# JobExecutionDecider로 동적 분기 — 런타임 비즈니스 조건으로 Flow 결정

---

## 🎯 핵심 질문

- `JobExecutionDecider`는 `ExitStatus`와 어떻게 다른가? 왜 별도로 필요한가?
- `decide(JobExecution, StepExecution)`의 두 파라미터를 어떻게 활용하는가?
- Decider는 Step과 달리 `StepExecution`을 생성하지 않는데, DB에는 어떻게 기록되는가?
- Decider가 반환하는 `FlowExecutionStatus`와 `ExitStatus`는 어떤 관계인가?
- 여러 조건을 조합한 복잡한 분기를 Decider 없이 ExitStatus만으로 구현할 때의 한계는?

---

## 🔍 왜 Decider가 필요한가

### 문제: ExitStatus만으로는 외부 조건 분기가 불가능하다

```
ExitStatus의 한계:

  ExitStatus는 Step 실행 결과로만 결정됨
  → "오늘이 월말인가?"
  → "파일이 존재하는가?"
  → "처리 건수가 0인가?"

  이런 조건을 ExitStatus로 표현하려면:
    Step 내부에서 조건을 확인하고 ExitStatus를 설정해야 함
    → Step Tasklet에 비즈니스 로직 + 흐름 제어 로직이 섞임

  Decider 해결책:
    조건 확인 로직을 별도 컴포넌트로 분리
    → Step은 자기 일만 (단일 책임)
    → Decider는 흐름 결정만 (런타임 조건 기반)
    → 테스트 용이성 향상
```

---

## 😱 흔한 실수

### Before: Step Tasklet에 흐름 제어 로직을 섞는다

```java
// ❌ Tasklet에 흐름 제어 로직 혼재
@Bean
public Step processStep() {
    return stepBuilderFactory.get("processStep")
        .tasklet((contribution, chunkContext) -> {
            int processedCount = process();

            // 흐름 제어 로직이 Tasklet 내부에!
            if (processedCount == 0) {
                contribution.setExitStatus(new ExitStatus("NO_DATA"));
            } else if (isMonthEnd()) {
                contribution.setExitStatus(new ExitStatus("MONTH_END"));
            }
            return RepeatStatus.FINISHED;
        })
        .build();
}
// → process() 테스트 시 흐름 제어 로직도 함께 검증해야 함
// → isMonthEnd() 조건이 바뀌면 Tasklet 코드 수정 필요

// ✅ Decider로 분리
@Bean
public Step processStep() {
    return stepBuilderFactory.get("processStep")
        .tasklet((contribution, chunkContext) -> {
            process();  // 순수 처리 로직만
            return RepeatStatus.FINISHED;
        })
        .build();
}
// Decider: processedCount, isMonthEnd() 확인 → FlowExecutionStatus 반환
```

---

## ✨ JobExecutionDecider 인터페이스

```java
// JobExecutionDecider
public interface JobExecutionDecider {

    /**
     * @param jobExecution  현재 Job 전체 실행 컨텍스트
     * @param stepExecution 직전에 실행된 Step의 실행 컨텍스트 (null 가능 — 첫 노드)
     * @return FlowExecutionStatus — 다음 전이를 결정하는 상태 코드
     */
    FlowExecutionStatus decide(JobExecution jobExecution,
                                @Nullable StepExecution stepExecution);
}

// FlowExecutionStatus 편의 상수
FlowExecutionStatus.COMPLETED   // "COMPLETED"
FlowExecutionStatus.FAILED      // "FAILED"
FlowExecutionStatus.STOPPED     // "STOPPED"
// 또는 커스텀:
new FlowExecutionStatus("NO_DATA")
new FlowExecutionStatus("MONTH_END_PROCESSING")
```

---

## 🔬 내부 동작 원리

### 1. Decider의 Flow 내 위치 — DecisionState

```java
// SimpleFlow 내부에서 Decider는 DecisionState로 표현됨
// StepState(Step 실행)와 달리 DecisionState는 StepExecution을 생성하지 않음

// FlowBuilder.next(decider) 또는 .start(decider) 호출 시:
DecisionState decisionState = new DecisionState(decider, stateName);

// DecisionState.handle()
@Override
public FlowExecutionStatus handle(FlowExecutor executor) throws Exception {
    JobExecution jobExecution = executor.getJobExecution();
    StepExecution stepExecution = executor.getStepExecution();

    // decide() 호출 — StepExecution 생성 없음
    FlowExecutionStatus status = decider.decide(jobExecution, stepExecution);

    // 반환값이 DB에 기록되지 않음 (StepExecution 없으므로)
    // → BATCH_STEP_EXECUTION에 Decider 이름으로 행이 생기지 않음
    return status;
}
```

### 2. Decider와 ExitStatus의 관계

```java
// FlowExecutionStatus는 내부적으로 String name을 갖는 래퍼
// .on() 패턴 매칭 시 FlowExecutionStatus.getName()이 사용됨

// Decider가 반환: new FlowExecutionStatus("MONTH_END")
// Flow 전이: .on("MONTH_END").to(monthEndStep())
// 매칭: "MONTH_END".equals("MONTH_END") → true → monthEndStep 실행

// ExitStatus와 동일한 패턴 매칭 알고리즘 사용
// (와일드카드 "*", "?" 동일하게 작동)
```

### 3. Decider 실행 타임라인

```
processStep 실행 완료
    │
    ▼ JobExecution, StepExecution(processStep) 전달
flowDecider.decide() 호출
    │
    ├─ ExecutionContext 조회 (processedCount 등)
    ├─ 외부 조건 확인 (날짜, 파일 존재 등)
    └─ FlowExecutionStatus 반환 ("MONTH_END" 등)
    │
    ▼ FlowExecutionStatus로 전이 규칙 매칭
    ├─ "MONTH_END"  → monthEndStep
    └─ "NORMAL"     → normalStep

DB에는 DecisionState 기록 없음
StepExecution은 processStep, monthEndStep 것만 생성됨
```

---

## 💻 실전 구현

### 처리 건수 기반 분기 Decider

```java
// Decider: 직전 Step의 처리 건수를 읽어 분기 결정
@Component
public class ProcessCountDecider implements JobExecutionDecider {

    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution,
                                       StepExecution stepExecution) {
        if (stepExecution == null) {
            // Decider가 첫 노드인 경우 (거의 없음)
            return new FlowExecutionStatus("NO_STEP");
        }

        long writeCount = stepExecution.getWriteCount();
        long filterCount = stepExecution.getFilterCount();

        if (writeCount == 0) {
            log.info("처리된 데이터 없음 — 알림 건너뜀");
            return new FlowExecutionStatus("NO_DATA");
        }

        if (filterCount > writeCount * 0.1) {
            // 필터 비율 10% 초과 → 경고 플로우
            log.info("필터 비율 높음: filtered={}, written={}", filterCount, writeCount);
            return new FlowExecutionStatus("HIGH_FILTER_RATE");
        }

        return FlowExecutionStatus.COMPLETED;
    }
}

// 월말 여부 + 공휴일 복합 조건 Decider
@Component
public class CalendarDecider implements JobExecutionDecider {

    @Autowired
    private HolidayService holidayService;

    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution,
                                       StepExecution stepExecution) {
        LocalDate today = LocalDate.now();
        boolean isLastDayOfMonth = today.equals(today.with(TemporalAdjusters.lastDayOfMonth()));
        boolean isHoliday = holidayService.isHoliday(today);

        if (isLastDayOfMonth && !isHoliday) {
            return new FlowExecutionStatus("MONTH_END");
        }
        if (isHoliday) {
            return new FlowExecutionStatus("HOLIDAY");
        }
        return FlowExecutionStatus.COMPLETED;
    }
}

// Job 구성 — Decider를 노드로 삽입
@Bean
public Job settlementJob() {
    return jobBuilderFactory.get("settlementJob")
        .start(processStep())
            .on("COMPLETED").to(processCountDecider())  // ① Step 후 Decider
            .on("FAILED").to(errorStep())
        // ② Decider 분기
        .from(processCountDecider())
            .on("NO_DATA").end("NO_DATA")
            .on("HIGH_FILTER_RATE").to(warningStep())
            .on("COMPLETED").to(calendarDecider())      // ③ 중첩 Decider
        .from(calendarDecider())
            .on("MONTH_END").to(monthEndReportStep())
            .on("HOLIDAY").end("HOLIDAY_SKIP")
            .on("COMPLETED").to(normalReportStep())
        .from(warningStep()).on("*").to(normalReportStep())
        .from(monthEndReportStep()).on("*").end()
        .from(normalReportStep()).on("*").end()
        .end()
        .build();
}
```

### 파일 존재 여부 Decider

```java
@Component
@StepScope  // JobParameters Late Binding
public class FileExistenceDecider implements JobExecutionDecider {

    @Override
    public FlowExecutionStatus decide(JobExecution jobExecution,
                                       StepExecution stepExecution) {
        String filePath = jobExecution.getJobParameters()
            .getString("inputFile");

        if (filePath == null || !new File(filePath).exists()) {
            log.warn("입력 파일 없음: {}", filePath);
            return new FlowExecutionStatus("FILE_NOT_FOUND");
        }

        long fileSize = new File(filePath).length();
        if (fileSize == 0) {
            return new FlowExecutionStatus("EMPTY_FILE");
        }

        return FlowExecutionStatus.COMPLETED;
    }
}

// Job 처음 시작 전 파일 확인
@Bean
public Job fileProcessingJob() {
    return jobBuilderFactory.get("fileProcessingJob")
        .start(fileExistenceDecider())  // Decider가 첫 번째 노드
            .on("COMPLETED").to(parseStep())
            .on("FILE_NOT_FOUND").to(fileDownloadStep())  // 없으면 먼저 다운로드
            .on("EMPTY_FILE").end("EMPTY_FILE_SKIP")
        .from(fileDownloadStep())
            .on("COMPLETED").to(parseStep())
            .on("*").fail()
        .from(parseStep())
            .on("*").end()
        .end()
        .build();
}
```

---

## 📊 Decider vs ExitStatus 비교

```
┌──────────────────────────┬──────────────────────────────┬──────────────────────────┐
│ 항목                      │ ExitStatus (Step 설정)        │ JobExecutionDecider      │
├──────────────────────────┼──────────────────────────────┼──────────────────────────┤
│ 조건 확인 위치              │ Step 내부 (Tasklet/Listener)   │ 독립 컴포넌트               │
│ StepExecution 생성        │ ✅                           │ ❌ (DB 기록 없음)          │
│ 외부 상태 접근              │ 제한적 (EC 통해서만)             │ 자유롭게 (Bean 주입)         │
│ 단일 책임 원칙              │ 혼재 가능성                     │ ✅ (흐름 결정만)            │
│ 테스트 용이성               │ Step과 함께 테스트 필요           │ 독립 단위 테스트 가능         │
│ 복잡한 조건 조합             │ 어려움                         │ 자유로운 Java 코드          │
└──────────────────────────┴──────────────────────────────┴──────────────────────────┘

Decider 사용 권장 시나리오:
  - 처리 건수 기반 분기 (0건, 일반, 대량)
  - 날짜/시간 기반 분기 (월말, 공휴일, 특정 요일)
  - 외부 파일/시스템 상태 기반 분기
  - 여러 Step 결과를 종합한 복합 조건 분기
```

---

## ⚖️ 트레이드오프

```
JobExecutionDecider:

  장점
    Step과 흐름 로직의 완전한 분리
    외부 시스템 조회, 복잡한 조건 자유롭게 구현
    단위 테스트 용이 (Mock 주입 가능)
    DB에 기록이 없어 메타데이터 깔끔

  단점
    DB에 실행 기록 없음 → 디버깅 시 어느 분기를 탔는지 로그 의존
    decide()에서 예외 발생 시 어느 Decider에서 실패했는지 파악 어려움
    → 명시적 로깅 필수
    @StepScope와 함께 사용 시 Decider 빈 스코프 주의
```

---

## 📌 핵심 정리

```
Decider 동작 핵심

  FlowExecutionStatus 반환 → .on() 패턴 매칭 → 다음 Step 결정
  StepExecution 생성 없음 → BATCH_STEP_EXECUTION 행 없음
  직전 StepExecution을 두 번째 파라미터로 받음 → writeCount, EC 접근 가능
  JobExecution을 첫 번째 파라미터로 받음 → JobParameters, 전체 EC 접근

Decider 활용 패턴

  처리 결과 분기:  stepExecution.getWriteCount() == 0 → NO_DATA
  날짜 분기:       LocalDate.now().getDayOfMonth() == lastDay → MONTH_END
  파일 확인:       new File(path).exists() → FILE_FOUND / FILE_NOT_FOUND
  복합 조건:       여러 Bean 주입 후 자유로운 Java 로직

Decider vs ExitStatus 선택
  Step이 자신의 결과를 알고 있을 때 → ExitStatus (Step 내에서 설정)
  외부 조건 또는 여러 Step 결과 종합 → Decider (독립 컴포넌트)
```

---

## 🤔 생각해볼 문제

**Q1.** Decider가 `FlowExecutionStatus.FAILED`를 반환했을 때, 해당 분기에 `.on("FAILED").fail()`이 설정돼 있다면 Job은 어떤 상태로 종료되는가? 이 경우 재시작 시 어느 State부터 시작되는가?

**Q2.** `decide(JobExecution, StepExecution)`에서 두 번째 파라미터 `StepExecution`이 `null`인 경우는 언제인가? 이 경우 어떻게 안전하게 처리해야 하는가?

**Q3.** Decider를 `@StepScope`로 등록하는 것이 의미가 있는가? Decider는 `StepExecution`을 생성하지 않는데, `@StepScope` Bean의 라이프사이클과 어떤 관계가 있는가?

> 💡 **해설**
>
> **Q1.** Decider가 `FAILED`를 반환하고 `.on("FAILED").fail()`이 설정돼 있으면 `Job.BatchStatus = FAILED`, `Job.ExitStatus = FAILED`로 종료됩니다. 재시작 시에는 마지막으로 실행된 `StepExecution`이 있는 Step부터 재시작됩니다. Decider 자체는 `StepExecution`을 생성하지 않으므로, Decider 직전에 실행된 Step이 재시작 대상이 됩니다. 단, 그 Step이 `COMPLETED`라면 `allowStartIfComplete` 설정에 따라 건너뛸 수 있습니다. `FlowJob`에서 재시작 시에는 이미 `COMPLETED`된 Step들을 건너뛰고, 마지막 `FAILED` 또는 미실행 Step부터 재개합니다.
>
> **Q2.** `StepExecution`이 `null`인 경우는 Decider가 **Job의 첫 번째 노드**로 배치됐을 때입니다. `.start(decider)`처럼 Job이 Decider로 시작하면 아직 어떤 Step도 실행되지 않았으므로 직전 `StepExecution`이 없습니다. 안전한 처리법: `if (stepExecution == null) { return new FlowExecutionStatus("INITIAL"); }`로 null을 명시적으로 처리하거나, `Optional.ofNullable(stepExecution).map(se -> se.getWriteCount()).orElse(0L)`처럼 null-safe하게 접근합니다.
>
> **Q3.** Decider를 `@StepScope`로 등록하는 것은 일반적으로 의미가 없습니다. `@StepScope`는 Step 실행 컨텍스트(`StepSynchronizationManager`)를 기반으로 Bean을 관리하는데, Decider는 `DecisionState.handle()`에서 직접 호출되며 `StepContext`가 활성화된 환경이 아닙니다. 만약 Decider에서 `@Value("#{jobParameters['key']}")`로 JobParameters를 Late Binding하려면 `@JobScope`를 사용하는 것이 적절합니다. `@JobScope`는 JobExecution 컨텍스트를 기반으로 동작하므로 Decider의 실행 환경과 일치합니다.

---

<div align="center">

**[⬅️ 이전: Conditional Flow](./02-conditional-flow.md)** | **[홈으로 🏠](../README.md)** | **[다음: Parallel Steps ➡️](./04-parallel-steps-split.md)**

</div>
