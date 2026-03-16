# Partitioning 개념 — Manager Step이 Worker Step들을 분배·실행하는 전 과정

---

## 🎯 핵심 질문

- `PartitionStep`(Manager)과 Worker Step은 역할이 어떻게 나뉘는가?
- `Partitioner.partition(gridSize)`는 무엇을 반환하며 어떻게 활용되는가?
- `PartitionHandler`는 Worker Step들을 어떻게 실행하는가?
- Partitioning과 Split(Ch3-04)의 근본적인 차이는 무엇인가?
- 각 Worker Step이 어떻게 자신의 데이터 범위를 알고 처리하는가?

---

## 🔍 왜 Partitioning이 필요한가

### 문제: 1,000만 건을 단일 쓰레드로 처리하면 너무 느리다

```
단일 쓰레드 처리 (Chunk 1000건):
  1,000만 건 / 1,000건 = 10,000번 Chunk
  Chunk당 처리 시간 100ms → 총 1,000초 ≈ 17분

Partitioning (4개 Worker):
  각 Worker: 250만 건 처리
  Worker당 처리 시간: 250초
  병렬 실행 → 총 ~250초 ≈ 4분

Split(Ch3-04) vs Partitioning 차이:

  Split: 서로 다른 종류의 작업 병렬화
    emailFlow: 이메일 집계
    smsFlow:   SMS 집계  ← 완전히 다른 Reader/Writer
    → 이종 작업

  Partitioning: 같은 작업을 데이터 범위로 분할
    Worker1: id 1~250만 처리    ← 동일 Step, 다른 데이터 범위
    Worker2: id 250~500만 처리
    Worker3: id 500~750만 처리
    Worker4: id 750~1000만 처리
    → 동종 작업, 데이터 분할
```

---

## 😱 흔한 실수

### Before: Partitioner가 GridSize와 무관하게 고정된 파티션을 생성한다

```java
// ❌ gridSize 파라미터를 무시하고 고정 파티션 생성
@Component
public class FixedPartitioner implements Partitioner {
    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        // gridSize 무시! 항상 4개 파티션 생성
        for (int i = 0; i < 4; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", i * 250000L + 1);
            ctx.putLong("maxId", (i + 1) * 250000L);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    }
}
// gridSize=8로 설정해도 실제 4개 Worker만 생성
// → 설정과 실제 동작 불일치

// ✅ gridSize 파라미터 반영
@Override
public Map<String, ExecutionContext> partition(int gridSize) {
    long totalCount = orderRepository.count();
    long rangeSize = totalCount / gridSize;  // gridSize에 맞게 계산

    Map<String, ExecutionContext> partitions = new LinkedHashMap<>();
    for (int i = 0; i < gridSize; i++) {
        // ...
    }
    return partitions;
}
```

### Before: Worker Step에 @StepScope를 붙이지 않아 파티션 데이터를 읽지 못한다

```java
// ❌ @StepScope 없음 → ExecutionContext 데이터 주입 불가
@Bean
public ItemReader<Order> workerReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("SELECT o FROM Order o WHERE o.id BETWEEN :min AND :max")
        // minId, maxId를 어떻게 가져오나? → @StepScope 없으면 Late Binding 불가
        .build();
}

// ✅ @StepScope로 각 Worker의 ExecutionContext에서 Late Binding
@Bean
@StepScope
public ItemReader<Order> workerReader(
        EntityManagerFactory emf,
        @Value("#{stepExecutionContext['minId']}") Long minId,   // ← Worker EC에서
        @Value("#{stepExecutionContext['maxId']}") Long maxId) { // ← Worker EC에서
    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("SELECT o FROM Order o WHERE o.id BETWEEN :min AND :max")
        .parameterValues(Map.of("min", minId, "max", maxId))
        .build();
}
```

---

## ✨ Partitioning 전체 구조

```java
@Bean
public Job partitionedJob() {
    return jobBuilderFactory.get("partitionedJob")
        .start(managerStep())  // Manager Step (PartitionStep)
        .build();
}

@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", rangePartitioner())    // ① Partitioner 등록
        .partitionHandler(partitionHandler())             // ② PartitionHandler 등록
        .build();
}

@Bean
public PartitionHandler partitionHandler() {
    TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
    handler.setStep(workerStep());                        // ③ Worker Step 지정
    handler.setTaskExecutor(taskExecutor());              // ④ 병렬 실행 Executor
    handler.setGridSize(4);                               // ⑤ 파티션 수
    return handler;
}

@Bean
public Step workerStep() {
    return stepBuilderFactory.get("workerStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(workerReader(null, null, null))  // ③ @StepScope 빈 — null은 Proxy
        .processor(processor())
        .writer(writer())
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. PartitionStep.doExecute() — Manager의 전체 흐름

```java
// PartitionStep.java (Manager Step 구현체)
public class PartitionStep extends AbstractStep {

    private Partitioner partitioner;
    private PartitionHandler partitionHandler;
    private StepExecutionSplitter stepExecutionSplitter;
    private String stepName;  // Worker Step 이름

    @Override
    protected void doExecute(StepExecution stepExecution) throws Exception {

        // ① Partitioner.partition(gridSize) 호출
        //    → Map<String, ExecutionContext> 생성
        //       "workerStep:partition0" → {minId=1, maxId=250만}
        //       "workerStep:partition1" → {minId=250만+1, maxId=500만}
        //       "workerStep:partition2" → {minId=500만+1, maxId=750만}
        //       "workerStep:partition3" → {minId=750만+1, maxId=1000만}

        // ② StepExecutionSplitter가 각 파티션에 StepExecution 생성
        Set<StepExecution> workerStepExecutions =
            stepExecutionSplitter.split(stepExecution, gridSize);
        //    BATCH_STEP_EXECUTION에 4개 행 생성:
        //    step_name="workerStep:partition0", job_execution_id=현재
        //    step_name="workerStep:partition1", ...

        // ③ PartitionHandler에게 Worker StepExecution들을 분배 + 실행 요청
        Collection<StepExecution> finishedStepExecutions =
            partitionHandler.handle(stepExecutionSplitter, stepExecution);

        // ④ 모든 Worker 완료 대기 (블로킹)
        // ⑤ 각 Worker StepExecution 상태를 집계
        //    하나라도 FAILED → Manager Step FAILED
        stepExecution.upgradeStatus(findStatus(finishedStepExecutions));
    }
}
```

### 2. StepExecutionSplitter — Worker별 StepExecution 생성

```java
// SimpleStepExecutionSplitter.split()
public Set<StepExecution> split(StepExecution stepExecution, int gridSize)
        throws JobExecutionException {

    // ① Partitioner 호출
    Map<String, ExecutionContext> contexts = partitioner.partition(gridSize);

    Set<StepExecution> set = new HashSet<>();
    for (Entry<String, ExecutionContext> context : contexts.entrySet()) {
        // ② 파티션 이름 (key) = Worker Step 이름
        // "workerStep:partition0", "workerStep:partition1", ...
        String stepName = context.getKey();
        ExecutionContext workerContext = context.getValue();

        // ③ 재시작 확인: 이미 COMPLETED인 파티션 건너뜀
        StepExecution lastStepExecution = jobRepository.getLastStepExecution(
            stepExecution.getJobExecution().getJobInstance(), stepName);

        if (lastStepExecution != null
                && lastStepExecution.getStatus() == BatchStatus.COMPLETED
                && !shouldRestart) {
            // 이미 완료된 Worker → 재사용 (재처리 안 함)
            set.add(lastStepExecution);
            continue;
        }

        // ④ 새 StepExecution 생성 + EC에 파티션 데이터 설정
        StepExecution workerStepExecution = stepExecution.getJobExecution()
            .createStepExecution(stepName);
        workerStepExecution.setExecutionContext(workerContext);
        //   EC: {minId=1, maxId=2500000}

        // ⑤ DB 저장 (BATCH_STEP_EXECUTION)
        jobRepository.add(workerStepExecution);
        set.add(workerStepExecution);
    }

    return set;
}
```

### 3. TaskExecutorPartitionHandler — Worker 병렬 실행

```java
// TaskExecutorPartitionHandler.handle()
public Collection<StepExecution> handle(
        StepExecutionSplitter stepSplitter,
        StepExecution managerStepExecution) throws Exception {

    // ① Worker StepExecution들 수신
    Set<StepExecution> workerExecutions = stepSplitter.split(
        managerStepExecution, gridSize);

    // ② 각 Worker를 Future로 제출
    List<Future<StepExecution>> futures = new ArrayList<>();
    for (StepExecution workerExecution : workerExecutions) {
        FutureTask<StepExecution> task = new FutureTask<>(() -> {
            // Worker Step 실행 (각자의 EC 포함)
            step.execute(workerExecution);
            return workerExecution;
        });
        futures.add(task);
        taskExecutor.execute(task);  // 병렬 제출
    }

    // ③ 모든 Worker 완료 대기 (블로킹)
    List<StepExecution> results = new ArrayList<>();
    for (Future<StepExecution> future : futures) {
        results.add(future.get());  // 블로킹 대기
    }

    return results;
}
```

### 4. Worker Step에서 stepExecutionContext 접근

```java
// Worker Step 실행 중 Reader가 자신의 파티션 범위를 어떻게 아는가?

// AbstractStep.execute()에서:
// StepContext 활성화 → stepExecutionContext 접근 가능
StepSynchronizationManager.register(workerStepExecution);
// → @Value("#{stepExecutionContext['minId']}") 이 시점에 평가됨
// → workerStepExecution.executionContext = {minId=1, maxId=2500000}
// → minId = 1, maxId = 2500000 주입

// 즉, 각 Worker는 자신의 StepExecution.executionContext에서
// Partitioner가 설정한 범위를 읽어 사용
```

### 5. Partitioning 실행 타임라인 ASCII

```
Manager Step (PartitionStep) 실행:
    │
    ▼ ① Partitioner.partition(gridSize=4)
    │   → Map 생성: partition0~3, 각각 minId/maxId
    │
    ▼ ② StepExecutionSplitter.split()
    │   → BATCH_STEP_EXECUTION 4개 행 생성
    │   → 각 행: workerStep:partition0~3, EC에 범위 포함
    │
    ▼ ③ TaskExecutorPartitionHandler.handle()
    │
    ├─ Thread 1 ──── workerStep:partition0 ─────────────────────────────────┐
    │   minId=1, maxId=2,500,000                                            │
    │   [Read 250만건] [Process] [Write] COMPLETED                           │
    │                                                                       │
    ├─ Thread 2 ──── workerStep:partition1 ───────────────────────────────┐ │
    │   minId=2,500,001, maxId=5,000,000                                  │ │
    │   [Read 250만건] [Process] [Write] COMPLETED                         │ │
    │                                                                     │ │
    ├─ Thread 3 ──── workerStep:partition2 ─────────────────────────────┐ │ │
    │   minId=5,000,001, maxId=7,500,000                                │ │ │
    │   [Read 250만건] [Process] [Write] COMPLETED                       │ │ │
    │                                                                   │ │ │
    ├─ Thread 4 ──── workerStep:partition3 ─────────────────────────┐   │ │ │
    │   minId=7,500,001, maxId=10,000,000                           │   │ │ │
    │   [Read 250만건] [Process] [Write] COMPLETED                   │   │ │ │
    │                                                               │   │ │ │
    ◀────────────── 모든 Worker 완료 대기 (Future.get()) ──────────────┘───┘──┘─┘
    │
    ▼ ④ 결과 집계: 모두 COMPLETED → Manager Step COMPLETED
    │
    ▼ Job COMPLETED
```

---

## 💻 실전 구현

### 최소한의 완전한 Partitioning 구성

```java
@Configuration
public class PartitioningJobConfig {

    @Bean
    public Job partitionedSettlementJob() {
        return jobBuilderFactory.get("partitionedSettlementJob")
            .start(managerStep())
            .build();
    }

    // Manager Step
    @Bean
    public Step managerStep() {
        return stepBuilderFactory.get("managerStep")
            .partitioner("workerStep", rangePartitioner())
            .partitionHandler(partitionHandler())
            .build();
    }

    // Partitioner: ID 범위 분할
    @Bean
    public Partitioner rangePartitioner() {
        return gridSize -> {
            long minId = orderRepository.findMinId();
            long maxId = orderRepository.findMaxId();
            long rangeSize = (maxId - minId) / gridSize + 1;

            Map<String, ExecutionContext> partitions = new LinkedHashMap<>();
            for (int i = 0; i < gridSize; i++) {
                ExecutionContext ctx = new ExecutionContext();
                ctx.putLong("minId", minId + rangeSize * i);
                ctx.putLong("maxId", i == gridSize - 1
                    ? maxId
                    : minId + rangeSize * (i + 1) - 1);
                ctx.putInt("partitionIndex", i);
                partitions.put("workerStep:partition" + i, ctx);
            }
            return partitions;
        };
    }

    // PartitionHandler
    @Bean
    public PartitionHandler partitionHandler() {
        TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
        handler.setStep(workerStep());
        handler.setTaskExecutor(partitionTaskExecutor());
        handler.setGridSize(4);
        return handler;
    }

    @Bean
    public TaskExecutor partitionTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(0);
        executor.setThreadNamePrefix("partition-worker-");
        executor.initialize();
        return executor;
    }

    // Worker Step
    @Bean
    public Step workerStep() {
        return stepBuilderFactory.get("workerStep")
            .<Order, SettledOrder>chunk(1000)
            .reader(workerReader(null, null, null))
            .processor(processor())
            .writer(writer())
            .build();
    }

    // Worker Reader: stepExecutionContext에서 범위 읽기
    @Bean
    @StepScope
    public JpaPagingItemReader<Order> workerReader(
            EntityManagerFactory emf,
            @Value("#{stepExecutionContext['minId']}") Long minId,
            @Value("#{stepExecutionContext['maxId']}") Long maxId) {
        return new JpaPagingItemReaderBuilder<Order>()
            .name("workerOrderReader")
            .entityManagerFactory(emf)
            .queryString("""
                SELECT o FROM Order o
                WHERE o.id BETWEEN :minId AND :maxId
                  AND o.status = 'PENDING'
                ORDER BY o.id
                """)
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(1000)
            .build();
    }
}
```

---

## 📊 Partitioning vs Split 비교

```
┌──────────────────────────┬────────────────────────────────┬─────────────────────────────┐
│ 항목                      │ Partitioning                   │ Split                       │
├──────────────────────────┼────────────────────────────────┼─────────────────────────────┤
│ 작업 유형                  │ 동종 (같은 Step, 다른 데이터)       │ 이종 (다른 Step, 다른 작업)      │
│ 데이터 분할                 │ Partitioner가 범위 계산          │ 각 Flow가 독립 데이터            │
│ StepExecution            │ Worker마다 독립 StepExecution    │ 각 Step에 StepExecution       │
│ 재시작                     │ 실패 Worker만 재실행              │ 실패 Branch의 Step 재실행       │
│ 확장성                     │ GridSize로 쉽게 조절              │ Flow 수 변경 = 코드 수정        │
│ 적합한 경우                 │ 대용량 단일 종류 데이터 처리          │ 독립적인 여러 도메인 처리         │
└──────────────────────────┴────────────────────────────────┴─────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Partitioning:

  장점
    GridSize 조절만으로 병렬도 변경 (코드 수정 없음)
    각 Worker가 독립 StepExecution → 재시작 시 실패 Worker만 재실행
    Remote Partitioning으로 여러 서버에 분산 가능

  단점
    Partitioner 구현 필요 (데이터 분할 로직)
    Worker Reader를 @StepScope로 구성해야 함
    DB 커넥션 풀이 GridSize 수만큼 필요 (각 Worker가 커넥션 사용)
    파티션 간 데이터 경계 불균형 시 특정 Worker만 오래 걸림
```

---

## 📌 핵심 정리

```
Partitioning 3요소

  Partitioner      → gridSize개의 ExecutionContext 맵 생성
                     각 EC에 Worker의 데이터 범위 (minId, maxId) 저장
  PartitionHandler → Worker Step들을 병렬 실행
  Worker Step      → @StepScope Bean으로 EC에서 범위 읽어 처리

PartitionStep 실행 순서
  1. Partitioner.partition(gridSize) → Map<String, EC>
  2. StepExecutionSplitter.split() → Worker StepExecution들 DB 저장
  3. PartitionHandler.handle() → Worker들 병렬 실행
  4. Future.get()으로 모든 Worker 완료 대기
  5. 결과 집계 → Manager Step 상태 결정

Worker Step의 범위 접근
  @StepScope + @Value("#{stepExecutionContext['minId']}")
  → AbstractStep 실행 시 StepContext 활성화 → EC 값 Late Binding
```

---

## 🤔 생각해볼 문제

**Q1.** Manager Step이 실행되는 동안 `PartitionStep.doExecute()`는 Worker들이 모두 완료될 때까지 블로킹됩니다. 이 블로킹 시간 동안 Manager Step의 `StepExecution` 상태는 무엇인가?

**Q2.** `gridSize=4`로 설정했는데 데이터가 3개 파티션 밖에 안 되는 상황(총 3건)이면 어떻게 되는가? `Partitioner.partition(4)`에서 3개의 Entry만 반환하면?

**Q3.** 4개 Worker 중 Worker 2가 FAILED됐을 때 재시작하면, Worker 1, 3, 4는 이미 COMPLETED이므로 건너뛰어집니다. `StepExecutionSplitter`가 이를 판단하는 정확한 조건은 무엇인가?

> 💡 **해설**
>
> **Q1.** Manager Step의 `StepExecution.status = STARTED`입니다. `PartitionStep.doExecute()`가 호출된 상태이므로 STARTED 상태가 유지됩니다. `BATCH_STEP_EXECUTION` 테이블에서 Manager Step은 `START_TIME`이 설정됐지만 `END_TIME`은 `null`인 채로 Worker들이 완료될 때까지 대기합니다. Worker들의 `StepExecution`도 각각 독립적으로 STARTED → COMPLETED 상태로 변경됩니다.
>
> **Q2.** `Partitioner.partition(4)`에서 3개의 Entry만 반환하면, `StepExecutionSplitter`는 3개의 Worker `StepExecution`만 생성합니다. `TaskExecutorPartitionHandler`는 3개의 Worker만 실행하고 4번째는 없으므로 정상 동작합니다. `gridSize`는 권장 파티션 수이지 강제 사항이 아닙니다. 단, `PartitionHandler.setGridSize(4)`는 Partitioner 호출 시 전달되는 hint일 뿐이며, 실제 파티션 수는 Partitioner 반환값으로 결정됩니다.
>
> **Q3.** `SimpleStepExecutionSplitter.split()` 내부에서 `jobRepository.getLastStepExecution(jobInstance, stepName)`으로 각 Worker의 마지막 StepExecution을 조회합니다. Worker의 step_name이 "workerStep:partition1"처럼 고유하므로 각 Worker를 독립적으로 확인할 수 있습니다. `lastStepExecution != null && lastStepExecution.status == COMPLETED && !shouldRestart(step)`이면 해당 Worker를 건너뜁니다. `shouldRestart`는 Step의 `allowStartIfComplete` 설정에 따라 결정됩니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Partitioner 구현 ➡️](./02-partitioner-implementation.md)**

</div>
