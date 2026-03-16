# Skip vs Retry 선택 기준 — 오류 성격으로 전략을 결정하는 방법

---

## 🎯 핵심 질문

- 일시적 오류와 영구적 오류를 구분하는 실무 기준은 무엇인가?
- Skip과 Retry를 동시에 설정했을 때 실행 순서는 어떻게 되는가?
- `noSkip()`과 `noRetry()`는 각각 어떤 예외에 적용해야 하는가?
- Retry가 소진된 후 자동으로 Skip으로 전환되는가?
- 같은 예외 클래스에 Skip과 Retry를 동시에 설정하면 어떻게 동작하는가?

---

## 🔍 오류 분류가 먼저다

### 문제: 모든 예외를 같은 방식으로 처리하면 안 된다

```
예외 분류 기준:

  일시적 오류 (Transient) → Retry 우선
    특징: 잠시 후 동일 요청이 성공할 가능성 있음
    예시:
      - TransientDataAccessException (DB 일시 연결 오류)
      - SocketTimeoutException (네트워크 순간 장애)
      - ServiceUnavailableException (외부 API 503)
      - LockAcquisitionException (락 획득 타임아웃)
      - DeadlockLoserDataAccessException (데드락)

  영구적 오류 (Permanent) → Skip 우선
    특징: 동일 요청을 몇 번 반복해도 동일하게 실패
    예시:
      - DataIntegrityViolationException (PK/FK 충돌)
      - ParseException (데이터 형식 오류)
      - NumberFormatException (숫자 파싱 실패)
      - ConstraintViolationException (Not Null 등 DB 제약 위반)
      - ClassCastException (타입 불일치)

  절대 Skip/Retry 금지 → noSkip + noRetry
    특징: 발생 시 배치를 즉시 중단해야 하는 치명적 오류
    예시:
      - InsufficientBalanceException (잔액 부족 — 비즈니스 로직 오류)
      - AccountFrozenException (계좌 동결 — 운영 개입 필요)
      - OutOfMemoryError (JVM 위기 — 즉시 중단)
```

---

## 😱 흔한 실수

### Before: Retry와 Skip을 무분별하게 같은 예외에 중복 적용한다

```java
// ❌ 같은 예외에 Skip과 Retry 중복 — 의도와 다른 동작
.faultTolerant()
    .retry(DataIntegrityViolationException.class)  // Retry 설정
    .skip(DataIntegrityViolationException.class)   // Skip도 설정
    .retryLimit(3)
    .skipLimit(10)
// 동작:
// DataIntegrityViolationException 발생 → Retry 3번 → 모두 실패
// → 자동으로 Skip 전환? ← 아님! retryLimit 초과 = Step FAILED

// ✅ 의도를 명확히 분리
// 일시적 DB 오류: Retry (데드락 등)
.retry(DeadlockLoserDataAccessException.class).retryLimit(3)
// 영구적 DB 오류: Skip (무결성 위반 등)
.skip(DataIntegrityViolationException.class).skipLimit(10)
```

### Before: Retry 소진 = 자동 Skip이라고 생각한다

```java
// ❌ 잘못된 이해
// "retryLimit 도달 → 자동으로 Skip으로 처리"

// 실제:
// retryLimit 초과 → RetryCacheMissException 또는 원래 예외 throw
// → FaultTolerantChunkProcessor가 shouldSkip() 확인
//   → 해당 예외가 skip() 목록에 있으면 Skip
//   → 없으면 Step FAILED

// 즉, Retry 소진 후 Skip으로 전환하려면
// 해당 예외를 retry()와 skip() 양쪽에 등록해야 함

.faultTolerant()
    .retry(ApiException.class)
    .retryLimit(3)
    .skip(ApiException.class)   // ← retry 소진 후 skip으로 전환
    .skipLimit(5)
```

---

## ✨ Skip + Retry 동시 설정 시 실행 순서

### 실행 흐름 상세

```java
// 같은 예외에 retry + skip 설정
.faultTolerant()
    .retry(TransientDataAccessException.class)
    .retryLimit(3)
    .skip(TransientDataAccessException.class)
    .skipLimit(10)

// 실행 흐름:
// ① TransientDataAccessException 발생 (1회차)
// ② RetryPolicy.canRetry()? retryCount(0) < retryLimit(3) → YES
// ③ BackOff 대기 → 재시도 (2회차)
// ④ 또 실패 → canRetry()? retryCount(1) < 3 → YES → 재시도 (3회차)
// ⑤ 또 실패 → canRetry()? retryCount(2) < 3 → NO (3회 소진)
// ⑥ 예외 throw → FaultTolerantChunkProcessor에서 shouldSkip() 확인
// ⑦ TransientDataAccessException이 skip 목록에 있음 → skipCount++ → Skip
// ⑧ onSkipInWrite(item, e) 호출

// 핵심: Retry가 모두 소진된 후 → 그 예외가 Skip 목록에도 있으면 → Skip 전환
```

---

## 🔬 내부 동작 원리

### 1. FaultTolerantChunkProcessor 예외 처리 결정 트리

```java
// FaultTolerantChunkProcessor.scan() 내 예외 처리
catch (Exception e) {
    // 결정 순서:
    // ① noRetry 예외인가? → 재시도 없이 shouldSkip() 확인
    // ② noSkip 예외인가? → Skip 불가 → throw
    // ③ Retry 가능하고 retryCount < retryLimit? → 재시도
    // ④ Retry 소진 → shouldSkip()? → Skip or throw

    if (isNoRetry(e)) {
        // ① 직접 Skip 여부 확인
        if (shouldSkip(skipPolicy, e, skipCount)) {
            skip(item, e);
        } else {
            throw e;
        }
    } else if (isRetryable(e) && context.getRetryCount() < retryLimit) {
        // ③ 재시도
        throw e;  // RetryTemplate이 catch해 재시도
    } else {
        // ④ 재시도 소진 or 재시도 불가 → Skip 확인
        if (shouldSkip(skipPolicy, e, skipCount)) {
            skip(item, e);
        } else {
            throw e;  // 최종 실패
        }
    }
}
```

### 2. noSkip과 noRetry의 내부 구현

```java
// noSkip: SkipPolicy에서 해당 예외를 항상 false 반환
// noRetry: RetryPolicy에서 해당 예외를 항상 false 반환

// LimitCheckingItemSkipPolicy.shouldSkip():
public boolean shouldSkip(Throwable t, long skipCount) {
    // noSkip으로 지정된 예외: 즉시 false 반환 (skipLimit과 무관)
    if (nonSkippableExceptions.contains(t.getClass())) {
        return false;
    }
    // 일반 skip 가능 예외: skipLimit 확인
    if (skippableExceptions.contains(t.getClass())) {
        if (skipCount < skipLimit) return true;
        throw new SkipLimitExceededException(skipLimit, t);
    }
    return false;
}
```

### 3. 예외 계층 구조와 isAssignableFrom

```java
// Spring Batch는 예외 계층 구조를 고려함
// DataAccessException (상위)
//   └── TransientDataAccessException (중간)
//         └── DeadlockLoserDataAccessException (하위)

.skip(TransientDataAccessException.class)
// → DeadlockLoserDataAccessException도 skip 가능 (상위 클래스 포함)

.noSkip(DeadlockLoserDataAccessException.class)
// → 더 구체적인 noSkip이 우선 적용
// DeadlockLoserDataAccessException → noSkip 우선 → Skip 불가
// 그 외 TransientDataAccessException → skip 가능

// 우선순위: noSkip > noRetry > skip > retry
// 더 구체적인 설정이 더 일반적인 설정보다 우선
```

### 4. 예외별 전략 결정 가이드

```
                     오류 발생
                        │
                ┌───────┴────────┐
                ▼                ▼
          일시적 오류?        영구적 오류?
          (재시도 시 성공 가능)  (재시도 의미 없음)
                │                │
                ▼                ▼
        Retry + BackOff      데이터 격리 가능?
                │            ┌──┴──┐
                │            ▼     ▼
                │         YES      NO
                │     Skip으로   noSkip + noRetry
                │     격리       → Step FAILED
                │     (오류 기록)   (운영 개입)
                │
                ▼
        Retry 소진 시 어떻게?
          ┌─────┴──────┐
          ▼            ▼
       격리 가능     치명적
       Skip으로      noSkip 추가
       전환          → Step FAILED
```

---

## 💻 실전 구현

### 오류 유형별 완전한 전략 설정

```java
@Bean
public Step robustSettlementStep() {
    return stepBuilderFactory.get("robustSettlementStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(orderReader())
        .processor(settlementProcessor())
        .writer(settlementWriter())
        .faultTolerant()

            // === Retry 전략: 일시적 오류 ===
            .retry(TransientDataAccessException.class)   // DB 일시 오류
            .retry(DeadlockLoserDataAccessException.class) // 데드락
            .retry(ApiTimeoutException.class)             // API 타임아웃
            .retryLimit(3)
            .backOffPolicy(exponentialRandomBackOff())

            // === Skip 전략: 영구적 데이터 오류 ===
            .skip(DataIntegrityViolationException.class) // PK/FK 충돌
            .skip(ParseException.class)                  // 형식 오류
            .skip(NumberFormatException.class)           // 숫자 파싱 오류
            // Retry 소진 후 Skip 전환용 (retry 목록과 교차)
            .skip(ApiTimeoutException.class)

            .skipLimit(100)  // 전체의 0.01% (100만건 기준)

            // === 절대 Skip/Retry 금지: 치명적 오류 ===
            .noSkip(InsufficientBalanceException.class)
            .noSkip(AccountFrozenException.class)
            .noRetry(DataIntegrityViolationException.class) // 데이터 오류 재시도 무의미

            .listener(errorStrategyListener())
        .build();
}

// 오류 전략 Listener: 전략 결정 로그
@Bean
public ItemSkipListener<Order, SettledOrder> errorStrategyListener() {
    return new ItemListenerSupport<>() {
        @Override
        public void onSkipInProcess(Order item, Throwable t) {
            log.warn("[Process Skip] orderId={}, type={}: {}",
                item.getId(), t.getClass().getSimpleName(), t.getMessage());
            // 오류 유형 분류 기록
            auditService.recordSkip("PROCESS", item.getId(),
                classifyError(t), t.getMessage());
        }

        @Override
        public void onSkipInWrite(Object item, Throwable t) {
            SettledOrder order = (SettledOrder) item;
            log.warn("[Write Skip] orderId={}, type={}: {}",
                order.getOrderId(), t.getClass().getSimpleName(), t.getMessage());
            auditService.recordSkip("WRITE", order.getOrderId(),
                classifyError(t), t.getMessage());
        }

        private String classifyError(Throwable t) {
            if (t instanceof TransientDataAccessException) return "TRANSIENT_DB";
            if (t instanceof DataIntegrityViolationException) return "INTEGRITY";
            if (t instanceof ParseException) return "FORMAT_ERROR";
            return "UNKNOWN";
        }
    };
}
```

---

## 📊 예외 유형별 전략 정리

```
┌───────────────────────────────────┬────────────┬──────────┬──────────────────────────────┐
│ 예외 유형                           │ Retry      │ Skip     │ 이유                          │
├───────────────────────────────────┼────────────┼──────────┼──────────────────────────────┤
│ TransientDataAccessException      │ ✅ 3회     │ ✅ 소진   │ 일시적 DB 오류                  │
│ DeadlockLoserDataAccessException  │ ✅ 3회     │ ✅ 소진   │ 잠시 후 재시도 시 성공 가능        │
│ SocketTimeoutException            │ ✅ 3회     │ ✅ 소진   │ 네트워크 일시 불안정              │
│ DataIntegrityViolationException   │ ❌        │ ✅       │ 재시도해도 동일 실패              │
│ ParseException                    │ ❌        │ ✅       │ 데이터 자체 오류                 │
│ InsufficientBalanceException      │ ❌ noRetry│ ❌ noSkip│ 비즈니스 오류, 운영 개입           │
│ OutOfMemoryError                  │ ❌        │ ❌       │ JVM 위기, 즉시 중단              │
└───────────────────────────────────┴────────────┴──────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Skip + Retry 조합:

  장점
    일시적 오류: 현재 위치에서 자동 복구 (Retry)
    영구적 오류: 격리하고 계속 진행 (Skip)
    치명적 오류: 즉시 중단 (noSkip + noRetry)
    → 오류 성격에 맞는 최적 처리

  단점
    설정 복잡도 증가 (skip/retry/noSkip/noRetry 조합)
    예외 계층 구조 이해 필요 (상위/하위 클래스 충돌)
    동일 예외에 양쪽 설정 시 의도치 않은 동작 가능

설정 원칙:
  1. 예외를 일시적/영구적/치명적으로 분류
  2. 일시적 → retry + (소진 시 skip 가능하면 skip도 추가)
  3. 영구적 → skip만
  4. 치명적 → noSkip + noRetry
  5. 예외 계층 구조 확인 (하위 예외에 noRetry → 상위 retry 무력화)
```

---

## 📌 핵심 정리

```
Skip vs Retry 결정 기준

  일시적 오류 (재시도 성공 가능) → Retry + BackOff
  영구적 오류 (재시도 무의미)    → Skip (데이터 격리)
  치명적 오류 (운영 개입 필요)   → noSkip + noRetry → Step FAILED

Retry 소진 후 Skip 전환
  같은 예외를 retry()와 skip() 양쪽에 등록
  → Retry 3회 소진 → shouldSkip() 확인 → Skip 전환

실행 우선순위
  noSkip > noRetry > skip > retry
  더 구체적인 예외 설정이 더 일반적인 설정보다 우선

핵심: 예외를 먼저 분류하라
  예외 종류 → 오류 성격 파악 → 전략 결정
  설정 먼저가 아닌 분류 먼저
```

---

## 🤔 생각해볼 문제

**Q1.** `retry(TransientDataAccessException.class).retryLimit(3)` + `skip(TransientDataAccessException.class).skipLimit(10)` 설정에서, Skip이 이미 10건 발생한 상황입니다. 11번째 아이템에서 Retry 3회 모두 실패했을 때 어떻게 되는가?

**Q2.** `noSkip(DataIntegrityViolationException.class)`를 설정했는데, `DataIntegrityViolationException`의 상위 클래스인 `DataAccessException`을 `skip(DataAccessException.class)`에 등록했습니다. `DataIntegrityViolationException` 발생 시 어떻게 처리되는가?

**Q3.** Retry 중 BackOff 대기 시간이 길어지면 Step의 트랜잭션 타임아웃이 발생할 수 있습니다. `ExponentialBackOffPolicy(initial=1초, multiplier=2, max=30초)`와 `retryLimit(5)`인 경우 최대 대기 시간과 트랜잭션 타임아웃 설정의 관계는?

> 💡 **해설**
>
> **Q1.** 11번째 아이템에서 Retry 3회 소진 → `shouldSkip()` 확인 → 현재 `skipCount=10` ≥ `skipLimit(10)` → `SkipLimitExceededException` 발생 → Step FAILED입니다. `skipLimit`은 "총 Skip 허용 건수"이므로 이미 10건이 Skip됐으면 더 이상 Skip이 불가합니다. Retry 소진 후 Skip 전환이 되려면 `skipCount < skipLimit` 조건을 만족해야 합니다.
>
> **Q2.** Spring Batch의 Skip/noSkip은 예외 계층 구조와 구체성을 고려합니다. `noSkip(DataIntegrityViolationException.class)`는 더 구체적인 설정이고, `skip(DataAccessException.class)`는 더 일반적인 설정입니다. 구체적인 설정이 우선 적용됩니다 — `DataIntegrityViolationException`이 발생하면 `noSkip`이 적용되어 Skip 불가, 예외 전파 → Step FAILED입니다. `DataAccessException`의 다른 하위 클래스들은 `skip`이 적용됩니다.
>
> **Q3.** Retry 5회의 최대 대기: 1초 + 2초 + 4초 + 8초 + 16초(30초 상한 미달) = 31초입니다. 트랜잭션 타임아웃(Spring `@Transactional(timeout=...)` 또는 DB 세션 타임아웃)이 30초라면 Retry 대기 중 트랜잭션이 만료될 수 있습니다. 해결책: (1) 트랜잭션 타임아웃을 Retry 총 대기시간보다 크게 설정합니다 (예: 60초). (2) `maxInterval`을 줄여 총 대기시간을 제한합니다. (3) `retryLimit`을 줄여서 BackOff 총 시간을 줄입니다. `ExponentialRandomBackOffPolicy`를 사용하면 최대 대기가 예측 불가하므로 특히 주의가 필요합니다.

---

<div align="center">

**[⬅️ 이전: Retry 전략](./02-retry-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: Job 재시작과 Restartability ➡️](./04-job-restart.md)**

</div>
