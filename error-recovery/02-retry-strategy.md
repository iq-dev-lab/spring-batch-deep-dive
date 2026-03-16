# Retry 전략 — RetryTemplate과 BackOffPolicy로 일시적 오류 복구

---

## 🎯 핵심 질문

- `RetryTemplate`은 `RetryPolicy`, `BackOffPolicy`, `RetryCallback`을 어떻게 조합하는가?
- `SimpleRetryPolicy`와 `ExceptionClassifierRetryPolicy`는 언제 각각 사용하는가?
- 지수 백오프(`ExponentialBackOffPolicy`)가 선형 백오프보다 나은 이유는?
- Retry 중 트랜잭션은 어떻게 되는가? Retry마다 새 트랜잭션이 시작되는가?
- `retryLimit`을 초과했을 때 Skip과 연동하면 어떻게 되는가?

---

## 🔍 왜 Retry가 필요한가

### 문제: 일시적 오류는 즉시 실패 처리하면 안 된다

```
일시적 오류 vs 영구적 오류:

  일시적 오류 (Retry 적합):
    - 네트워크 타임아웃 (DB 연결 잠깐 끊김)
    - 동시성 충돌 (DeadlockLoserDataAccessException)
    - 외부 API 일시적 503/429 오류
    - DB 커넥션 풀 고갈 (잠시 후 복구)
    → 잠시 기다렸다가 재시도하면 성공 가능

  영구적 오류 (Retry 무의미):
    - 데이터 형식 오류 (ParseException)
    - DB 무결성 위반 (ConstraintViolationException)
    - 비즈니스 규칙 위반 (InsufficientBalanceException)
    → 재시도해도 동일 오류 발생 → Skip으로 처리

  즉시 실패 처리하면:
    네트워크 순간 장애 → 전체 배치 실패
    재시작 시 이미 처리된 수천 건 재처리
    → Retry로 일시적 오류는 그 자리에서 복구
```

---

## 😱 흔한 실수

### Before: 영구적 오류에 Retry를 걸어 처리 시간을 낭비한다

```java
// ❌ 데이터 오류에 Retry — 재시도해도 동일 실패
.faultTolerant()
    .retry(Exception.class)       // 모든 예외에 Retry
    .retryLimit(3)
    // ConstraintViolationException → 3번 재시도 → 3번 다 실패
    // → 각 Retry가 BackOff 대기 시간만큼 시간 낭비

// ✅ 일시적 오류만 Retry
.faultTolerant()
    .retry(TransientDataAccessException.class)  // 일시적 DB 오류만
    .retry(SocketTimeoutException.class)         // 네트워크 타임아웃만
    .noRetry(DataIntegrityViolationException.class)  // 데이터 오류 제외
    .retryLimit(3)
```

### Before: BackOff 없이 즉시 재시도해 외부 시스템에 부하를 준다

```java
// ❌ BackOff 없음 → 실패 즉시 재시도
.faultTolerant()
    .retry(ApiRateLimitException.class)
    .retryLimit(5)
    // API가 429 응답 → 즉시 재시도 → 또 429 → 무한 반복처럼 보임
    // 외부 API 서버에 순간 대량 요청 집중

// ✅ 지수 백오프로 점진적 대기
.faultTolerant()
    .retry(ApiRateLimitException.class)
    .retryLimit(5)
    .backOffPolicy(exponentialBackOffPolicy())
    // 1회 실패: 1초 대기 → 2회: 2초 → 3회: 4초 → 4회: 8초
```

---

## ✨ Retry 설정 완전 구문

```java
@Bean
public Step retryableStep() {
    return stepBuilderFactory.get("retryableStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .faultTolerant()
            .retry(TransientDataAccessException.class)   // 일시적 DB 오류
            .retry(SocketTimeoutException.class)          // 네트워크 타임아웃
            .noRetry(DataIntegrityViolationException.class) // 데이터 무결성 오류는 제외
            .retryLimit(3)                                // 최대 3회 재시도
            .backOffPolicy(exponentialBackOffPolicy())    // 지수 백오프
        .build();
}

@Bean
public BackOffPolicy exponentialBackOffPolicy() {
    ExponentialBackOffPolicy policy = new ExponentialBackOffPolicy();
    policy.setInitialInterval(1000L);   // 첫 대기: 1초
    policy.setMultiplier(2.0);          // 배수: 2배씩 증가
    policy.setMaxInterval(30000L);      // 최대 대기: 30초
    // 1초 → 2초 → 4초 → 8초 → 16초 → 30초(상한) → ...
    return policy;
}
```

---

## 🔬 내부 동작 원리

### 1. RetryTemplate 실행 흐름

```java
// RetryTemplate.java
public <T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback)
        throws E {
    return doExecute(retryCallback, null, new RetryState(null));
}

protected <T, E extends Throwable> T doExecute(RetryCallback<T, E> retryCallback,
        RecoveryCallback<T> recoveryCallback, RetryState state) throws E {

    RetryPolicy retryPolicy = this.retryPolicy;
    BackOffPolicy backOffPolicy = this.backOffPolicy;

    RetryContext context = open(retryPolicy, state);

    try {
        while (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
            try {
                // ① RetryCallback 실행 (실제 처리 로직)
                return retryCallback.doWithRetry(context);

            } catch (Throwable e) {
                // ② 예외 등록 → RetryPolicy가 재시도 여부 결정
                registerThrowable(retryPolicy, state, context, e);

                if (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
                    // ③ BackOff 대기 (지수 백오프 등)
                    backOffPolicy.backOff(backOffContext);
                    // ← 이 시점에 스레드 sleep
                }
                // canRetry = false이면 루프 탈출 → 최종 예외 throw
            } finally {
                close(retryPolicy, context, state, ...);
            }
        }
        // ④ 재시도 소진 → RecoveryCallback 또는 예외 throw
        if (recoveryCallback != null) {
            return recover(recoveryCallback, context);
        }
        throw (E) context.getLastThrowable();
    } finally {
        close(retryPolicy, context, state, ...);
    }
}
```

### 2. Spring Batch에서 RetryTemplate이 적용되는 위치

```java
// FaultTolerantChunkProcessor.doWithRetry()
// → RetryTemplate이 ItemProcessor/ItemWriter를 감쌈

// Process 단계 Retry:
private O doProcess(I item) throws Exception {
    RetryCallback<O, Exception> retryCallback = context -> {
        return itemProcessor.process(item);
    };

    return retryTemplate.execute(retryCallback, new ItemProcessRetryCallback(item));
    // retryTemplate: retryPolicy + backOffPolicy로 구성
    // 최대 retryLimit번 재시도
}

// Write 단계 Retry (1건씩 재처리 중):
private void doWithRetry(RetryContext context) throws Exception {
    RetryCallback<Object, Exception> writeCallback = ctx -> {
        doWrite(Collections.singletonList(item));
        return null;
    };
    retryTemplate.execute(writeCallback);
}
```

### 3. RetryPolicy 종류

```java
// 1. SimpleRetryPolicy — 횟수 기반 + 예외 클래스 화이트리스트
SimpleRetryPolicy policy = new SimpleRetryPolicy(
    3,                                           // maxAttempts
    Map.of(TransientDataAccessException.class, true,
           SocketTimeoutException.class, true)   // retryable 예외 목록
);

// 2. ExceptionClassifierRetryPolicy — 예외 종류마다 다른 정책
ExceptionClassifierRetryPolicy classifierPolicy = new ExceptionClassifierRetryPolicy();
classifierPolicy.setExceptionClassifier(new Classifier<Throwable, RetryPolicy>() {
    @Override
    public RetryPolicy classify(Throwable classifiable) {
        if (classifiable instanceof TransientDataAccessException) {
            return new SimpleRetryPolicy(5);  // DB 오류: 5번
        } else if (classifiable instanceof SocketTimeoutException) {
            return new SimpleRetryPolicy(3);  // 네트워크: 3번
        }
        return new NeverRetryPolicy();         // 나머지: 재시도 안 함
    }
});

// 3. TimeoutRetryPolicy — 시간 기반 재시도
TimeoutRetryPolicy timeoutPolicy = new TimeoutRetryPolicy();
timeoutPolicy.setTimeout(30000L);  // 30초 동안 재시도 허용

// 4. CompositeRetryPolicy — 여러 정책 조합
CompositeRetryPolicy composite = new CompositeRetryPolicy();
composite.setPolicies(new RetryPolicy[]{
    new SimpleRetryPolicy(3),
    new TimeoutRetryPolicy()  // 둘 다 허용해야 재시도 (optimistic=false 기본)
});
```

### 4. Retry 중 트랜잭션 처리

```java
// 핵심: Retry는 트랜잭션 내부에서 발생

// TaskletStep의 트랜잭션 템플릿:
// [트랜잭션 시작]
//   ChunkOrientedTasklet.execute()
//     → FaultTolerantChunkProcessor.process()
//       → RetryTemplate.execute(writeCallback)
//         → 1회: write 실패 → BackOff 대기
//         → 2회: write 재시도 → 성공
//   [트랜잭션 COMMIT]

// → 동일 트랜잭션 내에서 재시도
// → 마지막 성공 시 Chunk 전체 COMMIT
// → 마지막 실패 시 Chunk 전체 ROLLBACK

// 주의: TransactionSynchronizationManager.isCurrentTransactionReadOnly()
// → Retry는 같은 트랜잭션에서 발생하므로 1차 캐시, 락 등이 유지됨
// → 데드락 재시도 시 같은 트랜잭션에서 재시도하면 또 데드락 가능
// → 해결: noRollback(DeadlockLoserDataAccessException.class)  → 새 트랜잭션에서 재시도
```

### 5. BackOff 정책 비교

```
선형 백오프 (FixedBackOffPolicy):
  interval=1000ms
  1회: 1초 대기 → 2회: 1초 대기 → 3회: 1초 대기
  → 총 대기: 3초
  → 단순하지만 외부 시스템 부하 집중 (많은 클라이언트가 동시에 1초 후 재시도)

지수 백오프 (ExponentialBackOffPolicy):
  initial=1000ms, multiplier=2, max=30000ms
  1회: 1초 → 2회: 2초 → 3회: 4초 → 4회: 8초
  → 총 대기: 15초
  → 점진적 압력 감소 (Thundering Herd 방지)

랜덤 지수 백오프 (ExponentialRandomBackOffPolicy):
  랜덤 지수: 각 클라이언트가 다른 시점에 재시도
  → 동시 재시도 집중 방지 (실무 권장)

@Bean
public BackOffPolicy randomExponentialBackOff() {
    ExponentialRandomBackOffPolicy policy = new ExponentialRandomBackOffPolicy();
    policy.setInitialInterval(500L);   // 최소 0.5초
    policy.setMultiplier(2.0);
    policy.setMaxInterval(10000L);     // 최대 10초
    return policy;
}
```

---

## 💻 실전 구현

### 외부 API 호출 Processor에 Retry 적용

```java
@Bean
public Step apiEnrichmentStep() {
    return stepBuilderFactory.get("apiEnrichmentStep")
        .<Order, EnrichedOrder>chunk(100)   // 외부 API → Chunk 작게
        .reader(reader())
        .processor(apiEnrichmentProcessor())
        .writer(writer())
        .faultTolerant()
            .retry(RestClientException.class)      // 네트워크 오류
            .retry(HttpServerErrorException.class) // 5xx 서버 오류
            .noRetry(HttpClientErrorException.class) // 4xx 클라이언트 오류 (재시도 의미 없음)
            .retryLimit(5)
            .backOffPolicy(randomExponentialBackOff())
            // Retry 모두 소진 시 Skip으로 전환
            .skip(RestClientException.class)
            .skip(HttpServerErrorException.class)
            .skipLimit(10)                          // 10건까지 API 실패 허용
        .build();
}

// RetryListener로 Retry 시도 추적
@Bean
public RetryListener retryAuditListener() {
    return new RetryListenerSupport() {
        @Override
        public <T, E extends Throwable> void onError(
                RetryContext context,
                RetryCallback<T, E> callback,
                Throwable throwable) {
            log.warn("[Retry #{}/{}] {}",
                context.getRetryCount(),
                maxAttempts,
                throwable.getMessage());
        }

        @Override
        public <T, E extends Throwable> void close(
                RetryContext context,
                RetryCallback<T, E> callback,
                Throwable throwable) {
            if (throwable != null && context.isExhaustedOnly()) {
                log.error("[Retry 소진] 최종 실패: {}", throwable.getMessage());
            }
        }
    };
}
```

---

## 📊 BackOff 정책 비교

```
┌──────────────────────────────┬────────────────────────────────┬─────────────────────┐
│ BackOffPolicy                │ 대기 패턴                        │ 적합한 상황            │
├──────────────────────────────┼────────────────────────────────┼─────────────────────┤
│ NoBackOffPolicy              │ 즉시 재시도 (0ms)                 │ 아주 빠른 일시 오류     │
│ FixedBackOffPolicy           │ 1초, 1초, 1초 (고정)              │ 단순 시스템           │
│ ExponentialBackOffPolicy     │ 1초, 2초, 4초, 8초...            │ 대부분의 경우          │
│ExponentialRandomBackOffPolicy│ 랜덤 지수                        │ 대용량 분산 배치        │
│ UniformRandomBackOffPolicy   │ 균등 분포 랜덤                    │ 짧은 대기 허용 시       │
└──────────────────────────────┴────────────────────────────────┴─────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Retry 전략:

  장점
    일시적 오류를 현재 위치에서 복구 (재시작 불필요)
    외부 시스템 일시 장애에 유연하게 대응
    BackOff로 외부 시스템 회복 시간 확보

  단점
    BackOff 대기 시간만큼 처리 시간 증가
    Retry 중 트랜잭션 유지 → 잠금(Lock) 시간 증가
    영구적 오류에 Retry 적용 시 시간 낭비

Retry + Skip 조합:
  Retry 먼저 실행 → 소진 시 Skip으로 전환
  → 일시적 오류: Retry로 자체 복구
  → 영구적 오류 또는 Retry 소진: Skip으로 격리
  상세 내용은 Ch4-03 참조
```

---

## 📌 핵심 정리

```
RetryTemplate 실행 순서
  1. RetryCallback 실행 (실제 처리)
  2. 예외 발생 → RetryPolicy.canRetry() 확인
  3. canRetry=true → BackOffPolicy.backOff() 대기
  4. 1번으로 돌아가 재시도
  5. canRetry=false → RecoveryCallback 또는 예외 throw

RetryPolicy 선택
  SimpleRetryPolicy          → 단순 횟수 + 예외 화이트리스트
  ExceptionClassifierRetryPolicy → 예외마다 다른 재시도 횟수
  TimeoutRetryPolicy         → 시간 기반 재시도 허용

BackOffPolicy 권장
  ExponentialRandomBackOffPolicy: 대부분의 실무 상황에 적합
  initial=500ms, multiplier=2, max=30초

Retry + 트랜잭션
  동일 트랜잭션 내에서 재시도 (기본)
  noRollback() 설정 시 재시도마다 새 트랜잭션 가능
```

---

## 🤔 생각해볼 문제

**Q1.** Retry가 동일 트랜잭션 내에서 발생하는 상황에서 `DeadlockLoserDataAccessException`이 발생했습니다. 같은 트랜잭션에서 재시도하면 또 데드락이 발생할 가능성이 높습니다. 이를 해결하는 방법은?

**Q2.** `retryLimit(3)`으로 설정했고 처음 2번 실패, 3번째에 성공했습니다. 이 아이템의 `retryCount`는 `BATCH_STEP_EXECUTION`에 어떻게 기록되는가?

**Q3.** `ExponentialBackOffPolicy`에서 `maxInterval(30000)`을 설정했는데 `retryLimit(100)`으로 설정하면 최악의 경우 총 대기 시간은 얼마인가?

> 💡 **해설**
>
> **Q1.** `noRollback(DeadlockLoserDataAccessException.class)` 설정을 사용합니다. 이 설정은 해당 예외 발생 시 트랜잭션을 롤백하지 않고 Retry를 시도하게 합니다. 단, 데드락은 트랜잭션이 롤백돼야 해소되므로, 더 나은 방법은 Chunk 크기를 줄여 잠금 범위를 축소하거나 처리 순서를 일관되게 하는 것입니다. 또한 `BackOffPolicy`를 추가해 잠시 대기 후 재시도하면 상대 트랜잭션이 먼저 완료될 가능성이 높아집니다.
>
> **Q2.** `BATCH_STEP_EXECUTION`에는 `retryCount` 관련 컬럼이 없습니다. Spring Batch는 Step 단위의 집계 통계(readCount, writeCount 등)만 기록하며, Retry 횟수는 집계하지 않습니다. Retry 시도 추적이 필요하다면 `RetryListener.onError()`에서 직접 로깅하거나 `ExecutionContext`에 누적해 저장해야 합니다. `ROLLBACK_COUNT` 컬럼이 Retry와 관련이 있지만, Retry 성공 시에는 최종적으로 커밋되므로 증가하지 않습니다.
>
> **Q3.** 최악의 경우: 1~3회는 1초 → 2초 → 4초, 4회부터 30초(상한)에 도달. 4회 이후는 모두 30초 대기. `retryLimit(100)`이면 총 100번 중 첫 5번(1+2+4+8+16=31초)을 제외한 95번이 30초씩 대기: 95 × 30초 = 2850초 + 31초 ≈ **49분**. 실무에서 `retryLimit`을 높게 설정할 때는 `maxInterval`과의 조합으로 총 대기 시간을 반드시 계산해야 합니다. 일반적으로 `retryLimit(3~5)`, `maxInterval(30초)` 정도가 실용적입니다.

---

<div align="center">

**[⬅️ 이전: Skip 전략](./01-skip-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: Skip vs Retry 선택 기준 ➡️](./03-skip-vs-retry.md)**

</div>
