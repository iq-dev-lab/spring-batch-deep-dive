# Multi-threaded Step — 단일 Step 내 여러 쓰레드로 Chunk 병렬 처리

---

## 🎯 핵심 질문

- `TaskExecutor`를 설정한 Step에서 여러 쓰레드가 어떻게 동작하는가?
- 여러 쓰레드가 동일한 `ItemReader`를 공유할 때 발생하는 Thread-safety 문제는?
- `SynchronizedItemStreamReader`가 어떻게 Thread-safety를 보장하는가?
- Multi-threaded Step과 Partitioning의 재시작 동작이 어떻게 다른가?
- `throttleLimit`은 어떤 역할을 하는가?

---

## 🔍 Multi-threaded Step vs Partitioning

### 언제 각각 선택하는가

```
Multi-threaded Step:
  하나의 Step에서 여러 쓰레드가 독립 Chunk를 처리
  → 쓰레드 1: Chunk 1(아이템 1~1000)
  → 쓰레드 2: Chunk 2(아이템 1001~2000)
  → 쓰레드 3: Chunk 3(아이템 2001~3000)
  → 쓰레드 4: Chunk 4(아이템 3001~4000)

  특징:
    Reader 공유 → Thread-safety 필요 (SynchronizedItemStreamReader)
    단일 StepExecution → 재시작 시 처음부터 (EC 저장 불가)
    설정 단순 (Partitioner 불필요)
    같은 데이터 소스에서 순서대로 읽어 분배

Partitioning:
  여러 StepExecution이 독립 데이터 범위 처리
  → Worker 1: id 1~250만
  → Worker 2: id 250만~500만
  → 각자 독립 Reader

  특징:
    Reader 독립 → Thread-safety 문제 없음
    독립 StepExecution → 재시작 시 실패 Worker만 재실행
    설정 복잡 (Partitioner + @StepScope 필요)

선택 기준:
  재시작이 중요하다 → Partitioning
  빠른 구현이 중요하다 → Multi-threaded Step
  Reader가 Thread-safe하다 (DB 뷰, REST API) → Multi-threaded Step
  대용량 (수백만 건 이상) → Partitioning 권장
```

---

## 😱 흔한 실수

### Before: ItemReader의 상태를 공유해 데이터 중복/누락이 발생한다

```java
// ❌ JpaPagingItemReader를 Multi-threaded Step에서 직접 사용
@Bean
public Step multiThreadedStep() {
    return stepBuilderFactory.get("multiThreadedStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(jpaPagingReader())        // Thread-unsafe!
        .processor(processor())
        .writer(writer())
        .taskExecutor(new SimpleAsyncTaskExecutor())  // 멀티 쓰레드 활성화
        .build();
}

@Bean
@StepScope
public JpaPagingItemReader<Order> jpaPagingReader() {
    // JpaPagingItemReader 내부 상태:
    //   page: 현재 페이지 번호
    //   results: 현재 페이지 결과 목록
    //   current: 현재 결과 인덱스
    // → 여러 쓰레드가 동시에 page/current를 수정하면 데이터 누락/중복!
    return new JpaPagingItemReaderBuilder<Order>().build();
}

// ✅ SynchronizedItemStreamReader로 래핑
@Bean
@StepScope
public SynchronizedItemStreamReader<Order> syncReader() {
    SynchronizedItemStreamReader<Order> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(jpaPagingReader());
    return reader;
}
```

### Before: saveState=true로 Multi-threaded Step을 설정해 EC가 꼬인다

```java
// ❌ Multi-threaded Step에서 saveState=true (기본값)
@Bean
@StepScope
public JpaPagingItemReader<Order> reader() {
    return new JpaPagingItemReaderBuilder<Order>()
        .name("orderReader")
        // saveState=true (기본값)
        // → 여러 쓰레드가 동시에 update() 호출
        // → EC에 read.count 덮어쓰기 경쟁 → 잘못된 재시작 포인트
        .build();
}

// ✅ Multi-threaded Step에서는 saveState=false
@Bean
@StepScope
public JpaPagingItemReader<Order> reader() {
    return new JpaPagingItemReaderBuilder<Order>()
        .name("orderReader")
        .saveState(false)    // EC 저장 비활성화 → 재시작 불가하지만 안전
        .build();
}
```

---

## ✨ Multi-threaded Step 설정

```java
@Bean
public Step multiThreadedStep() {
    return stepBuilderFactory.get("multiThreadedStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(synchronizedReader())      // Thread-safe Reader
        .processor(processor())            // 상태 없는 Processor (Thread-safe)
        .writer(writer())                  // JdbcBatchItemWriter (Thread-safe)
        .taskExecutor(stepTaskExecutor())  // 쓰레드 풀 설정
        .throttleLimit(4)                  // 동시 쓰레드 수 제한 (기본값 4)
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. taskExecutor 설정 시 Chunk 실행 방식 변화

```java
// TaskletStep.doExecute() — TaskExecutor가 있을 때
protected void doExecute(StepExecution stepExecution) throws Exception {

    if (taskExecutor instanceof SyncTaskExecutor) {
        // 동기 실행: 현재 쓰레드에서 순차 처리
        while (true) {
            RepeatStatus status = executeChunk(stepExecution);
            if (status == RepeatStatus.FINISHED) break;
        }
    } else {
        // 비동기 실행: ThrottledTaskExecutor 사용
        RepeatOperations repeatOperations =
            new TaskExecutorRepeatTemplate();
        ((TaskExecutorRepeatTemplate) repeatOperations)
            .setTaskExecutor(taskExecutor);
        ((TaskExecutorRepeatTemplate) repeatOperations)
            .setThrottleLimit(throttleLimit);

        repeatOperations.iterate(context -> {
            // 각 Chunk를 별도 쓰레드에서 실행
            // Chunk = Read 1000건 + Process + Write
            executeChunk(stepExecution);
            return RepeatStatus.CONTINUABLE;
        });
    }
}
```

### 2. SynchronizedItemStreamReader — read() 동기화

```java
// SynchronizedItemStreamReader.java
public class SynchronizedItemStreamReader<T>
        implements ItemStreamReader<T>, InitializingBean {

    private ItemStreamReader<T> delegate;

    // synchronized: 한 번에 한 쓰레드만 read() 진입
    @Override
    public synchronized T read() throws Exception, UnexpectedInputException,
            ParseException, NonTransientResourceException {
        return delegate.read();
        // 여러 쓰레드가 동시에 read() 호출해도
        // 실제로는 순서대로 하나씩 실행됨
        // → 각 쓰레드는 다른 아이템을 받음 (중복 없음)
    }

    @Override
    public void open(ExecutionContext executionContext) {
        delegate.open(executionContext);  // 동기화 없음 (한 번만 호출)
    }

    @Override
    public void update(ExecutionContext executionContext) {
        delegate.update(executionContext);  // 동기화 없음
        // ← 이 메서드가 여러 쓰레드에서 동시 호출되면 EC 덮어쓰기 위험
        // → saveState=false로 설정해 update() 자체를 비활성화 권장
    }
}
```

### 3. throttleLimit 동작

```java
// ThrottledTaskExecutor (내부 구현)
// throttleLimit = 최대 동시 실행 Chunk 수

// throttleLimit=4 설정:
// Semaphore semaphore = new Semaphore(4);
// 각 Chunk 실행 전: semaphore.acquire() — 4개 초과 시 블로킹
// 각 Chunk 완료 후: semaphore.release()

// 예: throttleLimit=4, 쓰레드 풀 크기=10
// → 최대 4개 Chunk만 동시 실행
// → 나머지 6개 쓰레드는 throttle 대기
// → DB 커넥션 풀, Lock 경쟁 제어 가능

// throttleLimit 기본값:
// TaskExecutorRepeatTemplate.DEFAULT_THROTTLE_LIMIT = 4
```

### 4. Multi-threaded Step 실행 타임라인

```
4개 쓰레드, throttleLimit=4, Chunk=1000건:

T=0:
  Thread 1: Read 1~1000번 (synchronized → 순서대로 가져감)
  Thread 2: Read 1001~2000번 (Thread 1이 read() 완료 후 접근)
  Thread 3: Read 2001~3000번
  Thread 4: Read 3001~4000번

T=일부:
  Thread 1: Process 1~1000번 (비동기 병렬)
  Thread 2: Process 1001~2000번 (비동기 병렬)
  Thread 3: Process 2001~3000번
  Thread 4: Process 3001~4000번

T=later:
  Thread 1 완료 → 새 Chunk: Read 4001~5000번
  Thread 2 완료 → 새 Chunk: Read 5001~6000번
  ...

주의: Thread 4가 Thread 1보다 먼저 Write를 완료할 수 있음
→ Write 순서 보장 안 됨 (Writer가 순서 무관해야 함)
```

---

## 💻 실전 구현

### 완전한 Multi-threaded Step 구성

```java
@Configuration
public class MultiThreadedStepConfig {

    @Bean
    public Job multiThreadedJob() {
        return jobBuilderFactory.get("multiThreadedJob")
            .start(multiThreadedSettlementStep())
            .build();
    }

    @Bean
    public Step multiThreadedSettlementStep() {
        return stepBuilderFactory.get("multiThreadedSettlementStep")
            .<Order, SettledOrder>chunk(500)    // Chunk 크기: 쓰레드 수 고려
            .reader(synchronizedOrderReader())   // Thread-safe Reader
            .processor(statelessSettlementProcessor())  // 상태 없는 Processor
            .writer(settlementWriter())          // Thread-safe Writer
            .taskExecutor(stepTaskExecutor())
            .throttleLimit(4)                    // 동시 4개 Chunk
            .faultTolerant()
                .skip(DataIntegrityViolationException.class)
                .skipLimit(100)
            .build();
    }

    @Bean
    @StepScope
    public SynchronizedItemStreamReader<Order> synchronizedOrderReader(
            EntityManagerFactory emf,
            @Value("#{jobParameters['targetDate']}") String targetDate) {

        JpaPagingItemReader<Order> pagingReader = new JpaPagingItemReaderBuilder<Order>()
            .name("multiThreadOrderReader")
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o WHERE o.date = :date ORDER BY o.id")
            .parameterValues(Map.of("date", LocalDate.parse(targetDate)))
            .pageSize(500)
            .saveState(false)   // ← 필수! Multi-threaded에서 EC 저장 비활성화
            .build();

        SynchronizedItemStreamReader<Order> syncReader = new SynchronizedItemStreamReader<>();
        syncReader.setDelegate(pagingReader);
        return syncReader;
    }

    // 상태 없는(stateless) Processor — Thread-safe 보장
    @Bean
    public ItemProcessor<Order, SettledOrder> statelessSettlementProcessor() {
        return order -> {
            // 인스턴스 변수 없음 → 쓰레드 간 상태 공유 없음
            BigDecimal fee = order.getAmount().multiply(new BigDecimal("0.015"));
            return new SettledOrder(order.getId(), order.getAmount(), fee);
        };
    }

    @Bean
    public TaskExecutor stepTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(10);
        executor.setThreadNamePrefix("step-thread-");
        executor.initialize();
        return executor;
    }
}
```

---

## 📊 Multi-threaded Step vs Partitioning 비교

```
┌──────────────────────────┬──────────────────────────────┬──────────────────────────────┐
│ 항목                      │ Multi-threaded Step          │ Partitioning                 │
├──────────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ StepExecution            │ 1개 (공유)                     │ Worker마다 독립                │
│ ItemReader               │ 공유 (동기화 필요)               │ Worker마다 독립                │
│ 재시작                     │ 처음부터 (saveState=false)     │ 실패 Worker만 재실행            │
│ EC 저장                   │ saveState=false 권장          │ 각 Worker 독립 저장             │
│ 설정 복잡도                 │ 낮음                          │ 높음                          │
│ Thread-safety 주의 대상    │ ItemReader                   │ 없음 (독립 Reader)              │
│ 적합한 데이터 규모           │ 수십만~수백만 건                 │ 수백만 건 이상                   │
│ 재시작 지원                 │ 제한적                        │ 완전 지원                       │
└──────────────────────────┴──────────────────────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Multi-threaded Step:

  장점
    Partitioner 불필요 → 설정 단순
    기존 Step에 taskExecutor 추가만으로 병렬화
    단일 StepExecution → 간단한 모니터링

  단점
    saveState=false 필수 → 재시작 시 처음부터
    ItemReader Thread-safety 직접 보장 필요
    처리 순서 보장 안 됨 (Writer가 순서 무관해야 함)
    쓰레드 간 ItemReader 경합 → 처리 속도 향상 제한적

throttleLimit vs 쓰레드 수:
  쓰레드 수: TaskExecutor 풀 크기 (최대 동시 쓰레드)
  throttleLimit: 동시 처리 Chunk 수 상한
  → throttleLimit ≤ 쓰레드 수 권장 (초과해도 실제 동시 실행 = 쓰레드 수)
```

---

## 📌 핵심 정리

```
Multi-threaded Step 핵심 설정

  .taskExecutor(executor)       → 병렬 실행 활성화
  .throttleLimit(N)             → 동시 처리 Chunk 수 제한 (기본값 4)
  SynchronizedItemStreamReader  → ItemReader Thread-safety 보장
  saveState(false)              → EC 저장 비활성화 (재시작 시 처음부터)

Thread-safety 체크리스트

  ItemReader: 상태 있음 → SynchronizedItemStreamReader 필수
  ItemProcessor: 상태 없음 (인스턴스 변수 없음) → 기본 Thread-safe
  ItemWriter: JdbcBatchItemWriter/JpaItemWriter → Thread-safe
              FlatFileItemWriter → Thread-unsafe (동시 파일 쓰기 충돌)

재시작 전략

  재시작 지원 필요 → Partitioning 선택
  재시작 불필요 or 처음부터 재처리 감수 → Multi-threaded Step
  Writer가 멱등(UPSERT) → 재시작 시 중복 처리 안전
```

---

## 🤔 생각해볼 문제

**Q1.** Multi-threaded Step에서 `saveState=false`로 설정하면 재시작 시 처음부터 처리합니다. 이 경우 Writer가 `ON DUPLICATE KEY UPDATE`로 설정돼 있지 않으면 어떤 문제가 발생하는가?

**Q2.** `SynchronizedItemStreamReader`의 `read()`가 `synchronized` 메서드입니다. 4개 쓰레드가 동시에 `read()`를 호출하면 실제로는 순차적으로 실행됩니다. 이 경우 성능 향상 효과는 어디서 오는가?

**Q3.** Multi-threaded Step에서 `FlatFileItemWriter`를 사용하면 왜 안 되는가? 반드시 사용해야 한다면 어떻게 해결하는가?

> 💡 **해설**
>
> **Q1.** Writer가 단순 INSERT이고 PK가 있다면, 재시작 시 이미 DB에 저장된 데이터를 다시 INSERT하려다 `DuplicateKeyException`이 발생합니다. `faultTolerant().skip(DuplicateKeyException.class)`로 처리할 수 있지만 불필요한 오류가 대량 발생합니다. 해결: (1) `ON DUPLICATE KEY UPDATE` 사용으로 멱등 처리. (2) 재시작 전 이전에 저장된 데이터를 삭제하는 Cleanup Step 추가. (3) Multi-threaded Step 대신 재시작 지원이 완전한 Partitioning 사용.
>
> **Q2.** 성능 향상은 `read()` 이후의 Process와 Write 단계에서 옵니다. `read()`가 순차적이어도 각 쓰레드는 가져온 Chunk를 독립적으로 Process하고 Write합니다. Process가 CPU 집약적이거나 Write가 비동기라면 병렬화 효과가 큽니다. 반면 `read()` 자체가 병목이라면 Multi-threaded Step의 효과가 제한됩니다. `read()` 병목 해결은 Partitioning이 더 효과적입니다.
>
> **Q3.** `FlatFileItemWriter`는 내부적으로 `BufferedWriter`와 파일 포인터를 유지하는 상태를 가집니다. 여러 쓰레드가 동시에 `write()`를 호출하면 파일 내용이 뒤섞입니다(interleaving). 해결: (1) 각 쓰레드가 별도 임시 파일에 쓰고 마지막에 합치는 방식 — `@StepScope` `FlatFileItemWriter`를 ThreadLocal로 관리하거나, Writer를 쓰레드별로 분리합니다. (2) `synchronized` 블록으로 write()를 보호 — 성능 저하 발생. (3) DB에 먼저 저장 후 별도 Step에서 파일 Export — 가장 권장.

---

<div align="center">

**[⬅️ 이전: Async ItemProcessor](./01-async-item-processor.md)** | **[홈으로 🏠](../README.md)** | **[다음: JobScope와 StepScope ➡️](./03-job-scope-step-scope.md)**

</div>
