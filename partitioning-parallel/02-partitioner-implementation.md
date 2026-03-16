# Partitioner 구현 — ID 범위·파일·날짜 기반 데이터 분할 전략

---

## 🎯 핵심 질문

- ID 범위 기반 분할에서 데이터 불균형을 어떻게 방지하는가?
- 파일 목록 기반 분할에서 각 Worker가 자신의 파일을 어떻게 읽는가?
- 날짜 기반 분할에서 파티션 경계를 어떻게 설정하는가?
- Partitioner가 반환하는 Map의 Key 이름이 Worker Step 이름과 어떻게 연관되는가?
- 데이터가 없는 파티션(빈 범위)이 생기면 어떻게 처리되는가?

---

## 🔍 분할 전략의 핵심 원칙

### 좋은 Partitioner의 3가지 조건

```
1. 균등 분할 (Load Balancing)
   Worker 1: 250만 건 처리
   Worker 2: 250만 건 처리
   Worker 3: 250만 건 처리
   Worker 4: 250만 건 처리
   → 총 처리 시간 = max(Worker 처리 시간) ≈ 250만 건 처리 시간
   
   불균등 분할:
   Worker 1: 10만 건
   Worker 2: 50만 건
   Worker 3: 400만 건  ← 병목
   Worker 4: 300만 건
   → 총 처리 시간 = Worker 3 처리 시간 ≈ 400만 건 처리 시간
   → Worker 1이 일찍 끝나도 전체는 Worker 3을 기다림

2. 겹치지 않는 범위 (No Overlap)
   Worker 1: id 1~250만, Worker 2: id 250만+1~500만
   → 중복 처리 없음
   → Writer에서 PK 충돌 없음

3. 재시작 가능 (Restartable)
   각 Worker의 처리 위치를 ExecutionContext에 저장
   → FAILED Worker만 재실행
```

---

## 😱 흔한 실수

### Before: AUTO_INCREMENT ID 기반 균등 분할이 실제로 균등하지 않다

```java
// ❌ ID를 균등하게 나눠도 실제 데이터가 편중될 수 있음
// ID 범위: 1 ~ 10,000,000 (총 1,000만)
// 하지만 실제 데이터:
//   id 1~2,500,000: 100만 건 (소프트 딜리트 등으로 id 공백 많음)
//   id 2,500,001~5,000,000: 300만 건
//   id 5,000,001~7,500,000: 250만 건
//   id 7,500,001~10,000,000: 350만 건

// 단순 범위 분할:
// Worker 1: id 1~2,500,000 → 실제 100만 건
// Worker 4: id 7,500,001~10,000,000 → 실제 350만 건
// → 3.5배 불균형!

// ✅ 실제 데이터 수 기반 분할 (더 정확하지만 비용 발생)
long totalCount = orderRepository.count();  // 실제 건수
// 또는 분위수(percentile) 기반 분할 → 각 파티션 실제 건수 동일하게
```

---

## 🔬 분할 전략 구현

### 1. ID 범위 기반 Partitioner (가장 일반적)

```java
@Component
public class IdRangePartitioner implements Partitioner {

    @Autowired
    private OrderRepository orderRepository;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        // ① 전체 ID 범위 조회 (인덱스 스캔으로 빠름)
        long minId = orderRepository.findMinIdByStatus("PENDING");
        long maxId = orderRepository.findMaxIdByStatus("PENDING");

        if (minId > maxId) {
            // 처리할 데이터 없음 → 빈 파티션 방지
            log.info("처리할 데이터 없음. 단일 빈 파티션 생성");
            Map<String, ExecutionContext> emptyPartition = new LinkedHashMap<>();
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", 0L);
            ctx.putLong("maxId", -1L);  // maxId < minId → Reader가 빈 결과 반환
            emptyPartition.put("workerStep:partition0", ctx);
            return emptyPartition;
        }

        // ② gridSize에 맞게 범위 계산
        long totalRange = maxId - minId + 1;
        long rangeSize = (totalRange + gridSize - 1) / gridSize;  // 올림 나눗셈

        Map<String, ExecutionContext> partitions = new LinkedHashMap<>();
        for (int i = 0; i < gridSize; i++) {
            long partitionMin = minId + rangeSize * i;
            long partitionMax = Math.min(minId + rangeSize * (i + 1) - 1, maxId);

            if (partitionMin > maxId) break;  // 실제 데이터보다 파티션이 많은 경우

            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", partitionMin);
            ctx.putLong("maxId", partitionMax);
            ctx.putInt("partitionIndex", i);
            ctx.putString("partitionName", "partition" + i);

            partitions.put("workerStep:partition" + i, ctx);
            log.info("파티션 {}: minId={}, maxId={}", i, partitionMin, partitionMax);
        }

        return partitions;
    }
}
```

### 2. 실제 건수 기반 균등 분할 (정확하지만 추가 쿼리 필요)

```java
@Component
public class CountBasedPartitioner implements Partitioner {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        // NTILE 함수로 실제 건수 기반 균등 분할
        // 각 파티션의 실제 데이터 건수가 동일하도록 경계 계산
        List<Long> boundaries = jdbcTemplate.queryForList(
            """
            SELECT MAX(id) as boundary_id
            FROM (
                SELECT id,
                       NTILE(?) OVER (ORDER BY id) as bucket
                FROM orders
                WHERE status = 'PENDING'
            ) t
            GROUP BY bucket
            ORDER BY bucket
            """,
            Long.class,
            gridSize
        );

        Map<String, ExecutionContext> partitions = new LinkedHashMap<>();
        long prevBoundary = 0;

        for (int i = 0; i < boundaries.size(); i++) {
            long boundary = boundaries.get(i);
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", prevBoundary + 1);
            ctx.putLong("maxId", boundary);
            partitions.put("workerStep:partition" + i, ctx);
            prevBoundary = boundary;
        }

        return partitions;
    }
}
```

### 3. 파일 목록 기반 Partitioner

```java
@Component
public class FileListPartitioner implements Partitioner {

    @Value("${batch.input.dir:/data/input}")
    private String inputDir;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        // ① 처리할 파일 목록 수집
        File[] files = new File(inputDir).listFiles(
            file -> file.getName().endsWith(".csv") && file.isFile());

        if (files == null || files.length == 0) {
            throw new IllegalStateException("처리할 파일 없음: " + inputDir);
        }

        // ② 각 파일을 하나의 파티션으로 (gridSize 무시 — 파일 수가 파티션 수)
        Map<String, ExecutionContext> partitions = new LinkedHashMap<>();
        for (int i = 0; i < files.length; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putString("filePath", files[i].getAbsolutePath());
            ctx.putString("fileName", files[i].getName());
            ctx.putLong("fileSize", files[i].length());
            partitions.put("workerStep:file-" + files[i].getName(), ctx);
        }

        log.info("파일 파티셔닝: {}개 파일", partitions.size());
        return partitions;
    }
}

// Worker Reader: 파일 경로 Late Binding
@Bean
@StepScope
public FlatFileItemReader<OrderCsvRow> fileWorkerReader(
        @Value("#{stepExecutionContext['filePath']}") String filePath,
        @Value("#{stepExecutionContext['fileName']}") String fileName) {
    log.info("Worker Reader 초기화: {}", fileName);
    return new FlatFileItemReaderBuilder<OrderCsvRow>()
        .name("fileWorkerReader-" + fileName)  // 이름에 파일명 포함 (EC 키 충돌 방지)
        .resource(new FileSystemResource(filePath))
        .linesToSkip(1)
        .delimited().names("orderId", "amount", "customerId")
        .targetType(OrderCsvRow.class)
        .build();
}
```

### 4. 날짜 기반 Partitioner (일별/월별 처리)

```java
@Component
public class DateRangePartitioner implements Partitioner {

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        // 예: 최근 N일치 데이터를 각 Worker에 배분
        LocalDate endDate = LocalDate.now().minusDays(1);
        LocalDate startDate = endDate.minusDays(gridSize - 1);

        Map<String, ExecutionContext> partitions = new LinkedHashMap<>();
        LocalDate currentDate = startDate;

        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putString("targetDate", currentDate.toString());
            ctx.putString("startTime", currentDate.atStartOfDay().toString());
            ctx.putString("endTime", currentDate.plusDays(1).atStartOfDay().toString());
            partitions.put("workerStep:date-" + currentDate, ctx);
            currentDate = currentDate.plusDays(1);
        }

        return partitions;
    }
}

// Worker Reader: 날짜 기반 쿼리
@Bean
@StepScope
public JpaPagingItemReader<Order> dateWorkerReader(
        EntityManagerFactory emf,
        @Value("#{stepExecutionContext['startTime']}") String startTime,
        @Value("#{stepExecutionContext['endTime']}") String endTime) {

    LocalDateTime start = LocalDateTime.parse(startTime);
    LocalDateTime end = LocalDateTime.parse(endTime);

    return new JpaPagingItemReaderBuilder<Order>()
        .name("dateWorkerReader-" + startTime.substring(0, 10))
        .entityManagerFactory(emf)
        .queryString("""
            SELECT o FROM Order o
            WHERE o.createdAt >= :start AND o.createdAt < :end
            ORDER BY o.id
            """)
        .parameterValues(Map.of("start", start, "end", end))
        .pageSize(1000)
        .build();
}
```

---

## 💻 실전 구현

### 재시작 안전한 Worker Reader — EC에 진행 위치 저장

```java
// @StepScope Worker Reader는 자동으로 EC에 read.count 저장
// → 재시작 시 "workerStep:partition0.read.count" 에서 복원

// Worker Reader 이름을 파티션별로 고유하게 설정하는 이유:
@Bean
@StepScope
public JpaPagingItemReader<Order> workerReader(
        EntityManagerFactory emf,
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId,
        @Value("#{stepExecutionContext['partitionIndex']}") Integer partitionIndex) {

    return new JpaPagingItemReaderBuilder<Order>()
        // 이름에 파티션 인덱스 포함 → EC 키 충돌 방지
        // EC 저장: "workerReader-0.read.count" vs "workerReader-1.read.count"
        .name("workerReader-" + partitionIndex)
        .entityManagerFactory(emf)
        .queryString("""
            SELECT o FROM Order o
            WHERE o.id BETWEEN :minId AND :maxId
            ORDER BY o.id
            """)
        .parameterValues(Map.of("minId", minId, "maxId", maxId))
        .pageSize(1000)
        .saveState(true)  // 재시작 가능 (기본값)
        .build();
}
```

---

## 📊 분할 전략 비교

```
┌────────────────────┬────────────────────────┬────────────────┬──────────────────────────────┐
│ 전략                │ 분할 기준                │ 균등성           │ 적합한 경우                     │
├────────────────────┼────────────────────────┼────────────────┼──────────────────────────────┤
│ ID 범위 (단순)       │ max-min / gridSize     │ △ (ID 공백 시)   │ ID 연속적이고 공백 적은 경우       │
│ 실제 건수 (NTILE)    │ 실제 데이터 균등 분할       │ ✅             │ ID 공백 많은 경우                │
│ 파일 기반            │ 파일 1개 = 파티션 1개      │ 파일 크기 의존     │ 여러 파일 병렬 처리              │
│ 날짜 기반            │ 날짜 1일 = 파티션 1개      │ 날짜별 건수 의존   │ 일별 배치 처리                  │
│ 커스텀 컬럼 기반       │ 도메인 컬럼 (지역, 등급)    │ 도메인 의존       │ 비즈니스 파티셔닝                │
└────────────────────┴────────────────────────┴────────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
ID 범위 분할 (단순):
  장점: DB 쿼리 최소 (MIN/MAX 2번), 구현 단순
  단점: ID 공백 많으면 불균등, 소프트 딜리트 데이터 고려 필요

실제 건수 분할 (NTILE):
  장점: 균등 보장, 처리 시간 예측 가능
  단점: 전체 데이터 스캔 쿼리 필요 (대용량 시 비용)

파일 기반:
  장점: 직관적 (파일 수 = Worker 수)
  단점: 파일 크기 불균등 시 Worker 처리 시간 불균등
       gridSize 파라미터 의미 없어짐

날짜 기반:
  장점: 비즈니스 의미 있는 분할
  단점: 특정 날짜에 데이터 쏠림 시 불균등
```

---

## 📌 핵심 정리

```
Partitioner 반환값 구조
  Map<String, ExecutionContext>
    Key: "workerStep:partition0" (Worker Step 이름 prefix 포함)
    Value: EC에 Worker의 데이터 범위 정보
           {minId=1, maxId=2500000, partitionIndex=0}

Worker Reader 구성 필수 사항
  @StepScope 필수 → EC에서 Late Binding
  @Value("#{stepExecutionContext['minId']}") → 각 Worker 범위 접근
  .name("reader-" + partitionIndex) → EC 키 충돌 방지 (중요!)

빈 파티션 처리
  minId > maxId인 파티션 → Reader가 빈 결과 반환
  → 해당 Worker: readCount=0, writeCount=0 → COMPLETED
  → 정상 처리 (오류 아님)

재시작 안전성
  .saveState(true) 기본값 → Chunk 커밋마다 EC 저장
  "readerName.read.count" = 현재까지 읽은 건수
  재시작 시 해당 Worker의 마지막 커밋 위치부터 재개
```

---

## 🤔 생각해볼 문제

**Q1.** `IdRangePartitioner`에서 `minId=1, maxId=10,000,000, gridSize=4`로 설정했을 때, Worker 1이 처리 중 새로운 id=8,000,000 주문이 INSERT됩니다. 이 새 주문은 어느 Worker에서 처리되는가?

**Q2.** `FileListPartitioner`에서 `gridSize` 파라미터를 완전히 무시하고 파일 수만큼 Worker를 생성합니다. 파일이 100개라면 100개의 Worker가 동시에 실행될 수 있습니다. 이를 제한하는 방법은?

**Q3.** 날짜 기반 Partitioner에서 Worker 1이 "2024-01-01" 데이터를 처리하다 실패했습니다. 재시작 시 Worker 1의 `StepExecution` 이름은 "workerStep:date-2024-01-01"입니다. 재시작 후 `StepExecutionSplitter`가 이 Worker를 재실행하는 조건은?

> 💡 **해설**
>
> **Q1.** 파티셔닝은 배치 시작 시 한 번만 수행됩니다. `IdRangePartitioner.partition()`이 호출된 시점의 `maxId`를 기준으로 분할합니다. `id=8,000,000`이 배치 시작 후 INSERT됐다면, Worker 4의 범위(7,500,001~10,000,000)에 포함되므로 Worker 4에서 처리됩니다. 단, Worker 4의 `JpaPagingItemReader`가 이미 `id=8,000,000`을 넘어선 페이지를 처리 중이었다면 이 주문은 처리되지 않을 수 있습니다 — 이것이 배치 처리 중 실시간 데이터 변경을 피해야 하는 이유입니다.
>
> **Q2.** `TaskExecutorPartitionHandler`에서 `ThreadPoolTaskExecutor`의 `corePoolSize`와 큐 설정으로 제한합니다. `corePoolSize=8, queueCapacity=0`으로 설정하면 최대 8개 쓰레드만 동시 실행되고 나머지는 대기합니다. 단, `queueCapacity=0`이면 쓰레드 풀이 가득 찼을 때 `TaskRejectedException`이 발생할 수 있으므로 `queueCapacity`를 적절히 설정해야 합니다. 또는 `gridSize`를 파일 수와 무관하게 실제 동시 실행 수로 제한하고, Partitioner에서 여러 파일을 하나의 파티션에 묶는 방법도 있습니다.
>
> **Q3.** `StepExecutionSplitter.split()`에서 `jobRepository.getLastStepExecution(jobInstance, "workerStep:date-2024-01-01")`을 조회합니다. 반환된 StepExecution의 `status == FAILED`이므로 `shouldStart()` = true, 해당 Worker를 재실행합니다. Worker 1의 이전 `ExecutionContext`(`BATCH_STEP_EXECUTION_CONTEXT`에 저장된)가 새 StepExecution에 복사돼, 마지막 커밋 위치(예: read.count=50000)부터 재개합니다.

---

<div align="center">

**[⬅️ 이전: Partitioning 개념](./01-partitioning-concept.md)** | **[홈으로 🏠](../README.md)** | **[다음: Grid Size와 Thread Pool 설정 ➡️](./03-grid-size-thread-pool.md)**

</div>
