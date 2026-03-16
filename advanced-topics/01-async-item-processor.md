# Async ItemProcessor·ItemWriter — Process 단계를 비동기로 오프로드하기

---

## 🎯 핵심 질문

- `AsyncItemProcessor`가 `Future<O>`를 반환하는 원리는 무엇인가?
- `AsyncItemWriter`가 `Future` 목록을 어떻게 unwrap해 Write하는가?
- `AsyncItemProcessor` + `AsyncItemWriter` 없이 비동기 처리를 구현하면 어떤 문제가 생기는가?
- Partitioning과 Async Processing을 비교했을 때 각각 어떤 상황에 적합한가?
- `AsyncItemProcessor`의 `TaskExecutor`와 Step의 `TaskExecutor`는 어떻게 다른가?

---

## 🔍 왜 Async Processing이 필요한가

### 문제: Process 단계가 느릴 때 Read와 Write가 유휴 상태로 기다린다

```
동기 처리 흐름 (외부 API 호출 포함):

  [Read 1000건] → [Process: 각 건당 외부 API 100ms] → [Write 1000건]
  Read:    ~1초
  Process: 100ms × 1000건 = 100초  ← 병목!
  Write:   ~2초
  총:      ~103초

  문제: Read가 끝나고 Write는 준비됐는데 Process가 100초 기다림

비동기 처리 (AsyncItemProcessor):

  [Read 1000건]
      ↓
  [AsyncItemProcessor: 1000건을 별도 쓰레드 풀에 동시 제출]
      ↓ Future<O> × 1000개 즉시 반환
  [AsyncItemWriter: Future 1000개를 unwrap → 실제 값 수집 후 Write]
  
  Read:    ~1초
  Process: 1000건 / 쓰레드 수 (병렬) ≈ 100ms (50개 쓰레드 기준)
  Write:   ~2초
  총:      ~3초 → 34배 향상
```

---

## 😱 흔한 실수

### Before: AsyncItemProcessor만 사용하고 AsyncItemWriter를 안 쓴다

```java
// ❌ AsyncItemProcessor + 일반 ItemWriter 조합
@Bean
public Step asyncStep() {
    return stepBuilderFactory.get("asyncStep")
        .<Order, Future<SettledOrder>>chunk(1000)  // 타입이 Future<O>
        .reader(reader())
        .processor(asyncProcessor())  // Future<SettledOrder> 반환
        .writer(settledOrderWriter())  // SettledOrder를 받아야 하는데 Future<SettledOrder>가 전달됨!
        // 컴파일 오류 또는 런타임 ClassCastException
        .build();
}

// ✅ AsyncItemProcessor + AsyncItemWriter 반드시 쌍으로 사용
@Bean
public Step asyncStep() {
    return stepBuilderFactory.get("asyncStep")
        .<Order, Future<SettledOrder>>chunk(1000)
        .reader(reader())
        .processor(asyncProcessor())          // Future<SettledOrder> 반환
        .writer(asyncWriter())                // Future<SettledOrder>를 unwrap
        .build();
}
```

### Before: TaskExecutor 쓰레드 수를 Chunk Size보다 작게 설정한다

```java
// ❌ 쓰레드 2개 / Chunk 1000건 → 500건씩 순차 처리 (비동기 효과 반감)
@Bean
public AsyncItemProcessor<Order, SettledOrder> asyncProcessor() {
    AsyncItemProcessor<Order, SettledOrder> processor = new AsyncItemProcessor<>();
    processor.setDelegate(actualProcessor());
    processor.setTaskExecutor(new SimpleAsyncTaskExecutor());  // 쓰레드 무제한 생성
    // SimpleAsyncTaskExecutor는 매 작업마다 새 쓰레드 생성 → GC 압력
    return processor;
}

// ✅ 쓰레드 풀 크기 = 최적 병렬 처리 수 (외부 API 동시 연결 허용 수 기준)
@Bean
public TaskExecutor asyncProcessorExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(50);   // 외부 API 동시 연결 50개
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(1000);  // Chunk Size 이상으로 설정
    executor.setThreadNamePrefix("async-processor-");
    executor.initialize();
    return executor;
}
```

---

## ✨ AsyncItemProcessor + AsyncItemWriter 동작 원리

```java
// AsyncItemProcessor: 실제 Processor를 별도 쓰레드에서 실행
// 반환값: Future<O> (즉시 반환 — 처리가 완료되지 않아도)

// AsyncItemWriter: Future<O> 목록을 받아 unwrap 후 실제 Writer에 전달
// Future.get() 호출 → 모든 비동기 처리 완료 대기 → 실제 값 수집 → Write

// 타입 흐름:
// ItemReader<Order>
//   ↓ Order
// AsyncItemProcessor<Order, SettledOrder>
//   ↓ Future<SettledOrder>  (즉시)
// AsyncItemWriter<SettledOrder>  (내부: List<Future<SettledOrder>>를 받음)
//   ↓ unwrap: future.get() 호출 × 1000건
//   ↓ List<SettledOrder> (실제 값)
// 실제 ItemWriter<SettledOrder>
```

---

## 🔬 내부 동작 원리

### 1. AsyncItemProcessor 소스

```java
// AsyncItemProcessor.java
public class AsyncItemProcessor<I, O> implements ItemProcessor<I, Future<O>>,
        InitializingBean {

    private ItemProcessor<I, O> delegate;   // 실제 처리 Processor
    private TaskExecutor taskExecutor;       // 비동기 실행 Executor

    @Override
    public Future<O> process(final I item) throws Exception {
        // ① FutureTask에 실제 처리 로직 래핑
        FutureTask<O> task = new FutureTask<>(() -> {
            // ← 별도 쓰레드에서 실행될 코드
            return delegate.process(item);
        });

        // ② TaskExecutor에 즉시 제출 (블로킹 없음)
        taskExecutor.execute(task);

        // ③ Future 즉시 반환 (처리 완료 전)
        return task;
        // → ChunkProcessor는 Future<O>를 outputs Chunk에 추가
        // → 1000건 모두 제출 후 AsyncItemWriter로 전달
    }
}
```

### 2. AsyncItemWriter 소스

```java
// AsyncItemWriter.java
public class AsyncItemWriter<T> implements ItemStreamWriter<Future<T>>,
        InitializingBean {

    private ItemWriter<T> delegate;   // 실제 Write 로직

    @Override
    public void write(Chunk<? extends Future<T>> items) throws Exception {
        List<T> list = new ArrayList<>(items.size());

        // ① 모든 Future 완료 대기 + unwrap
        for (Future<T> future : items) {
            T item = future.get();    // ← 블로킹 대기
            if (item != null) {
                list.add(item);
            }
            // null이면 필터링 (AsyncItemProcessor 위임 Processor가 null 반환 시)
        }

        // ② unwrap된 실제 값들로 Write
        delegate.write(new Chunk<>(list));
    }
}
```

### 3. 비동기 처리 타임라인

```
Chunk 1000건 처리 (외부 API 건당 100ms, 쓰레드 50개):

T=0ms:    ItemReader.read() × 1000건 완료 → 1초

T=1000ms: AsyncItemProcessor.process() × 1000건 제출
          쓰레드 풀에 1000개 FutureTask 제출 (즉시 반환)

T=1000ms~3000ms: 병렬 처리
          쓰레드 1~50번: 1~50번 아이템 외부 API 호출 (각 100ms)
          T=1100ms: 1~50번 완료 → 51~100번 시작
          T=1200ms: 51~100번 완료 → 101~150번 시작
          ...
          T=3000ms: 951~1000번 완료

T=3000ms: AsyncItemWriter.write() 호출
          → future.get() × 1000건 (이미 완료됐으므로 즉시 반환)
          → JdbcBatchItemWriter.write(1000건) ~2초

총 처리: ~5초 vs 동기 103초 → 20배 향상
```

### 4. Async Processing vs Partitioning 비교

```
AsyncItemProcessor:
  병렬화 대상: Process 단계 (하나의 Chunk 내 아이템들을 병렬 처리)
  쓰레드 수:   Chunk 내 병렬 처리 수 (TaskExecutor 설정)
  적합한 경우: 외부 API 호출, CPU 집약적 처리 (I/O Bound)
  재시작:      EC 기반 (일반 Chunk와 동일)
  설정:        @StepScope 불필요 (Reader/Writer는 단일 쓰레드)

Partitioning:
  병렬화 대상: 데이터 범위 (서로 다른 데이터를 각 Worker가 처리)
  쓰레드 수:   Worker 수 = GridSize
  적합한 경우: 대용량 데이터 처리, 독립적 범위 분할 가능
  재시작:      Worker StepExecution 단위로 재시작
  설정:        Worker Reader에 @StepScope 필수
```

---

## 💻 실전 구현

### 외부 API 호출 Processor에 Async 적용

```java
@Configuration
public class AsyncProcessingConfig {

    @Bean
    public Job asyncEnrichmentJob() {
        return jobBuilderFactory.get("asyncEnrichmentJob")
            .start(asyncEnrichmentStep())
            .build();
    }

    @Bean
    public Step asyncEnrichmentStep() {
        return stepBuilderFactory.get("asyncEnrichmentStep")
            .<Order, Future<EnrichedOrder>>chunk(100)   // Chunk 작게 (API 호출 고려)
            .reader(orderReader())
            .processor(asyncEnrichmentProcessor())
            .writer(asyncEnrichedOrderWriter())
            .build();
    }

    @Bean
    public AsyncItemProcessor<Order, EnrichedOrder> asyncEnrichmentProcessor() {
        AsyncItemProcessor<Order, EnrichedOrder> asyncProcessor =
            new AsyncItemProcessor<>();
        asyncProcessor.setDelegate(enrichmentProcessor());  // 실제 API 호출 로직
        asyncProcessor.setTaskExecutor(apiCallExecutor());  // 별도 쓰레드 풀
        return asyncProcessor;
    }

    @Bean
    public ItemProcessor<Order, EnrichedOrder> enrichmentProcessor() {
        return order -> {
            // 외부 API 호출 (100ms)
            CustomerProfile profile = customerApiClient.getProfile(
                order.getCustomerId());
            return EnrichedOrder.of(order, profile);
        };
    }

    @Bean
    public AsyncItemWriter<EnrichedOrder> asyncEnrichedOrderWriter() {
        AsyncItemWriter<EnrichedOrder> asyncWriter = new AsyncItemWriter<>();
        asyncWriter.setDelegate(enrichedOrderWriter());  // 실제 DB 저장 로직
        return asyncWriter;
    }

    @Bean
    public JdbcBatchItemWriter<EnrichedOrder> enrichedOrderWriter() {
        return new JdbcBatchItemWriterBuilder<EnrichedOrder>()
            .dataSource(dataSource)
            .sql("INSERT INTO enriched_orders (order_id, customer_name, tier) " +
                 "VALUES (:orderId, :customerName, :tier)")
            .beanMapped()
            .build();
    }

    @Bean
    public TaskExecutor apiCallExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(50);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);   // Chunk 100건 × 2배 여유
        executor.setThreadNamePrefix("api-async-");
        executor.initialize();
        return executor;
    }
}
```

---

## 📊 동기 vs Async 성능 비교

```
외부 API 호출 (건당 100ms), 10만 건 처리:

┌───────────────────────────────┬──────────────┬──────────────────────────────────────┐
│ 처리 방식                       │ 처리 시간       │ 특이사항                               │
├───────────────────────────────┼──────────────┼──────────────────────────────────────┤
│ 동기 처리 (Chunk 100건)          │ ~2.8시간      │ 100ms × 10만 = 10,000초               │
│ AsyncItemProcessor (50쓰레드)   │ ~240초       │ 10만 / 50 × 100ms = 200초 + 오버헤드     │
│ Partitioning (4 Workers)      │ ~720초        │ 각 Worker 2.5만 건 동기 처리             │
│ Partitioning + Async 조합      │ ~80초         │ 4 Workers × 50쓰레드 = 200 동시         │
└───────────────────────────────┴──────────────┴──────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
AsyncItemProcessor + AsyncItemWriter:

  장점
    Process 병목(외부 API, CPU 집약)을 병렬화
    Reader/Writer는 단일 쓰레드 유지 → Thread-safety 문제 없음
    구현 단순 (기존 Processor를 delegate로 감싸기만 하면 됨)

  단점
    Future.get() 호출 시 모든 처리 완료 대기 (AsyncItemWriter)
    → 가장 느린 아이템이 Chunk 전체를 대기시킴 (Long Tail 문제)
    처리 순서 보장 안 됨 (병렬 처리 특성)
    쓰레드 풀 + Chunk 조합으로 메모리 관리 복잡

Async Processing이 효과적인 경우:
  ✓ 외부 API 호출 (I/O Bound, 병렬 연결 허용)
  ✓ 독립적인 CPU 연산 (각 아이템이 서로 무관)
  ✗ 아이템 간 순서 의존성 있는 경우
  ✗ 처리 순서가 비즈니스 로직에 영향을 미치는 경우
```

---

## 📌 핵심 정리

```
AsyncItemProcessor 동작
  delegate.process(item)를 FutureTask에 래핑
  TaskExecutor에 즉시 제출 → Future<O> 즉시 반환
  1000건을 동시에 쓰레드 풀에 제출 → 병렬 처리

AsyncItemWriter 동작
  List<Future<O>> 수신
  각 Future.get()으로 완료 대기 + unwrap
  실제 List<O>를 delegate Writer에 전달

반드시 쌍으로 사용
  AsyncItemProcessor → AsyncItemWriter
  Step 타입: <I, Future<O>>로 선언

Async vs Partitioning 선택
  Process 단계가 병목 (API, CPU) → AsyncItemProcessor
  데이터 양 자체가 문제 (1,000만+ 건) → Partitioning
  둘 다 필요 → Partitioning + Async 조합
```

---

## 🤔 생각해볼 문제

**Q1.** `AsyncItemWriter.write()`에서 `future.get()`을 호출할 때 하나의 Future에서 예외가 발생하면 어떻게 되는가? 나머지 Future들은 어떻게 처리되는가?

**Q2.** `AsyncItemProcessor`의 `TaskExecutor` 쓰레드 수를 100으로 설정하고 Chunk Size를 50으로 설정했습니다. 이 조합이 효율적인가? 쓰레드 50개가 유휴 상태로 남지 않는가?

**Q3.** 외부 API가 초당 최대 20건 요청만 허용(Rate Limit)합니다. `AsyncItemProcessor`에서 이 제한을 어떻게 적용하는가?

> 💡 **해설**
>
> **Q1.** `AsyncItemWriter`에서 순서대로 `future.get()`을 호출하다가 예외가 발생하면, `ExecutionException`이 throw됩니다. 이 예외가 `write()` 밖으로 전파되면 Chunk 전체가 롤백됩니다. 이미 완료된 다른 Future들은 계산이 됐지만 결과가 Writer에 전달되지 않습니다. `faultTolerant().skip()`이 설정돼 있으면 스캐터-개더가 시작되는데, 이때 AsyncItemWriter가 아닌 일반 Writer로 1건씩 재처리합니다. 단, 이미 비동기로 처리된 결과를 다시 처리하는 문제가 발생할 수 있으므로, Skip 시나리오가 있다면 `FaultTolerant` + `AsyncItemProcessor/Writer` 조합을 신중히 설계해야 합니다.
>
> **Q2.** 쓰레드 100개, Chunk 50건이면 한 번에 50개 Future만 제출되므로 쓰레드 50개는 항상 유휴 상태입니다. 낭비입니다. 최적: 쓰레드 수 ≤ Chunk Size. 실제로는 `CorePoolSize = Chunk Size / 평균_처리_시간_ms × 허용_대기_시간_ms`로 계산합니다. 외부 API 100ms, Chunk 50건, 허용 지연 200ms이면 50 × (200/100) = 100개 동시 처리를 위해 쓰레드 50개가 적정합니다.
>
> **Q3.** `AsyncItemProcessor`의 `TaskExecutor`에서 Rate Limiting을 적용합니다. Guava의 `RateLimiter`를 delegate Processor에서 사용합니다: `rateLimiter.acquire()` 호출 후 API 요청. 단, 이렇게 하면 쓰레드들이 RateLimiter에서 블로킹되므로 쓰레드가 점유됩니다. 더 효율적인 방법: Bucket4j 같은 비동기 Rate Limiter를 사용하거나, 쓰레드 수를 Rate Limit에 맞게 제한합니다 (초당 20건 허용 → 쓰레드 20개 × API 1초/건 = 초당 20건 처리).

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Multi-threaded Step ➡️](./02-multi-threaded-step.md)**

</div>
