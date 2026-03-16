# Spring Batch Integration — 메시지 기반 배치 처리 아키텍처

---

## 🎯 핵심 질문

- `JobLaunchingGateway`는 어떻게 메시지 수신 시 Job을 자동으로 기동하는가?
- `ChunkMessageChannelItemWriter`는 Write 단계를 어떻게 메시지 큐에 위임하는가?
- 배치 Job과 실시간 이벤트 처리를 어떻게 결합하는가?
- Spring Integration의 `MessageSource` 기반 폴링으로 배치를 자동 트리거하는 방법은?
- `JobLaunchRequest`와 `JobExecution` 메시지의 구조는?

---

## 🔍 배치와 이벤트 기반 시스템을 결합하는 이유

### 문제: 시간 기반 스케줄러의 한계

```
시간 기반 스케줄러 문제:

  Scenario A:
    매일 자정에 배치 실행
    근데 입력 파일이 03:00에 도착한다면?
    → 00:00에 실행 → 파일 없음 → 실패
    → 03:00 이후 수동 재실행 필요

  Scenario B:
    카드사 정산 데이터가 FTP로 도착할 때마다 처리
    도착 시간: 불규칙 (14:22, 17:45, 22:10...)
    → 1분마다 폴링하면 대부분 불필요한 실행
    → 실시간 처리 불가

이벤트 기반 배치 트리거:

  파일 도착 이벤트 → 메시지 발행 → JobLaunchingGateway → Job 즉시 기동
  FTP 다운로드 완료 → RabbitMQ 메시지 → 배치 자동 시작
  REST API 요청 → 메시지 큐 → 비동기 배치 기동 + 상태 추적

장점:
  입력 준비 즉시 처리 (지연 없음)
  불필요한 폴링 제거
  이벤트 소스와 배치 처리 간 결합도 낮춤
```

---

## 😱 흔한 실수

### Before: JobLaunchingGateway에서 동시 실행 중복 처리를 방지하지 않는다

```java
// ❌ 같은 파라미터로 메시지가 중복 수신 시 중복 실행
@Bean
public IntegrationFlow fileArrivalFlow() {
    return IntegrationFlow
        .from(fileMessageSource(), spec -> spec.poller(Pollers.fixedDelay(5000)))
        .transform(fileToJobLaunchRequest())
        .handle(jobLaunchingGateway())  // 중복 메시지 → 중복 JobExecution
        .get();
}
// 같은 파일 이름으로 두 개의 메시지가 오면 같은 JobParameters → JobInstanceAlreadyCompleteException

// ✅ 타임스탬프로 고유성 보장 + 중복 메시지 필터
@Bean
public IntegrationFlow fileArrivalFlow() {
    return IntegrationFlow
        .from(fileMessageSource(), spec -> spec.poller(Pollers.fixedDelay(5000)))
        .filter(fileIdempotentFilter())    // 이미 처리된 파일 필터링
        .transform(fileToJobLaunchRequest())
        .handle(jobLaunchingGateway())
        .get();
}
```

---

## ✨ Spring Batch Integration 주요 컴포넌트

```
주요 컴포넌트:

  JobLaunchingGateway
    → 메시지 채널에서 JobLaunchRequest 수신
    → JobLauncher.run() 호출
    → JobExecution을 응답 메시지로 반환

  JobLaunchRequest
    → Job + JobParameters를 담은 메시지 페이로드
    → MessageChannel을 통해 전달

  ChunkMessageChannelItemWriter
    → Write 단계를 메시지 채널로 위임
    → 비동기 Write 처리 가능
    → Remote Partitioning Worker Writer로 활용

  JobExecutingMessageHandler
    → 메시지 수신 시 Job Step 실행
    → Remote Partitioning에서 Worker 역할
```

---

## 🔬 내부 동작 원리

### 1. JobLaunchingGateway — 메시지 → Job 기동

```java
// JobLaunchingGateway.java
public class JobLaunchingGateway extends AbstractReplyProducingMessageHandler {

    private final JobLauncher jobLauncher;

    @Override
    protected Object handleRequestMessage(Message<?> requestMessage) {
        // ① 메시지 페이로드에서 JobLaunchRequest 추출
        JobLaunchRequest jobLaunchRequest = (JobLaunchRequest) requestMessage.getPayload();
        Job job = jobLaunchRequest.getJob();
        JobParameters jobParameters = jobLaunchRequest.getJobParameters();

        JobExecution jobExecution;
        try {
            // ② JobLauncher로 Job 기동 (동기 또는 비동기)
            jobExecution = jobLauncher.run(job, jobParameters);
        } catch (JobExecutionAlreadyRunningException
                | JobRestartException
                | JobInstanceAlreadyCompleteException
                | JobParametersInvalidException e) {
            throw new MessageHandlingException(requestMessage,
                "Job 실행 실패: " + e.getMessage(), e);
        }

        // ③ JobExecution을 응답 메시지로 반환
        return jobExecution;
    }
}

// JobLaunchRequest 구조
public class JobLaunchRequest {
    private final Job job;
    private final JobParameters jobParameters;

    public JobLaunchRequest(Job job, JobParameters jobParameters) {
        this.job = job;
        this.jobParameters = jobParameters;
    }
}
```

### 2. 파일 도착 이벤트 → 배치 자동 기동

```java
@Configuration
@EnableIntegration
public class FileBasedBatchTriggerConfig {

    @Autowired
    private Job orderSettlementJob;

    // ① 파일 감시 MessageSource
    @Bean
    public MessageSource<File> fileMessageSource() {
        FileReadingMessageSource source = new FileReadingMessageSource();
        source.setDirectory(new File("/data/incoming"));
        source.setFilter(new AcceptOnceFileListFilter<>());  // 이미 처리된 파일 제외
        source.setScanEachPoll(true);
        return source;
    }

    // ② 파일 도착 → JobLaunchRequest 변환
    @Bean
    @Transformer
    public GenericTransformer<File, JobLaunchRequest> fileToJobLaunchRequest() {
        return file -> {
            JobParameters params = new JobParametersBuilder()
                .addString("inputFile", file.getAbsolutePath())
                .addString("fileName", file.getName())
                .addLong("fileTimestamp", file.lastModified())
                .toJobParameters();
            return new JobLaunchRequest(orderSettlementJob, params);
        };
    }

    // ③ JobLaunchingGateway
    @Bean
    public JobLaunchingGateway jobLaunchingGateway() {
        // 비동기 Launcher 사용 → 메시지 처리 스레드 블로킹 방지
        TaskExecutorJobLauncher asyncLauncher = new TaskExecutorJobLauncher();
        asyncLauncher.setJobRepository(jobRepository);
        asyncLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor());

        return new JobLaunchingGateway(asyncLauncher);
    }

    // ④ 전체 Integration Flow
    @Bean
    public IntegrationFlow fileBasedBatchFlow() {
        return IntegrationFlow
            .from(fileMessageSource(),
                spec -> spec.poller(Pollers.fixedDelay(30_000)))  // 30초마다 폴링
            .transform(fileToJobLaunchRequest())
            .handle(jobLaunchingGateway(),
                spec -> spec.advice(retryAdvice()))              // 실패 시 재시도
            .channel("jobResultChannel")                          // 결과 채널
            .get();
    }

    // ⑤ Job 실행 결과 처리 (알림, 로깅)
    @Bean
    public IntegrationFlow jobResultFlow() {
        return IntegrationFlow
            .from("jobResultChannel")
            .handle(message -> {
                JobExecution execution = (JobExecution) message.getPayload();
                if (execution.getStatus() == BatchStatus.FAILED) {
                    alertService.sendAlert("배치 실패: " + execution.getJobInstance().getJobName());
                }
                log.info("배치 완료: status={}", execution.getStatus());
            })
            .get();
    }
}
```

### 3. 메시지 기반 배치 아키텍처 전체 흐름

```
FTP 서버                    메시지 큐               배치 서버
   │                         │                       │
   │ 파일 업로드                │                       │
   │─────────────────────────▶                       │
   │                    RabbitMQ                     │
   │                    "file.arrived"               │
   │                    {filePath, timestamp}        │
   │                         │                       │
   │                         │ MessageSource 폴링     │
   │                         │◀──────────────────────┤
   │                         │                       │
   │                         │ 파일 도착 메시지          │
   │                         │──────────────────────▶│
   │                         │                       │ JobLaunchingGateway
   │                         │                       │ → JobLauncher.run()
   │                         │                       │ → BatchJob 실행 시작
   │                         │                       │   ↓
   │                         │                       │ Read/Process/Write
   │                         │                       │   ↓
   │                         │                       │ Job COMPLETED
   │                         │                       │
   │                         │ Job 완료 알림           │
   │                         │◀──────────────────────┤
   │                         │                       │
   │                    "job.completed"              │
   │                    {jobName, status, count}     │
   │                         │                       │
   모니터링 대시보드 ◀───────────┤                        │
```

### 4. REST API 요청 → 비동기 배치 → 상태 조회

```java
@Configuration
public class RestTriggeredBatchConfig {

    // HTTP 요청을 채널에 넣으면 Integration이 처리
    @Bean
    public IntegrationFlow httpInboundFlow() {
        return IntegrationFlow
            .from(Http.inboundGateway("/api/batch/trigger")
                .requestMapping(m -> m.methods(HttpMethod.POST))
                .requestPayloadType(BatchTriggerRequest.class))
            .transform(requestToJobLaunchRequest())    // 요청 → JobLaunchRequest 변환
            .handle(asyncJobLaunchingGateway())        // 비동기 Job 기동
            .transform(execution ->                    // 즉시 응답 (비동기 결과 추적용 ID)
                Map.of("executionId", ((JobExecution) execution).getId(),
                       "status", "STARTED"))
            .get();
    }
}
```

---

## 💻 실전 구현

### Kafka 이벤트로 배치 트리거 + 결과 이벤트 발행

```java
@Configuration
@EnableIntegration
public class KafkaTriggeredBatchConfig {

    @Autowired
    private Job orderProcessingJob;

    // Kafka에서 주문 완료 이벤트 수신 → 배치 트리거
    @Bean
    public IntegrationFlow kafkaBatchTriggerFlow(
            ConsumerFactory<String, OrderCompletedEvent> consumerFactory) {

        return IntegrationFlow
            .from(Kafka.messageDrivenChannelAdapter(
                consumerFactory,
                KafkaMessageDrivenChannelAdapter.ListenerMode.record,
                "order-completed-events"))
            .aggregate(aggSpec -> aggSpec
                .correlationStrategy(msg ->
                    ((OrderCompletedEvent) msg.getPayload()).getBatchDate())
                .releaseStrategy(group -> group.size() >= 1000 ||
                    Duration.between(group.getTimestamp(), Instant.now()).toMinutes() >= 5)
                // 1000건 모이거나 5분 경과 시 배치 기동
            )
            .transform(this::aggregatedEventsToJobLaunchRequest)
            .handle(jobLaunchingGateway())
            .handle(execution -> {
                // 배치 완료 후 결과 이벤트 Kafka에 발행
                publishBatchCompletedEvent((JobExecution) execution);
            })
            .get();
    }

    private JobLaunchRequest aggregatedEventsToJobLaunchRequest(
            List<OrderCompletedEvent> events) {
        String batchDate = events.get(0).getBatchDate();
        JobParameters params = new JobParametersBuilder()
            .addString("batchDate", batchDate)
            .addLong("eventCount", (long) events.size())
            .addLong("triggeredAt", System.currentTimeMillis())
            .toJobParameters();
        return new JobLaunchRequest(orderProcessingJob, params);
    }

    private void publishBatchCompletedEvent(JobExecution execution) {
        kafkaTemplate.send("batch-completed-events",
            new BatchCompletedEvent(
                execution.getJobInstance().getJobName(),
                execution.getStatus().toString(),
                execution.getWriteCount()
            ));
    }
}
```

---

## 📊 배치 트리거 방식 비교

```
┌────────────────────────────┬─────────────────────────┬──────────────────────────────────┐
│ 트리거 방식                   │ 적합한 경우                │ 주의사항                           │
├────────────────────────────┼─────────────────────────┼──────────────────────────────────┤
│ @Scheduled (시간 기반)       │ 고정 스케줄, 단순 배치       │ 입력 준비 여부와 무관 실행 위험         │
│ JobLaunchingGateway (이벤트)│ 파일/이벤트 도착 시          │ 메시지 중복 처리 방지 필요             │
│ REST API + 비동기 Launcher  │ 수동 트리거, UI 제어         │ 상태 폴링 API 별도 구현 필요          │
│ MessageSource 폴링         │ 조건 충족 시 자동 트리거       │ 폴링 주기 조정 필요                  │
│ Kafka/RabbitMQ 이벤트       │ 마이크로서비스 연동           │ 이벤트 순서, 멱등성 보장 필요          │
└────────────────────────────┴─────────────────────────┴──────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Spring Batch Integration:

  장점
    배치와 이벤트 시스템의 느슨한 결합
    파일/이벤트 준비 즉시 처리 (지연 최소화)
    메시지 큐의 내구성(durability)으로 트리거 손실 방지
    비동기 Job 기동 + 결과 이벤트 발행으로 반응형 아키텍처

  단점
    메시지 큐 인프라 필요 (복잡도 증가)
    메시지 중복 처리 방지 로직 필요 (IdempotentFilter 등)
    메시지 손실 시 배치 미실행 (Dead Letter Queue 처리 필요)
    디버깅 복잡 (메시지 추적, 배치 상태 추적 분리)

언제 사용:
  이미 메시지 큐 인프라가 있는 경우
  배치 입력이 이벤트 기반인 경우 (파일 도착, API 이벤트)
  마이크로서비스 환경에서 배치와 실시간 서비스 연동
  → 단순 스케줄 배치라면 @Scheduled가 더 단순
```

---

## 📌 핵심 정리

```
Spring Batch Integration 핵심 흐름

  이벤트 발생 (파일 도착, Kafka 메시지)
    → MessageSource 또는 Channel Adapter가 수신
    → Transform: 이벤트 → JobLaunchRequest (Job + JobParameters)
    → JobLaunchingGateway: JobLauncher.run() 호출
    → JobExecution 응답 → 결과 이벤트 발행 또는 알림

주요 컴포넌트

  JobLaunchingGateway  → 메시지 수신 → Job 기동 → JobExecution 반환
  JobLaunchRequest     → 메시지 페이로드 (Job + JobParameters)
  IntegrationFlow      → Source → Transform → Handler 파이프라인

중복 실행 방지

  AcceptOnceFileListFilter: 파일 기반 중복 제거
  IdempotentReceiverInterceptor: 메시지 ID 기반 중복 제거
  identifying=false 파라미터: JobInstance 식별 제외 파라미터로 타임스탬프 추가

배치 + 이벤트 결합 패턴

  이벤트 집계 → 임계값(건수/시간) 도달 시 배치 기동
  배치 완료 → 결과 이벤트 발행 → 다음 처리 시스템에 전달
```

---

## 🤔 생각해볼 문제

**Q1.** `JobLaunchingGateway`에 비동기 `JobLauncher`를 설정했습니다. 메시지 처리 쓰레드는 Job이 완료되기 전에 `JobExecution(status=STARTING)`을 응답 채널로 전송합니다. 응답 채널 핸들러가 최종 상태(COMPLETED/FAILED)를 알려면 어떻게 해야 하는가?

**Q2.** 파일 기반 Integration Flow에서 `AcceptOnceFileListFilter`를 사용합니다. 애플리케이션이 재시작되면 이 필터의 인메모리 상태가 초기화됩니다. 이미 처리된 파일이 다시 처리될 위험은 어떻게 방지하는가?

**Q3.** Kafka 이벤트 기반 배치에서 1000개 이벤트를 집계해 하나의 배치를 기동합니다. 집계 중 배치 서버가 재시작되면 집계 상태가 손실됩니다. 이를 어떻게 처리해야 하는가?

> 💡 **해설**
>
> **Q1.** 비동기 실행 시 즉시 반환된 `JobExecution`의 `getId()`를 클라이언트에게 전달합니다. 이후 (1) 폴링: 주기적으로 `JobExplorer.getJobExecution(id)`로 상태를 조회합니다. (2) 이벤트 기반: `JobExecutionListener.afterJob()`에서 완료 이벤트를 메시지 채널(Kafka 등)에 발행하고 구독자가 수신합니다. (3) WebSocket/SSE: 배치 서버가 완료 시 클라이언트에 Push합니다. 가장 일반적인 방법은 (2)로, 결과 채널에 `JobExecution` 최종 상태를 발행하는 `JobExecutionListener`를 등록합니다.
>
> **Q2.** `AcceptOnceFileListFilter`는 인메모리이므로 재시작 시 초기화됩니다. 영구적 필터 구현: (1) `AbstractPersistentAcceptOnceFileListFilter`를 상속해 처리된 파일 정보를 Redis나 DB에 저장합니다. (2) `FileSystemPersistentAcceptOnceFileListFilter`를 사용해 파일 시스템에 메타데이터를 저장합니다. (3) 처리된 파일을 별도 아카이브 디렉토리로 이동시켜 재처리 방지합니다. (4) 파일 이름을 DB 테이블에 기록하고 IntegrationFlow에서 처리 여부를 조회합니다.
>
> **Q3.** Kafka Aggregator의 인메모리 집계 상태 손실 처리: (1) Aggregator에 `MessageGroupStore`를 Redis나 DB로 교체해 집계 상태를 영구 저장합니다(`RedisMessageGroupStore` 사용). (2) Kafka Consumer Group 오프셋을 커밋하지 않아 재시작 시 미처리 메시지를 다시 수신하게 합니다 (At-least-once 처리). (3) 이벤트를 DB에 먼저 저장하고, 배치 기동 시 DB에서 집계하는 방식으로 전환합니다. 가장 실용적인 방법은 (2)와 (3)의 조합입니다 — Kafka 오프셋을 배치 기동 성공 후에만 커밋하고, 배치 기동 전 이벤트는 별도 DB 큐에 저장합니다.

---

<div align="center">

**[⬅️ 이전: JobScope와 StepScope](./03-job-scope-step-scope.md)** | **[홈으로 🏠](../README.md)**

*"Spring Batch를 쓰는 것과, Chunk 트랜잭션이 왜 그 경계에서 커밋되는지 아는 것은 다르다"*

</div>
