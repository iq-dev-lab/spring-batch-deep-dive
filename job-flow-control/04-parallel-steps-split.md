# Parallel Steps — Split으로 여러 Flow를 병렬 실행하기

---

## 🎯 핵심 질문

- `FlowBuilder.split(TaskExecutor)`는 내부에서 어떻게 병렬 실행을 구현하는가?
- 병렬 Branch 중 하나가 실패했을 때 나머지 Branch와 전체 Job 상태는?
- `FlowExecutionAggregator`가 여러 Branch의 `ExitStatus`를 어떻게 집계하는가?
- Split 내 각 Flow는 독립적인 트랜잭션을 갖는가?
- Split과 Partitioning의 근본적인 차이는 무엇이며 언제 각각을 사용하는가?

---

## 🔍 왜 Split이 필요한가

### 문제: 독립적인 처리를 순차로 실행하면 시간 낭비다

```
시나리오: 일일 정산 배치

  순차 실행 (현재):
    FTP 다운로드    — 30초
    이메일 집계     — 45초
    SMS 집계        — 40초
    쿠폰 처리       — 55초
    결과 집계 Step  — 10초
    총 시간: 180초 + 10초 = 190초

  세 집계 작업은 서로 독립적 (의존 관계 없음)
  → 병렬 실행하면:
    max(30, 45, 40, 55) = 55초 + 10초 = 65초
    → 3배 빠름

Split 사용 조건:
  ① 각 Branch가 독립적 (서로 다른 DB 테이블, 다른 파일)
  ② 공유 자원 충돌 없음 (같은 테이블에 동시 쓰기 주의)
  ③ 최종 집계 Step은 모든 Branch 완료 후 실행
```

---

## 😱 흔한 실수

### Before: Split에서 같은 자원을 동시에 쓰려 한다

```java
// ❌ 두 Flow가 같은 테이블에 동시 INSERT
Flow emailFlow = new FlowBuilder<Flow>("emailFlow")
    .start(emailStep())  // daily_stats 테이블에 INSERT
    .build();

Flow smsFlow = new FlowBuilder<Flow>("smsFlow")
    .start(smsStep())    // daily_stats 테이블에 같은 날짜로 INSERT
    .build();

// 동시 실행 → PK 충돌 or 데드락 위험!

// ✅ 각 Flow가 독립 테이블에 쓰거나 다른 파티션 사용
// emailStep → email_stats 테이블
// smsStep   → sms_stats 테이블
// 집계 Step에서 두 테이블을 읽어 최종 집계
```

### Before: Split의 TaskExecutor 쓰레드 수가 Branch 수보다 적다

```java
// ❌ 쓰레드 1개 → 실제로는 순차 실행
@Bean
public Job parallelJob() {
    TaskExecutor executor = new SyncTaskExecutor();  // 동기 = 1 쓰레드
    return jobBuilderFactory.get("parallelJob")
        .start(step1())
        .split(executor)           // ← 동기 Executor → 병렬 아님!
            .add(emailFlow(), smsFlow())
        .next(aggregateStep())
        .end()
        .build();
}

// ✅ 병렬 실행을 위한 적절한 Executor
TaskExecutor executor = new SimpleAsyncTaskExecutor("batch-split-");
// 또는 쓰레드 수 제한:
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
executor.setCorePoolSize(4);
executor.setMaxPoolSize(4);
executor.initialize();
```

---

## ✨ Split 구성 문법

```java
@Bean
public Job parallelFlowJob() {
    // 병렬 실행할 각 Flow 정의
    Flow emailAggregationFlow = new FlowBuilder<Flow>("emailAggregationFlow")
        .start(downloadEmailStep())
        .next(processEmailStep())
        .build();

    Flow smsAggregationFlow = new FlowBuilder<Flow>("smsAggregationFlow")
        .start(downloadSmsStep())
        .next(processSmsStep())
        .build();

    Flow couponFlow = new FlowBuilder<Flow>("couponFlow")
        .start(processCouponStep())
        .build();

    // TaskExecutor: 최소 Branch 수 이상의 쓰레드
    TaskExecutor splitExecutor = new SimpleAsyncTaskExecutor("split-");

    return jobBuilderFactory.get("parallelFlowJob")
        .start(initStep())                     // ① 공통 초기화 (순차)
        .split(splitExecutor)
            .add(emailAggregationFlow,         // ② 세 Flow 병렬 실행
                 smsAggregationFlow,
                 couponFlow)
        .next(aggregateStep())                 // ③ 집계 (모든 Branch 완료 후)
        .end()
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. SplitState — 병렬 실행 내부 구조

```java
// SplitState.java
public class SplitState extends AbstractState {

    private Collection<Flow> flows;
    private FlowExecutionAggregator aggregator;
    private TaskExecutor taskExecutor;

    @Override
    public FlowExecutionStatus handle(final FlowExecutor executor)
            throws FlowExecutionException {

        // ① 각 Flow를 Callable로 래핑
        List<Callable<FlowExecution>> tasks = new ArrayList<>();
        for (final Flow flow : flows) {
            tasks.add(() -> {
                // 각 Branch가 독립 JobFlowExecutor에서 실행
                // → 독립 트랜잭션 (각 Step이 자체 트랜잭션 관리)
                return flow.start(createFlowExecutor(executor));
            });
        }

        // ② TaskExecutor로 모든 Callable 제출
        Collection<FlowExecution> results;
        try {
            results = executor.execute(tasks, taskExecutor);
            // → submit() 후 Future.get() 으로 모든 완료 대기
        } catch (TaskRejectedException e) {
            throw new FlowExecutionException("Split 실행 실패", e);
        }

        // ③ FlowExecutionAggregator로 결과 집계
        return aggregator.aggregate(results);
    }
}
```

### 2. FlowExecutionAggregator — Branch 결과 집계

```java
// MaxValueFlowExecutionAggregator (기본값)
// 가장 "나쁜" 상태가 최종 상태로 결정됨
public class MaxValueFlowExecutionAggregator implements FlowExecutionAggregator {

    @Override
    public FlowExecutionStatus aggregate(Collection<FlowExecution> executions) {
        FlowExecutionStatus result = FlowExecutionStatus.COMPLETED;

        for (FlowExecution execution : executions) {
            FlowExecutionStatus status = execution.getStatus();

            // 우선순위 (높을수록 나쁨):
            // FAILED > STOPPED > COMPLETED
            if (status.isGreaterThan(result)) {
                result = status;
            }
        }

        // 하나라도 FAILED이면 최종 → FAILED
        // 모두 COMPLETED면 최종 → COMPLETED
        return result;
    }
}

// 집계 예시:
// Branch 1: COMPLETED
// Branch 2: FAILED
// Branch 3: COMPLETED
// 집계 결과: FAILED → Job FAILED
```

### 3. Split 실행 타임라인

```
Job 시작
    │
    ▼ initStep (순차)
    [▓▓▓▓▓▓▓▓▓▓] COMPLETED
    │
    ▼ SplitState 진입 — 3개 쓰레드 동시 시작
    │
    ├─ Thread 1 ──────── emailAggregationFlow ──────────────────────────────────┐
    │   downloadEmailStep [▓▓▓▓] procesEmailStep [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] COMPLETED│
    │                                                                           │
    ├─ Thread 2 ── smsAggregationFlow ─────────────────────────────────────┐    │
    │   downloadSmsStep [▓▓▓▓▓▓] processSmsStep [▓▓▓▓▓▓▓▓▓▓▓▓▓] COMPLETED  │    │
    │                                                                      │    │
    ├─ Thread 3 ── couponFlow ────────────────────────────────────┐        │    │
    │   processCouponStep [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] COMPLETED       │        │    │
    │                                                             │        │    │
    ◀──────────────────────────── 모든 완료 대기 ────────────────────┘────────┘────┘
    │
    ▼ aggregateStep (순차, 모든 Branch 완료 후)
    [▓▓▓▓▓▓▓▓▓▓] COMPLETED
    │
    ▼ Job COMPLETED
```

### 4. Branch 실패 시 나머지 처리

```java
// Branch 1: COMPLETED
// Branch 2: FAILED (예외 발생)
// Branch 3: 아직 실행 중

// 질문: Branch 3은 계속 실행되는가?
// 답: 예. 이미 시작된 Branch는 중단되지 않고 완료까지 실행됨
// → Future.get()으로 모든 완료를 기다림
// → 모두 완료 후 Aggregator가 FAILED 집계
// → aggregateStep은 실행 안 됨 (Job FAILED이므로)

// 실무 주의: Branch 2 실패 후 Branch 3이 완료까지 실행됨
// → 불필요한 리소스 낭비 / 잘못된 데이터 처리 위험
// → Branch 간 독립성이 중요한 이유
```

---

## 💻 실전 구현

### 쓰레드 풀 기반 Split + 타임아웃 처리

```java
@Configuration
public class ParallelFlowConfig {

    @Bean
    public TaskExecutor splitTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);      // Branch 수와 동일하게
        executor.setMaxPoolSize(3);
        executor.setQueueCapacity(0);     // 큐 없음 (즉시 실행)
        executor.setThreadNamePrefix("batch-split-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(300);  // 최대 5분 대기
        executor.initialize();
        return executor;
    }

    @Bean
    public Job dailyAggregationJob(TaskExecutor splitTaskExecutor) {

        Flow emailFlow = new FlowBuilder<Flow>("emailFlow")
            .start(emailAggregationStep())
            .build();

        Flow smsFlow = new FlowBuilder<Flow>("smsFlow")
            .start(smsAggregationStep())
            .build();

        Flow pushFlow = new FlowBuilder<Flow>("pushFlow")
            .start(pushAggregationStep())
            .build();

        return jobBuilderFactory.get("dailyAggregationJob")
            .incrementer(new RunIdIncrementer())
            .start(prepareStep())                  // 공통 준비
            .split(splitTaskExecutor)
                .add(emailFlow, smsFlow, pushFlow) // 병렬 집계
            .next(finalAggregationStep())          // 최종 합산
            .next(reportStep())
            .end()
            .build();
    }
}
```

### Split vs Partitioning 선택 가이드

```java
// Split: 서로 다른 종류의 작업을 병렬로 (이종 작업)
split(executor).add(
    emailFlow,    // 이메일 집계
    smsFlow,      // SMS 집계
    pushFlow      // 푸시 집계
)
// → 각 Flow는 완전히 다른 Reader/Writer/Step 사용

// Partitioning: 같은 종류의 작업을 데이터 분할 병렬로 (동종 작업, Ch5)
// PartitionStep → Worker Step × N (같은 Step, 다른 데이터 범위)
// id 1~100만: Worker1(1~25만), Worker2(25~50만), Worker3(50~75만), Worker4(75~100만)
```

---

## 📊 Split 설계 시 고려사항

```
┌────────────────────────────┬──────────────────────────────────────────────┐
│ 항목                        │ 권장 사항                                      │
├────────────────────────────┼──────────────────────────────────────────────┤
│ TaskExecutor 쓰레드 수       │ Branch 수 이상 (부족 시 순차 실행)                 │
│ DB 커넥션 풀 크기             │ Branch 수 × Step당 커넥션 수 + 여유분             │
│ 공유 자원                    │ 각 Branch가 독립 자원 사용 (테이블, 파일)           │
│ 실패 시 롤백 범위              │ Branch 별 독립 트랜잭션 (전체 롤백 안 됨)           │
│ Branch 간 데이터 전달         │ Job EC 사용 (동시 쓰기 Race Condition 주의)       │
└────────────────────────────┴──────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Split 병렬 실행:

  장점
    독립적인 작업을 병렬화 → 전체 처리 시간 감소
    서로 다른 도메인 처리를 하나의 Job에서 관리
    각 Branch가 독립 실패 → 다른 Branch에 영향 없음 (실행 중에는)

  단점
    하나라도 FAILED이면 Job FAILED (aggregator 기본 동작)
    Branch 간 공유 자원 충돌 위험
    재시작 시 이미 COMPLETED된 Branch가 건너뛰어지지 않음 (Flow 단위 재시작 미지원)
    → Split 내 Step이 allowStartIfComplete=false이면 재시작 시 다시 실행

Split vs Partitioning:

  Split     → 다른 종류의 작업 병렬 (이종 Flow)
  Partition → 같은 작업에 데이터 분할 병렬 (동종 Step, 더 체계적 재시작)
```

---

## 📌 핵심 정리

```
Split 실행 흐름

  SplitState → 각 Flow를 Callable로 제출 → TaskExecutor로 병렬 실행
  → 모든 Future.get() 완료 대기
  → FlowExecutionAggregator로 결과 집계
  → FAILED 있으면 FAILED, 모두 COMPLETED면 COMPLETED

Branch 실패 처리
  실패 Branch 발견 시 즉시 다른 Branch를 중단하지 않음
  → 모든 Branch 완료 후 FAILED 집계
  → 이후 Step (aggregateStep 등) 실행 안 됨

FlowExecutionAggregator 기본 동작
  FAILED > STOPPED > COMPLETED 우선순위
  하나라도 FAILED → 전체 FAILED

TaskExecutor 설정
  SyncTaskExecutor → 순차 실행 (병렬 아님)
  SimpleAsyncTaskExecutor → 쓰레드 제한 없음
  ThreadPoolTaskExecutor → 쓰레드 수 제어 권장
```

---

## 🤔 생각해볼 문제

**Q1.** Split 내 Branch 2가 `FAILED`로 끝났지만 Branch 1, 3은 `COMPLETED`로 끝났습니다. 이 Job을 재시작하면 Branch 1, 3의 Step들은 다시 실행되는가?

**Q2.** Split의 각 Branch는 독립 쓰레드에서 실행됩니다. 두 Branch가 동시에 `Job-scope ExecutionContext`에 서로 다른 키로 값을 저장하면 안전한가?

**Q3.** `FlowExecutionAggregator`를 커스텀하여 "Branch 중 하나가 FAILED여도 나머지가 COMPLETED면 Job 전체를 COMPLETED로 처리"하도록 만들 수 있는가? 어떤 위험이 있는가?

> 💡 **해설**
>
> **Q1.** Branch 1, 3의 Step들이 이미 `COMPLETED` 상태이고 `allowStartIfComplete=false`(기본값)이라면, 재시작 시 `SimpleStepHandler`가 이 Step들을 건너뜁니다. Branch 2의 `FAILED` Step만 재실행됩니다. 단, `SplitState`는 항상 모든 Branch를 다시 시작하므로, Branch 1과 3의 Step들은 Flow 내에서 시작 시도 후 건너뛰어집니다. `BATCH_STEP_EXECUTION`에서 각 Step의 상태를 확인해 재시작 여부를 결정합니다.
>
> **Q2.** `JobExecution.getExecutionContext()`는 `ExecutionContext` 객체를 반환하고, 이는 내부적으로 `LinkedHashMap`을 사용합니다. 개별 `put()` 연산은 Thread-safe하지 않습니다. 두 Branch가 동시에 다른 키로 저장하면 Race Condition이 발생할 수 있습니다. 더 큰 문제는 `TaskletStep`이 각 Chunk 커밋 시 `jobRepository.updateExecutionContext()`를 호출해 DB에 전체 EC를 저장하는데, 두 Branch가 동시에 저장하면 서로의 값을 덮어씁니다. 해결책: Branch 별 Step EC를 사용하거나, Split 완료 후 별도 집계 Step에서 각 Branch의 Step EC를 읽어 합산합니다.
>
> **Q3.** `FlowExecutionAggregator`를 구현해 "일부 FAILED 허용" 로직을 만들 수 있습니다. `JobBuilderHelper.aggregate(customAggregator)`로 등록합니다. 위험: (1) Job이 `COMPLETED`로 끝나면 같은 `JobInstance`를 다시 실행할 수 없습니다 — FAILED Branch를 수동으로 재처리해야 합니다. (2) FAILED Branch의 처리가 불완전한 상태에서 후속 Step이 실행되면 잘못된 데이터가 생성될 수 있습니다. (3) 운영자가 Job이 `COMPLETED`라고 판단해 실패를 인지하지 못할 수 있습니다. 대안: FAILED Branch를 `end("PARTIAL_FAILURE")`로 처리하고 커스텀 ExitStatus로 모니터링합니다.

---

<div align="center">

**[⬅️ 이전: JobExecutionDecider](./03-job-execution-decider.md)** | **[홈으로 🏠](../README.md)** | **[다음: Flow 외부화와 재사용 ➡️](./05-flow-externalization.md)**

</div>
