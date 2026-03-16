# Remote Partitioning — 메시지 큐로 Worker를 다른 서버에 분산

---

## 🎯 핵심 질문

- `MessageChannelPartitionHandler`가 `StepExecutionRequest`를 어떻게 메시지 큐에 전송하는가?
- Worker 프로세스가 응답을 Manager에게 어떻게 반환하는가?
- Local Partitioning과 Remote Partitioning의 아키텍처 차이는 무엇인가?
- 네트워크 장애로 Worker 응답이 오지 않을 때 어떻게 처리되는가?
- `pollInterval`과 `timeout` 설정은 무엇을 제어하는가?

---

## 🔍 왜 Remote Partitioning이 필요한가

### 문제: 단일 서버의 CPU/메모리 한계를 넘어서야 할 때

```
Local Partitioning 한계:

  단일 서버에서 gridSize=16으로 실행:
    CPU: 16개 쓰레드가 경쟁 → CPU 포화
    메모리: 16개 Chunk 동시 메모리 → OOM 위험
    DB 커넥션: 16 × 3 = 48개 커넥션 필요
    → 서버 1대의 리소스 한계에 종속

Remote Partitioning:

  Manager 서버 1대 + Worker 서버 N대
    Manager: Partitioner 실행, StepExecution 생성, 메시지 발행
    Worker 서버들: 각자 독립 프로세스로 자신의 파티션 처리
    → Worker 서버를 추가할수록 처리 용량 선형 확장
    → 각 서버가 독립 리소스 사용

  사용 시나리오:
    - 처리 데이터 수 천만~수억 건 (서버 1대로 불충분)
    - Kubernetes Pod 기반 수평 확장
    - 이미 메시지 큐 인프라가 있는 환경
```

---

## 😱 흔한 실수

### Before: timeout을 너무 짧게 설정해 Worker가 실패로 처리된다

```java
// ❌ 짧은 timeout → 정상 실행 중인 Worker가 timeout으로 실패
@Bean
public MessageChannelPartitionHandler partitionHandler() {
    MessageChannelPartitionHandler handler = new MessageChannelPartitionHandler();
    handler.setMessagingOperations(messagingTemplate);
    handler.setReplyChannel(replyChannel());
    handler.setTimeout(30_000L);  // 30초 → Worker가 대용량 처리 중이면 timeout!
    return handler;
}
// Worker가 250만 건 처리에 5분 걸리면 → 30초 후 Manager가 timeout 처리
// → Worker는 계속 실행 중인데 Manager는 FAILED 처리
// → 데이터 중복 처리 위험 (재시작 시 Worker 결과와 Manager 상태 불일치)

// ✅ Worker 처리 시간 + 여유 시간으로 설정
// Worker 최대 처리 시간 5분 + 여유 2분 = 7분
handler.setTimeout(7 * 60 * 1000L);  // 7분
```

---

## ✨ Remote Partitioning 아키텍처

```
Manager 프로세스:
  PartitionStep
    → Partitioner.partition(gridSize) [① 파티션 생성]
    → StepExecutionSplitter.split()   [② StepExecution DB 저장]
    → MessageChannelPartitionHandler  [③ 메시지 발행]
         ↓
    [메시지 큐: RabbitMQ/Kafka]
         ↓ StepExecutionRequest 메시지
Worker 프로세스들:
    StepExecutionRequestHandler     [④ 메시지 수신]
    → Worker Step 실행              [⑤ 데이터 처리]
    → StepExecution 상태 DB 갱신   [⑥ JobRepository 공유]
    → 응답 메시지 발송              [⑦ Manager에게 완료 알림]
         ↓ StepExecution 응답
Manager 프로세스:
    MessageChannelPartitionHandler  [⑧ 응답 수집]
    → 모든 Worker 완료 대기         [⑨ 집계]
    → Manager Step 완료             [⑩ 최종 상태 결정]
```

---

## 🔬 내부 동작 원리

### 1. MessageChannelPartitionHandler — Manager 측

```java
// MessageChannelPartitionHandler.java
public class MessageChannelPartitionHandler implements PartitionHandler {

    private MessagingOperations messagingOperations;
    private PollableChannel replyChannel;
    private long timeout = -1L;          // -1: 무한 대기
    private long pollInterval = 10_000L; // 10초마다 응답 확인

    @Override
    public Collection<StepExecution> handle(
            StepExecutionSplitter stepSplitter,
            StepExecution managerStepExecution) throws Exception {

        // ① Worker StepExecution 생성
        Set<StepExecution> workerExecutions =
            stepSplitter.split(managerStepExecution, gridSize);

        // ② 각 Worker에게 StepExecutionRequest 메시지 발송
        for (StepExecution workerExecution : workerExecutions) {
            Message<StepExecutionRequest> message = MessageBuilder
                .withPayload(new StepExecutionRequest(
                    stepName,
                    workerExecution.getJobExecutionId(),
                    workerExecution.getId()))
                .setReplyChannel(replyChannel)  // 응답 채널 설정
                .build();

            messagingOperations.send(requestChannel, message);
        }

        // ③ 모든 Worker 응답 대기 (pollInterval 주기로 확인)
        return receiveReplies(workerExecutions);
    }

    private Collection<StepExecution> receiveReplies(Set<StepExecution> expected)
            throws Exception {
        long startTime = System.currentTimeMillis();
        Set<StepExecution> received = new HashSet<>();

        while (received.size() < expected.size()) {
            // replyChannel에서 응답 메시지 수신
            Message<?> message = replyChannel.receive(pollInterval);

            if (message != null) {
                // ④ Worker로부터 받은 StepExecution 응답
                StepExecution workerExecution = (StepExecution) message.getPayload();
                received.add(workerExecution);
                log.info("Worker 완료: {}/{}", received.size(), expected.size());
            }

            // ⑤ timeout 체크
            if (timeout > 0 && System.currentTimeMillis() - startTime > timeout) {
                throw new TimeoutException("Worker 응답 대기 timeout");
            }
        }

        return received;
    }
}
```

### 2. Worker 측 — StepExecutionRequestHandler

```java
// StepExecutionRequestHandler.java (Worker 프로세스)
// @ServiceActivator로 메시지 채널에서 자동 호출됨

@MessageEndpoint
public class StepExecutionRequestHandler {

    @Autowired
    private JobExplorer jobExplorer;

    @Autowired
    private Step workerStep;

    @ServiceActivator
    public StepExecution handle(StepExecutionRequest request) throws Exception {
        // ① Manager가 DB에 미리 저장한 StepExecution 조회
        Long jobExecutionId = request.getJobExecutionId();
        Long stepExecutionId = request.getStepExecutionId();

        JobExecution jobExecution = jobExplorer.getJobExecution(jobExecutionId);
        StepExecution stepExecution = jobExecution.getStepExecutions().stream()
            .filter(se -> se.getId().equals(stepExecutionId))
            .findFirst()
            .orElseThrow();

        // ② Worker Step 실행
        // stepExecution.executionContext에 minId, maxId가 담겨있음
        workerStep.execute(stepExecution);

        // ③ 완료된 StepExecution 반환 (응답 메시지로 발송됨)
        return stepExecution;
    }
}
```

### 3. 공유 JobRepository — 핵심 설계

```
Remote Partitioning이 가능한 이유:

Manager 서버 ──────┐
                  │
Worker 서버 1 ─────┼──── 공유 JobRepository DB ────
Worker 서버 2 ─────┤         (MySQL/PostgreSQL)
Worker 서버 3 ─────┘

Manager와 Worker들이 같은 DB의 BATCH_STEP_EXECUTION을 공유:

Manager가 StepExecution 생성 (STARTING):
  INSERT INTO BATCH_STEP_EXECUTION (job_execution_id=100, step_name="workerStep:partition0", status=STARTING)

Worker가 처리 후 갱신:
  UPDATE BATCH_STEP_EXECUTION SET status=COMPLETED WHERE id=?

Manager가 최종 조회:
  SELECT * FROM BATCH_STEP_EXECUTION WHERE job_execution_id=100
  → 모든 Worker StepExecution 상태 확인

→ DB가 Manager-Worker 간 상태 공유 매체
```

---

## 💻 실전 구현

### RabbitMQ 기반 Remote Partitioning

```java
// Manager 설정
@Configuration
@EnableBatchProcessing
public class ManagerConfig {

    @Bean
    public DirectChannel requestChannel() {
        return new DirectChannel();  // Manager → Worker 요청 채널
    }

    @Bean
    public QueueChannel replyChannel() {
        return new QueueChannel();   // Worker → Manager 응답 채널
    }

    @Bean
    public AmqpOutboundEndpoint amqpOutbound(AmqpTemplate amqpTemplate) {
        AmqpOutboundEndpoint endpoint = new AmqpOutboundEndpoint(amqpTemplate);
        endpoint.setExchangeName("partition-exchange");
        endpoint.setRoutingKey("partition-request");
        endpoint.setExpectReply(false);
        return endpoint;
    }

    @Bean
    public IntegrationFlow requestFlow(AmqpOutboundEndpoint amqpOutbound) {
        return IntegrationFlow.from(requestChannel())
            .handle(amqpOutbound)
            .get();
    }

    @Bean
    public MessageChannelPartitionHandler partitionHandler(
            MessagingTemplate messagingTemplate) {
        MessageChannelPartitionHandler handler = new MessageChannelPartitionHandler();
        handler.setMessagingOperations(messagingTemplate);
        handler.setReplyChannel(replyChannel());
        handler.setGridSize(4);
        handler.setTimeout(10 * 60 * 1000L);  // 10분 타임아웃
        handler.setPollInterval(5_000L);       // 5초마다 확인
        return handler;
    }

    @Bean
    public Step managerStep(Partitioner rangePartitioner,
                             PartitionHandler partitionHandler) {
        return stepBuilderFactory.get("managerStep")
            .partitioner("workerStep", rangePartitioner)
            .partitionHandler(partitionHandler)
            .build();
    }
}

// Worker 설정 (별도 프로세스)
@Configuration
@EnableBatchProcessing
public class WorkerConfig {

    @Bean
    public AmqpInboundChannelAdapter amqpInbound(
            SimpleMessageListenerContainer listenerContainer) {
        AmqpInboundChannelAdapter adapter = new AmqpInboundChannelAdapter(
            listenerContainer);
        adapter.setOutputChannelName("requestChannel");
        return adapter;
    }

    @Bean
    public IntegrationFlow replyFlow() {
        return IntegrationFlow
            .from("requestChannel")
            .handle(stepExecutionRequestHandler(), "handle")
            .channel(amqpReplyChannel())  // RabbitMQ 응답 채널로 발송
            .get();
    }

    @Bean
    public StepExecutionRequestHandler stepExecutionRequestHandler() {
        StepExecutionRequestHandler handler = new StepExecutionRequestHandler();
        handler.setJobExplorer(jobExplorer);
        handler.setStep(workerStep());
        return handler;
    }
}
```

---

## 📊 Local vs Remote Partitioning 비교

```
┌───────────────────────┬─────────────────────────────┬──────────────────────────────┐
│ 항목                   │ Local Partitioning          │ Remote Partitioning          │
├───────────────────────┼─────────────────────────────┼──────────────────────────────┤
│ 인프라                  │ 단일 서버, 쓰레드 풀            │ 여러 서버, 메시지 큐              │
│ 확장성                  │ 서버 사양 한계                 │ Worker 서버 추가로 선형 확장      │
│ 설정 복잡도              │ 낮음                         │ 높음 (MQ, 네트워크 설정)         │
│ 장애 복원               │ 쓰레드 실패 = Step 실패         │ Worker 프로세스 독립 재시작       │
│ 공유 자원               │ 같은 프로세스 내 공유            │ 공유 DB만 (네트워크 분리)         │
│ 운영 복잡도              │ 낮음                         │ 높음 (Worker 프로세스 관리)      │
│ 적합한 경우              │ 수백만 건, 단일 서버 충분        │ 수천만 건 이상, 확장 필요          │
└───────────────────────┴─────────────────────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Remote Partitioning:

  장점
    서버 리소스 선형 확장 가능
    Worker 프로세스 독립 → 단일 Worker 장애가 다른 Worker에 영향 없음
    Kubernetes Pod 자동 스케일링 연동 가능

  단점
    메시지 큐 인프라 필요 (RabbitMQ, Kafka 등)
    네트워크 장애 처리 로직 필요 (timeout, 재시도)
    Manager-Worker 간 공유 DB 필수 (DB가 SPOF)
    디버깅 복잡 (여러 프로세스에 걸친 로그 추적)

timeout 설정:
  짧게 설정: Worker 정상 실행 중에도 timeout → 거짓 실패
  길게 설정: Worker 진짜 장애 시 오래 대기
  → Worker 처리 시간의 2~3배로 설정 권장
  → Dead Letter Queue로 응답 없는 메시지 감지
```

---

## 📌 핵심 정리

```
Remote Partitioning 흐름

  Manager: Partitioner → StepExecution DB 저장 → 메시지 발행
  메시지 큐: StepExecutionRequest 전달 (jobExecutionId, stepExecutionId)
  Worker: 메시지 수신 → JobRepository에서 StepExecution 조회
          → Worker Step 실행 → StepExecution 갱신 → 응답 발송
  Manager: 모든 응답 수신 → 결과 집계

공유 JobRepository의 역할
  Manager가 StepExecution 생성 (STARTING)
  Worker가 처리 후 갱신 (STARTED → COMPLETED/FAILED)
  Manager가 최종 상태 조회 → 집계

핵심 설정값
  timeout     → Worker 최대 처리 시간 × 2~3 (거짓 실패 방지)
  pollInterval → 응답 확인 주기 (10초 권장)
  gridSize    → Partitioner에 전달되는 파티션 수 힌트
```

---

## 🤔 생각해볼 문제

**Q1.** Worker 서버 1대가 처리 도중 죽었습니다. Manager는 `timeout` 후 `TimeoutException`을 발생시킵니다. 재시작 시 죽은 Worker의 파티션은 어떻게 처리되는가?

**Q2.** 메시지 큐(RabbitMQ)에서 `StepExecutionRequest` 메시지가 손실됐습니다(Broker 장애). Manager는 응답을 기다리지만 Worker는 메시지를 받지 못했습니다. 이 상황을 어떻게 탐지하고 복구하는가?

**Q3.** Remote Partitioning에서 Manager와 Worker가 공유 JobRepository DB를 사용합니다. 이 DB가 장애나면 전체 배치가 중단됩니다. 이를 어떻게 설계 개선할 수 있는가?

> 💡 **해설**
>
> **Q1.** Manager가 `TimeoutException`으로 Manager Step FAILED → Job FAILED 처리됩니다. 죽은 Worker의 `StepExecution`은 DB에 `STARTED` 상태로 남습니다(정상 종료가 안 됐으므로 UNKNOWN으로 변경 필요). 재시작 시 `UNKNOWN` StepExecution은 재실행 불가 상태이므로, 운영자가 수동으로 해당 `BATCH_STEP_EXECUTION.STATUS = 'FAILED'`로 변경 후 Job 재시작해야 합니다. 이 StepExecution만 재실행되고 나머지 COMPLETED Worker는 건너뜁니다. 이를 자동화하려면 Heartbeat 메커니즘으로 Worker 생존을 모니터링하고 자동 FAILED 처리하는 별도 로직이 필요합니다.
>
> **Q2.** Manager는 `pollInterval`(10초)마다 `replyChannel.receive()`를 시도하고 `timeout` 이후 `TimeoutException`을 발생시킵니다. 탐지는 timeout으로 가능하지만 복구는 수동이 필요합니다. 개선 방법: (1) RabbitMQ `x-delivery-count` 헤더로 재전송 횟수 추적 — 응답 없는 메시지를 Dead Letter Queue로 라우팅. (2) Manager에서 `StepExecution`이 `STARTING` 상태로 일정 시간 유지되면 해당 파티션의 메시지를 재발행. (3) Kafka를 사용할 경우 At-least-once 전달 보장으로 메시지 손실 방지.
>
> **Q3.** JobRepository DB가 SPOF인 것은 Remote Partitioning의 구조적 한계입니다. 개선 방법: (1) DB 이중화(Primary-Replica) + 자동 장애 조치(Failover). (2) `InMemoryJobRepository`를 사용하되 Manager가 `StepExecution` 상태를 별도 상태 저장소(Redis)에도 유지. 단, 재시작 기능이 제한됩니다. (3) Choreography 방식으로 전환 — DB 중앙 집중을 피하고 각 Worker가 독립적으로 상태를 관리하며 완료 이벤트를 발행, Manager가 이를 구독하는 이벤트 기반 아키텍처. 실제로는 (1)의 DB HA 설정이 가장 현실적인 접근입니다.

---

<div align="center">

**[⬅️ 이전: Grid Size와 Thread Pool](./03-grid-size-thread-pool.md)** | **[홈으로 🏠](../README.md)** | **[다음: Partitioning 성능 튜닝 ➡️](./05-partitioning-performance.md)**

</div>
