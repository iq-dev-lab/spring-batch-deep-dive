# Grid Size와 Thread Pool 설정 — 병렬 처리 성능과 안전성의 균형

---

## 🎯 핵심 질문

- `gridSize`와 실제 생성되는 Worker `StepExecution` 수의 정확한 관계는?
- `TaskExecutorPartitionHandler`의 `gridSize`와 `TaskExecutor` 쓰레드 수가 다르면 어떻게 되는가?
- DB 커넥션 풀 크기를 `gridSize`에 맞게 계산하는 공식은?
- `gridSize`를 늘려도 성능이 선형으로 증가하지 않는 이유는?
- `queueCapacity=0`과 `queueCapacity>0` 설정의 차이는?

---

## 🔍 GridSize, 쓰레드 수, 커넥션 풀의 관계

### 문제: 세 가지 설정이 맞지 않으면 교착상태 또는 성능 저하가 발생한다

```
설정 조합별 결과:

  상황 1: gridSize=8, 쓰레드=8, 커넥션 풀=10
    → 8개 Worker 동시 실행 (쓰레드 8개 모두 사용)
    → 각 Worker: 커넥션 최대 2개 (Reader 1 + Writer 1)
    → 필요 커넥션: 8 × 2 = 16개 > 풀 크기 10개
    → 커넥션 고갈 → 대기 → 타임아웃 → 배치 실패!

  상황 2: gridSize=8, 쓰레드=4, 커넥션 풀=20
    → 8개 Work는 있지만 4개만 동시 실행 (쓰레드 4개)
    → 필요 커넥션: 4 × 2 = 8개 < 풀 크기 20개 → 안전
    → But 8개 작업을 4개씩 나눠 실행 → 병렬도 제한

  이상적 설정:
    gridSize = 데이터 분할 수
    쓰레드 수 = 원하는 동시 실행 수 (≤ gridSize)
    커넥션 풀 = 쓰레드 수 × Worker당 커넥션 수 + 여유
```

---

## 😱 흔한 실수

### Before: queueCapacity=0으로 설정해 TaskRejectedException이 발생한다

```java
// ❌ queueCapacity=0 + gridSize > corePoolSize → 즉시 오류
@Bean
public TaskExecutor partitionTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(4);
    executor.setQueueCapacity(0);  // 큐 없음
    executor.initialize();
    return executor;
}

// gridSize=8 → 8개 Worker 생성 → 4개를 초과하는 순간:
// TaskRejectedException: Task rejected from ThreadPoolExecutor
// → PartitionHandler가 이를 처리하지 못하면 Manager Step FAILED

// ✅ queueCapacity를 gridSize 이상으로 설정
executor.setQueueCapacity(20);  // gridSize보다 크게 설정
// → corePoolSize(4)개 동시 실행, 나머지는 큐에서 대기
// → 순차적으로 처리됨 (완전 병렬 아님)

// 또는 maxPoolSize=gridSize로 설정
executor.setCorePoolSize(8);    // gridSize와 동일
executor.setMaxPoolSize(8);
executor.setQueueCapacity(0);   // 큐 없어도 8개 모두 즉시 실행
```

### Before: Worker당 커넥션 수를 과소 계산한다

```java
// ❌ Worker당 커넥션 1개로 계산
// Worker = Reader (커넥션 1) + Writer (커넥션 1) + JobRepository 갱신 (커넥션 1)
// 실제로는 커넥션 2~3개 필요
int poolSize = gridSize * 1;  // 잘못된 계산

// ✅ Worker당 필요 커넥션 계산:
// JpaPagingItemReader:   쿼리마다 커넥션 사용 (짧게 사용)
// JdbcBatchItemWriter:   배치 INSERT (짧게 사용)
// JobRepository 갱신:    Chunk 커밋마다 1번 (짧게 사용)
// JdbcCursorItemReader:  Step 전체 동안 1개 유지!

// 안전한 공식:
// pool_size = (동시 실행 Worker 수) × (Worker당 최대 동시 커넥션) + 여유(5~10)
// 일반적: pool_size = gridSize × 2 + 10
```

---

## ✨ GridSize와 StepExecution 수의 관계

```java
// gridSize는 두 곳에서 사용됨:

// ① Partitioner.partition(gridSize)에 전달 → EC 맵 개수 결정
// ② PartitionHandler.setGridSize(gridSize) → 힌트로만 사용

// 실제 Worker StepExecution 수 = Partitioner가 반환하는 Map 크기
// → gridSize=8 설정해도 Partitioner가 6개만 반환하면 Worker 6개

// TaskExecutorPartitionHandler:
public void setGridSize(int gridSize) {
    this.gridSize = gridSize;
    // StepExecutionSplitter에 전달되는 값
    // → Partitioner.partition(this.gridSize) 호출 시 사용
}

// 즉:
// gridSize = Partitioner에게 "몇 개로 나눠달라"는 요청
// 실제 Worker 수 = Partitioner 반환값의 entry 수
```

---

## 🔬 내부 동작 원리

### 1. ThreadPoolTaskExecutor 동작 원리

```
ThreadPoolTaskExecutor 구성:
  corePoolSize = 4
  maxPoolSize  = 8
  queueCapacity = 10

작업 제출 규칙:
  1. 현재 쓰레드 수 < corePoolSize → 새 쓰레드 생성
  2. 현재 쓰레드 수 ≥ corePoolSize → 큐에 추가
  3. 큐 가득 참 → 쓰레드 수 < maxPoolSize → 새 쓰레드 생성
  4. 쓰레드 수 = maxPoolSize, 큐 가득 참 → TaskRejectedException

Partitioning 최적 설정 (gridSize=8, 완전 병렬):
  corePoolSize = 8   (즉시 8개 쓰레드 생성)
  maxPoolSize  = 8
  queueCapacity = 0  (큐 없음, 즉시 실행)
  → 8개 Worker가 모두 동시에 시작

또는 제한된 병렬 (gridSize=8, 4개씩 처리):
  corePoolSize = 4
  maxPoolSize  = 4
  queueCapacity = 8  (나머지 4개는 큐에서 대기)
  → 4개 완료 후 다음 4개 실행
```

### 2. DB 커넥션 풀 계산

```java
// Worker당 커넥션 사용 패턴 (JpaPagingItemReader + JdbcBatchItemWriter 기준)

// 1. Read 단계: JpaPagingItemReader
//    페이지 쿼리마다 커넥션 1개 사용 → 페이지 로드 후 반환
//    (Paging은 짧게 사용)

// 2. Write 단계: JdbcBatchItemWriter
//    batchUpdate() 실행 중 커넥션 1개 사용

// 3. JobRepository 갱신: Chunk 커밋마다
//    BATCH_STEP_EXECUTION UPDATE → 커넥션 1개
//    (같은 DataSource면 트랜잭션 공유 가능)

// 실제 동시 최대 커넥션 (Paging Reader):
// 각 Chunk에서 Read + Write + JobRepository ≈ 2개 (트랜잭션 공유)
// → 동시 Worker 4개: 4 × 2 = 8개 커넥션

// JdbcCursorItemReader 사용 시 (커넥션 계속 유지):
// Reader 1개 (Step 전체 유지) + Write 1개 = 2개/Worker
// → 동시 Worker 4개: 4 × 2 = 8개 커넥션 (지속)

// 권장 커넥션 풀 크기:
int concurrentWorkers = 4;
int connectionsPerWorker = 3;  // 여유 포함
int extraConnections = 5;       // 관리 쿼리, 모니터링 등
int poolSize = concurrentWorkers * connectionsPerWorker + extraConnections;
// = 4 × 3 + 5 = 17 → 20으로 올림
```

### 3. GridSize 증가에 따른 성능 곡선

```
1,000만 건, 단일 Writer (MySQL), -Xmx2g:

┌──────────┬──────────┬─────────────┬──────────────┬─────────────────────────┐
│ GridSize │ 처리 시간  │ 선형 대비     │ 커넥션 사용     │ 비고                     │
├──────────┼──────────┼─────────────┼──────────────┼─────────────────────────┤
│ 1        │ 800초    │ 기준          │ 2개          │ 단일 쓰레드                │
│ 2        │ 430초    │ 1.86배 향상    │ 4개          │ 거의 선형                 │
│ 4        │ 230초    │ 3.48배 향상    │ 8개          │ 높은 병렬 효율             │
│ 8        │ 140초    │ 5.71배 향상    │ 16개         │ 수확 체감 시작             │
│ 16       │ 100초    │ 8배 향상       │ 32개         │ DB I/O 병목              │
│ 32       │ 95초     │ 8.42배 향상    │ 64개         │ 성능 수렴 (오버헤드 증가)    │
└──────────┴──────────┴─────────────┴──────────────┴─────────────────────────┘

성능 수렴 이유:
  - DB Write I/O 포화 (디스크 쓰기 대역폭)
  - Lock 경쟁 증가 (동일 테이블에 INSERT 경쟁)
  - GC 압력 증가 (활성 Chunk 수 증가)
  - CPU 컨텍스트 스위치 비용

권장 GridSize: DB I/O 병목 전 구간 (4~8 일반적)
```

---

## 💻 실전 구현

### 최적화된 ThreadPoolTaskExecutor 설정

```java
@Configuration
public class PartitionExecutorConfig {

    @Value("${batch.partition.grid-size:4}")
    private int gridSize;

    @Bean
    public TaskExecutor partitionTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // ① 쓰레드 수 = gridSize (완전 병렬)
        executor.setCorePoolSize(gridSize);
        executor.setMaxPoolSize(gridSize);

        // ② 큐: gridSize만큼 여유 (재시작 시 중복 제출 방지)
        executor.setQueueCapacity(gridSize);

        // ③ 쓰레드 이름 (디버깅, 로그 추적용)
        executor.setThreadNamePrefix("partition-worker-");

        // ④ 종료 시 실행 중인 작업 완료 대기
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(600);  // 최대 10분 대기

        executor.initialize();
        return executor;
    }

    // HikariCP 설정 (application.yml)
    // spring.datasource.hikari.maximum-pool-size: ${batch.partition.grid-size * 3 + 10}
}
```

### 동적 GridSize — 데이터 건수 기반 자동 결정

```java
@Bean
@JobScope
public Step dynamicManagerStep(
        @Value("#{jobParameters['targetDate']}") String targetDate) {

    // 실제 데이터 건수 기반 동적 gridSize 결정
    long dataCount = orderRepository.countByDateAndStatus(targetDate, "PENDING");
    int optimalGridSize = calculateOptimalGridSize(dataCount);

    log.info("동적 GridSize 결정: {}건 → gridSize={}", dataCount, optimalGridSize);

    TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
    handler.setStep(workerStep());
    handler.setGridSize(optimalGridSize);
    handler.setTaskExecutor(dynamicTaskExecutor(optimalGridSize));

    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", rangePartitioner())
        .partitionHandler(handler)
        .build();
}

private int calculateOptimalGridSize(long dataCount) {
    if (dataCount < 100_000) return 1;       // 10만 미만: 단일 쓰레드
    if (dataCount < 1_000_000) return 2;     // 100만 미만: 2개
    if (dataCount < 5_000_000) return 4;     // 500만 미만: 4개
    if (dataCount < 20_000_000) return 8;    // 2,000만 미만: 8개
    return 16;                                // 그 이상: 16개
}
```

---

## 📊 GridSize 결정 가이드

```
상황별 권장 GridSize:

┌─────────────────────────────────┬────────────────┬────────────────────────────────┐
│ 상황                             │ 권장 GridSize   │ 이유                            │
├─────────────────────────────────┼────────────────┼────────────────────────────────┤
│ 데이터 100만 건 이하                │ 1~2            │ 병렬화 오버헤드 > 이득              │
│ 데이터 100만~1,000만 건             │ 4              │ 효율적인 병렬화 구간               │
│ 데이터 1,000만~5,000만 건           │ 8              │ DB I/O 포화 전                  │
│ Processor가 무거운 경우 (API)       │ CPU 코어 수     │ CPU 병목                        │
│ Write 경합 많은 경우                │ 4 이하         │ Lock 경쟁 최소화                  │
│ Remote Partitioning              │ 서버 수 × 2    │ 서버 리소스 활용                   │
├─────────────────────────────────┼────────────────┼────────────────────────────────┤
│ DB 커넥션 풀 크기                  │ gridSize×3 +10 │ Worker당 최대 3개 + 여유           │
│ 쓰레드 수                         │ gridSize       │ 완전 병렬 (큐 필요 없음)            │
└─────────────────────────────────┴────────────────┴────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
GridSize 크게:
  장점  처리 속도 향상 (병렬도 증가)
  단점  DB 커넥션 많이 필요, Lock 경쟁, GC 압력
        파티션 불균등 시 특정 Worker만 오래 걸림 (전체 대기)

GridSize 작게:
  장점  리소스 절약, Lock 경쟁 최소, 안정적
  단점  처리 시간 증가

queueCapacity=0:
  장점  gridSize개 Worker가 즉시 모두 시작 (최대 병렬)
  단점  쓰레드 수 < gridSize이면 TaskRejectedException

queueCapacity>0:
  장점  TaskRejectedException 방지, 안정적
  단점  큐에 있는 Worker는 순차 대기 (완전 병렬 아님)
```

---

## 📌 핵심 정리

```
GridSize와 Worker 수의 관계
  gridSize → Partitioner.partition(gridSize)에 전달
  실제 Worker 수 = Partitioner가 반환하는 Map 크기
  두 값이 다를 수 있음 (Partitioner가 독자적으로 결정 가능)

쓰레드 수 설정 원칙
  완전 병렬: corePoolSize = maxPoolSize = gridSize, queueCapacity=0
  제한된 병렬: corePoolSize < gridSize, queueCapacity ≥ (gridSize - corePoolSize)

커넥션 풀 공식
  pool_size ≥ 동시_실행_Worker × Worker당_최대_커넥션 + 여유(5~10)
  일반적: gridSize × 3 + 10

성능 수렴
  GridSize 8~16 이후 수확 체감
  DB I/O 대역폭이 진짜 병목
  → gridSize를 늘리기보다 DB 최적화 (인덱스, 파티션 테이블)
```

---

## 🤔 생각해볼 문제

**Q1.** `gridSize=8, corePoolSize=4, queueCapacity=10`으로 설정했을 때, 8개 Worker가 어떤 순서로 실행되고 완료되는가? 총 처리 시간은 완전 병렬(corePoolSize=8)에 비해 얼마나 느린가?

**Q2.** DB 커넥션 풀을 `gridSize × 3 + 10`으로 설정했습니다. 그런데 Partitioner가 `gridSize=8`에 반해 10개의 파티션을 반환했습니다. 커넥션 풀이 부족할 수 있는가?

**Q3.** `gridSize=8`로 설정했을 때 파티션 1이 250만 건, 파티션 8이 50만 건을 처리합니다. 파티션 8이 먼저 완료돼도 파티션 1을 기다려야 합니다. `PartitionHandler`가 모든 Worker를 기다리는 대신, 완료된 Worker의 쓰레드를 재활용해 남은 파티션을 처리하는 방법은?

> 💡 **해설**
>
> **Q1.** `corePoolSize=4, queueCapacity=10`으로 8개 Worker 제출 시: Worker 1~4는 즉시 쓰레드 생성 후 실행, Worker 5~8은 큐에서 대기합니다. Worker 1이 완료되면 쓰레드가 반환되고 큐에서 Worker 5가 꺼내져 실행됩니다. 이런 식으로 순차적으로 처리됩니다. 총 처리 시간: `2 × max(각 Worker 처리 시간)` (4개씩 2라운드). 완전 병렬(corePoolSize=8)의 `1 × max(처리 시간)`에 비해 약 2배 느립니다. 정확한 차이는 파티션 균등도에 따라 달라집니다.
>
> **Q2.** 커넥션 풀 = `8 × 3 + 10 = 34`개. 실제 동시 실행 Worker는 `min(partitions, threads) = min(10, 4) = 4`개(corePoolSize=4 가정). 따라서 동시 필요 커넥션 = `4 × 3 = 12`개 < 34개로 안전합니다. 단, `corePoolSize=10`이면 10개 Worker 동시 실행 → `10 × 3 = 30`개 필요 → 풀 34개로 간신히 충족합니다. gridSize 계획 변경 시 커넥션 풀도 재계산해야 합니다.
>
> **Q3.** 현재 `TaskExecutorPartitionHandler`는 모든 Worker의 `Future.get()`을 순서대로 기다리므로, 마지막 Worker가 완료될 때까지 대기합니다. 쓰레드 재활용 전략은 기본 제공되지 않습니다. 개선 방법: (1) 파티셔닝을 더 세분화(gridSize=32)하고 corePoolSize=8로 제한해 균등 처리 확률을 높입니다. (2) `NTILE` 기반 건수 균등 분할로 Worker별 처리 시간을 비슷하게 만듭니다. (3) `ReactivePartitionHandler`(커스텀 구현)로 Worker 완료 즉시 새 파티션 처리하는 동적 분배를 구현합니다. 가장 실용적인 방법은 (2)입니다.

---

<div align="center">

**[⬅️ 이전: Partitioner 구현](./02-partitioner-implementation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Remote Partitioning ➡️](./04-remote-partitioning.md)**

</div>
