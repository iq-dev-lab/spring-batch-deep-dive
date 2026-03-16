# Custom Skip·Retry Policy 구현 — 비즈니스 규칙 기반 오류 처리

---

## 🎯 핵심 질문

- `SkipPolicy.shouldSkip(Throwable, long)`의 두 파라미터는 각각 무엇을 의미하는가?
- 예외 클래스 기반 정책과 비즈니스 규칙 기반 정책의 차이는?
- 비율 기반 Skip 정책(전체의 5% 이내만 허용)은 어떻게 구현하는가?
- Custom RetryPolicy에서 예외 내용에 따라 재시도 횟수를 다르게 적용하는 방법은?
- `ItemSkipListener`와 Custom SkipPolicy를 연동해 Skip 발생 시 알림을 보내는 패턴은?

---

## 🔍 왜 Custom Policy가 필요한가

### 문제: 기본 정책으로는 표현할 수 없는 비즈니스 규칙이 있다

```
기본 LimitCheckingItemSkipPolicy의 한계:

  제공 기능:
    - 예외 클래스 화이트리스트 (DataIntegrityViolationException → Skip)
    - 건수 상한 (skipLimit=100)

  표현할 수 없는 규칙:
    - "전체 처리 건수의 5% 이내만 Skip 허용"
      → 100만 건 처리 시 5만 건까지, 1만 건 처리 시 500건까지
    - "특정 필드(customer_grade='VIP')가 오염되면 절대 Skip 금지"
      → 예외 클래스가 아닌 아이템 내용으로 판단
    - "같은 고객의 주문이 3번 이상 Skip되면 해당 고객 전체 중단"
      → 상태 기반 복잡한 규칙
    - "오전 9시~11시 배치 시간대에는 skipLimit를 늘림"
      → 시간 기반 동적 정책

  → Custom SkipPolicy로 자유롭게 구현
```

---

## 😱 흔한 실수

### Before: SkipPolicy에서 예외를 삼켜 skipLimit를 우회한다

```java
// ❌ shouldSkip()에서 모든 예외를 true 반환 — skipLimit 무시
@Override
public boolean shouldSkip(Throwable t, long skipCount) {
    log.error("오류 발생, Skip 처리: {}", t.getMessage());
    return true;  // 항상 Skip → skipLimit 무시!
}
// skipLimit 체크 누락 → 무한정 Skip → 대규모 데이터 오류 조용히 무시

// ✅ skipLimit 체크 포함 필수
@Override
public boolean shouldSkip(Throwable t, long skipCount)
        throws SkipLimitExceededException {
    if (isSkippable(t)) {
        if (skipCount < skipLimit) {
            return true;
        }
        throw new SkipLimitExceededException(skipLimit, t);  // 한계 초과
    }
    return false;
}
```

---

## ✨ SkipPolicy 인터페이스 이해

```java
// SkipPolicy 인터페이스
public interface SkipPolicy {

    /**
     * @param t         발생한 예외
     * @param skipCount 현재 Step 실행에서 지금까지 Skip된 총 건수
     *                  (Read + Process + Write Skip 합산)
     * @return true  → 이 아이템을 Skip (계속 진행)
     *         false → Skip 거부 (예외 전파 → Step FAILED 가능성)
     * @throws SkipLimitExceededException → Skip 한계 초과 시 명시적 예외
     */
    boolean shouldSkip(Throwable t, long skipCount) throws SkipLimitExceededException;
}
```

---

## 🔬 내부 동작 원리

### 1. shouldSkip() 호출 위치와 시점

```java
// FaultTolerantChunkProvider (Read 단계):
catch (Exception e) {
    if (skipPolicy.shouldSkip(e, contribution.getStepSkipCount())) {
        contribution.incrementReadSkipCount();
        // → Read Skip 발생
    } else {
        throw e;
    }
}

// FaultTolerantChunkProcessor.scan() (Write 스캐터-개더):
catch (Exception e) {
    if (skipPolicy.shouldSkip(e, contribution.getStepSkipCount())) {
        contribution.incrementWriteSkipCount();
        // → Write Skip 발생
    } else {
        throw e;
    }
}

// skipCount 파라미터:
// contribution.getStepSkipCount() = readSkipCount + processSkipCount + writeSkipCount
// 현재 Step 실행에서 지금까지 발생한 모든 Skip의 합산
```

---

## 💻 실전 구현

### 1. 비율 기반 Skip Policy

```java
/**
 * 전체 처리 건수 대비 일정 비율 이내만 Skip 허용
 * 예: 5% 비율 → 100만 건 처리 시 5만 건까지 Skip 허용
 */
@Component
public class RatioBasedSkipPolicy implements SkipPolicy {

    private final double skipRatio;            // 허용 비율 (예: 0.05 = 5%)
    private final int absoluteMinSkipLimit;    // 비율 계산 전 최소 허용 건수
    private final Set<Class<? extends Throwable>> skippableExceptions;

    public RatioBasedSkipPolicy(double skipRatio,
                                  int absoluteMinSkipLimit,
                                  Set<Class<? extends Throwable>> skippableExceptions) {
        this.skipRatio = skipRatio;
        this.absoluteMinSkipLimit = absoluteMinSkipLimit;
        this.skippableExceptions = skippableExceptions;
    }

    @Override
    public boolean shouldSkip(Throwable t, long skipCount)
            throws SkipLimitExceededException {

        if (!isSkippable(t)) {
            return false;  // Skip 대상 예외가 아님
        }

        // StepExecution에서 현재까지 처리된 총 건수 조회
        // (실제 구현에서는 StepExecution 주입 필요)
        long totalRead = getCurrentReadCount();

        // 최소 건수 보장: 처리 건수가 적을 때 비율 계산 무의미
        long dynamicLimit = Math.max(
            absoluteMinSkipLimit,
            (long) (totalRead * skipRatio)
        );

        if (skipCount < dynamicLimit) {
            log.debug("Skip 허용: skipCount={}/{} (ratio={}%, totalRead={})",
                skipCount + 1, dynamicLimit,
                String.format("%.1f", skipRatio * 100),
                totalRead);
            return true;
        }

        log.error("Skip 한계 초과: skipCount={}, limit={}, ratio={}%",
            skipCount, dynamicLimit, String.format("%.1f", skipRatio * 100));
        throw new SkipLimitExceededException((int) dynamicLimit, t);
    }

    private boolean isSkippable(Throwable t) {
        return skippableExceptions.stream()
            .anyMatch(clazz -> clazz.isAssignableFrom(t.getClass()));
    }

    private long getCurrentReadCount() {
        // StepSynchronizationManager에서 현재 StepExecution 접근
        StepContext context = StepSynchronizationManager.getContext();
        if (context == null) return 0;
        return context.getStepExecution().getReadCount()
            + context.getStepExecution().getReadSkipCount();
    }
}

// 설정 예시:
@Bean
public SkipPolicy ratioSkipPolicy() {
    return new RatioBasedSkipPolicy(
        0.05,  // 5% 비율
        10,    // 최소 10건은 항상 허용
        Set.of(DataIntegrityViolationException.class, ParseException.class)
    );
}

@Bean
public Step ratioSkipStep() {
    return stepBuilderFactory.get("ratioSkipStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader()).writer(writer())
        .faultTolerant()
            .skipPolicy(ratioSkipPolicy())  // 비율 기반 정책 적용
        .build();
}
```

### 2. 아이템 내용 기반 Skip Policy (예외 내용 분석)

```java
/**
 * 예외의 원인(원본 아이템 내용)에 따라 Skip 여부 결정
 * VIP 고객 관련 오류는 절대 Skip 금지
 */
@Component
public class BusinessRuleSkipPolicy implements SkipPolicy {

    private static final int DEFAULT_SKIP_LIMIT = 100;

    @Override
    public boolean shouldSkip(Throwable t, long skipCount)
            throws SkipLimitExceededException {

        // ① 치명적 예외: 절대 Skip 금지
        if (t instanceof CriticalBusinessException) {
            log.error("치명적 예외 — Skip 거부: {}", t.getMessage());
            return false;  // 예외 전파 → Step FAILED
        }

        // ② VIP 고객 오류: 절대 Skip 금지
        if (t instanceof OrderProcessingException) {
            OrderProcessingException ope = (OrderProcessingException) t;
            if ("VIP".equals(ope.getCustomerGrade())) {
                log.error("VIP 고객 처리 오류 — Skip 거부: customerId={}",
                    ope.getCustomerId());
                return false;  // VIP는 Skip 안 함
            }
        }

        // ③ 일반 데이터 오류: skipLimit 내에서 Skip 허용
        if (t instanceof DataIntegrityViolationException
                || t instanceof ParseException) {
            if (skipCount < DEFAULT_SKIP_LIMIT) {
                return true;
            }
            throw new SkipLimitExceededException(DEFAULT_SKIP_LIMIT, t);
        }

        return false;  // 기타 예외: Skip 거부
    }
}
```

### 3. Custom RetryPolicy — 예외 내용별 재시도 횟수 차별화

```java
/**
 * HTTP 상태 코드에 따라 재시도 횟수를 다르게 적용
 */
@Component
public class HttpStatusAwareRetryPolicy implements RetryPolicy {

    @Override
    public boolean canRetry(RetryContext context) {
        Throwable lastThrowable = context.getLastThrowable();

        if (lastThrowable instanceof HttpClientException) {
            HttpClientException e = (HttpClientException) lastThrowable;
            int statusCode = e.getStatusCode();

            if (statusCode == 429) {  // Too Many Requests
                // 429: 속도 제한 → 많이 재시도
                return context.getRetryCount() < 10;
            } else if (statusCode == 503) {  // Service Unavailable
                // 503: 서버 일시 불가 → 중간 재시도
                return context.getRetryCount() < 5;
            } else if (statusCode >= 500) {  // 기타 서버 오류
                // 5xx: 3회
                return context.getRetryCount() < 3;
            } else {
                // 4xx: 재시도 의미 없음
                return false;
            }
        }

        if (lastThrowable instanceof SocketTimeoutException) {
            return context.getRetryCount() < 3;  // 네트워크 타임아웃: 3회
        }

        return false;  // 기타: 재시도 안 함
    }

    @Override
    public RetryContext open(RetryContext parent) {
        return new SimpleRetryContext(parent);
    }

    @Override
    public void close(RetryContext context) {}

    @Override
    public void registerThrowable(RetryContext context, Throwable throwable) {
        ((SimpleRetryContext) context).registerThrowable(throwable);
    }
}
```

### 4. ItemSkipListener 연동 — Skip 알림 + 오류 아이템 저장

```java
/**
 * Skip 발생 시:
 * 1. 오류 아이템을 dead_letter 테이블에 저장
 * 2. Skip 건수 임계값 초과 시 알림
 * 3. 오류 분류 통계 수집
 */
@Component
public class AuditSkipListener implements SkipListener<Order, SettledOrder> {

    @Autowired private DeadLetterRepository deadLetterRepo;
    @Autowired private AlertService alertService;
    @Autowired private MetricsService metricsService;

    private final AtomicInteger skipCount = new AtomicInteger(0);
    private static final int ALERT_THRESHOLD = 10;

    @Override
    public void onSkipInRead(Throwable t) {
        // Read Skip: 아이템 내용 모름
        int count = skipCount.incrementAndGet();
        log.warn("[Read Skip #{}] 원인: {}", count, t.getMessage());
        metricsService.incrementCounter("batch.skip.read");

        if (count >= ALERT_THRESHOLD) {
            alertService.sendAlert("Read Skip " + count + "건 발생");
        }
    }

    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        int count = skipCount.incrementAndGet();
        log.warn("[Process Skip #{}] orderId={}, 원인: {}",
            count, item.getId(), t.getMessage());

        // dead_letter 테이블에 저장
        deadLetterRepo.save(DeadLetter.builder()
            .sourceTable("orders")
            .sourceId(item.getId().toString())
            .errorStage("PROCESS")
            .errorType(t.getClass().getSimpleName())
            .errorMessage(t.getMessage())
            .rawData(item.toString())
            .build());

        metricsService.incrementCounter("batch.skip.process",
            "errorType", t.getClass().getSimpleName());
    }

    @Override
    public void onSkipInWrite(SettledOrder item, Throwable t) {
        int count = skipCount.incrementAndGet();
        log.warn("[Write Skip #{}] orderId={}, 원인: {}",
            count, item.getOrderId(), t.getMessage());

        deadLetterRepo.save(DeadLetter.builder()
            .sourceTable("settled_orders")
            .sourceId(item.getOrderId().toString())
            .errorStage("WRITE")
            .errorType(t.getClass().getSimpleName())
            .errorMessage(t.getMessage())
            .rawData(item.toString())
            .build());

        // 임계값 초과 시 즉각 알림
        if (count >= ALERT_THRESHOLD) {
            alertService.sendUrgentAlert(
                "Write Skip " + count + "건: " + t.getMessage());
        }
    }
}
```

---

## 📊 Custom Policy 유형 비교

```
┌─────────────────────────────┬───────────────────────────────┬──────────────────────┐
│ 정책 유형                     │ 판단 기준                       │ 사용 시나리오            │
├─────────────────────────────┼───────────────────────────────┼──────────────────────┤
│ LimitCheckingItemSkipPolicy │ 예외 클래스 + 건수 상한            │ 단순 케이스 (기본값)     │
│ RatioBasedSkipPolicy        │ 처리 건수 대비 비율               │ 데이터 품질 기준         │
│ BusinessRuleSkipPolicy      │ 아이템 내용 (VIP, 등급 등)        │ 도메인 규칙 기반         │
│ HttpStatusAwareRetryPolicy  │ HTTP 상태 코드                  │ 외부 API 통합          │
│ CompositeSkipPolicy         │ 여러 정책 조합 (AND/OR)          │ 복합 규칙              │
└─────────────────────────────┴───────────────────────────────┴──────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Custom SkipPolicy:

  장점
    비즈니스 규칙을 코드로 직접 표현
    아이템 내용 기반 세밀한 판단 가능
    단위 테스트 용이 (shouldSkip() 독립 테스트)

  단점
    SkipPolicy 내에서 StepExecution 접근이 필요 (복잡)
    예외에서 아이템 내용 접근 제한
    → 예외 메시지에 아이템 ID를 포함해야 하거나
      ThreadLocal로 아이템을 전달하는 별도 구현 필요

ItemSkipListener 연동:

  장점  Skip 발생 시 아이템 + 예외 정보 모두 접근 가능
        dead_letter 저장, 알림, 통계 수집 모두 가능
  단점  SkipPolicy와 Listener 두 곳에서 Skip 관련 로직 분산
       → SkipPolicy: Skip 여부 결정
       → Listener: Skip 발생 후 처리 (명확히 역할 분리)
```

---

## 📌 핵심 정리

```
shouldSkip() 두 파라미터
  Throwable t     → 발생한 예외 (예외 클래스, 메시지로 판단)
  long skipCount  → 현재 Step에서 누적 Skip 건수 (한계 체크용)

Custom SkipPolicy 필수 구현 사항
  isSkippable(t)  → 이 예외가 Skip 대상인지 확인
  skipLimit 체크  → skipCount >= limit → SkipLimitExceededException throw
  false 반환      → 예외 전파 (Step FAILED 가능)

비율 기반 Skip 구현 패턴
  dynamicLimit = max(minLimit, totalRead * ratio)
  StepSynchronizationManager.getContext() → StepExecution → readCount 접근

ItemSkipListener 역할 분리
  SkipPolicy   → Skip 여부 결정 (shouldSkip 반환값)
  SkipListener → Skip 발생 후 처리 (dead_letter, 알림, 통계)
  → SkipPolicy에서 side effect 금지
    (SkipPolicy는 순수하게 예외/카운트만 보고 boolean 반환)
```

---

## 🤔 생각해볼 문제

**Q1.** `BusinessRuleSkipPolicy`에서 VIP 고객 오류를 `false`로 반환해 Skip을 거부합니다. 그런데 `retry(OrderProcessingException.class).retryLimit(3)`도 설정돼 있다면, VIP 오류는 Retry 3회 후 어떻게 처리되는가?

**Q2.** `RatioBasedSkipPolicy`에서 `StepSynchronizationManager.getContext()`로 `StepExecution`에 접근합니다. 이 방법이 Thread-safe한가? Multi-threaded Step에서 사용할 수 있는가?

**Q3.** `AuditSkipListener.onSkipInWrite()`에서 `deadLetterRepo.save()`를 호출합니다. 이 저장은 어떤 트랜잭션 컨텍스트에서 실행되는가? Write Skip이 발생하면 Chunk의 트랜잭션이 롤백되는데, dead_letter 저장도 롤백되는가?

> 💡 **해설**
>
> **Q1.** Retry 3회 후에도 VIP 오류가 지속되면, `FaultTolerantChunkProcessor`가 `shouldSkip()` 확인 → `BusinessRuleSkipPolicy.shouldSkip()`에서 VIP이면 `false` 반환 → 예외 전파 → Step FAILED. Retry 소진 후 자동 Skip 전환은 `shouldSkip()`이 `true`를 반환할 때만 이루어집니다. VIP 오류는 `false`이므로 Skip 없이 Step FAILED입니다. 이것이 의도된 동작 — VIP 고객 오류는 운영자가 직접 확인해야 한다는 비즈니스 규칙이 코드로 표현된 것입니다.
>
> **Q2.** `StepSynchronizationManager.getContext()`는 Thread-local 기반으로 각 쓰레드의 현재 `StepContext`를 반환합니다. Multi-threaded Step에서 각 쓰레드는 독립적인 `ChunkContext`를 가지지만, `StepExecution`은 공유됩니다. `getContext().getStepExecution().getReadCount()`는 공유된 `StepExecution`을 접근하므로, 동시에 여러 쓰레드가 읽으면 최신값이 아닐 수 있습니다. 비율 계산은 대략적인 기준이므로 큰 문제는 없지만, 정확한 동시성이 필요하다면 `AtomicLong`을 Policy 인스턴스 변수로 관리하는 것이 더 안전합니다.
>
> **Q3.** `onSkipInWrite()`는 스캐터-개더 중 Write가 실패한 직후, 롤백된 이후에 호출됩니다. 스캐터-개더에서 1건씩 별도 트랜잭션으로 처리하므로, 이미 해당 Write 트랜잭션은 롤백됐습니다. `onSkipInWrite()` 시점에는 활성 트랜잭션이 없습니다. 따라서 `deadLetterRepo.save()`는 **새로운 트랜잭션**에서 실행됩니다(`@Transactional(propagation=REQUIRES_NEW)` 또는 기본 `@Transactional`). dead_letter 저장은 Skip된 아이템의 롤백과 무관하게 영구 저장됩니다. 단, `deadLetterRepo.save()` 자체가 실패하면 예외가 전파되어 배치가 중단될 수 있으므로 예외 처리를 추가하는 것이 좋습니다.

---

<div align="center">

**[⬅️ 이전: Fault Tolerance 설정 완전 가이드](./06-fault-tolerance-config.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Partitioning 개념 ➡️](../partitioning-parallel/01-partitioning-concept.md)**

</div>
