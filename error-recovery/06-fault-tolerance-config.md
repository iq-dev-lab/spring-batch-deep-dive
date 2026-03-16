# Fault Tolerance 설정 완전 가이드 — FaultTolerant ChunkProcessor로의 전환

---

## 🎯 핵심 질문

- `faultTolerant()` 호출 시 `ChunkOrientedTasklet` 내부에서 무엇이 교체되는가?
- `FaultTolerantChunkProvider`와 `FaultTolerantChunkProcessor`는 기본 구현체와 어떻게 다른가?
- `noRollback(Exception.class)`는 어떤 시나리오에서 사용하며 내부에서 어떻게 동작하는가?
- `processorNonTransactional()`은 무엇을 의미하며 성능에 어떤 영향을 미치는가?
- `keyGenerator`를 사용한 Item 캐싱은 스캐터-개더에서 어떤 역할을 하는가?

---

## 🔍 faultTolerant()가 교체하는 것

### 문제: 기본 ChunkProcessor는 예외 발생 시 바로 실패한다

```
기본 구성 (faultTolerant() 없음):
  SimpleChunkProvider    → Read 예외 → 즉시 Step FAILED
  SimpleChunkProcessor   → Process/Write 예외 → 즉시 Chunk 롤백 + Step FAILED
  → 단 1건의 오류로 전체 배치 실패

faultTolerant() 호출 후:
  FaultTolerantChunkProvider    → Read 예외 → Skip 정책 확인
  FaultTolerantChunkProcessor   → Process/Write 예외 → Retry + Skip 정책 확인
  → 오류 아이템만 격리, 나머지 정상 처리

교체 시점:
  StepBuilderHelper가 faultTolerant()로 FaultTolerantStepFactoryBean 선택
  → Step 빌드 시 FaultTolerantChunkOrientedTasklet 생성
  → FaultTolerantChunkProvider + FaultTolerantChunkProcessor 조합
```

---

## 😱 흔한 실수

### Before: noRollback을 무분별하게 사용해 데이터 불일치를 유발한다

```java
// ❌ noRollback을 광범위하게 적용
.faultTolerant()
    .noRollback(Exception.class)  // 모든 예외에 롤백 안 함!
    // → Write 실패해도 트랜잭션 롤백 없음
    // → 이미 쓰여진 999건과 실패한 1건이 같은 커밋 상태
    // → 데이터 불일치 가능성 매우 높음

// ✅ noRollback은 매우 제한적으로 사용
// 적합한 경우: 예외가 발생해도 트랜잭션 상태가 여전히 유효한 경우
.faultTolerant()
    .noRollback(ValidationException.class)
    // ValidationException은 DB 상태를 변경하지 않는 순수 검증 예외
    // → 롤백 없이 Retry 시도 가능
```

### Before: processorNonTransactional()을 이해 없이 설정한다

```java
// ❌ processorNonTransactional 의미를 모르고 설정
.faultTolerant()
    .skip(Exception.class)
    .skipLimit(10)
    .processorNonTransactional()
    // 의도: Processor 성능 최적화
    // 실제 효과:
    //   스캐터-개더(Write Skip) 시 Processor를 재실행하지 않음
    //   → 이미 캐싱된 Processor 결과를 재사용
    //   → Processor에 side effect(외부 API 호출 등)가 있으면 재실행 안 됨!

// ✅ Processor가 순수 함수(side effect 없음)일 때만 사용
.processorNonTransactional()
// → 스캐터-개더 시 Process 단계 생략 → Write만 1건씩 재시도
// → Processor가 DB 조회나 API 호출을 포함하면 사용 금지
```

---

## ✨ faultTolerant() 전체 설정 옵션

```java
@Bean
public Step fullFaultTolerantStep() {
    return stepBuilderFactory.get("fullFaultTolerantStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .faultTolerant()

            // === Skip 설정 ===
            .skip(DataIntegrityViolationException.class)
            .skip(ParseException.class)
            .noSkip(CriticalException.class)
            .skipLimit(50)
            .skipPolicy(new CustomSkipPolicy())  // 또는 커스텀

            // === Retry 설정 ===
            .retry(TransientDataAccessException.class)
            .noRetry(DataIntegrityViolationException.class)
            .retryLimit(3)
            .backOffPolicy(exponentialBackOff())
            .retryContextCache(new MapRetryContextCache())

            // === 트랜잭션 제어 ===
            .noRollback(ValidationException.class) // 이 예외는 롤백 안 함
            .transactionAttribute(customTxAttribute) // 트랜잭션 격리 수준

            // === 성능 최적화 ===
            .processorNonTransactional() // Processor 캐싱 (side effect 없을 때만)

            // === 리스너 ===
            .listener(skipListener())
            .listener(retryListener())
            .listener(chunkListener())

        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. faultTolerant() 호출 시 내부 교체 과정

```java
// FaultTolerantStepBuilder.build()
@Override
public TaskletStep build() {
    // ① FaultTolerantChunkProvider 생성
    FaultTolerantChunkProvider<I> chunkProvider =
        new FaultTolerantChunkProvider<>(itemReader, repeatOperations);
    chunkProvider.setSkipPolicy(createSkipPolicy());  // Skip 정책 설정
    chunkProvider.setRollbackClassifier(rollbackClassifier); // noRollback 설정

    // ② FaultTolerantChunkProcessor 생성
    FaultTolerantChunkProcessor<I, O> chunkProcessor =
        new FaultTolerantChunkProcessor<>(
            itemProcessor,
            itemWriter,
            createRetryOperations());  // RetryTemplate 생성
    chunkProcessor.setSkipPolicy(createSkipPolicy());
    chunkProcessor.setProcessorTransactional(!processorNonTransactional);

    // ③ FaultTolerantChunkOrientedTasklet 조합
    ChunkOrientedTasklet<I> tasklet =
        new ChunkOrientedTasklet<>(chunkProvider, chunkProcessor);

    // ④ TaskletStep에 등록
    return (TaskletStep) super.build();
}
```

### 2. noRollback — 트랜잭션 롤백 방지 메커니즘

```java
// BinaryExceptionClassifier — noRollback 예외 분류
// true: 롤백 필요, false: 롤백 불필요

// FaultTolerantChunkProcessor.write() 내부:
catch (Exception e) {
    // ① noRollback 예외인지 확인
    if (!rollbackClassifier.classify(e)) {
        // noRollback 예외 → 트랜잭션 롤백하지 않음
        // 단, 이 아이템은 Skip 처리
        contribution.incrementWriteSkipCount();
        listener.onSkipInWrite(item, e);
        // 트랜잭션은 계속 유효 → 다음 아이템 처리 가능
    } else {
        // 일반 예외 → 트랜잭션 롤백 필요
        // 스캐터-개더 모드 진입
        throw e;
    }
}

// noRollback 적합한 예외:
// - ValidationException: DB 상태 변경 없음
// - BusinessRuleException: 순수 검증 실패
//
// noRollback 부적합한 예외:
// - DataAccessException: DB 상태가 불확실
// - RuntimeException (일반): 어떤 DB 변경이 있었는지 불명확
```

### 3. processorNonTransactional — Processor 캐싱

```java
// FaultTolerantChunkProcessor에서 processorTransactional 플래그

// processorTransactional=true (기본값):
// 스캐터-개더 시 각 아이템에 대해 Process + Write 모두 재실행
// → 순서: 1건 Process → 1건 Write → 실패 시 Skip

// processorTransactional=false (processorNonTransactional() 설정):
// 스캐터-개더 시 이미 Process된 결과(캐싱)를 재사용 → Write만 재실행
// → 순서: Chunk 전체 Process 완료 후 캐싱 → Write 실패 시 캐시에서 가져와 1건씩 Write

// 캐싱 구현:
private final Map<Object, O> cache = new LinkedHashMap<>();

// 비교:
// processorTransactional=true: Process 1000번 + Write 1000번 (스캐터-개더)
// processorTransactional=false: Process 1000번 한 번 + Write 1000번 (캐시 재사용)
// → 성능 향상 but Processor side effect 재실행 없음 → 주의 필요
```

### 4. keyGenerator — 아이템 식별자 생성

```java
// 스캐터-개더 시 어떤 아이템이 어떤 출력과 대응하는지 추적
// 기본: DefaultKeyGenerator (Object.hashCode() + Object.equals())

// 커스텀 keyGenerator: 도메인 ID 기반 정확한 매핑
.keyGenerator(new KeyGenerator() {
    @Override
    public Object generateKey(Object target) {
        if (target instanceof Order) {
            return ((Order) target).getId();  // 주문 ID를 키로 사용
        }
        return target.hashCode();
    }
})
// → 스캐터-개더 중 inputs와 outputs 매핑 정확성 보장
// → 특히 processorNonTransactional 설정 시 캐시 키로 활용
```

---

## 💻 실전 구현

### 완전한 faultTolerant Step 구성 + 성능 튜닝

```java
@Bean
public Step productionFaultTolerantStep() {
    return stepBuilderFactory.get("productionStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(orderReader())
        .processor(settlementProcessor())   // 순수 함수, side effect 없음
        .writer(settlementWriter())
        .faultTolerant()

            // 영구적 데이터 오류: Skip
            .skip(DataIntegrityViolationException.class)
            .skip(ParseException.class)
            .skip(NumberFormatException.class)
            .skipLimit(100)

            // 일시적 DB 오류: Retry
            .retry(TransientDataAccessException.class)
            .retry(DeadlockLoserDataAccessException.class)
            .retryLimit(3)
            .backOffPolicy(exponentialRandomBackOffPolicy())

            // 치명적 오류 보호
            .noSkip(InsufficientBalanceException.class)
            .noRetry(DataIntegrityViolationException.class)

            // 순수 검증 예외: 롤백 불필요
            .noRollback(FieldValidationException.class)

            // Processor가 순수 함수 → 캐싱으로 성능 향상
            .processorNonTransactional()

            // 리스너
            .listener(skipAuditListener())
            .listener(retryMonitoringListener())
        .build();
}

// 성능 모니터링: Retry와 Skip 현황 추적
@Bean
public CompositeItemSkipListener<Order, SettledOrder> skipAuditListener() {
    return new CompositeItemSkipListener<>() {{
        register(new ItemListenerSupport<Order, SettledOrder>() {
            private final AtomicLong totalSkips = new AtomicLong();

            @Override
            public void onSkipInWrite(Object item, Throwable t) {
                long count = totalSkips.incrementAndGet();
                log.warn("[Skip #{}] Write: {} - {}",
                    count, ((SettledOrder) item).getOrderId(), t.getMessage());
                metricsService.recordSkip("WRITE", t.getClass().getSimpleName());

                // 임계값 초과 시 알림
                if (count % 10 == 0) {
                    alertService.sendSkipAlert(count);
                }
            }
        });
    }};
}
```

---

## 📊 FaultTolerant 설정 옵션 요약

```
┌─────────────────────────────────┬──────────────────────────────────────────────────┐
│ 설정                             │ 효과                                              │
├─────────────────────────────────┼──────────────────────────────────────────────────┤
│ skip(Exception.class)           │ 해당 예외 발생 시 해당 아이템 건너뜀                     │
│ noSkip(Exception.class)         │ 해당 예외는 절대 Skip 안 함 → Step FAILED             │
│ skipLimit(N)                    │ 총 N건까지 Skip 허용                                │
│ skipPolicy(policy)              │ 커스텀 Skip 정책 적용                                │
│ retry(Exception.class)          │ 해당 예외 시 Retry                                  │
│ noRetry(Exception.class)        │ 해당 예외는 Retry 안 함                              │
│ retryLimit(N)                   │ 최대 N번 Retry                                    │
│ backOffPolicy(policy)           │ Retry 대기 전략                                    │
│ noRollback(Exception.class)     │ 해당 예외 시 트랜잭션 롤백 안 함                        │
│ processorNonTransactional()     │ 스캐터-개더 시 Processor 재실행 안 함 (캐싱)            │
│ keyGenerator(generator)         │ 아이템 캐시 키 생성 방식 지정                           │
└─────────────────────────────────┴──────────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
FaultTolerant 전체 활성화:

  장점  일시적/영구적 오류를 세밀하게 처리
        단일 오류로 전체 배치 실패 방지
        noRollback/processorNonTransactional로 성능 최적화 가능

  단점  설정 복잡도 급증
        스캐터-개더 오버헤드 (Write Skip 시 1건씩 재처리)
        noRollback 잘못 설정 시 데이터 불일치

processorNonTransactional() 사용 기준:
  Processor가 순수 함수 (동일 입력 → 항상 동일 출력, side effect 없음)
    → 사용 가능 (성능 향상)
  Processor가 외부 API 호출, DB 조회 등을 포함
    → 사용 금지 (재실행 안 되므로 최신 데이터 반영 안 됨)
```

---

## 📌 핵심 정리

```
faultTolerant() 교체 내용
  SimpleChunkProvider → FaultTolerantChunkProvider
  SimpleChunkProcessor → FaultTolerantChunkProcessor
  → Skip/Retry 정책 적용 가능 상태로 전환

noRollback 사용 기준
  예외 발생 시 DB 상태가 변경되지 않는 순수 검증 예외에만 적용
  → 트랜잭션 상태가 유효한 상태에서 Retry 또는 Skip 가능
  잘못 사용 시 데이터 불일치 → 매우 신중하게 사용

processorNonTransactional 사용 기준
  Processor가 순수 함수일 때만 사용
  → 스캐터-개더 시 Process 단계 생략으로 성능 향상
  side effect 있는 Processor에 사용 시 부작용 재실행 안 됨

전체 설정 순서
  faultTolerant()           → FaultTolerant 모드 진입
  → skip/noSkip/skipLimit   → 영구적 오류 처리
  → retry/noRetry/retryLimit/backOffPolicy → 일시적 오류 처리
  → noRollback              → 트랜잭션 최적화 (선택적)
  → processorNonTransactional → 성능 최적화 (선택적)
```

---

## 🤔 생각해볼 문제

**Q1.** `noRollback(ValidationException.class)` 설정 후 Write 단계에서 `ValidationException`이 발생했습니다. 트랜잭션이 롤백되지 않는다면, 이 Chunk의 나머지 999건은 어떻게 처리되는가?

**Q2.** `processorNonTransactional()` 설정으로 Processor 결과가 캐싱됩니다. 스캐터-개더 중 캐시에서 가져온 데이터를 Write 재시도합니다. Write 재시도 중 해당 데이터의 기반이 되는 외부 데이터가 변경됐다면 어떤 문제가 생기는가?

**Q3.** `faultTolerant()` 설정 없이 `chunk(1000).reader().writer().build()`로 구성한 Step에서 Write 중 `DataIntegrityViolationException`이 발생했습니다. 재시작 시 이 Step은 어디서부터 시작되는가?

> 💡 **해설**
>
> **Q1.** `noRollback(ValidationException.class)` 설정 후 Write 실패 시, FaultTolerantChunkProcessor가 트랜잭션을 롤백하지 않고 해당 아이템만 Skip합니다. 그리고 나머지 999건에 대해 계속 Write를 시도합니다. 트랜잭션이 유효한 상태이므로 나머지 999건의 Write가 성공하면 Chunk가 정상 커밋됩니다. 단, `noRollback` 예외 발생 시에도 Skip 목록에 등록돼 있어야 Skip됩니다. `noRollback + skip` 조합이어야 합니다. `noRollback`만 있고 `skip`이 없으면 예외가 그대로 전파됩니다.
>
> **Q2.** `processorNonTransactional()`로 캐싱된 Processor 결과는 처음 Process된 시점의 데이터입니다. Write 재시도 중 외부 데이터(예: 환율, 재고 수량)가 변경됐다면, 캐시된 데이터는 이전 상태를 기반으로 계산된 값입니다. 예를 들어 Process 시점의 환율로 계산된 금액이 Write 재시도 시 이미 변경된 환율을 반영하지 못합니다. 이는 데이터 정합성 문제로 이어질 수 있습니다. 따라서 외부 데이터를 조회하는 Processor에는 `processorNonTransactional()`을 절대 사용하지 말아야 합니다.
>
> **Q3.** `faultTolerant()` 없이 Write 중 예외 발생 시, `SimpleChunkProcessor`는 예외를 그대로 전파합니다 → Chunk 전체 롤백 → `TaskletStep`이 예외를 받아 `StepExecution.status = FAILED` 처리합니다. 재시작 시 이 Step의 마지막 정상 커밋된 `ExecutionContext`의 `read.count` 기준으로 Reader가 위치를 복원합니다. 예를 들어 5000건까지 커밋되고 6000번째 Write에서 실패했다면, 재시작 시 5001번째부터 재처리합니다. 단, `faultTolerant()` 없으면 Skip 없이 동일 예외가 다시 발생할 수 있으므로 오류 원인을 먼저 해결해야 합니다.

---

<div align="center">

**[⬅️ 이전: ExecutionContext를 통한 상태 저장](./05-execution-context-state.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Skip·Retry Policy 구현 ➡️](./07-custom-policy.md)**

</div>
