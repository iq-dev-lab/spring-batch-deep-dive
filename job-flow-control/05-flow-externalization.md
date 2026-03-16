# Flow 외부화와 재사용 — 공통 에러 처리와 Sub-flow 분리

---

## 🎯 핵심 질문

- `Flow` Bean을 독립적으로 정의하면 어떤 구조적 이점이 있는가?
- `FlowStep`이 Sub-flow를 하나의 Step처럼 취급하는 것이 무엇을 의미하는가?
- 여러 Job이 같은 Flow를 공유할 때 `StepExecution` 이름 충돌은 어떻게 방지하는가?
- 공통 에러 처리 Flow를 분리하면 재시작 시 어떤 복잡성이 생기는가?
- `JobFlowExecutor`와 일반 `FlowExecutor`의 차이는 무엇인가?

---

## 🔍 왜 Flow를 외부화하는가

### 문제: 여러 Job에 동일한 처리 패턴이 반복된다

```
공통 패턴 예시:

  Job A (주문 정산):
    processStep → errorNotifyStep → cleanupStep (실패 시)
    processStep → reportStep → cleanupStep (성공 시)

  Job B (정산 집계):
    aggregateStep → errorNotifyStep → cleanupStep (실패 시)
    aggregateStep → reportStep → cleanupStep (성공 시)

  공통 부분: errorNotifyStep + cleanupStep
  → 두 Job에 동일 코드 중복
  → errorNotifyStep 로직 변경 시 두 곳 수정 필요

  Flow 외부화 해결:
    @Bean commonErrorFlow: errorNotifyStep → cleanupStep
    Job A: processStep → commonErrorFlow (실패 시)
    Job B: aggregateStep → commonErrorFlow (실패 시)
    → 공통 로직 한 곳에서 관리
```

---

## 😱 흔한 실수

### Before: FlowStep의 StepExecution 명명 충돌을 고려하지 않는다

```java
// ❌ 두 Job이 같은 Flow를 공유할 때 Step 이름 충돌
@Bean
public Flow commonErrorFlow() {
    return new FlowBuilder<Flow>("commonErrorFlow")
        .start(notifyStep())    // Step 이름: "notifyStep"
        .next(cleanupStep())    // Step 이름: "cleanupStep"
        .build();
}

// Job A와 Job B 모두 commonErrorFlow를 사용
// BATCH_STEP_EXECUTION에 "notifyStep" 이름으로 행 생성
// → 두 Job의 StepExecution이 같은 Step 이름 공유
// → 재시작 시 다른 Job의 StepExecution을 조회할 수 있음

// ✅ Job별 고유 이름을 사용하거나 Step 이름에 Job 접두사 추가
@Bean
public Step jobANotifyStep() {
    return stepBuilderFactory.get("jobA.notifyStep")  // 접두사로 구분
        .tasklet(notifyTasklet())
        .build();
}
```

---

## ✨ Flow 외부화 방법

### 방법 1: Flow Bean을 직접 Job에 포함

```java
// 공통 에러 처리 Flow 정의
@Bean
public Flow errorHandlingFlow() {
    return new FlowBuilder<Flow>("errorHandlingFlow")
        .start(sendErrorNotificationStep())
        .next(cleanupTempFilesStep())
        .next(markJobFailedStep())
        .build();
}

// 여러 Job에서 재사용
@Bean
public Job orderJob() {
    return jobBuilderFactory.get("orderJob")
        .start(processOrderStep())
            .on("COMPLETED").to(reportStep())
            .on("*").to(errorHandlingFlow())  // ← Flow를 다음 노드로 지정
        .from(reportStep()).on("*").end()
        .from(errorHandlingFlow()).end()
        .end()
        .build();
}
```

### 방법 2: FlowStep — Sub-flow를 하나의 Step처럼

```java
// FlowStep: Flow를 Step처럼 다룸
// → BATCH_STEP_EXECUTION에 FlowStep 이름으로 행 생성
// → Flow 내 Step들은 FlowStep의 하위로 실행됨

@Bean
public Step errorHandlingAsStep() {
    // FlowStepBuilder 사용
    return stepBuilderFactory.get("errorHandlingStep")
        .flow(errorHandlingFlow())   // Flow를 Step으로 감쌈
        .build();
}

// Job에서 일반 Step처럼 사용
@Bean
public Job orderJob() {
    return jobBuilderFactory.get("orderJob")
        .start(processOrderStep())
            .on("COMPLETED").to(reportStep())
            .on("*").to(errorHandlingAsStep())   // ← Step처럼 사용
        .from(reportStep()).on("*").end()
        .from(errorHandlingAsStep()).on("*").end()
        .end()
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. FlowStep 내부 — Flow를 Step으로 감싸는 방식

```java
// FlowStep.java
public class FlowStep extends AbstractStep {

    private Flow flow;  // 내부에서 실행할 Flow

    @Override
    protected void doExecute(StepExecution stepExecution) throws Exception {
        // ① StepExecution을 JobExecution에 등록
        StepContext stepContext = StepSynchronizationManager.register(stepExecution);

        try {
            // ② 내부 Flow 실행 (별도 FlowExecutor 사용)
            FlowExecutor executor = new JobFlowExecutor(getJobRepository(),
                new SimpleStepHandler(getJobRepository(), transactionManager),
                stepExecution.getJobExecution()) {
                    @Override
                    public String getJobName() {
                        // FlowStep 이름을 prefix로 사용해 내부 Step 이름 구분
                        return FlowStep.this.getName();
                    }
                };

            FlowExecution flowExecution = flow.start(executor);

            // ③ Flow 최종 상태를 FlowStep의 ExitStatus로 설정
            stepExecution.upgradeStatus(flowExecution.getStatus().getBatchStatus());
            stepExecution.setExitStatus(
                new ExitStatus(flowExecution.getStatus().getName()));

        } finally {
            StepSynchronizationManager.release();
        }
    }
}
```

### 2. 공통 Flow Bean이 여러 Job에서 공유될 때의 동작

```java
// 동일 Flow Bean을 Job A와 Job B가 공유
@Bean
public Flow commonCleanupFlow() {
    return new FlowBuilder<Flow>("commonCleanupFlow")
        .start(cleanupStep())  // "cleanupStep" 이름
        .build();
}

// Job A에서:
// BATCH_STEP_EXECUTION.STEP_NAME = "cleanupStep"
// BATCH_JOB_EXECUTION_ID = Job A의 실행 ID

// Job B에서:
// BATCH_STEP_EXECUTION.STEP_NAME = "cleanupStep"
// BATCH_JOB_EXECUTION_ID = Job B의 실행 ID

// → 서로 다른 JOB_EXECUTION_ID로 구분 → 충돌 없음
// 단, 같은 JobInstance 내에서는 Step 이름으로 마지막 실행 조회
// → 동일 Job에서 같은 Flow를 두 번 실행하면 문제 발생
```

### 3. 공통 에러 처리 Flow 재시작 복잡성

```
Job 첫 실행 (processStep FAILED):
  processStep     → FAILED
  errorHandlingFlow 실행됨:
    notifyStep    → COMPLETED
    cleanupStep   → COMPLETED

재시작:
  processStep     → FAILED (다시 실패)
  errorHandlingFlow 재실행:
    notifyStep    → allowStartIfComplete=false → 건너뜀! (이미 COMPLETED)
    cleanupStep   → 건너뜀! (이미 COMPLETED)
  → errorNotification이 두 번째 실패에서는 발송 안 됨!

해결책: 공통 Flow의 Step에 allowStartIfComplete(true) 설정
    또는 매 실행마다 새 JobInstance 사용 (RunIdIncrementer)
```

---

## 💻 실전 구현

### 공통 에러/성공 처리 Flow 패턴

```java
@Configuration
public class CommonFlowConfig {

    // ============ 공통 성공 후처리 Flow ============
    @Bean
    public Flow successPostProcessingFlow() {
        return new FlowBuilder<Flow>("successPostProcessingFlow")
            .start(sendSuccessNotificationStep())
            .next(archiveInputFilesStep())
            .next(updateJobStatusStep())
            .build();
    }

    // ============ 공통 실패 후처리 Flow ============
    @Bean
    public Flow failurePostProcessingFlow() {
        return new FlowBuilder<Flow>("failurePostProcessingFlow")
            .start(sendAlertNotificationStep())
            .next(cleanupTempFilesStep())
            .next(rollbackPartialResultsStep())
            .build();
    }

    // allowStartIfComplete(true) 설정으로 재실행 보장
    @Bean
    public Step sendAlertNotificationStep() {
        return stepBuilderFactory.get("sendAlertNotificationStep")
            .tasklet((contribution, chunkContext) -> {
                String jobName = chunkContext.getStepContext()
                    .getJobName();
                Long executionId = chunkContext.getStepContext()
                    .getStepExecution().getJobExecutionId();
                alertService.sendAlert(jobName, executionId);
                return RepeatStatus.FINISHED;
            })
            .allowStartIfComplete(true)  // 재실행마다 알림 발송
            .build();
    }
}

// 공통 Flow를 사용하는 Job
@Bean
public Job orderJob(Flow successPostProcessingFlow,
                     Flow failurePostProcessingFlow) {
    return jobBuilderFactory.get("orderJob")
        .start(validateStep())
            .on("COMPLETED").to(processStep())
            .on("*").to(failurePostProcessingFlow)
        .from(processStep())
            .on("COMPLETED").to(successPostProcessingFlow)
            .on("*").to(failurePostProcessingFlow)
        .from(successPostProcessingFlow)
            .on("*").end()
        .from(failurePostProcessingFlow)
            .on("*").end("FAILED_WITH_NOTIFICATION")
        .end()
        .build();
}
```

### 모듈 방식 Flow 조합 — FlowStep 활용

```java
// 각 도메인별 Flow를 독립 @Configuration으로 관리
@Configuration
public class OrderFlowConfig {

    @Bean
    public Flow orderProcessingFlow() {
        return new FlowBuilder<Flow>("orderProcessingFlow")
            .start(validateOrderStep())
            .next(calculateFeeStep())
            .next(persistSettlementStep())
            .build();
    }

    // FlowStep으로 래핑 — Job에서 일반 Step처럼 사용
    @Bean
    public Step orderProcessingFlowStep() {
        return stepBuilderFactory.get("orderProcessingFlowStep")
            .flow(orderProcessingFlow())
            .build();
    }
}

@Configuration
public class ReportFlowConfig {

    @Bean
    public Step reportFlowStep() {
        Flow reportFlow = new FlowBuilder<Flow>("reportFlow")
            .start(generateCsvStep())
            .next(uploadToS3Step())
            .next(sendEmailStep())
            .build();
        return stepBuilderFactory.get("reportFlowStep")
            .flow(reportFlow)
            .build();
    }
}

// 최종 Job은 FlowStep만 나열 — 가독성 향상
@Bean
public Job dailyJob(Step orderProcessingFlowStep, Step reportFlowStep) {
    return jobBuilderFactory.get("dailyJob")
        .start(orderProcessingFlowStep)   // 내부: 3개 Step
        .next(reportFlowStep)             // 내부: 3개 Step
        .build();
}
```

---

## 📊 Flow 재사용 방식 비교

```
┌─────────────────────────┬──────────────────────────┬──────────────────────────────┐
│ 방식                     │ StepExecution 생성        │ 재시작 동작                     │
├─────────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Flow Bean 직접 사용       │ Flow 내 Step들이 개별 생성   │ 각 Step 개별 재시작             │
│ FlowStep으로 래핑         │ FlowStep 1개 + 내부 Step들 │ FlowStep 단위 재시작            │
│ Split에서 Flow 사용       │ 각 Branch의 Step들 개별     │ Branch 내 각 Step 개별 재시작    │
└─────────────────────────┴──────────────────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Flow 외부화:

  장점
    공통 로직 중앙화 → 한 곳에서 유지보수
    여러 Job에서 재사용 → 코드 중복 제거
    도메인별 Flow 분리 → 가독성/모듈성 향상

  단점
    공통 Step의 재시작 동작 (allowStartIfComplete 설정 주의)
    Flow 내 Step 이름 충돌 가능성 (같은 Job 내 중복 Flow 사용 시)
    복잡한 Job일수록 Flow 그래프 파악이 어려워짐

FlowStep 사용:

  장점  Job 설정이 간결해짐 (FlowStep만 나열)
        FlowStep 단위로 재시작 제어 가능
  단점  FlowStep 내부 Step의 개별 재시작 제어 복잡
        FlowStep FAILED → 내부 어느 Step에서 실패했는지 파악에 추가 쿼리 필요
```

---

## 📌 핵심 정리

```
Flow 외부화 두 가지 방법

  Flow Bean 직접 참조
    → Flow 내 Step들이 Job의 StepExecution으로 직접 생성
    → 세밀한 재시작 제어 가능

  FlowStep (Flow를 Step으로 래핑)
    → FlowStep 단위로 StepExecution 1개 생성
    → 내부 Step들도 별도 StepExecution 생성
    → Job 설정이 간결해짐

공통 Flow 재시작 주의사항
  공통 에러 Flow 내 Step에 allowStartIfComplete(true) 설정 권장
  → 재실행마다 에러 알림, 정리 작업이 실행되도록 보장

Flow 이름 네이밍 전략
  FlowBuilder 이름 ≠ 반드시 고유해야 함
  Step 이름만 고유하면 StepExecution 충돌 없음
  단, 같은 Job 내에서 동일 Step 이름 중복 시 재시작 오동작
```

---

## 🤔 생각해볼 문제

**Q1.** `FlowStep`으로 래핑된 `errorHandlingFlowStep`이 `FAILED`로 끝났을 때(내부 Step 중 하나 실패), 재시작 시 `FlowStep` 전체가 재실행되는가 아니면 내부 실패 Step만 재실행되는가?

**Q2.** 동일 Job에서 같은 `commonCleanupFlow`를 두 번 참조한다고 가정합니다(예: 성공 분기와 실패 분기 모두에서). 두 번 중 첫 번째만 실행됐을 때, 두 번째 분기의 Flow에 같은 이름의 Step이 있으면 어떻게 되는가?

**Q3.** `FlowBuilder.start(step).next(step).build()`로 만든 `Flow` Bean의 내부 상태(steps 목록 등)는 공유 가능한가? 여러 쓰레드(Split의 여러 Branch)가 같은 `Flow` Bean을 동시에 실행해도 안전한가?

> 💡 **해설**
>
> **Q1.** `FlowStep`의 `FAILED` 재시작 시 `FlowStep` 전체가 재실행됩니다. `FlowStep` 자체가 하나의 `StepExecution`을 가지며, 이 `StepExecution.status = FAILED`이면 재시작 시 `FlowStep`이 다시 실행됩니다. `FlowStep` 내부 Step들도 각자의 `StepExecution`을 가지므로, 이미 `COMPLETED`인 내부 Step들은 `allowStartIfComplete` 설정에 따라 건너뛰어집니다. 결과적으로 내부 실패 Step부터 재개되는 효과가 있지만, `FlowStep` 단위로 재시작이 트리거됩니다.
>
> **Q2.** 같은 Job 내에서 같은 Step 이름이 두 번 사용되면, `SimpleStepHandler`가 `jobRepository.getLastStepExecution(jobInstance, stepName)`을 조회할 때 첫 번째 실행 분기에서 `COMPLETED`된 `StepExecution`을 반환합니다. 두 번째 분기에서 이 Step을 실행하려 할 때, `COMPLETED`이고 `allowStartIfComplete=false`(기본값)이면 건너뜁니다. 이는 의도치 않은 동작입니다. 해결책: 각 사용 맥락마다 다른 Step 이름 사용 (예: `successCleanupStep`, `failureCleanupStep`).
>
> **Q3.** `Flow` Bean은 Step 목록과 State 그래프를 가지지만, 실행 상태(현재 어느 State인지)는 `FlowExecutor`가 관리합니다. `Flow.start(executor)`를 호출할 때 각 `executor`가 독립적인 실행 상태를 유지합니다. 따라서 여러 쓰레드가 같은 `Flow` Bean의 `start()`를 동시에 호출해도 각 호출이 별도의 `FlowExecutor`를 사용하므로 Thread-safe합니다. `Flow` Bean 자체는 불변(immutable)한 설정 정보만 담고 있기 때문입니다.

---

<div align="center">

**[⬅️ 이전: Parallel Steps](./04-parallel-steps-split.md)** | **[홈으로 🏠](../README.md)** | **[다음: Job·Step Listener 활용 ➡️](./06-job-step-listener.md)**

</div>
