# Sequential Flow — SimpleJob이 Step을 순서대로 실행하는 내부 흐름

---

## 🎯 핵심 질문

- `SimpleJob.doExecute()`는 Step 목록을 어떻게 순서대로 실행하는가?
- 앞 Step이 `FAILED`일 때 다음 Step으로 넘어가지 않는 이유는?
- `allowStartIfComplete(true)`는 어떤 상황에서 필요한가?
- Step의 `ExitStatus`와 `BatchStatus`는 어떻게 다르며, Flow 제어에서 어느 것이 기준인가?
- `STOPPED` 상태인 Job을 재시작하면 어느 Step부터 실행되는가?

---

## 🔍 왜 Sequential Flow가 기본 동작인가

### 문제: 배치 단계는 의존 관계가 있다

```
주문 정산 배치 의존 관계:

  1단계: FTP 다운로드   (실패 시 2단계 불가)
  2단계: CSV 파싱 + 저장 (실패 시 3단계 불가 — 데이터 없음)
  3단계: 정산 계산
  4단계: 리포트 이메일 발송

앞 단계 실패 시 뒤 단계를 실행하면:
  2단계 실패 → 3단계 실행 → 데이터 없어 잘못된 정산
  → 잘못된 금액으로 이메일 발송

기본 동작: 앞 Step FAILED → 즉시 Job FAILED, 나머지 Step 실행 안 함
→ 데이터 의존성 보호
→ 부분 성공 상태에서 잘못된 처리 방지
```

---

## 😱 흔한 실수

### Before: COMPLETED Step이 재시작 시 다시 실행되길 원하는데 안 된다

```java
// ❌ allowStartIfComplete 설정 없이 멱등 Step 재실행 시도
@Bean
public Step cleanupStep() {
    return stepBuilderFactory.get("cleanupStep")
        .tasklet(cleanupTasklet())
        // allowStartIfComplete 미설정 (기본값 false)
        .build();
}
// Job 재시작 시 cleanupStep이 이미 COMPLETED → 건너뜀
// 그러나 cleanupStep은 매 실행마다 임시 파일을 지워야 함

// ✅ 멱등 Step은 allowStartIfComplete(true) 설정
@Bean
public Step cleanupStep() {
    return stepBuilderFactory.get("cleanupStep")
        .tasklet(cleanupTasklet())
        .allowStartIfComplete(true)   // COMPLETED여도 재실행
        .build();
}
```

### Before: ExitStatus와 BatchStatus를 혼동해 Flow를 잘못 구성한다

```java
// ❌ BatchStatus로 분기하려는 시도 — 작동 안 함
jobBuilderFactory.get("job")
    .start(step1())
    .on("FAILED")      // ← 이것은 ExitStatus 패턴
    .to(step2())
    .end()
    .build();

// BatchStatus vs ExitStatus:
// BatchStatus: 프레임워크가 관리하는 실행 상태 (STARTING, STARTED, COMPLETED, FAILED...)
//              JobExecution/StepExecution의 내부 상태 머신
// ExitStatus:  Step의 비즈니스 결과 코드 (COMPLETED, FAILED, STOPPED, 커스텀 코드)
//              Flow 전이의 매칭 대상 ← on()에서 비교하는 것
//
// on("FAILED") → step.getExitStatus().getExitCode().equals("FAILED") 매칭
```

---

## ✨ 올바른 이해와 사용

### SimpleJob Step 실행 흐름

```java
// 기본 Sequential Job 구성
@Bean
public Job orderJob() {
    return jobBuilderFactory.get("orderJob")
        .start(downloadStep())    // Step 1
        .next(processStep())      // Step 2
        .next(reportStep())       // Step 3
        .build();
    // 내부적으로 SimpleJob 생성
    // steps = [downloadStep, processStep, reportStep]
}
```

---

## 🔬 내부 동작 원리

### 1. SimpleJob.doExecute() — Step 순차 실행 소스

```java
// SimpleJob.java
public class SimpleJob extends AbstractJob {

    private List<Step> steps = new ArrayList<>();

    @Override
    protected void doExecute(JobExecution execution)
            throws JobInterruptedException, JobRestartException, StartLimitExceededException {

        StepExecution stepExecution = null;

        for (Step step : steps) {

            // ① 이전 Step 상태 확인
            if (stepExecution != null) {
                if (stepExecution.getStatus() == BatchStatus.STOPPED) {
                    // STOPPED → Job도 STOPPED, 남은 Step 실행 안 함
                    execution.setStatus(BatchStatus.STOPPED);
                    execution.setExitStatus(ExitStatus.STOPPED);
                    return;
                }
                if (stepExecution.getStatus() != BatchStatus.COMPLETED) {
                    // FAILED 등 비정상 → Job FAILED, 즉시 종료
                    execution.upgradeStatus(stepExecution.getStatus());
                    execution.setExitStatus(stepExecution.getExitStatus());
                    return;
                }
            }

            // ② Step 실행 (StepHandler에 위임)
            stepExecution = handleStep(step, execution);
        }

        // ③ 모든 Step COMPLETED → Job COMPLETED
        execution.upgradeStatus(BatchStatus.COMPLETED);
        execution.setExitStatus(ExitStatus.COMPLETED.and(
            stepExecution != null ? stepExecution.getExitStatus() : ExitStatus.COMPLETED));
    }
}
```

### 2. SimpleStepHandler.handleStep() — 재시작 판단

```java
// SimpleStepHandler.java
public StepExecution handleStep(Step step, JobExecution execution)
        throws JobInterruptedException, JobRestartException, StartLimitExceededException {

    // ① 이 Job 실행에서 이미 실행된 StepExecution 확인
    StepExecution currentStepExecution =
        getStepExecution(execution, step.getName());

    if (currentStepExecution != null) {
        // 현재 JobExecution에 이미 이 Step이 있음 → 건너뜀
        return currentStepExecution;
    }

    // ② 이전 JobExecution에서 이 Step의 마지막 실행 확인 (재시작용)
    StepExecution lastStepExecution = jobRepository.getLastStepExecution(
        execution.getJobInstance(), step.getName());

    // ③ allowStartIfComplete 확인
    if (stepExecutionAlreadyExists(execution) || !shouldStart(lastStepExecution, step)) {
        log.info("Step [{}]: 건너뜀 (COMPLETED 또는 startLimit 초과)",
            step.getName());
        return lastStepExecution;
    }

    // ④ 새 StepExecution 생성 + Step 실행
    currentStepExecution = execution.createStepExecution(step.getName());
    jobRepository.add(currentStepExecution);
    step.execute(currentStepExecution);
    return currentStepExecution;
}

// shouldStart() 판단 로직
private boolean shouldStart(StepExecution lastStepExecution, Step step) {
    if (lastStepExecution == null) return true;  // 첫 실행

    BatchStatus lastStatus = lastStepExecution.getStatus();

    if (lastStatus == BatchStatus.UNKNOWN) {
        throw new JobRestartException("Step [" + step.getName() +
            "] is of status UNKNOWN");
    }

    if (lastStatus == BatchStatus.COMPLETED) {
        if (!step.isAllowStartIfComplete()) {
            return false;  // COMPLETED + allowStartIfComplete=false → 건너뜀
        }
    }

    // startLimit 초과 확인 (기본값 Integer.MAX_VALUE)
    int startCount = jobRepository.getStepExecutionCount(
        lastStepExecution.getJobExecution().getJobInstance(), step.getName());
    if (startCount >= step.getStartLimit()) {
        throw new StartLimitExceededException(...);
    }

    return true;
}
```

### 3. BatchStatus vs ExitStatus — 역할 분리

```
BatchStatus (프레임워크 상태 머신):
  STARTING → STARTED → COMPLETED
                     → FAILED
                     → STOPPED
                     → STOPPING
                     → ABANDONED
                     → UNKNOWN

  JobExecution/StepExecution이 현재 어떤 단계에 있는지를 나타냄
  프레임워크가 내부적으로 관리

ExitStatus (비즈니스 결과 코드):
  기본값: UNKNOWN, EXECUTING, COMPLETED, NOOP, FAILED, STOPPED
  커스텀: "VALIDATION_FAILED", "NO_DATA", "PARTIAL_SUCCESS" 등

  Step이 끝났을 때 "어떤 결과였는가"를 나타냄
  Flow 전이(.on())에서 매칭 대상

관계:
  BatchStatus.COMPLETED → ExitStatus.COMPLETED (기본 매핑)
  BatchStatus.FAILED    → ExitStatus.FAILED    (기본 매핑)
  커스텀 ExitStatus는 BatchStatus와 무관하게 설정 가능
```

### 4. STOPPED Job 재시작 흐름

```
Job 실행 1 (STOPPED):
  Step 1 downloadStep → COMPLETED
  Step 2 processStep  → JobOperator.stop() 호출 → STOPPED
  Step 3 reportStep   → 실행 안 됨

Job 실행 2 (재시작):
  SimpleStepHandler.shouldStart() 확인:
    downloadStep: lastStatus=COMPLETED, allowStartIfComplete=false → 건너뜀 ✓
    processStep:  lastStatus=STOPPED → 재실행 가능 → 재시작 ✓
    reportStep:   lastStepExecution=null (실행된 적 없음) → 실행 ✓

  결과: processStep부터 재개
```

### 5. Step 실행 상태 타임라인 ASCII 다이어그램

```
Job 최초 실행 (processStep에서 FAILED):

downloadStep  [▓▓▓▓▓▓▓▓▓▓] COMPLETED
processStep   [▓▓▓▓▓▓░░░░] FAILED ← 여기서 멈춤
reportStep    [          ] 실행 안 됨

Job FAILED → BATCH_JOB_EXECUTION.STATUS = 'FAILED'
BATCH_STEP_EXECUTION: downloadStep=COMPLETED, processStep=FAILED

재시작:

downloadStep  [──────────] COMPLETED (건너뜀 — 이미 완료)
processStep   [▓▓▓▓▓▓▓▓▓▓] COMPLETED ← 새 StepExecution으로 재실행
reportStep    [▓▓▓▓▓▓▓▓▓▓] COMPLETED

Job COMPLETED
```

---

## 💻 실전 구현

### startLimit으로 재시도 횟수 제한

```java
@Bean
public Step retryLimitedStep() {
    return stepBuilderFactory.get("retryLimitedStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .writer(writer())
        .startLimit(3)   // 이 Step은 최대 3번만 실행 가능
        // 4번째 시도 시 StartLimitExceededException 발생
        .build();
}
```

### 커스텀 ExitStatus로 세밀한 흐름 제어

```java
// Step 결과에 따라 커스텀 ExitStatus 설정
@Bean
public Step validationStep() {
    return stepBuilderFactory.get("validationStep")
        .tasklet((contribution, chunkContext) -> {
            int invalidCount = validateData();
            if (invalidCount == 0) {
                // 모든 데이터 정상 → 기본 COMPLETED
                return RepeatStatus.FINISHED;
            } else if (invalidCount < 100) {
                // 일부 오류 → 계속 진행하되 커스텀 ExitStatus
                contribution.setExitStatus(new ExitStatus("PARTIAL_VALID"));
                return RepeatStatus.FINISHED;
            } else {
                // 대규모 오류 → 처리 중단
                contribution.setExitStatus(new ExitStatus("INVALID"));
                return RepeatStatus.FINISHED;
            }
        })
        .build();
}

// Flow에서 커스텀 ExitStatus 활용 (Ch3-02에서 상세)
jobBuilderFactory.get("job")
    .start(validationStep())
        .on("COMPLETED").to(processStep())
        .on("PARTIAL_VALID").to(partialProcessStep())
        .on("INVALID").to(errorNotificationStep())
    .end()
    .build();
```

---

## 📊 Step 상태별 Job 동작 요약

```
Step 종료 상태별 SimpleJob 동작:

┌──────────────────┬──────────────────────────────────────────────────────┐
│ Step BatchStatus │ SimpleJob 동작                                        │
├──────────────────┼──────────────────────────────────────────────────────┤
│ COMPLETED        │ 다음 Step 실행                                         │
│ FAILED           │ 즉시 Job FAILED 반환, 나머지 Step 실행 안 함               │
│ STOPPED          │ 즉시 Job STOPPED 반환, 나머지 Step 실행 안 함              │
│ ABANDONED        │ Job FAILED로 처리                                      │
│ UNKNOWN          │ JobRestartException 발생 (재시작 불가)                   │
└──────────────────┴──────────────────────────────────────────────────────┘

재시작 시 Step 건너뜀 조건:
  - lastStatus == COMPLETED && allowStartIfComplete == false (기본값)
  - 현재 JobExecution에 이미 이 Step의 StepExecution 존재
```

---

## ⚖️ 트레이드오프

```
Sequential Flow (SimpleJob) 기본 설계:

  장점
    단순 — Job 설정이 직관적
    안전 — 앞 Step 실패 시 뒤 Step 자동 차단
    재시작 효율 — COMPLETED Step 건너뜀

  단점
    유연성 부족 — 성공/실패 분기 불가 (→ FlowJob 필요)
    병렬 처리 불가 (→ Split 필요)

allowStartIfComplete(true) 사용 주의:

  필요한 경우: 멱등 Step (매 실행마다 같은 작업, 예: 임시 파일 삭제)
  주의: 비멱등 Step에 설정하면 같은 데이터를 두 번 처리하는 문제 발생
```

---

## 📌 핵심 정리

```
SimpleJob Sequential 실행 규칙

  Step COMPLETED → 다음 Step 실행
  Step FAILED    → Job 즉시 FAILED, 이후 Step 건너뜀
  Step STOPPED   → Job 즉시 STOPPED, 이후 Step 건너뜀

재시작 시 Step 처리

  COMPLETED + allowStartIfComplete=false → 건너뜀 (기본값)
  COMPLETED + allowStartIfComplete=true  → 재실행
  FAILED / STOPPED                       → 새 StepExecution으로 재실행
  startLimit 초과                        → StartLimitExceededException

ExitStatus vs BatchStatus

  BatchStatus  → 프레임워크 상태 머신 (내부)
  ExitStatus   → 비즈니스 결과 코드 (Flow 전이 매칭 대상)
  .on("FAILED") → ExitStatus.exitCode == "FAILED" 매칭
```

---

## 🤔 생각해볼 문제

**Q1.** `SimpleJob`에서 3개 Step이 있고 2번째 Step이 `STOPPED`로 끝났을 때, 3번째 Step은 실행되는가? 재시작 시 2번째 Step은 새 `StepExecution`으로 처음부터 실행되는가?

**Q2.** `allowStartIfComplete(true)`로 설정된 Step이 재시작 시 실행됩니다. 이 Step의 `ExecutionContext`는 어떻게 되는가? 이전 실행의 `ExecutionContext`가 복원되는가 새 것으로 시작되는가?

**Q3.** Step에 `startLimit(2)`를 설정했습니다. 첫 번째 실행 성공(COMPLETED) 후, 두 번째 재시작 시 `allowStartIfComplete(true)` 설정으로 다시 실행됩니다. 세 번째 재시작 시 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 3번째 Step은 실행되지 않습니다. `SimpleJob.doExecute()`에서 이전 Step의 `BatchStatus`가 `STOPPED`이면 `return`으로 즉시 종료합니다. 재시작 시 2번째 Step은 `lastStatus == STOPPED`이므로 `shouldStart()`가 `true`를 반환해 새 `StepExecution`을 생성하고 처음부터 실행합니다. 단, `ItemStream`이 구현돼 있다면 `ExecutionContext`에서 이전 처리 위치를 복원해 중단된 지점부터 재개할 수 있습니다.
>
> **Q2.** `allowStartIfComplete(true)` Step의 재시작 시, 새 `StepExecution`이 생성됩니다. `AbstractStep.execute()`가 `jobRepository.add(stepExecution)`으로 새 `StepExecution`을 저장하고, 새 `ExecutionContext`(비어있는 상태)로 시작합니다. 이전 실행의 `ExecutionContext`는 복원되지 않습니다. 이전 EC 값이 필요하다면 `StepExecutionListener.beforeStep()`에서 `jobRepository.getLastStepExecution()`으로 이전 EC를 조회해야 합니다.
>
> **Q3.** `startLimit(2)`는 이 Step이 실행된 `StepExecution` 총 개수가 `startLimit`에 도달하면 더 이상 실행을 허용하지 않습니다. 세 번째 재시작 시 `shouldStart()`에서 `startCount(2) >= startLimit(2)` 조건을 만족해 `StartLimitExceededException`이 발생하고 Job이 FAILED로 종료됩니다. `startLimit`은 성공/실패와 무관하게 실행 횟수 자체를 제한합니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Conditional Flow ➡️](./02-conditional-flow.md)**

</div>
