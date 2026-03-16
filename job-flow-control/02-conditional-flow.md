# Conditional Flow — ExitStatus 패턴 매칭으로 Step 분기

---

## 🎯 핵심 질문

- `FlowJob`의 전이 규칙(Transition)은 `ExitStatus`를 어떻게 패턴 매칭하는가?
- `.on("*")`, `.on("FAILED")`, `.on("PARTIAL_*")` 와일드카드 우선순위는 어떻게 결정되는가?
- `.end()`, `.fail()`, `.stopAndRestart()` 세 가지 종료 전이의 차이는?
- `FlowJob`의 전이 그래프에서 무한 루프가 발생하는 조건은?
- `NOOP` ExitStatus는 언제 발생하며 Flow에서 어떻게 처리해야 하는가?

---

## 🔍 왜 Conditional Flow가 필요한가

### 문제: 실패해도 계속 진행해야 하는 배치 시나리오

```
시나리오: 데이터 검증 → 처리 → 알림 배치

  검증 결과에 따라 다른 경로:
    검증 통과 (COMPLETED)     → 정상 처리 → 성공 알림
    일부 오류 (PARTIAL_VALID) → 오류 제외 처리 → 경고 알림
    전부 오류 (INVALID)       → 처리 건너뜀 → 실패 알림

  SimpleJob으로는 불가능:
    validationStep FAILED → Job 즉시 종료 → 알림 발송 불가

  FlowJob Conditional Flow:
    validationStep.on("COMPLETED").to(processStep)
    validationStep.on("PARTIAL_*").to(partialProcessStep)  ← 와일드카드
    validationStep.on("INVALID").to(skipProcessStep)
    → 어떤 결과든 알림 Step까지 실행 가능
```

---

## 😱 흔한 실수

### Before: .on() 패턴을 등록 순서에 의존한다

```java
// ❌ 패턴 우선순위를 모르고 구성
jobBuilderFactory.get("job")
    .start(step1())
        .on("*")           // 모든 상태 매칭
        .to(defaultStep())
        .from(step1())
        .on("FAILED")      // FAILED만 매칭
        .to(errorStep())
    .end()
    .build();

// 문제: "*"가 "FAILED"보다 먼저 등록되면 FAILED도 "*"에 매칭될 수 있음
// Spring Batch는 더 구체적인 패턴이 더 높은 우선순위를 가짐
// 하지만 같은 구체성이면 등록 순서가 영향을 줄 수 있어 혼란 발생

// ✅ 구체적인 패턴을 먼저 등록 (의도 명확화)
jobBuilderFactory.get("job")
    .start(step1())
        .on("FAILED").to(errorStep())  // 구체적 먼저
        .from(step1())
        .on("*").to(defaultStep())     // 와일드카드 나중에
    .end()
    .build();
```

### Before: .end() 없이 FlowJob을 완성하려 한다

```java
// ❌ .end() 누락 → 컴파일 오류 또는 런타임 예외
jobBuilderFactory.get("job")
    .start(step1())
        .on("COMPLETED").to(step2())
    // .end() 누락!
    .build();  // JobBuilderException 발생

// ✅ 모든 분기 경로에 종료 선언 필요
jobBuilderFactory.get("job")
    .start(step1())
        .on("COMPLETED").to(step2())
        .from(step1())
        .on("*").fail()
    .end()     // 전체 Flow 종료 선언
    .build();
```

---

## ✨ 올바른 이해와 사용

### FlowJob 전이 구성 문법

```java
@Bean
public Job conditionalJob() {
    return jobBuilderFactory.get("conditionalJob")
        .start(validationStep())
            .on("COMPLETED").to(processStep())   // COMPLETED → processStep
            .on("PARTIAL_*").to(partialStep())   // PARTIAL_로 시작 → partialStep
            .on("FAILED").fail()                 // FAILED → Job FAILED 종료
        .from(processStep())
            .on("COMPLETED").to(reportStep())
            .on("*").to(errorStep())             // 나머지 모두 → errorStep
        .from(partialStep())
            .on("*").to(reportStep())            // 어떤 결과든 → reportStep
        .from(reportStep())
            .on("*").end()                       // reportStep 이후 → 정상 종료
        .from(errorStep())
            .on("*").end()                       // errorStep 이후 → 정상 종료
        .end()
        .build();
}
```

### 세 가지 종료 전이

```java
// 1. .end() — Job COMPLETED로 정상 종료
.on("COMPLETED").end()
.on("COMPLETED").end("CUSTOM_EXIT")  // 커스텀 ExitStatus로 종료

// 2. .fail() — Job FAILED로 실패 종료
.on("FAILED").fail()
// JobExecution.status = FAILED, exitStatus = FAILED
// → 재시작 가능 (FAILED 상태이므로)

// 3. .stopAndRestart(step) — Job STOPPED + 재시작 시 특정 Step부터
.on("STOPPED").stopAndRestart(processStep())
// JobExecution.status = STOPPED
// 재시작 시 processStep부터 재개
// → "중간 점검 후 이어서 처리" 패턴에 유용
```

---

## 🔬 내부 동작 원리

### 1. FlowJob 전이 그래프 내부 구조

```java
// FlowJob.java
public class FlowJob extends AbstractJob {

    private Flow flow;  // State들의 그래프

    @Override
    protected void doExecute(JobExecution execution) throws Exception {
        // FlowExecutor에 실행 위임
        FlowExecutor executor = new JobFlowExecutor(getJobRepository(),
            new SimpleStepHandler(getJobRepository(), ...),
            execution);

        // Flow 실행 (State 전이 그래프 순회)
        FlowExecution flowExecution = flow.start(executor);

        // Flow 최종 상태 → Job ExitStatus 결정
        execution.upgradeStatus(flowExecution.getStatus().getBatchStatus());
        execution.setExitStatus(new ExitStatus(flowExecution.getStatus().getName()));
    }
}

// SimpleFlow.java — 전이 그래프 실행
public FlowExecution start(FlowExecutor executor) throws FlowExecutionException {
    State state = startState;  // 시작 State (첫 번째 Step)

    while (state != null) {
        FlowExecutionStatus status = state.handle(executor);
        // → Step 실행 후 ExitStatus 반환

        // 현재 State의 전이 규칙에서 매칭되는 다음 State 탐색
        state = nextState(state, status, executor);
        // → null이면 Flow 종료
    }

    return new FlowExecution("COMPLETED", ...);
}
```

### 2. ExitStatus 와일드카드 패턴 매칭 — 우선순위 규칙

```java
// StateTransition.java — 패턴 매칭
public static boolean matches(String str, String pattern) {
    // "*"  → 모든 문자열 매칭
    // "?"  → 단일 문자 매칭
    // 나머지 → 정확 일치
    return PatternMatcher.match(pattern, str);
}

// 우선순위 결정 (StateTransition.compareTo):
// 더 구체적인 패턴이 높은 우선순위
// 1. 와일드카드 없는 패턴 (정확 일치) → 최고 우선순위
// 2. "?" 포함 패턴                      → 중간
// 3. "*" 포함 패턴                      → 낮음
// 4. "*" 단독                           → 최저 우선순위

// 예시: ExitStatus = "PARTIAL_VALID"
// 매칭 후보:
//   "PARTIAL_VALID" (정확 일치)         → 1순위
//   "PARTIAL_*"    (와일드카드)          → 2순위
//   "PARTIAL_?????"                      → 3순위
//   "*"            (전체 와일드카드)     → 4순위
```

### 3. .on() 전이 규칙 등록 내부 — StateTransition 목록

```java
// JobFlowBuilder 내부: on() 호출 → StateTransition 등록
// from(step1).on("FAILED").to(errorStep) 는 다음 객체를 만듦:

StateTransition.createStateTransition(
    new StepState(step1),      // 현재 State
    "FAILED",                   // ExitStatus 패턴
    new StepState(errorStep)    // 다음 State
)

// 등록된 전이 목록 (step1에서 출발하는 것들):
// [FAILED → errorStep, PARTIAL_* → partialStep, * → defaultStep]
// → ExitStatus 확인 시 더 구체적인 패턴부터 매칭 시도
```

### 4. NOOP ExitStatus — 발생 조건과 처리

```java
// NOOP은 Step이 실행되지 않았을 때 발생
// SimpleStepHandler.handleStep()에서:
if (shouldNotStart) {
    stepExecution.setExitStatus(ExitStatus.NOOP);
    // BATCH_STEP_EXECUTION.EXIT_CODE = 'NOOP'
}

// 발생 시나리오:
// 재시작 시 COMPLETED Step을 건너뛸 때
// allowStartIfComplete=false + COMPLETED Step

// FlowJob에서 NOOP 처리:
from(step1())
    .on("NOOP").to(nextStep())   // COMPLETED Step 건너뜀 후 다음 Step으로
    .on("COMPLETED").to(processStep())
    .on("*").fail()
```

### 5. Conditional Flow ASCII 다이어그램

```
validationStep
    │
    ├─ ExitStatus "COMPLETED"
    │       │
    │       ▼
    │   processStep ──── ExitStatus "COMPLETED" ──▶ reportStep ──▶ Job END
    │                                                    ↑
    ├─ ExitStatus "PARTIAL_*"                            │
    │       │                                            │
    │       ▼                                            │
    │   partialStep ──── ExitStatus "*" ─────────────────┘
    │
    └─ ExitStatus "FAILED"
            │
            ▼
         Job FAILED (재시작 가능)
```

---

## 💻 실전 구현

### 데이터 유효성 검사 → 조건 분기 전체 예시

```java
@Configuration
public class ValidationFlowJobConfig {

    @Bean
    public Job validationFlowJob() {
        return jobBuilderFactory.get("validationFlowJob")
            .start(dataValidationStep())
                .on("COMPLETED").to(fullProcessStep())
                .on("PARTIAL_VALID").to(partialProcessStep())
                .on("NO_DATA").end("NO_DATA_PROCESSED")  // 데이터 없음 → 정상 종료
                .on("*").fail()
            .from(fullProcessStep())
                .on("COMPLETED").to(successNotificationStep())
                .on("*").to(errorNotificationStep())
            .from(partialProcessStep())
                .on("*").to(warningNotificationStep())
            .from(successNotificationStep()).on("*").end()
            .from(warningNotificationStep()).on("*").end()
            .from(errorNotificationStep()).on("*").end()
            .end()
            .build();
    }

    @Bean
    public Step dataValidationStep() {
        return stepBuilderFactory.get("dataValidationStep")
            .tasklet((contribution, chunkContext) -> {
                ValidationResult result = validator.validate();

                switch (result.getStatus()) {
                    case ALL_VALID:
                        // 기본 COMPLETED
                        return RepeatStatus.FINISHED;
                    case PARTIAL_VALID:
                        contribution.setExitStatus(new ExitStatus("PARTIAL_VALID"));
                        // Job EC에 오류 건수 저장 (다음 Step 활용)
                        chunkContext.getStepContext()
                            .getJobExecutionContext()
                            .put("invalidCount", result.getInvalidCount());
                        return RepeatStatus.FINISHED;
                    case NO_DATA:
                        contribution.setExitStatus(new ExitStatus("NO_DATA"));
                        return RepeatStatus.FINISHED;
                    default:
                        throw new ValidationException("검증 실패: " + result);
                }
            })
            .build();
    }
}
```

### stopAndRestart 패턴 — 수동 확인 후 재개

```java
// 대용량 처리 중 중간 점검이 필요한 경우
@Bean
public Job manualCheckpointJob() {
    return jobBuilderFactory.get("manualCheckpointJob")
        .start(processStep())
            .on("COMPLETED").to(approvalCheckStep())
            .on("*").fail()
        .from(approvalCheckStep())
            .on("APPROVED").to(finalizeStep())
            // 미승인 시: STOPPED로 종료, 재시작 시 approvalCheckStep부터
            .on("PENDING").stopAndRestart(approvalCheckStep())
        .from(finalizeStep())
            .on("*").end()
        .end()
        .build();
}

// approvalCheckStep: 외부 시스템에서 승인 여부 확인
@Bean
public Step approvalCheckStep() {
    return stepBuilderFactory.get("approvalCheckStep")
        .tasklet((contribution, chunkContext) -> {
            ApprovalStatus status = approvalService.checkStatus(
                chunkContext.getStepContext().getJobParameters().get("batchId"));

            if (status == ApprovalStatus.APPROVED) {
                // ExitStatus 기본값 COMPLETED
            } else {
                contribution.setExitStatus(new ExitStatus("PENDING"));
            }
            return RepeatStatus.FINISHED;
        })
        .allowStartIfComplete(true)  // 재시작마다 재실행
        .build();
}
```

---

## 📊 종료 전이 비교

```
┌───────────────────┬──────────────────┬────────────────────┬────────────────────────┐
│ 종료 전이           │ Job BatchStatus  │ Job ExitStatus     │ 재시작 가능 여부           │
├───────────────────┼──────────────────┼────────────────────┼────────────────────────┤
│ .end()            │ COMPLETED        │ COMPLETED          │ ❌ (COMPLETED 재시작 불가)│
│ .end("CUSTOM")    │ COMPLETED        │ CUSTOM             │ ❌                     │
│ .fail()           │ FAILED           │ FAILED             │ ✅                     │
│ .stopAndRestart() │ STOPPED          │ STOPPED            │ ✅ (지정 Step부터)       │
└───────────────────┴──────────────────┴────────────────────┴────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
FlowJob Conditional 설계:

  장점
    어떤 Step 결과에서도 후속 처리 가능 (알림, 정리 등)
    커스텀 ExitStatus로 세밀한 비즈니스 흐름 표현
    .end()/.fail()/.stopAndRestart()로 종료 상태 명확히 제어

  단점
    복잡한 분기는 가독성 저하 (많은 .from().on().to())
    ExitStatus 패턴 우선순위 이해 필요
    모든 분기 경로에 .end()/.fail() 없으면 런타임 오류
    테스트 케이스 수가 분기 수에 비례해 증가

커스텀 ExitStatus:

  장점  비즈니스 언어로 흐름 표현 가능
  단점  문자열 기반이라 오타 시 런타임에서야 발견
       → ExitStatus 상수 클래스 정의 권장
```

---

## 📌 핵심 정리

```
FlowJob 전이 핵심

  .on(pattern)        → ExitStatus.exitCode와 패턴 매칭
  .to(step)           → 다음 Step으로 이동
  .end()              → Job COMPLETED 종료
  .fail()             → Job FAILED 종료 (재시작 가능)
  .stopAndRestart(s)  → Job STOPPED 종료, 재시작 시 s Step부터

와일드카드 우선순위
  정확 일치 > "?" 포함 > "*" 포함 > "*" 단독
  → 구체적인 패턴이 항상 먼저 매칭됨
  → "*"는 나머지 모든 케이스의 기본 처리용

NOOP ExitStatus
  COMPLETED Step을 재시작 시 건너뛸 때 발생
  → .on("NOOP")으로 명시적 처리 가능

.end() 필수
  모든 가능한 분기 경로의 최종 노드에 종료 선언 필요
  → 누락 시 JobBuilderException
```

---

## 🤔 생각해볼 문제

**Q1.** `.on("COMPLETED").end()`를 설정하면 `Job.ExitStatus = COMPLETED`이고, `.on("COMPLETED").end("PARTIAL_SUCCESS")`를 설정하면 `Job.ExitStatus = PARTIAL_SUCCESS`입니다. 그런데 두 경우 모두 `BatchStatus = COMPLETED`입니다. 외부에서 이 Job의 최종 결과를 구분하려면 어떻게 해야 하는가?

**Q2.** `FlowJob`에서 A → B → C의 순환 참조가 생기면(C에서 다시 A로 전이) 어떻게 되는가? Spring Batch가 이를 감지하는가?

**Q3.** `.stopAndRestart(approvalCheckStep())`으로 설정된 Job에서, `approvalCheckStep`에 `allowStartIfComplete(true)` 설정이 없다면 재시작 시 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `JobExecution.getExitStatus().getExitCode()`로 구분합니다. `BatchStatus`는 항상 `COMPLETED`이지만 `ExitStatus`는 다릅니다. `JobExplorer.getJobExecution(id).getExitStatus()`로 조회하거나, `JobExecutionListener.afterJob()`에서 `execution.getExitStatus().getExitCode()`를 확인합니다. 모니터링 시스템에서는 `BATCH_JOB_EXECUTION.EXIT_CODE` 컬럼을 조회하면 됩니다.
>
> **Q2.** Spring Batch는 순환 참조를 빌드 시점에 감지하지 않습니다. `FlowJob` 실행 중 A → B → C → A로 무한 루프가 발생하면 `StackOverflowError` 또는 무한 실행 상태가 됩니다. 이를 방지하려면 방문 횟수를 추적하는 `JobExecutionDecider`를 사용하거나, `startLimit`으로 특정 Step의 최대 실행 횟수를 제한해야 합니다.
>
> **Q3.** `stopAndRestart(approvalCheckStep())`으로 Job이 STOPPED됐을 때, 재시작 시 `approvalCheckStep`을 다시 실행해야 합니다. 그런데 `approvalCheckStep`이 이미 첫 실행에서 어떤 상태로 끝났다면 문제가 생깁니다. 만약 `COMPLETED` 상태였다면 `allowStartIfComplete=false`(기본값)이므로 `shouldStart()`가 `false`를 반환해 건너뛰게 됩니다. 따라서 `stopAndRestart()`로 지정된 Step은 반드시 `allowStartIfComplete(true)`를 설정해야 재시작 시 정상 동작합니다.

---

<div align="center">

**[⬅️ 이전: Sequential Flow](./01-sequential-flow.md)** | **[홈으로 🏠](../README.md)** | **[다음: JobExecutionDecider로 동적 분기 ➡️](./03-job-execution-decider.md)**

</div>
