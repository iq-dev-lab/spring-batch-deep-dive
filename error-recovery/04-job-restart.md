# Job 재시작과 Restartability — FAILED Job을 정확히 그 지점부터 재개하기

---

## 🎯 핵심 질문

- `JobRepository`가 `FAILED` `JobExecution`을 어떻게 찾아 재시작을 결정하는가?
- `restartable(false)` 설정 시 `JobRestartException`이 발생하는 정확한 시점은?
- 재시작 시 이미 `COMPLETED`인 Step을 건너뛰는 판단 로직은?
- `UNKNOWN` 상태의 `JobExecution`은 왜 재시작할 수 없으며 어떻게 해결하는가?
- 재시작 vs 새 실행의 차이는 무엇이며, 언제 새 실행이 더 나은가?

---

## 🔍 재시작의 핵심: 정확히 어디서부터 재개하는가

### 문제: 50만 건 처리 중 실패했을 때 처음부터 다시 처리하면 너무 비효율적이다

```
재시작 없이:
  1회 실행: 1~500,000번 처리 → 500,001번에서 DB 오류 → FAILED
  재실행:   1번부터 다시 시작
  → 1~500,000번 중복 처리 (이미 DB에 있는 데이터를 다시 INSERT)
  → 중복 처리 방지 로직 없으면 데이터 오염

재시작 with Spring Batch:
  1회 실행: Chunk마다 EC 저장 → 500,001번에서 FAILED
    BATCH_STEP_EXECUTION_CONTEXT: read.count=500000
  재시작: read.count=500000 복원 → 500,001번부터 재개
  → 중복 처리 없음
  → COMPLETED Step은 건너뜀 (다운로드, 검증 등)
```

---

## 😱 흔한 실수

### Before: 재시작과 새 실행을 구분하지 않는다

```java
// ❌ 재시작 의도인데 새 실행으로 처리
// FAILED 상태인 같은 파라미터로 실행 → 재시작 (기존 JobInstance 재사용)
JobParameters params = new JobParametersBuilder()
    .addString("targetDate", "2024-01-01")
    .toJobParameters();

// 이것은 재시작:
jobLauncher.run(job, params);  // 기존 FAILED JobExecution에 이어서 실행

// 만약 새로 처음부터 실행하고 싶다면:
// → RunIdIncrementer로 새 JobInstance 생성해야 함
JobParameters newParams = new JobParametersBuilder()
    .addString("targetDate", "2024-01-01")
    .addLong("runId", System.currentTimeMillis())  // 새 파라미터 = 새 JobInstance
    .toJobParameters();
```

### Before: UNKNOWN 상태를 방치하고 재시작을 시도한다

```java
// ❌ UNKNOWN 상태 JobExecution 재시작 시도
// → JobRestartException: "Cannot restart job from UNKNOWN status"

// UNKNOWN 발생 원인: JVM kill -9, 서버 강제 종료
// 해결 방법:
UPDATE BATCH_JOB_EXECUTION
SET STATUS = 'FAILED',
    EXIT_CODE = 'FAILED',
    EXIT_MESSAGE = 'Manually reset from UNKNOWN for restart'
WHERE JOB_EXECUTION_ID = ?
  AND STATUS = 'UNKNOWN';
// FAILED로 변경 후 재시작 시도
```

---

## ✨ 재시작 동작 완전 이해

### JobExecution 상태별 재시작 가능 여부

```
JobExecution 상태 → 재시작 시 동작:

  COMPLETED   → JobInstanceAlreadyCompleteException (재시작 불가)
  FAILED      → 재시작 가능 (새 JobExecution 생성, 같은 JobInstance)
  STOPPED     → 재시작 가능 (새 JobExecution 생성)
  ABANDONED   → JobRestartException (재시작 불가)
  UNKNOWN     → JobRestartException (자동 재시작 불가, 수동 FAILED 변경 필요)
  STARTING/   → JobExecutionAlreadyRunningException (실행 중)
  STARTED
```

---

## 🔬 내부 동작 원리

### 1. 재시작 시 JobRepository의 JobExecution 처리

```java
// SimpleJobRepository.createJobExecution()
public JobExecution createJobExecution(String jobName, JobParameters jobParameters) {

    // ① 기존 JobInstance 조회
    JobInstance jobInstance = jobInstanceDao.getJobInstance(jobName, jobParameters);

    if (jobInstance == null) {
        // 신규 실행: 새 JobInstance + 새 JobExecution 생성
        jobInstance = jobInstanceDao.createJobInstance(jobName, jobParameters);
    } else {
        // ② 기존 JobInstance → 마지막 JobExecution 상태 확인
        List<JobExecution> executions = jobExecutionDao.findJobExecutions(jobInstance);
        JobExecution lastExecution = executions.isEmpty() ? null : executions.get(0);

        if (lastExecution != null) {
            // 실행 중이면 중복 실행 방지
            if (lastExecution.isRunning()) {
                throw new JobExecutionAlreadyRunningException(...);
            }

            // ③ UNKNOWN → 재시작 금지 (데이터 정합성 불확실)
            if (lastExecution.getStatus() == BatchStatus.UNKNOWN) {
                throw new JobRestartException(
                    "Cannot restart from UNKNOWN status. " +
                    "Manual intervention required.");
            }

            // ④ COMPLETED → 재시작 불가 (정상 완료된 작업 재실행 불가)
            if (lastExecution.getStatus() == BatchStatus.COMPLETED) {
                throw new JobInstanceAlreadyCompleteException(
                    "A job instance already exists and is complete...");
            }
            // FAILED, STOPPED → 재시작 허용
        }
    }

    // ⑤ 새 JobExecution 생성 (기존 JobInstance에 연결)
    JobExecution jobExecution = jobInstance.createJobExecution();
    jobExecutionDao.saveJobExecution(jobExecution);  // status=STARTING
    return jobExecution;
}
```

### 2. 재시작 시 Step 건너뜀 판단 — SimpleStepHandler

```java
// SimpleStepHandler.shouldStart()
private boolean shouldStart(StepExecution lastStepExecution, Step step)
        throws JobRestartException, StartLimitExceededException {

    if (lastStepExecution == null) {
        // 첫 실행 → 무조건 시작
        return true;
    }

    BatchStatus stepStatus = lastStepExecution.getStatus();

    // ① UNKNOWN Step → 재시작 금지
    if (stepStatus == BatchStatus.UNKNOWN) {
        throw new JobRestartException(
            "Step [" + step.getName() + "] is of status UNKNOWN");
    }

    // ② COMPLETED + allowStartIfComplete=false → 건너뜀
    if (stepStatus == BatchStatus.COMPLETED) {
        if (!step.isAllowStartIfComplete()) {
            return false;  // 이미 성공 → 건너뜀
        }
        // allowStartIfComplete=true → 재실행 허용
    }

    // ③ startLimit 확인
    int startCount = jobRepository.getStepExecutionCount(
        lastStepExecution.getJobExecution().getJobInstance(), step.getName());
    if (step.getStartLimit() > 0 && startCount >= step.getStartLimit()) {
        throw new StartLimitExceededException(...);
    }

    return true;
}
```

### 3. restartable(false) 설정 동작

```java
// Job을 재시작 불가로 설정
@Bean
public Job nonRestartableJob() {
    return jobBuilderFactory.get("nonRestartableJob")
        .preventRestart()  // = restartable(false)
        .start(step1())
        .build();
}

// SimpleJobLauncher.run() 에서:
JobExecution lastExecution = jobRepository.getLastJobExecution(jobName, params);
if (lastExecution != null) {
    if (!job.isRestartable()) {
        // FAILED 상태여도 재시작 금지
        throw new JobRestartException(
            "JobInstance already exists and is not restartable");
    }
    // ...
}

// 사용 시나리오:
// - 매일 새로 실행해야 하는 배치 (어제 실패한 것을 오늘 재실행하면 안 됨)
// - RunIdIncrementer로 항상 새 JobInstance 생성하는 경우
// - 배치가 멱등하지 않아 중복 처리 위험이 있는 경우
```

### 4. 재시작 시 ExecutionContext 복원 흐름

```
재시작 실행 흐름:

JobLauncher.run(job, params)
  → JobRepository: FAILED JobExecution 확인 → 새 JobExecution 생성
  → SimpleJob.doExecute()
      → Step 1 (downloadStep): 마지막 실행 COMPLETED → 건너뜀 ✓
      → Step 2 (processStep): 마지막 실행 FAILED → 재실행
          → StepExecution 새로 생성
          → AbstractStep.execute()
              → stream.open(executionContext)
                  ← BATCH_STEP_EXECUTION_CONTEXT에서 EC 복원
                  ← "orderReader.read.count": 500000
              → ItemReader.open() → EC에서 read.count=500000 읽음
                  → jumpToItem(500000) or seek to offset
              → 500,001번 아이템부터 처리 재개
```

---

## 💻 실전 구현

### 재시작 가능한 Job 완전 설정

```java
@Configuration
public class RestartableJobConfig {

    @Bean
    public Job restartableSettlementJob() {
        return jobBuilderFactory.get("settlementJob")
            // .preventRestart() 없음 → 기본값 재시작 가능
            .start(downloadStep())
                .on("COMPLETED").to(processStep())
                .on("*").fail()
            .from(processStep())
                .on("COMPLETED").to(reportStep())
                .on("*").fail()
            .from(reportStep())
                .on("*").end()
            .end()
            .build();
    }

    @Bean
    public Step downloadStep() {
        return stepBuilderFactory.get("downloadStep")
            .tasklet(downloadTasklet())
            .allowStartIfComplete(false)  // 기본값: 재시작 시 성공한 것 건너뜀
            .build();
    }

    @Bean
    public Step processStep() {
        return stepBuilderFactory.get("processStep")
            .<Order, SettledOrder>chunk(1000)
            .reader(orderReader())    // ItemStream 구현 → EC에 read.count 저장
            .processor(processor())
            .writer(writer())
            .faultTolerant()
                .skip(DataIntegrityViolationException.class)
                .skipLimit(50)
            .build();
    }

    @Bean
    public Step reportStep() {
        return stepBuilderFactory.get("reportStep")
            .tasklet(reportTasklet())
            .allowStartIfComplete(true)  // 재시작마다 리포트 재생성
            .build();
    }
}
```

### UNKNOWN 상태 자동 감지 및 복구

```java
@Component
public class JobHealthChecker {

    @Autowired
    private JobExplorer jobExplorer;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    // 애플리케이션 시작 시 UNKNOWN 상태 JobExecution 감지
    @PostConstruct
    public void checkAndRecoverUnknownJobs() {
        String sql = """
            SELECT JOB_EXECUTION_ID, JOB_INSTANCE_ID
            FROM BATCH_JOB_EXECUTION
            WHERE STATUS = 'UNKNOWN'
              AND LAST_UPDATED < DATE_SUB(NOW(), INTERVAL 10 MINUTE)
            """;
        // 10분 이상 된 UNKNOWN은 강제 종료로 판단

        List<Long> unknownIds = jdbcTemplate.queryForList(sql, Long.class);
        if (!unknownIds.isEmpty()) {
            log.warn("UNKNOWN 상태 JobExecution 발견: {}개", unknownIds.size());

            // FAILED로 변경 (재시작 가능 상태로)
            jdbcTemplate.update("""
                UPDATE BATCH_JOB_EXECUTION
                SET STATUS = 'FAILED',
                    EXIT_CODE = 'FAILED',
                    EXIT_MESSAGE = 'Auto-recovered from UNKNOWN. Server may have crashed.',
                    LAST_UPDATED = NOW()
                WHERE JOB_EXECUTION_ID IN (?)
                  AND STATUS = 'UNKNOWN'
                """, unknownIds.get(0));  // 실제로는 IN절로 처리

            alertService.sendAlert("UNKNOWN 상태 JobExecution이 FAILED로 복구됨");
        }
    }
}
```

---

## 📊 재시작 vs 새 실행 비교

```
┌─────────────────────────────┬───────────────────────────────┬────────────────────────────┐
│ 항목                         │ 재시작 (같은 JobParameters)      │ 새 실행 (다른 JobParameters)  │
├─────────────────────────────┼───────────────────────────────┼────────────────────────────┤
│ JobInstance                 │ 재사용 (기존 Instance)           │ 새로 생성                    │
│ JobExecution                │ 새로 생성                        │ 새로 생성                   │
│ COMPLETED Step              │ 건너뜀                          │ 처음부터 실행                 │
│ EC (처리 위치)                │ 이전 EC 복원 (이어서 처리)         │ 초기화 (처음부터)              │
│ 중복 처리 위험                 │ 없음 (EC 기반 재개)              │ 있음 (처음부터 재처리)           │
│ 사용 시나리오                  │ 동일 배치 재실행                  │ 강제 재처리 또는 새 날짜         │
└─────────────────────────────┴───────────────────────────────┴────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
재시작 가능한 Job 설계:

  장점
    실패 시 중단 지점부터 재개 → 효율적
    이미 처리된 데이터 재처리 방지
    COMPLETED Step 건너뜀 → 시간 절약

  단점
    ItemStream 구현 필수 (EC에 위치 저장)
    재시작 로직 복잡성 증가
    멱등하지 않은 Step에 allowStartIfComplete 설정 주의 필요

preventRestart() 사용 시나리오:
  - 배치가 멱등하지 않아 재실행 시 데이터 오염 우려
  - 운영 정책상 실패 시 수동 점검 후 새 배치 실행
  - RunIdIncrementer로 항상 새 JobInstance 생성하는 경우
```

---

## 📌 핵심 정리

```
재시작 가능 조건
  마지막 JobExecution이 FAILED 또는 STOPPED 상태
  Job이 restartable=true (기본값)
  UNKNOWN → 수동으로 FAILED 변경 필요

재시작 실행 흐름
  같은 JobParameters로 run() 호출
  → 기존 JobInstance에 새 JobExecution 생성 (status=STARTING)
  → COMPLETED Step 건너뜀 (allowStartIfComplete=false 기본값)
  → FAILED/STOPPED Step → 새 StepExecution으로 재실행
  → EC에서 이전 처리 위치 복원 → 이어서 처리

UNKNOWN 상태
  JVM 강제 종료 시 발생
  자동 재시작 불가 (데이터 정합성 불확실)
  수동 FAILED 변경 후 재시작 가능
  → 운영 자동화: PostConstruct에서 UNKNOWN 감지 + FAILED 변경

재시작 vs 새 실행
  같은 파라미터 → 재시작 (이어서)
  다른 파라미터 → 새 JobInstance (처음부터)
```

---

## 🤔 생각해볼 문제

**Q1.** 재시작 시 `processStep`이 FAILED였고 새 `StepExecution`으로 재실행됩니다. 이전 `StepExecution`의 `ExecutionContext`는 어떻게 새 `StepExecution`에서 복원되는가?

**Q2.** 재시작 시 COMPLETED된 `downloadStep`을 건너뜁니다. 그런데 `downloadStep`이 다운로드한 파일이 삭제됐다면 `processStep`이 실패합니다. 이 문제를 어떻게 설계로 해결하는가?

**Q3.** `preventRestart()`와 `startLimit(1)`은 어떻게 다른가? 두 설정의 결과가 같아 보이는데 차이점은?

> 💡 **해설**
>
> **Q1.** `AbstractStep.execute()`에서 `jobRepository.getLastStepExecution(jobInstance, stepName)`으로 FAILED된 이전 `StepExecution`을 조회합니다. 이 `StepExecution`의 `ExecutionContext`(`BATCH_STEP_EXECUTION_CONTEXT`에 저장된)를 새 `StepExecution`에 복사합니다. 이후 `CompositeItemStream.open(executionContext)`이 호출되면, 각 `ItemStream`이 이 EC에서 자신의 재시작 포인트를 읽어 이전 위치로 복원합니다.
>
> **Q2.** 설계 해결책들: (1) `downloadStep`에 `allowStartIfComplete(true)` 설정 — 재시작마다 다시 다운로드합니다. 단, 이미 있는 파일이면 덮어쓰기 또는 건너뜀 로직 추가. (2) 파일 존재 여부를 확인하는 `JobExecutionDecider`를 `downloadStep`과 `processStep` 사이에 배치 — 파일 없으면 `downloadStep` 재실행. (3) `downloadStep`에서 다운로드한 파일을 아카이브 디렉토리로 이동 후 원본 보관 — 삭제되지 않도록 관리. 가장 실용적인 방법은 (1)입니다.
>
> **Q3.** `preventRestart()`는 `job.isRestartable() = false`로 설정해 **첫 실행 실패 후 재실행 자체를 차단**합니다 — `JobRestartException`을 던집니다. `startLimit(1)`은 Step 단위 설정으로 **해당 Step이 1번만 실행 가능** — 두 번째 시도 시 `StartLimitExceededException`을 던집니다. 차이점: `preventRestart()`는 Job 전체에 적용되며 FAILED Job의 재실행을 막습니다. `startLimit(1)`은 특정 Step에만 적용되며, Job은 재시작 가능하지만 그 Step만 다시 실행 불가합니다. `allowStartIfComplete=false` + `startLimit(1)`은 "처음 성공/실패 후 두 번 다시는 실행 안 함"과 같습니다.

---

<div align="center">

**[⬅️ 이전: Skip vs Retry 선택 기준](./03-skip-vs-retry.md)** | **[홈으로 🏠](../README.md)** | **[다음: ExecutionContext를 통한 상태 저장 ➡️](./05-execution-context-state.md)**

</div>
