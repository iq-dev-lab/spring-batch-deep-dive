# Job·Step Listener — 실행 전후 훅과 Step 간 데이터 전달

---

## 🎯 핵심 질문

- `JobExecutionListener`와 `StepExecutionListener`의 각 메서드는 정확히 언제 호출되는가?
- `@BeforeJob`, `@AfterStep` 어노테이션 기반 등록 방식은 어떻게 동작하는가?
- `afterStep()`에서 반환하는 `ExitStatus`가 Step의 최종 ExitStatus를 변경하는 원리는?
- Listener에서 `ExecutionContext`를 통해 Step 간 데이터를 전달하는 안전한 패턴은?
- `ChunkListener`와 `ItemReadListener` / `ItemWriteListener`는 언제 각각 사용하는가?

---

## 🔍 Listener 계층과 역할

### 문제: Step 실행 전후에 공통 작업이 필요하다

```
Listener가 없으면:

  Step Tasklet에 공통 로직 혼재:
    public RepeatStatus execute(...) {
        // 실행 전: 처리 시작 로그, 락 획득
        log.info("Step 시작");
        acquireLock();
        try {
            // 실제 처리
            process();
        } finally {
            // 실행 후: 락 해제, 완료 시간 기록
            releaseLock();
            recordEndTime();
        }
    }
  → 모든 Step이 동일 코드 반복
  → Chunk 기반 Step에는 finally 훅이 없음

Listener 해결:
  @BeforeStep   → 락 획득, 시작 로그
  @AfterStep    → 락 해제, 시간 기록, Step 결과를 Job EC에 저장
  @BeforeChunk  → Chunk 시작 시간 기록
  @AfterChunk   → Chunk 처리 시간 로깅, 진행률 계산
```

---

## 😱 흔한 실수

### Before: afterJob()에서 JobExecution 상태를 변경하려 한다

```java
// ❌ afterJob()에서 Status를 변경해도 DB에 반영 안 됨
@Override
public void afterJob(JobExecution jobExecution) {
    if (jobExecution.getStatus() == BatchStatus.FAILED) {
        // 이 변경은 DB에 저장되지 않음!
        jobExecution.setStatus(BatchStatus.COMPLETED);
        jobExecution.setExitStatus(ExitStatus.COMPLETED);
    }
}
// afterJob()은 JobRepository.update(jobExecution)이 이미 호출된 후
// → 상태 변경해도 DB에 반영되지 않음

// ✅ afterStep()에서 ExitStatus 변경 (DB 반영됨)
@Override
public ExitStatus afterStep(StepExecution stepExecution) {
    // afterStep()의 반환값이 Step의 최종 ExitStatus로 사용됨
    // → AbstractStep.execute()에서 이 값을 StepExecution에 적용 후 DB 저장
    if (someCondition) {
        return new ExitStatus("CUSTOM_CODE");
    }
    return stepExecution.getExitStatus();  // 기존 값 유지
}
```

### Before: Listener에서 checked 예외를 무시한다

```java
// ❌ Listener 예외를 삼켜버림
@Override
public void beforeJob(JobExecution jobExecution) {
    try {
        sendJobStartNotification();
    } catch (Exception e) {
        log.error("알림 실패", e);
        // 예외를 throw하지 않음 → Job은 계속 실행됨
    }
}
// 알림 실패가 조용히 무시됨 → 운영자가 Job 시작을 모름

// ✅ 알림 실패가 치명적이면 예외를 전파
@Override
public void beforeJob(JobExecution jobExecution) {
    sendJobStartNotification();
    // 예외 발생 시 Job 시작 자체를 중단 (선택적)
}
```

---

## ✨ Listener 종류와 호출 시점

```java
// 호출 순서 타임라인:

Job 시작
  └─ JobExecutionListener.beforeJob()          ← ①

  Step 1 시작
    └─ StepExecutionListener.beforeStep()      ← ②

    Chunk 1 시작
      └─ ChunkListener.beforeChunk()           ← ③

      아이템 읽기/처리/쓰기 루프:
        ItemReadListener.beforeRead()          ← ④
        ItemReadListener.afterRead(item)
        ItemProcessListener.beforeProcess()
        ItemProcessListener.afterProcess()
        ItemWriteListener.beforeWrite()
        ItemWriteListener.afterWrite()

      └─ ChunkListener.afterChunk()            ← ⑤

    Chunk N 완료...

    └─ StepExecutionListener.afterStep()       ← ⑥ (ExitStatus 반환 가능)
  Step 1 완료

└─ JobExecutionListener.afterJob()             ← ⑦
Job 완료
```

---

## 🔬 내부 동작 원리

### 1. afterStep() 반환값이 ExitStatus를 변경하는 원리

```java
// AbstractStep.execute() — afterStep() 호출 후 처리
public final void execute(StepExecution stepExecution) {
    // ... Step 실행 ...

    ExitStatus exitStatus = stepExecution.getExitStatus();

    try {
        // ① StepExecutionListener.afterStep() 호출
        // 반환된 ExitStatus가 null이 아니면 교체
        exitStatus = listener.afterStep(stepExecution);
        if (exitStatus != null) {
            stepExecution.setExitStatus(exitStatus);
            // ← 이 시점에 ExitStatus가 변경됨
        }
    } catch (Exception e) {
        log.error("afterStep 예외", e);
        stepExecution.setStatus(BatchStatus.UNKNOWN);
    } finally {
        // ② DB에 최종 StepExecution 저장 (변경된 ExitStatus 포함)
        getJobRepository().update(stepExecution);
    }
}

// 활용: 처리 건수 0이면 ExitStatus를 "NO_DATA"로 변경
@Override
public ExitStatus afterStep(StepExecution stepExecution) {
    if (stepExecution.getWriteCount() == 0) {
        return new ExitStatus("NO_DATA");
    }
    return null;  // null 반환 시 기존 ExitStatus 유지
}
```

### 2. @BeforeJob / @AfterStep 어노테이션 기반 등록

```java
// 어노테이션 기반: JobExecutionListener 인터페이스 구현 없이도 동작
@Component
public class JobAuditListener {

    @BeforeJob
    public void beforeJob(JobExecution jobExecution) {
        log.info("[JOB 시작] {} | params: {}",
            jobExecution.getJobInstance().getJobName(),
            jobExecution.getJobParameters());
    }

    @AfterJob
    public void afterJob(JobExecution jobExecution) {
        long durationMs = jobExecution.getEndTime().getTime()
            - jobExecution.getStartTime().getTime();
        log.info("[JOB 완료] {} | status: {} | {}ms",
            jobExecution.getJobInstance().getJobName(),
            jobExecution.getStatus(),
            durationMs);
    }
}

// Job에 등록 방법:
// 1. JobBuilder.listener()
@Bean
public Job myJob(JobAuditListener jobAuditListener) {
    return jobBuilderFactory.get("myJob")
        .listener(jobAuditListener)   // ← 어노테이션 기반 Listener 등록
        .start(step1())
        .build();
}

// 2. StepBuilder.listener()
@Bean
public Step myStep(StepAuditListener stepAuditListener) {
    return stepBuilderFactory.get("myStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .writer(writer())
        .listener(stepAuditListener)  // ← Step Listener 등록
        .build();
}
```

### 3. Listener에서 Step 간 데이터 전달

```java
// Step 1의 afterStep에서 Job EC에 데이터 저장
@Component
public class Step1ResultListener implements StepExecutionListener {

    @Override
    public void beforeStep(StepExecution stepExecution) {
        // Step 시작 시간 기록 (진행 모니터링용)
        stepExecution.getExecutionContext()
            .put("step1.startTime", System.currentTimeMillis());
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        long writeCount = stepExecution.getWriteCount();
        long filterCount = stepExecution.getFilterCount();
        long elapsed = System.currentTimeMillis()
            - stepExecution.getExecutionContext().getLong("step1.startTime");

        // ← Job EC에 저장 (다음 Step에서 @Value로 접근 가능)
        stepExecution.getJobExecution().getExecutionContext()
            .putLong("step1.processedCount", writeCount)
            .put("step1.filterRate",
                writeCount > 0 ? (double) filterCount / writeCount : 0)
            .putLong("step1.elapsedMs", elapsed);

        log.info("[Step1 완료] processed={}, filtered={}, {}ms",
            writeCount, filterCount, elapsed);

        // 처리 건수 0이면 후속 Step을 건너뛸 수 있도록 ExitStatus 변경
        if (writeCount == 0) {
            return new ExitStatus("NO_DATA");
        }
        return null;
    }
}

// Step 2에서 Step 1 결과 활용
@Bean
@StepScope
public Tasklet step2Tasklet(
        @Value("#{jobExecutionContext['step1.processedCount']}") Long processedCount,
        @Value("#{jobExecutionContext['step1.filterRate']}") Double filterRate) {

    return (contribution, chunkContext) -> {
        log.info("Step1 처리 결과 활용: count={}, filterRate={}",
            processedCount, filterRate);
        if (filterRate > 0.3) {
            // 30% 이상 필터링 → 경고 알림
            alertService.sendHighFilterWarning(filterRate);
        }
        return RepeatStatus.FINISHED;
    };
}
```

### 4. ChunkListener — 처리 진행률 추적

```java
@Component
public class ProgressChunkListener implements ChunkListener {

    private static final int LOG_INTERVAL = 10;  // 10 Chunk마다 로그
    private long chunkCount = 0;
    private long startTime;

    @Override
    public void beforeChunk(ChunkContext context) {
        if (chunkCount == 0) {
            startTime = System.currentTimeMillis();
        }
    }

    @Override
    public void afterChunk(ChunkContext context) {
        chunkCount++;
        StepExecution se = context.getStepContext().getStepExecution();

        if (chunkCount % LOG_INTERVAL == 0) {
            long totalWritten = se.getWriteCount();
            long elapsed = System.currentTimeMillis() - startTime;
            double tps = elapsed > 0 ? totalWritten / (elapsed / 1000.0) : 0;

            log.info("[진행률] Chunk #{} | 누적: {}건 | TPS: {:.1f}",
                chunkCount, totalWritten, tps);
        }
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        log.error("[Chunk 오류] Chunk #{} 실패", chunkCount);
    }
}
```

---

## 💻 실전 구현

### 완전한 Listener 조합 예시

```java
@Configuration
public class ListenerConfig {

    // Job 전체 모니터링
    @Bean
    public JobExecutionListener jobMonitoringListener() {
        return new JobExecutionListenerSupport() {

            @Override
            public void beforeJob(JobExecution jobExecution) {
                metricsService.recordJobStart(jobExecution.getJobInstance().getJobName());
                log.info("[JOB 시작] id={}", jobExecution.getId());
            }

            @Override
            public void afterJob(JobExecution jobExecution) {
                metricsService.recordJobEnd(
                    jobExecution.getJobInstance().getJobName(),
                    jobExecution.getStatus());

                // afterJob에서는 ExitStatus 변경이 DB 반영 안 됨
                // → 알림/모니터링 목적으로만 사용
                if (jobExecution.getStatus() == BatchStatus.FAILED) {
                    alertService.sendJobFailureAlert(jobExecution);
                }
            }
        };
    }

    // Step별 처리 통계 수집
    @Bean
    public StepExecutionListener stepStatisticsListener() {
        return new StepExecutionListenerSupport() {

            @Override
            public ExitStatus afterStep(StepExecution stepExecution) {
                statisticsService.record(
                    stepExecution.getStepName(),
                    stepExecution.getReadCount(),
                    stepExecution.getWriteCount(),
                    stepExecution.getFilterCount(),
                    stepExecution.getSkipCount()
                );

                // 처리 결과 → Job EC 저장 (다음 Step 활용)
                stepExecution.getJobExecution().getExecutionContext()
                    .putLong(stepExecution.getStepName() + ".writeCount",
                        stepExecution.getWriteCount());

                return null;  // ExitStatus 변경 없음
            }
        };
    }

    // Chunk 에러 발생 시 알림
    @Bean
    public ChunkListener chunkErrorListener() {
        return new ChunkListenerSupport() {
            @Override
            public void afterChunkError(ChunkContext context) {
                Throwable error = (Throwable) context.getAttribute(
                    ChunkListener.ROLLBACK_EXCEPTION_KEY);
                log.error("[Chunk 롤백] 원인: {}", error != null ? error.getMessage() : "unknown");
            }
        };
    }
}

// Job에 모든 Listener 등록
@Bean
public Job monitoredJob(JobExecutionListener jobMonitoringListener) {
    return jobBuilderFactory.get("monitoredJob")
        .listener(jobMonitoringListener)
        .start(monitoredStep())
        .build();
}

@Bean
public Step monitoredStep(StepExecutionListener stepStatisticsListener,
                            ChunkListener chunkErrorListener,
                            ChunkListener progressChunkListener) {
    return stepBuilderFactory.get("monitoredStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .listener(stepStatisticsListener)
        .listener(chunkErrorListener)
        .listener(progressChunkListener)
        .build();
}
```

---

## 📊 Listener 종류 요약

```
┌──────────────────────────┬─────────────────────────────────┬──────────────────────────┐
│ Listener                 │ 주요 메서드                        │ 주 용도                    │
├──────────────────────────┼─────────────────────────────────┼──────────────────────────┤
│ JobExecutionListener     │ beforeJob, afterJob             │ Job 시작/종료 알림, 감사      │
│ StepExecutionListener    │ beforeStep, afterStep*          │ Step 통계, EC 데이터 전달    │
│ ChunkListener            │ beforeChunk, afterChunk, error  │ 진행률 추적, Chunk 오류      │
│ ItemReadListener         │ beforeRead, afterRead, onError  │ 읽기 오류 추적              │
│ ItemProcessListener      │ beforeProcess, afterProcess     │ 처리 변환 추적              │
│ ItemWriteListener        │ beforeWrite, afterWrite, onError│ 쓰기 오류 추적              │
│ SkipListener             │ onSkipInRead/Process/Write      │ Skip 건 알림, 오류 로그      │
│ RetryListener            │ open, onError, close            │ Retry 시도 추적            │
└──────────────────────────┴─────────────────────────────────┴──────────────────────────┘
* afterStep()은 ExitStatus를 반환해 Step의 최종 ExitStatus 변경 가능
```

---

## ⚖️ 트레이드오프

```
Listener 사용:

  장점
    Step/Job 로직과 횡단 관심사(로깅, 알림, 통계) 분리
    afterStep()으로 ExitStatus 동적 변경
    EC를 통한 Step 간 데이터 전달 표준화

  단점
    Listener 등록 순서가 호출 순서에 영향 (여러 Listener 등록 시)
    afterJob()에서는 DB 반영 불가 → ExitStatus 변경 불가
    Listener 내부 예외가 원래 예외를 덮어쓸 수 있음

afterStep() ExitStatus 변경:

  장점  Tasklet/Chunk 처리 로직에서 흐름 제어 코드 제거
  단점  Step의 실제 처리와 흐름 결정이 분리 → 디버깅 시 Listener도 확인해야 함
```

---

## 📌 핵심 정리

```
Listener 호출 순서
  beforeJob → beforeStep → beforeChunk → (read/process/write 루프) → afterChunk → afterStep → afterJob

afterStep() 특이점
  반환값 (ExitStatus)이 Step의 최종 ExitStatus로 사용됨
  null 반환 시 기존 ExitStatus 유지
  afterJob()은 반환값 없음, DB 반영 불가

Step 간 데이터 전달 패턴
  afterStep()에서 stepExecution.getJobExecution().getExecutionContext()에 저장
  다음 Step에서 @Value("#{jobExecutionContext['key']}") 또는 EC 직접 접근

Listener 등록 방법
  인터페이스 구현: JobExecutionListener, StepExecutionListener 등
  어노테이션: @BeforeJob, @AfterJob, @BeforeStep, @AfterStep (인터페이스 불필요)
  Job/Step Builder의 .listener() 메서드로 등록
```

---

## 🤔 생각해볼 문제

**Q1.** `StepExecutionListener`가 여러 개 등록된 경우, `afterStep()` 메서드가 여러 번 호출됩니다. 첫 번째 Listener가 `ExitStatus("CUSTOM")`을 반환하고, 두 번째 Listener가 `ExitStatus.COMPLETED`를 반환하면 최종 `ExitStatus`는 무엇인가?

**Q2.** `JobExecutionListener.afterJob()`이 호출되는 시점에 `JobExecution.status = FAILED`입니다. 이 안에서 `jobExecution.setExitStatus(new ExitStatus("HANDLED"))`를 호출해도 DB에 반영되지 않는다고 했습니다. 그렇다면 `afterJob()` 호출 이후 DB에 저장하는 방법은?

**Q3.** `ItemSkipListener.onSkipInWrite(item, t)`가 호출될 때, 이 아이템은 이미 어떤 처리 단계에 있는가? `FaultTolerantChunkProcessor`가 Skip을 처리하는 흐름에서 이 Listener는 어느 시점에 호출되는가?

> 💡 **해설**
>
> **Q1.** `CompositeStepExecutionListener`가 여러 Listener를 순서대로 호출합니다. 각 `afterStep()` 반환값으로 `StepExecution.exitStatus`를 업데이트합니다. 두 Listener가 각각 `"CUSTOM"`과 `"COMPLETED"`를 반환하면, 나중에 호출된 Listener의 반환값이 최종적으로 적용됩니다. 즉, 등록 순서가 마지막인 Listener가 `ExitStatus.COMPLETED`를 반환하면 최종 ExitStatus는 `COMPLETED`가 됩니다. 따라서 ExitStatus 변경 Listener는 등록 순서를 신중히 결정해야 합니다.
>
> **Q2.** `afterJob()` 호출 시점 이후 DB에 반영하려면 `jobRepository.update(jobExecution)`을 직접 호출해야 합니다. `JobRepository` Bean을 Listener에 주입받아: `jobExecution.setExitStatus(new ExitStatus("HANDLED")); jobRepository.update(jobExecution);` 순서로 호출합니다. 단, 이는 Spring Batch의 생명주기를 우회하는 것이므로, 가급적 `afterStep()`에서 `ExitStatus`를 제어하는 것이 권장됩니다. 대안으로 `JobExecutionListener`를 `AbstractJob.execute()` 내부 `finally` 블록 이전에 실행되도록 순서를 조정하는 방법도 있지만, 내부 구현에 의존적입니다.
>
> **Q3.** `FaultTolerantChunkProcessor`가 Write 단계에서 예외 발생 시 Chunk를 1건씩 재처리합니다(스캐터-개더). 각 아이템을 개별 트랜잭션으로 실행하고, 다시 예외가 발생하면 해당 아이템을 Skip 처리합니다. `onSkipInWrite(item, t)`는 이 개별 재처리 트랜잭션이 롤백된 직후, `skip` 여부가 확정된 시점에 호출됩니다. 즉, 이미 해당 아이템의 트랜잭션이 롤백된 상태이며, `skipCount`가 증가하기 직전입니다. 이 Listener에서 Skip된 아이템을 별도 테이블에 기록하거나 알림을 보낼 수 있습니다.

---

<div align="center">

**[⬅️ 이전: Flow 외부화와 재사용](./05-flow-externalization.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Skip 전략 ➡️](../error-recovery/01-skip-strategy.md)**

</div>
