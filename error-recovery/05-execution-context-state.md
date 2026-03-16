# ExecutionContext를 통한 상태 저장 — 재시작 포인트의 저장과 복원

---

## 🎯 핵심 질문

- `ItemStream.update(ExecutionContext)`가 정확히 언제 호출되는가?
- `ExecutionContext`가 `BATCH_STEP_EXECUTION_CONTEXT`에 직렬화되는 방식은?
- `SHORT_CONTEXT`와 `SERIALIZED_CONTEXT` 컬럼은 어떻게 구분해서 사용되는가?
- 재시작 시 `ItemStream.open(ExecutionContext)`으로 이전 위치를 복원하는 정확한 경로는?
- `saveState(false)` 설정은 어떤 영향을 미치는가?

---

## 🔍 ExecutionContext가 재시작의 핵심인 이유

### 문제: 재시작 지점을 어디에 저장하는가

```
배치 재시작의 핵심 질문:
  "500,000번째 아이템을 처리하다 실패했을 때,
   재시작 시 500,001번째부터 정확히 어떻게 시작하는가?"

단순 메모리 변수로는 불가:
  currentRow = 500000;  // 인스턴스 변수
  → 서버 재시작 시 사라짐

DB에 저장해야 함:
  BATCH_STEP_EXECUTION_CONTEXT.SHORT_CONTEXT =
    '{"orderReader.read.count": 500000}'
  → 서버 재시작 후에도 영구 보존
  → 재시작 시 이 값으로 Reader를 500,001번째로 이동

ExecutionContext = 배치의 "책갈피"
  Chunk 커밋마다 현재 읽기 위치를 DB에 저장
  재시작 시 마지막 저장 위치부터 재개
```

---

## 😱 흔한 실수

### Before: update()에서 대용량 데이터를 저장해 성능 저하를 유발한다

```java
// ❌ 처리된 모든 ID를 EC에 저장
private List<Long> processedIds = new ArrayList<>();

@Override
public void update(ExecutionContext executionContext) {
    // Chunk 커밋마다 호출됨
    // 100만 건 처리 시 processedIds 크기가 100만 개로 성장
    executionContext.put("processedIds", processedIds);
    // → 직렬화 크기: 수십 MB
    // → 매 Chunk 커밋마다 수십 MB를 DB에 UPDATE
    // → 심각한 성능 저하
}

// ✅ 최소한의 재시작 포인트만 저장
@Override
public void update(ExecutionContext executionContext) {
    // 재시작에 필요한 것: "어디까지 읽었는가" 하나만
    executionContext.putLong("lastProcessedId", lastProcessedId);
    // 직렬화: 수십 바이트 수준
    // 재시작 시 WHERE id > lastProcessedId 쿼리로 이어서 처리
}
```

### Before: 여러 Reader를 사용할 때 EC 키 충돌을 고려하지 않는다

```java
// ❌ 이름 없는 Reader (기본 이름 충돌)
@Bean
public JpaPagingItemReader<Order> reader1() {
    return new JpaPagingItemReaderBuilder<Order>()
        // .name() 없음 → 기본 이름 = 클래스명
        .build();
}

@Bean
public JpaPagingItemReader<Product> reader2() {
    return new JpaPagingItemReaderBuilder<Product>()
        // .name() 없음 → 같은 기본 이름 충돌!
        .build();
}
// EC에 "JpaPagingItemReader.read.count" 키가 두 개 저장 → 덮어씌움

// ✅ 명시적 이름으로 EC 키 분리
@Bean
public JpaPagingItemReader<Order> reader1() {
    return new JpaPagingItemReaderBuilder<Order>()
        .name("orderReader")     // EC 키: "orderReader.read.count"
        .build();
}

@Bean
public JpaPagingItemReader<Product> reader2() {
    return new JpaPagingItemReaderBuilder<Product>()
        .name("productReader")   // EC 키: "productReader.read.count"
        .build();
}
```

---

## ✨ ItemStream의 3메서드 역할

```java
// ItemStream 구현 시 완전한 재시작 지원
@Component
@StepScope
public class RestartableOrderReader implements ItemStreamReader<Order> {

    private Long lastId = 0L;
    private List<Order> buffer = new ArrayList<>();
    private static final String LAST_ID_KEY = "lastId";
    private static final String READER_NAME = "restartableOrderReader";

    // ① Step 시작 시 1회 호출 — 이전 상태 복원
    @Override
    public void open(ExecutionContext executionContext) {
        String key = READER_NAME + "." + LAST_ID_KEY;
        if (executionContext.containsKey(key)) {
            lastId = executionContext.getLong(key);
            log.info("재시작 감지: lastId={} 부터 재개", lastId);
        } else {
            log.info("신규 실행: 처음부터 시작");
        }
        loadNextBatch();
    }

    // ② Chunk 커밋마다 호출 — 현재 상태 저장
    @Override
    public void update(ExecutionContext executionContext) {
        // 재시작에 필요한 최소 정보만 저장
        executionContext.putLong(READER_NAME + "." + LAST_ID_KEY, lastId);
        // → BATCH_STEP_EXECUTION_CONTEXT에 저장
    }

    // ③ Step 종료 시 1회 호출 — 리소스 정리
    @Override
    public void close() {
        buffer.clear();
    }

    @Override
    public Order read() throws Exception {
        if (buffer.isEmpty()) {
            if (!loadNextBatch()) return null;
        }
        Order order = buffer.remove(0);
        lastId = order.getId();  // 마지막 처리 ID 추적
        return order;
    }

    private boolean loadNextBatch() {
        List<Order> batch = orderRepository.findByIdGreaterThan(lastId,
            PageRequest.of(0, 1000, Sort.by("id")));
        if (batch.isEmpty()) return false;
        buffer.addAll(batch);
        return true;
    }
}
```

---

## 🔬 내부 동작 원리

### 1. update() 호출 시점 — Chunk 커밋 시퀀스

```java
// TaskletStep.doExecute() 내 트랜잭션 템플릿
status = transactionTemplate.execute(txStatus -> {
    StepContribution contribution = stepExecution.createStepContribution();

    // ① Chunk 처리 (Read + Process + Write)
    RepeatStatus result = tasklet.execute(contribution, chunkContext);

    // ② StepExecution 카운터 업데이트
    stepExecution.apply(contribution);

    // ③ CompositeItemStream.update() 호출
    //    → 등록된 모든 ItemStream의 update() 순서대로 호출
    //    → EC에 read.count 등 저장
    stream.update(stepExecution.getExecutionContext());

    // ④ JobRepository에 EC 저장 (DB UPDATE)
    jobRepository.updateExecutionContext(stepExecution);
    // → BATCH_STEP_EXECUTION_CONTEXT.SHORT_CONTEXT 갱신

    // ⑤ StepExecution 상태 저장
    jobRepository.update(stepExecution);

    return result;
});
// ⑥ 트랜잭션 COMMIT
// ← ③과 ④가 트랜잭션 내에 있음 → EC 저장과 비즈니스 데이터 저장이 원자적
```

### 2. ExecutionContext 직렬화 — Jackson 기반

```java
// Jackson2ExecutionContextStringSerializer
// ExecutionContext → JSON 문자열 → DB 저장

// 예시 EC 내용:
// {"FlatFileItemReader.read.count": 5000,
//  "JpaPagingItemReader.read.count": 5000,
//  "lastProcessedId": 12345}

// SHORT_CONTEXT 컬럼 (VARCHAR 2500): 2500자 이내면 여기에 저장
// SERIALIZED_CONTEXT 컬럼 (TEXT/CLOB): 2500자 초과 시 여기에 저장

// 직렬화 코드 (단순화):
public void serialize(Map<String, Object> ec, OutputStream os) throws IOException {
    objectMapper.writeValue(os, ec);
    // {"@class": "java.util.HashMap", "key1": value1, ...}
    // @class 타입 정보 포함 (역직렬화 시 타입 복원용)
}
```

### 3. 재시작 시 open() 복원 흐름

```java
// AbstractStep.execute() 에서:

// ① 이전 StepExecution의 EC 로드
ExecutionContext executionContext;
if (isRestart) {
    // 이전 FAILED StepExecution의 EC 조회
    StepExecution lastStepExecution = jobRepository.getLastStepExecution(
        jobInstance, stepName);
    executionContext = lastStepExecution.getExecutionContext();
    // ← BATCH_STEP_EXECUTION_CONTEXT에서 역직렬화
} else {
    executionContext = new ExecutionContext();  // 빈 EC
}
stepExecution.setExecutionContext(executionContext);

// ② 모든 등록된 ItemStream에 open() 호출
stream.open(stepExecution.getExecutionContext());
// → JpaPagingItemReader.open(ec)
//     → ec.containsKey("orderReader.read.count") → true
//     → itemCount = ec.getInt("orderReader.read.count") → 5000
//     → jumpToItem(5000) → 5001번째 아이템부터 읽기 시작
```

### 4. jumpToItem() 구현 방식 (Reader별 차이)

```java
// JpaPagingItemReader.jumpToItem()
@Override
protected void jumpToItem(int itemIndex) throws Exception {
    // 페이지 계산: 5000 / pageSize(1000) = 5번째 페이지
    int page = itemIndex / getPageSize();
    int offset = itemIndex % getPageSize();

    // 5번째 페이지까지는 쿼리로 직접 이동 (offset=0으로 각 페이지 쿼리)
    for (int i = 0; i < page; i++) {
        doReadPage();  // 각 페이지를 읽고 버림
    }
    current = offset;  // 페이지 내 위치 설정
    // → 5001번째 아이템: 5번째 페이지(4001~5000) 다음, 0번 offset
}

// FlatFileItemReader.jumpToItem()
@Override
protected void jumpToItem(int itemIndex) throws Exception {
    // 파일을 처음부터 itemIndex번 읽고 버림
    for (int i = 0; i < itemIndex; i++) {
        String line = readLine();  // 읽고 버림
        if (line == null) break;
    }
    // → 5001번째 줄부터 실제 read() 시작
    // 비효율적 (파일 처음부터 5000줄 스캔)
}

// JdbcCursorItemReader: 커서 기반이므로 jumpToItem 없음
// 재시작 시 read.count만큼 rs.next() 반복 호출로 위치 이동
```

---

## 💻 실전 구현

### 파일 Writer의 재시작 포인트 저장

```java
// FlatFileItemWriter — 파일 바이트 위치 기반 재시작
// AbstractFileItemWriter.update()
@Override
public void update(ExecutionContext executionContext) {
    if (state != null && isSaveState()) {
        // 현재 파일 포인터 위치 (바이트)를 EC에 저장
        executionContext.putLong(
            getExecutionContextKey(RESTART_DATA_KEY),
            state.position());
        // 예: {"settlementWriter.current.count.written": 5243000}
        // → 5,243,000 바이트까지 쓴 상태
    }
}

// 재시작 시 open():
@Override
public void open(ExecutionContext executionContext) {
    if (executionContext.containsKey(getExecutionContextKey(RESTART_DATA_KEY))) {
        long filePosition = executionContext.getLong(
            getExecutionContextKey(RESTART_DATA_KEY));
        // 파일을 filePosition까지만 유지하고 이후 내용 truncate
        // → 이전에 잘못 쓴 부분 제거
        initializeBufferedWriter(resource, true, filePosition);
    }
}
```

### saveState(false) — 재시작 불필요한 대용량 처리

```java
// saveState=false 설정 시:
// → update()에서 EC에 저장하지 않음
// → 재시작 불가하지만 성능 향상
// (매 Chunk마다 EC DB UPDATE 생략)

@Bean
@StepScope
public JpaPagingItemReader<Order> nonRestartableReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Order>()
        .name("orderReader")
        .entityManagerFactory(emf)
        .queryString("SELECT o FROM Order o")
        .pageSize(5000)
        .saveState(false)   // EC 저장 비활성화
        // 사용 시나리오:
        // - 실패 시 항상 처음부터 재처리하는 것이 더 안전한 경우
        // - Multi-threaded Step (EC 동기화 문제 방지)
        // - 읽기 속도가 매우 빠르고 재처리 비용이 낮은 경우
        .build();
}
```

---

## 📊 EC 저장 크기별 영향

```
EC 저장 크기와 성능 영향 (Chunk 1000건, 100만 건 처리):

┌─────────────────────────────────┬────────────────┬──────────────────────┐
│ EC 저장 내용                      │ EC 크기         │ 커밋당 DB UPDATE 비용   │
├─────────────────────────────────┼────────────────┼──────────────────────┤
│ read.count 하나만 (권장)           │ ~50 bytes      │ 미미                  │
│ lastId + 메타데이터 몇 가지          │ ~200 bytes     │ 낮음                  │
│ 처리된 ID 목록 1000개              │ ~10 KB          │ 중간                  │
│ 처리된 ID 목록 10만개               │ ~1 MB          │ 높음                  │
│ 직렬화된 도메인 객체 목록             │ 수십 MB         │ 매우 높음 → 성능 저하     │
└─────────────────────────────────┴────────────────┴──────────────────────┘

권장: EC에는 "재시작에 꼭 필요한 최소 정보"만 저장
     처리 결과는 DB 테이블에 저장하고 EC에는 위치 정보만
```

---

## ⚖️ 트레이드오프

```
ExecutionContext 상태 저장:

  장점
    Chunk 단위 원자적 저장 (비즈니스 데이터 + EC 같은 트랜잭션)
    서버 재시작 후에도 안전하게 복원
    클러스터 환경에서도 DB 기반으로 공유 가능

  단점
    Chunk 커밋마다 EC UPDATE → 추가 DB I/O
    EC 크기가 크면 성능 저하
    복잡한 Reader의 jumpToItem() 구현 어려움 (파일 Reader의 경우)

saveState(false) 사용 기준:
  재처리 비용 < EC 저장 비용  → saveState=false
  재처리 비용 > EC 저장 비용  → saveState=true (기본값)
  Multi-threaded Step         → saveState=false 권장
```

---

## 📌 핵심 정리

```
update() 호출 시점
  Chunk 커밋 직전 (같은 트랜잭션 내)
  비즈니스 데이터 COMMIT과 EC 저장이 원자적

저장 최적화 원칙
  최소 정보만 저장: 재시작 위치 (ID, 페이지 번호, 바이트 오프셋)
  처리 결과는 비즈니스 테이블에 → EC는 위치 포인터 역할만

직렬화 규칙
  2500자 이하 → SHORT_CONTEXT (VARCHAR)
  2500자 초과 → SERIALIZED_CONTEXT (TEXT/CLOB)
  Jackson 기반, @class 타입 정보 포함

Reader 이름 필수
  .name("uniqueReaderName") 설정
  EC 키: "name.read.count"
  여러 Reader 사용 시 이름 충돌 방지 필수

saveState(false)
  EC 저장 비활성화 → 재시작 불가
  성능 향상, Multi-threaded Step 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `update(executionContext)`는 Chunk 커밋 직전에 트랜잭션 내에서 호출됩니다. 만약 `update()` 내부에서 예외가 발생하면 어떻게 되는가? 비즈니스 데이터 WRITE는 성공했는데 EC 저장이 실패하면?

**Q2.** Multi-threaded Step에서 4개 쓰레드가 동시에 같은 `StepExecution`의 `ExecutionContext`를 `update()`하면 어떤 문제가 생기는가? `saveState(true)`와 `saveState(false)` 중 어느 것이 안전한가?

**Q3.** 재시작 시 `JpaPagingItemReader.jumpToItem(500000)`이 호출되면, 내부적으로 500페이지(pageSize=1000 기준)의 쿼리를 실행합니다. 이 500번의 쿼리는 새로운 `StepExecution`의 `readCount`에 포함되는가?

> 💡 **해설**
>
> **Q1.** `update()`는 트랜잭션 내에서 호출되며, `update()` 내부 예외는 트랜잭션 롤백을 트리거합니다. 비즈니스 데이터 WRITE도 같은 트랜잭션에 있으므로 함께 롤백됩니다. 따라서 "WRITE 성공 + EC 저장 실패"는 발생하지 않습니다 — 원자적으로 처리됩니다. 단, `update()` 내부에서 예외를 catch해서 삼켜버리면 EC가 저장되지 않아도 트랜잭션이 커밋될 수 있으므로, `update()` 구현 시 예외를 잡지 않거나 `ItemStreamException`으로 감싸서 전파해야 합니다.
>
> **Q2.** `ExecutionContext`는 내부적으로 `HashMap`을 사용하며 Thread-safe하지 않습니다. 4개 쓰레드가 동시에 `put()`하면 Race Condition이 발생합니다. 또한 `jobRepository.updateExecutionContext()`가 동시에 호출되면 마지막 저장이 이전 값을 덮어씁니다. `saveState(false)` 설정이 안전합니다 — EC를 아예 저장하지 않으므로 동기화 문제가 없습니다. Multi-threaded Step에서 재시작이 필요하다면 Partitioning을 사용하는 것이 권장됩니다 — 각 Worker Step이 독립 `StepExecution`을 가지므로 EC 충돌이 없습니다.
>
> **Q3.** `jumpToItem(500000)` 중 버려지는 500번의 쿼리로 읽은 데이터는 `readCount`에 포함되지 않습니다. `jumpToItem()`은 `doRead()`가 아닌 내부 메서드(`doReadPage()`)를 직접 호출하며, `ItemReadListener.afterRead()`가 호출되지 않습니다. `contribution.incrementReadCount()`도 호출되지 않으므로 새 `StepExecution`의 `readCount`는 jumpToItem 이후부터 카운트됩니다. 다만 500번의 쿼리는 DB 부하를 발생시키므로 대용량 재시작 시 `JdbcCursorItemReader`처럼 커서 위치를 직접 이동하는 방식이 더 효율적입니다.

---

<div align="center">

**[⬅️ 이전: Job 재시작과 Restartability](./04-job-restart.md)** | **[홈으로 🏠](../README.md)** | **[다음: Fault Tolerance 설정 완전 가이드 ➡️](./06-fault-tolerance-config.md)**

</div>
