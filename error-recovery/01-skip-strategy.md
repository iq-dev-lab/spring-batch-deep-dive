# Skip 전략 — FaultTolerantChunkProcessor의 스캐터-개더 패턴

---

## 🎯 핵심 질문

- `FaultTolerantChunkProcessor`가 Write 실패 시 Chunk를 어떻게 1건씩 재처리하는가?
- `skipLimit`에 도달했을 때 `SkipLimitExceededException`은 어느 시점에 발생하는가?
- Read/Process/Write 각 단계에서 Skip이 발생할 때 처리 흐름이 어떻게 다른가?
- Skip된 아이템은 `BATCH_STEP_EXECUTION`의 어떤 컬럼에 기록되는가?
- `noSkip(Exception.class)`로 보호된 예외가 발생하면 어떻게 되는가?

---

## 🔍 왜 스캐터-개더 패턴이 필요한가

### 문제: Write 실패 시 어떤 아이템이 문제인지 알 수 없다

```
Write 실패 시나리오:

  Chunk 1000건을 ItemWriter.write()에 전달
  → DB INSERT 중 500번째 아이템에서 ConstraintViolationException 발생
  → JDBC는 배치 단위로 예외를 던짐
  → 어느 아이템이 문제인지 알 수 없음

  단순 접근: Chunk 전체를 FAILED로 처리
  → 1건 문제로 999건도 롤백 → 재처리 비효율

  스캐터-개더 해결책:
    1. Chunk 전체 롤백 (정상)
    2. Chunk를 1건씩 분해해 각각 별도 트랜잭션으로 재처리
    3. 각 아이템 재처리 중 예외 → skipPolicy 확인
       - skip 가능: skipCount++, 해당 아이템만 건너뜀
       - skip 불가: 예외 전파 → Step FAILED
    4. 문제 아이템만 Skip, 나머지 999건은 정상 처리
```

---

## 😱 흔한 실수

### Before: faultTolerant() 없이 skip()을 설정한다

```java
// ❌ faultTolerant() 없이 skip() — 작동 안 함
@Bean
public Step processStep() {
    return stepBuilderFactory.get("processStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .writer(writer())
        .skip(DataIntegrityViolationException.class)  // ← 무시됨!
        .skipLimit(10)
        .build();
}
// skip()은 faultTolerant() 이후에만 효과 있음
// faultTolerant() 없이는 일반 ChunkProcessor → Skip 동작 없음

// ✅ faultTolerant() 먼저
return stepBuilderFactory.get("processStep")
    .<Order, SettledOrder>chunk(1000)
    .reader(reader())
    .writer(writer())
    .faultTolerant()
        .skip(DataIntegrityViolationException.class)
        .skipLimit(10)
    .build();
```

### Before: skipLimit을 너무 낮게 설정해 배치가 조기 종료된다

```java
// ❌ skipLimit=1이면 첫 Skip 발생 즉시 SkipLimitExceededException
.faultTolerant()
    .skip(Exception.class)
    .skipLimit(1)   // 1건만 허용 → 두 번째 오류에서 Step FAILED

// 실무에서는 데이터 품질 기준으로 설정:
.skipLimit(100)    // 100건까지는 Skip 허용 (전체의 0.01%라면 허용)
// 또는 비율 기반 커스텀 SkipPolicy 사용 (07 문서 참조)
```

---

## ✨ Skip 설정 완전 구문

```java
@Bean
public Step faultTolerantStep() {
    return stepBuilderFactory.get("faultTolerantStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .faultTolerant()
            // Skip 허용 예외
            .skip(DataIntegrityViolationException.class)
            .skip(ParseException.class)
            // Skip 금지 예외 (이 예외는 절대 Skip 안 함 → Step FAILED)
            .noSkip(CriticalBusinessException.class)
            .skipLimit(50)                    // 총 50건까지 Skip 허용
            // 또는 커스텀 정책
            // .skipPolicy(new CustomSkipPolicy())
            .listener(skipListener())          // Skip 발생 시 알림
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. FaultTolerantChunkProcessor — Write 실패 시 스캐터-개더

```java
// FaultTolerantChunkProcessor.java (단순화)
protected void write(StepContribution contribution,
                      Chunk<I> inputs,
                      Chunk<O> outputs) throws Exception {
    try {
        // ① 일반 Write 시도 (1000건 배치)
        doWrite(outputs.getItems());
    } catch (Exception e) {
        // ② Write 실패 → 스캐터-개더 모드 진입
        if (shouldSkip(skipPolicy, e, contribution.getStepSkipCount())) {
            // 예외 자체가 Skip 가능하고 Chunk 전체가 1건이면 즉시 Skip
        } else {
            // ③ Chunk를 1건씩 재처리
            scan(contribution, inputs, outputs, chunk);
        }
    }
}

private void scan(StepContribution contribution,
                   Chunk<I> inputs,
                   Chunk<O> outputs,
                   ChunkContext chunkContext) throws Exception {

    Iterator<O> outputIterator = outputs.iterator();

    while (outputIterator.hasNext()) {
        O item = outputIterator.next();

        try {
            // ④ 아이템 1건씩 별도 트랜잭션으로 Write 시도
            doWrite(Collections.singletonList(item));
        } catch (Exception e) {
            // ⑤ 개별 아이템 Write 실패
            if (shouldSkip(skipPolicy, e, contribution.getStepSkipCount())) {
                // Skip 가능 → 해당 아이템 제외
                outputIterator.remove();
                contribution.incrementWriteSkipCount();
                // ItemSkipListener.onSkipInWrite(item, e) 호출
                listener.onSkipInWrite(item, e);
                log.debug("Skip in write: item=" + item, e);
            } else {
                // Skip 불가 → 예외 전파 → Step FAILED
                throw e;
            }
        }
    }
}
```

### 2. Read/Process 단계 Skip 처리

```java
// FaultTolerantChunkProvider — Read 단계 Skip
protected I read(StepContribution contribution, Chunk<I> chunk) throws Exception {
    while (true) {
        try {
            return doRead();
        } catch (Exception e) {
            if (shouldSkip(skipPolicy, e, contribution.getStepSkipCount())) {
                // Read Skip → 해당 아이템 없는 것으로 처리
                contribution.incrementReadSkipCount();
                listener.onSkipInRead(e);
                // 다음 아이템 Read 시도 (루프 계속)
                continue;
            }
            throw e;
        }
    }
}

// SimpleChunkProcessor — Process 단계 Skip
protected Chunk<O> transform(StepContribution contribution, Chunk<I> inputs) {
    for (ChunkIterator<I> iterator = inputs.iterator(); iterator.hasNext(); ) {
        I item = iterator.next();
        try {
            O output = doProcess(item);
            if (output != null) outputs.add(output);
        } catch (Exception e) {
            if (shouldSkip(skipPolicy, e, contribution.getStepSkipCount())) {
                // Process Skip → inputs에서 제거, outputs에 추가 안 함
                iterator.remove(e);
                contribution.incrementProcessSkipCount();
                listener.onSkipInProcess(item, e);
            } else {
                throw e;
            }
        }
    }
    return outputs;
}
```

### 3. skipLimit 초과 처리

```java
// SkipPolicy.shouldSkip() 에서 limit 초과 확인
public class LimitCheckingItemSkipPolicy implements SkipPolicy {

    private final int skipLimit;
    private final Map<Class<? extends Throwable>, Boolean> skippableExceptions;

    @Override
    public boolean shouldSkip(Throwable t, long skipCount) throws SkipLimitExceededException {
        if (isSkippable(t)) {
            if (skipCount < skipLimit) {
                return true;  // Skip 허용
            } else {
                // skipLimit 도달 → SkipLimitExceededException 발생
                // → Step FAILED (Skip이 아님!)
                throw new SkipLimitExceededException(skipLimit, t);
            }
        }
        return false;  // Skip 불가 예외 → 원래 예외 전파
    }
}
```

### 4. Skip 단계별 카운터와 DB 컬럼

```
Skip 발생 단계별 BATCH_STEP_EXECUTION 컬럼:

  Read  Skip: READ_SKIP_COUNT  컬럼 증가
  Process Skip: PROCESS_SKIP_COUNT 컬럼 증가
  Write Skip: WRITE_SKIP_COUNT 컬럼 증가

  전체 Skip 수 = READ_SKIP_COUNT + PROCESS_SKIP_COUNT + WRITE_SKIP_COUNT
  skipLimit은 이 합산값을 기준으로 체크

  BATCH_STEP_EXECUTION 예시:
  READ_COUNT=1000, WRITE_COUNT=990, FILTER_COUNT=5,
  READ_SKIP_COUNT=2, PROCESS_SKIP_COUNT=1, WRITE_SKIP_COUNT=2
  → 총 Skip: 5건
```

### 5. 스캐터-개더 실행 타임라인

```
Chunk 1000건 Write 실패 (500번 아이템 ConstraintViolation):

  ① Chunk 전체 롤백 (1000건 모두)

  ② 스캐터-개더 모드:
     아이템 1번: 별도 트랜잭션 → INSERT 성공 → COMMIT
     아이템 2번: 별도 트랜잭션 → INSERT 성공 → COMMIT
     ...
     아이템 500번: 별도 트랜잭션 → INSERT 실패
       → shouldSkip() = true → writeSkipCount++ → ROLLBACK (1건만)
       → onSkipInWrite(item500, e) 호출
     아이템 501번: 별도 트랜잭션 → INSERT 성공 → COMMIT
     ...
     아이템 1000번: 별도 트랜잭션 → INSERT 성공 → COMMIT

  ③ 결과:
     writeCount += 999
     writeSkipCount += 1
     skipLimit 체크: 1 < 50 → 계속 처리

  ④ 원래 Chunk는 정상 완료로 처리됨
```

---

## 💻 실전 구현

### 데이터 무결성 위반 Skip + 로그 기록

```java
@Configuration
public class FaultTolerantJobConfig {

    @Bean
    public Step orderSettlementStep() {
        return stepBuilderFactory.get("orderSettlementStep")
            .<Order, SettledOrder>chunk(1000)
            .reader(orderReader())
            .processor(settlementProcessor())
            .writer(settlementWriter())
            .faultTolerant()
                // 데이터 오류: Skip 허용
                .skip(DataIntegrityViolationException.class)
                .skip(ParseException.class)
                .skip(NumberFormatException.class)
                // 비즈니스 예외: Skip 금지
                .noSkip(InsufficientBalanceException.class)
                .noSkip(AccountFrozenException.class)
                .skipLimit(100)
                .listener(skipAuditListener())
            .build();
    }

    @Bean
    public SkipListener<Order, SettledOrder> skipAuditListener() {
        return new ItemListenerSupport<Order, SettledOrder>() {

            @Override
            public void onSkipInRead(Throwable t) {
                log.error("[Read Skip] 원인: {}", t.getMessage());
                // 파일 오류 등 Read Skip은 아이템 정보 없음
            }

            @Override
            public void onSkipInProcess(Order item, Throwable t) {
                log.warn("[Process Skip] orderId={}, 원인: {}",
                    item.getId(), t.getMessage());
                // 오류 아이템을 별도 테이블에 기록
                skipAuditRepository.save(new SkipAudit(
                    "PROCESS", item.getId(), t.getMessage()));
            }

            @Override
            public void onSkipInWrite(Object item, Throwable t) {
                SettledOrder order = (SettledOrder) item;
                log.warn("[Write Skip] orderId={}, 원인: {}",
                    order.getOrderId(), t.getMessage());
                skipAuditRepository.save(new SkipAudit(
                    "WRITE", order.getOrderId(), t.getMessage()));
                // 알림: 5건 이상 Skip 발생 시 경고
                if (skipCount.incrementAndGet() >= 5) {
                    alertService.sendSkipWarning(skipCount.get());
                }
            }
        };
    }
}
```

---

## 📊 Skip 단계별 특성 비교

```
┌──────────┬───────────────────────────────┬───────────────────────────────────────┐
│ 단계      │ Skip 시 동작                    │ 특이사항                                │
├──────────┼───────────────────────────────┼───────────────────────────────────────┤
│ Read     │ 해당 아이템 없는 것으로 처리         │ item 정보 없음 (아직 읽지 못함)            │
│          │ Read 루프 계속                  │ READ_SKIP_COUNT 증가                   │
├──────────┼───────────────────────────────┼───────────────────────────────────────┤
│ Process  │ inputs에서 제거                 │ item은 있음 (processor 입력)             │
│          │ outputs에 추가 안 함             │ PROCESS_SKIP_COUNT 증가                │
├──────────┼───────────────────────────────┼───────────────────────────────────────┤
│ Write    │ Chunk 전체 롤백                 │ item은 있음 (writer 입력)                │
│          │ → 스캐터-개더 1건씩 재처리         │ WRITE_SKIP_COUNT 증가                   │
│          │ 문제 아이템만 롤백                │ 가장 비용 큰 Skip (재처리 오버헤드)          │
└──────────┴───────────────────────────────┴───────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Skip 전략:

  장점
    일부 불량 데이터로 전체 배치 실패 방지
    문제 데이터만 격리, 나머지는 정상 처리
    ItemSkipListener로 Skip 건 추적 및 후처리

  단점
    Write Skip 시 스캐터-개더 → 1건씩 별도 트랜잭션
    → Chunk 1000건 중 1건 Skip = 최대 1000번 별도 커밋
    → 성능 저하 (Skip이 빈번하면 매우 느려짐)

skipLimit 설정 전략:
  너무 낮음 → 조기 Step 실패
  너무 높음 → 대규모 데이터 오류를 조용히 무시
  → 비율 기반 Custom SkipPolicy 권장 (07 문서)
```

---

## 📌 핵심 정리

```
faultTolerant() 필수
  skip() / retry()는 faultTolerant() 이후에만 동작
  → ChunkOrientedTasklet이 FaultTolerantChunkProcessor로 교체됨

스캐터-개더 (Write Skip 시)
  Chunk 전체 롤백 → 1건씩 별도 트랜잭션 재처리
  문제 아이템: shouldSkip()=true → writeSkipCount++, 해당 1건만 롤백
  나머지: 정상 COMMIT

skipLimit 동작
  Read+Process+Write Skip 합산 기준
  skipLimit 초과 시 → SkipLimitExceededException → Step FAILED (Skip 아님)

noSkip()
  지정 예외 발생 시 shouldSkip()이 무조건 false 반환
  → 예외 전파 → Step FAILED
  → 치명적 비즈니스 예외 보호에 사용
```

---

## 🤔 생각해볼 문제

**Q1.** Chunk 1000건 Write 중 예외 발생 → 스캐터-개더 시작. 1건씩 재처리 중 10번째 아이템에서 `noSkip()`으로 보호된 예외가 발생하면 어떻게 되는가? 이미 성공한 1~9번 아이템은 커밋된 상태인가?

**Q2.** Read Skip이 발생했을 때 `ItemSkipListener.onSkipInRead(e)`에는 `item` 파라미터가 없습니다. Skip된 아이템의 내용을 알 수 없는데, 이를 추적할 방법이 있는가?

**Q3.** `skipLimit(50)`으로 설정된 Step이 재시작될 때, 이미 이전 실행에서 30건이 Skip됐습니다. 재시작 후 남은 허용 Skip 건수는 20건인가 50건인가?

> 💡 **해설**
>
> **Q1.** 1~9번 아이템은 각각 별도 트랜잭션으로 이미 COMMIT됐습니다. 10번째 아이템에서 `noSkip()` 예외 발생 → `shouldSkip()` = false → 예외 전파 → 스캐터-개더 루프 종료 → Step FAILED. 1~9번의 데이터는 DB에 영구 반영된 상태입니다. 이것이 스캐터-개더의 특성입니다 — 1건씩 독립 트랜잭션이므로 중간 실패 시 부분 커밋 상태가 됩니다. 재시작 시 1~9번은 중복 처리될 수 있으므로, Writer의 `ON DUPLICATE KEY UPDATE` 또는 멱등 설계가 중요합니다.
>
> **Q2.** Read Skip된 아이템은 아직 `ItemReader.read()`가 완전히 성공하지 못한 상태이므로 내용을 알기 어렵습니다. 추적 방법: (1) `ItemReader.read()` 내부에서 직접 예외 발생 직전의 데이터를 로깅합니다. (2) Custom ItemReader를 구현해 마지막으로 시도한 원본 데이터를 ThreadLocal 등에 임시 저장하고, `onSkipInRead()`에서 이를 조회합니다. (3) FlatFileItemReader라면 `LineCallbackHandler`로 Skip된 줄을 로깅할 수 있습니다.
>
> **Q3.** 재시작 후 남은 허용 Skip 건수는 **20건**입니다. `LimitCheckingItemSkipPolicy.shouldSkip(t, skipCount)` 메서드의 두 번째 파라미터 `skipCount`는 `StepExecution.getSkipCount()`를 사용합니다. 재시작 시 새 `StepExecution`이 생성되므로 `skipCount`는 0부터 시작합니다. 하지만 `skipLimit` 체크는 현재 `StepExecution`의 누적 skip count를 기준으로 합니다. 재시작 후 20건이 추가로 Skip되면 `skipCount = 20 < 50`이므로 허용됩니다. 이전 실행의 30건은 새 `StepExecution`과 무관합니다. 즉, 재시작마다 skipLimit이 리셋됩니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Retry 전략 ➡️](./02-retry-strategy.md)**

</div>
