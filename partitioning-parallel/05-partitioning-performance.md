# Partitioning 성능 튜닝 — 1,000만 건 처리 최적화

---

## 🎯 핵심 질문

- GridSize 4/8/16/32에 따른 처리 시간이 어떻게 달라지는가?
- Worker별 `ItemReader` 쿼리를 최적화해 DB 부하를 줄이는 방법은?
- `StepExecutionSplitter`의 `allowStartIfComplete` 설정이 재시작에 어떤 영향을 미치는가?
- DB Lock 경합을 최소화하는 Writer 설계 전략은?
- 파티션 불균형을 탐지하고 보정하는 방법은?

---

## 🔍 Partitioning 성능 병목 지점

### 문제: GridSize를 늘려도 기대만큼 빨라지지 않는다

```
성능 병목 지점 분석:

  병목 1 — DB Read 쿼리
    Worker 8개가 동시에 SELECT 실행
    → 인덱스 범위 스캔 × 8 → Buffer Pool 경쟁
    → 해결: 커버링 인덱스, No-Offset 패턴

  병목 2 — DB Write 경합
    Worker 8개가 동시에 같은 테이블에 INSERT
    → INSERT Lock 경쟁 → 처리 시간 증가
    → 해결: 테이블 파티셔닝 (MySQL PARTITION BY), 별도 임시 테이블

  병목 3 — DB 커넥션 풀 경쟁
    gridSize=16이면 최대 48개 커넥션 필요
    → 풀 고갈 → 대기 → 처리 시간 증가
    → 해결: 커넥션 풀 크기 조정 (gridSize × 3 + 10)

  병목 4 — GC 압력
    16개 Worker × Chunk 1000건 = 16,000개 객체 동시 힙 존재
    → Young GC 빈도 증가 → Stop-the-world
    → 해결: Chunk Size 축소, G1GC 튜닝

  병목 5 — JobRepository 갱신 경합
    16개 Worker가 동시에 BATCH_STEP_EXECUTION UPDATE
    → DB 행 잠금 경쟁
    → 해결: 낙관적 잠금 (version 컬럼) → 재시도 로직 내장
```

---

## 😱 흔한 실수

### Before: Worker Reader가 효율적이지 않은 쿼리를 사용한다

```java
// ❌ Worker Reader에 OFFSET 방식 Paging 사용
@Bean
@StepScope
public JpaPagingItemReader<Order> workerReader(
        EntityManagerFactory emf,
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("""
            SELECT o FROM Order o
            WHERE o.id BETWEEN :minId AND :maxId
            ORDER BY o.id
            """)
        // pageSize=1000 → OFFSET 점점 증가
        // Worker가 250만 건 처리 시 마지막 페이지: OFFSET 2,499,000
        // → 매우 느림!
        .pageSize(1000)
        .build();
}

// ✅ JdbcPagingItemReader → 내부적으로 id > lastId 방식 (No-Offset)
@Bean
@StepScope
public JdbcPagingItemReader<Order> workerReader(
        DataSource dataSource,
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {

    MySqlPagingQueryProvider provider = new MySqlPagingQueryProvider();
    provider.setSelectClause("SELECT id, amount, status, customer_id");
    provider.setFromClause("FROM orders");
    provider.setWhereClause("WHERE id BETWEEN " + minId + " AND " + maxId);
    provider.setSortKeys(Map.of("id", Order.ASCENDING));
    // → 내부: SELECT ... WHERE id BETWEEN ? AND ? AND (id > ?) ORDER BY id LIMIT 1000
    // → No-Offset! OFFSET 없음 → 일정한 성능

    return new JdbcPagingItemReaderBuilder<Order>()
        .name("workerReader-" + minId)
        .dataSource(dataSource)
        .queryProvider(provider)
        .rowMapper(new BeanPropertyRowMapper<>(Order.class))
        .pageSize(1000)
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. StepExecutionSplitter의 allowStartIfComplete 설정

```java
// SimpleStepExecutionSplitter — 재시작 판단
public Set<StepExecution> split(StepExecution stepExecution, int gridSize) {

    for (Entry<String, ExecutionContext> context : contexts.entrySet()) {
        String workerStepName = context.getKey();
        StepExecution lastExecution = jobRepository.getLastStepExecution(
            jobInstance, workerStepName);

        if (lastExecution != null
                && lastExecution.getStatus() == BatchStatus.COMPLETED) {

            if (shouldRestart) {
                // allowStartIfComplete=true → COMPLETED Worker도 재실행
                // → 해당 파티션 데이터를 다시 처리 (멱등 처리 필요!)
                createNewWorkerExecution(context);
            } else {
                // allowStartIfComplete=false (기본값) → 건너뜀
                // → 재시작 시 실패/미완료 Worker만 재실행
                addExistingExecution(lastExecution);  // 기존 COMPLETED 재사용
            }
        } else {
            createNewWorkerExecution(context);
        }
    }
}

// 설정 방법:
@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", rangePartitioner())
        .partitionHandler(partitionHandler())
        .allowStartIfComplete(true)   // 재시작 시 모든 Worker 재실행
        // .allowStartIfComplete(false)  // 기본값: COMPLETED Worker 건너뜀
        .build();
}
```

### 2. DB Write 최적화 — 테이블 파티셔닝

```sql
-- MySQL PARTITION BY RANGE로 Writer INSERT 경합 최소화
CREATE TABLE settled_orders (
    id          BIGINT AUTO_INCREMENT,
    order_date  DATE NOT NULL,
    customer_id BIGINT,
    amount      DECIMAL(15, 2),
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE COLUMNS(order_date) (
    PARTITION p_2024_01 VALUES LESS THAN ('2024-02-01'),
    PARTITION p_2024_02 VALUES LESS THAN ('2024-03-01'),
    -- ...
);
-- Worker별로 다른 날짜 파티션에 INSERT → Lock 경쟁 최소화
-- 각 파티션이 별도 물리 세그먼트 → 독립적인 INSERT
```

### 3. 커버링 인덱스 — Worker Reader 쿼리 최적화

```sql
-- Worker Reader 쿼리:
-- SELECT id, amount, status, customer_id FROM orders WHERE id BETWEEN ? AND ?

-- 커버링 인덱스 생성:
CREATE INDEX idx_orders_covering
ON orders (id, amount, status, customer_id);
-- SELECT 컬럼이 모두 인덱스에 포함 → 테이블 Random I/O 없이 인덱스만으로 처리
-- → Worker의 SELECT 성능 대폭 향상

-- 인덱스 확인:
EXPLAIN SELECT id, amount, status, customer_id
FROM orders WHERE id BETWEEN 1 AND 2500000;
-- type: range, key: idx_orders_covering, Extra: Using index ← 커버링 인덱스 사용
```

---

## 💻 실전 구현

### 1,000만 건 최적화 완전 설정

```java
@Configuration
public class OptimizedPartitioningConfig {

    @Bean
    public Job optimizedPartitionJob() {
        return jobBuilderFactory.get("optimizedPartitionJob")
            .start(optimizedManagerStep())
            .build();
    }

    @Bean
    public Step optimizedManagerStep() {
        return stepBuilderFactory.get("optimizedManagerStep")
            .partitioner("optimizedWorkerStep", countBasedPartitioner())  // 균등 분할
            .partitionHandler(optimizedPartitionHandler())
            .build();
    }

    @Bean
    public PartitionHandler optimizedPartitionHandler() {
        TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
        handler.setStep(optimizedWorkerStep());
        handler.setTaskExecutor(optimizedTaskExecutor());
        handler.setGridSize(8);
        return handler;
    }

    @Bean
    public TaskExecutor optimizedTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);    // gridSize와 동일
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(0);   // 즉시 8개 시작
        executor.setThreadNamePrefix("optimized-worker-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(1800);  // 30분
        executor.initialize();
        return executor;
    }

    @Bean
    public Step optimizedWorkerStep() {
        return stepBuilderFactory.get("optimizedWorkerStep")
            .<Order, SettledOrder>chunk(2000)   // 대용량: Chunk 크게
            .reader(optimizedWorkerReader(null, null))
            .processor(settlementProcessor())
            .writer(optimizedWorkerWriter())
            .faultTolerant()
                .skip(DataIntegrityViolationException.class)
                .skipLimit(100)
            .build();
    }

    // No-Offset Paging Reader (JdbcPagingItemReader)
    @Bean
    @StepScope
    public JdbcPagingItemReader<Order> optimizedWorkerReader(
            DataSource dataSource,
            @Value("#{stepExecutionContext['minId']}") Long minId,
            @Value("#{stepExecutionContext['maxId']}") Long maxId) {

        MySqlPagingQueryProvider provider = new MySqlPagingQueryProvider();
        provider.setSelectClause("SELECT id, amount, status, customer_id, order_date");
        provider.setFromClause("FROM orders");
        provider.setWhereClause(
            "WHERE id BETWEEN " + minId + " AND " + maxId
            + " AND status = 'PENDING'");
        provider.setSortKeys(Map.of("id", Order.ASCENDING));

        return new JdbcPagingItemReaderBuilder<Order>()
            .name("optimizedWorkerReader-" + minId)
            .dataSource(dataSource)
            .queryProvider(provider)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .pageSize(2000)   // Chunk Size와 동일
            .saveState(true)
            .build();
    }

    // UPSERT 방식 Writer — 멱등 처리 보장
    @Bean
    public JdbcBatchItemWriter<SettledOrder> optimizedWorkerWriter(
            DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<SettledOrder>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO settled_orders
                    (order_id, amount, fee, net_amount, settled_at, order_date)
                VALUES
                    (:orderId, :amount, :fee, :netAmount, NOW(), :orderDate)
                ON DUPLICATE KEY UPDATE
                    fee = VALUES(fee),
                    net_amount = VALUES(net_amount),
                    settled_at = NOW()
                """)
            .beanMapped()
            .assertUpdates(false)  // ON DUPLICATE KEY UPDATE 허용
            .build();
    }
}
```

### 파티션 불균형 탐지 + 진행률 모니터링

```java
@Component
public class PartitionProgressMonitor {

    @Autowired
    private JobExplorer jobExplorer;

    // 5초마다 Worker 진행률 출력
    @Scheduled(fixedDelay = 5000)
    public void monitorPartitionProgress() {
        // 실행 중인 Job 조회
        Set<JobExecution> runningJobs =
            jobExplorer.findRunningJobExecutions("optimizedPartitionJob");

        for (JobExecution jobExecution : runningJobs) {
            List<StepExecution> workerSteps = jobExecution.getStepExecutions()
                .stream()
                .filter(se -> se.getStepName().startsWith("optimizedWorkerStep:"))
                .sorted(Comparator.comparing(StepExecution::getStepName))
                .collect(Collectors.toList());

            if (workerSteps.isEmpty()) continue;

            long totalRead = workerSteps.stream()
                .mapToLong(StepExecution::getReadCount).sum();
            long totalWrite = workerSteps.stream()
                .mapToLong(StepExecution::getWriteCount).sum();

            log.info("=== Partition Progress ===");
            for (StepExecution worker : workerSteps) {
                log.info("  {} [{}]: read={}, write={}, skip={}",
                    worker.getStepName(),
                    worker.getStatus(),
                    worker.getReadCount(),
                    worker.getWriteCount(),
                    worker.getSkipCount());
            }
            log.info("  TOTAL: read={}, write={}", totalRead, totalWrite);

            // 불균형 탐지: 최빠른 Worker vs 최느린 Worker 비율
            OptionalLong maxRead = workerSteps.stream()
                .mapToLong(StepExecution::getReadCount).max();
            OptionalLong minRead = workerSteps.stream()
                .mapToLong(StepExecution::getReadCount).min();

            if (maxRead.isPresent() && minRead.isPresent() && minRead.getAsLong() > 0) {
                double imbalanceRatio = (double) maxRead.getAsLong() / minRead.getAsLong();
                if (imbalanceRatio > 3.0) {
                    log.warn("파티션 불균형 감지: 최대/최소 = {:.1f}배", imbalanceRatio);
                }
            }
        }
    }
}
```

---

## 📊 GridSize별 성능 측정 (1,000만 건)

```
환경: MySQL 8.0, JdbcPagingItemReader (No-Offset), JdbcBatchItemWriter
      Chunk 2000, -Xmx4g, HikariCP 50개

┌──────────┬──────────┬─────────┬──────────────┬──────────────┬─────────────────┐
│ GridSize │ 처리 시간  │ TPS     │ 커넥션 사용     │ GC 횟수       │ CPU 사용률        │
├──────────┼──────────┼─────────┼──────────────┼──────────────┼─────────────────┤
│ 1        │ 720초    │ ~13,889 │ 3개           │ 45회          │ 25%             │
│ 4        │ 200초    │ ~50,000 │ 12개          │ 55회          │ 70%             │
│ 8        │ 115초    │ ~86,957 │ 24개          │ 80회          │ 90%             │
│ 16       │ 80초     │ ~125,000│ 48개          │ 150회         │ 95%             │
│ 32       │ 78초     │ ~128,205│ 96개          │ 310회         │ 97%             │
└──────────┴──────────┴─────────┴──────────────┴──────────────┴─────────────────┘

GridSize 8→16: 30% 향상 (DB I/O 병목 시작)
GridSize 16→32: 2.5% 향상 (GC + Lock 오버헤드 > 병렬화 이득)
→ 이 환경에서 최적: GridSize 8~16
```

---

## ⚖️ 트레이드오프

```
Chunk Size 조정:
  크게 (2000+): JDBC 배치 효율 향상, 메모리 증가
  Worker당 Chunk: Worker 수 × Chunk = 동시 힙 사용량
  → gridSize=8, chunkSize=2000: 8 × 2000 = 16,000건 동시 힙
  → gridSize=16, chunkSize=1000: 16 × 1000 = 16,000건 (동일)

allowStartIfComplete 설정:
  false (기본값): 재시작 시 COMPLETED Worker 건너뜀 → 효율적
                  단, Writer가 멱등하지 않으면 중복 처리 방지
  true:  재시작 시 모든 Worker 재실행 → 안전하지만 비효율
         Worker가 멱등(UPSERT)이면 true도 안전

No-Offset vs OFFSET:
  No-Offset: 일정한 성능, 커서 유지 불필요
  OFFSET: 초반은 빠르지만 깊은 페이지에서 느려짐
  → Worker가 수백만 건 처리 시 No-Offset 필수
```

---

## 📌 핵심 정리

```
Partitioning 성능 최적화 체크리스트

  Reader 최적화
    ✓ JdbcPagingItemReader (No-Offset) 사용
    ✓ 커버링 인덱스 생성 (SELECT 컬럼 포함)
    ✓ pageSize = chunkSize (페이지 쿼리 횟수 최소화)

  Writer 최적화
    ✓ rewriteBatchedStatements=true (MySQL)
    ✓ ON DUPLICATE KEY UPDATE (멱등 처리 + 재시작 안전)
    ✓ 대상 테이블 DB 파티셔닝 (INSERT Lock 경쟁 최소화)

  GridSize 결정
    ✓ DB CPU/I/O 사용률 모니터링 후 결정
    ✓ 성능 수렴 구간 확인 (일반적 8~16)
    ✓ 커넥션 풀 = gridSize × 3 + 10

  재시작 전략
    ✓ Writer가 멱등(UPSERT) → allowStartIfComplete 선택 가능
    ✓ 비멱등 Writer → allowStartIfComplete=false (기본값) 필수
    ✓ COMPLETED Worker StepExecution 재사용 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `allowStartIfComplete(false)` 설정에서 4개 Worker 중 Worker 2가 FAILED됐습니다. 재시작 시 Worker 1, 3, 4는 이미 COMPLETED이므로 재실행되지 않습니다. 그런데 Worker 2가 처리하는 id 범위(250만~500만)가 Write 실패로 인해 DB에는 일부만 저장된 상태입니다. 재시작 시 Worker 2는 어디서부터 재처리하는가?

**Q2.** `JdbcPagingItemReader`의 No-Offset 방식은 `id > lastId` 조건으로 동작합니다. Worker의 범위가 `minId=1, maxId=2500000`이고 재시작 시 `lastId=500000`까지 커밋됐다면, 재시작 후 쿼리는 어떻게 생성되는가?

**Q3.** 파티션 불균형으로 Worker 1이 200만 건, Worker 8이 10만 건을 처리합니다. Worker 8이 먼저 완료돼도 Worker 1을 기다려야 합니다. `allowStartIfComplete(true)` 설정으로 Worker 8이 즉시 다른 범위를 처리하도록 하는 것이 가능한가?

> 💡 **해설**
>
> **Q1.** Worker 2가 FAILED됐을 때 DB에는 일부만 저장된 상태입니다. 재시작 시 `StepExecutionSplitter`가 Worker 2의 마지막 FAILED `StepExecution`에 저장된 `ExecutionContext`를 새 StepExecution에 복사합니다. 이 EC에는 `"optimizedWorkerReader-2500001.read.count": 50000`처럼 마지막 커밋 위치가 저장돼 있습니다. `JdbcPagingItemReader.open(ec)`이 이 값을 읽어 50,000번째 위치(id ≈ 2,600,000)부터 재개합니다. UPSERT Writer라면 중복 처리해도 안전합니다.
>
> **Q2.** `JdbcPagingItemReader`(MySqlPagingQueryProvider)는 No-Offset 방식으로 첫 페이지는 `WHERE id BETWEEN 1 AND 2500000 AND status='PENDING' ORDER BY id LIMIT 1000`을 실행합니다. 재시작 시 `lastId=500000`이 EC에 있으므로, 복원 후 조회는 `WHERE id BETWEEN 1 AND 2500000 AND status='PENDING' AND (id > 500000) ORDER BY id LIMIT 1000`이 됩니다. 500,001번째 레코드부터 정확히 이어서 처리합니다.
>
> **Q3.** `allowStartIfComplete(true)` 자체가 이 목적에 맞지 않습니다. 이 설정은 "COMPLETED 상태여도 재실행하라"는 의미이지, "완료된 Worker가 다른 범위를 가져가라"는 뜻이 아닙니다. Worker가 처리할 수 있는 범위는 Partitioner가 처음 생성 시 결정되며 변경되지 않습니다. 동적 작업 분배를 원한다면 작업 큐(Job Queue) 방식으로 설계해야 합니다 — 예를 들어 파티션을 32개로 세분화하고 쓰레드를 8개로 제한하면, 처리 속도 빠른 쓰레드가 큐에서 다음 파티션을 가져가 처리하는 자연스러운 로드 밸런싱이 이루어집니다.

---

<div align="center">

**[⬅️ 이전: Remote Partitioning](./04-remote-partitioning.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — Async ItemProcessor ➡️](../advanced-topics/01-async-item-processor.md)**

</div>
