<div align="center">

# ⚙️ Spring Batch Deep Dive

**"수백만 건 데이터를 안정적으로 처리하는 메커니즘"**

<br/>

> *"Spring Batch를 쓰는 것과, Chunk 트랜잭션이 왜 그 경계에서 커밋되는지 아는 것은 다르다"*

JobLauncher.run() 한 줄씩 분석부터 ChunkOrientedTasklet → ItemReader → ItemProcessor → ItemWriter 전체 체인,  
실패한 Job이 어떻게 정확히 그 지점부터 재시작되는지, Partitioning으로 데이터를 병렬 분산하는 메커니즘까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring Batch 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring Batch](https://img.shields.io/badge/Spring_Batch-5.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-batch/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-35개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring Batch에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@EnableBatchProcessing`을 붙이면 배치가 동작합니다" | `BatchAutoConfiguration`이 `JobRepository`, `JobLauncher`, `JobOperator`를 어떻게 초기화하는지, `DataSource`가 없을 때 `Map` 기반으로 폴백되는 이유 |
| "`chunkSize(100)`으로 Chunk 크기를 설정합니다" | `ChunkOrientedTasklet.execute()`에서 Read 루프가 돌고, Chunk가 찼을 때 트랜잭션이 커밋되는 정확한 시점, OOM 없이 대용량을 처리할 수 있는 이유 |
| "`skip(Exception.class)`으로 오류를 건너뜁니다" | `FaultTolerantChunkProcessor`가 Skip된 아이템을 개별 트랜잭션으로 재처리하는 과정, Skip이 Chunk 전체를 롤백하지 않는 이유 |
| "실패한 Job을 다시 실행하면 됩니다" | `JobRepository`의 `BATCH_JOB_EXECUTION` / `BATCH_STEP_EXECUTION` 메타데이터가 어떻게 재시작 지점을 결정하는가, `ExecutionContext`에 저장된 상태로 정확히 어디서부터 재개되는가 |
| "`Partitioner`로 병렬 처리합니다" | `PartitionStep`이 Manager Step에서 Worker Step의 `StepExecution`을 생성하고 `TaskExecutor`로 분산하는 내부 흐름, Grid Size와 실제 쓰레드 수의 관계 |
| 이론 나열 | 실행 가능한 Job 코드 + Spring Batch 소스코드 직접 추적 + 100만 건 성능 측정 + JobRepository 메타데이터 분석 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Arch](https://img.shields.io/badge/🔹_Batch_Architecture-Job·Step·Tasklet_관계와_구조-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./batch-architecture/01-job-step-tasklet-structure.md)
[![Chunk](https://img.shields.io/badge/🔹_Chunk_Processing-Chunk_처리_원리_(Read→Process→Write)-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./chunk-processing/01-chunk-oriented-processing.md)
[![Flow](https://img.shields.io/badge/🔹_Job_Flow_Control-Sequential_&_Conditional_Flow-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./job-flow-control/01-sequential-flow.md)
[![Error](https://img.shields.io/badge/🔹_Error_Recovery-Skip·Retry·Restart_전략-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./error-recovery/01-skip-strategy.md)
[![Partition](https://img.shields.io/badge/🔹_Partitioning-Manager_Step_&_Worker_Steps-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./partitioning-parallel/01-partitioning-concept.md)
[![Advanced](https://img.shields.io/badge/🔹_Advanced_Topics-Async_ItemProcessor_&_Multi--threaded_Step-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./advanced-topics/01-async-item-processor.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Batch Architecture

> **핵심 질문:** `JobLauncher.run()`을 호출했을 때, `Job` → `Step` → `Tasklet`은 정확히 어떤 순서로 실행되고, 실행 기록은 어디에 어떻게 남는가?

<details>
<summary><b>Job 초기화부터 메타데이터 관리까지 완전 분해 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Job·Step·Tasklet 관계와 구조](./batch-architecture/01-job-step-tasklet-structure.md) | `Job`이 여러 `Step`을 순서대로 실행하는 구조, `Tasklet` vs `ChunkOrientedTasklet` 선택 기준, 단순 파일 이동 작업과 대용량 처리 작업을 분리하는 이유 |
| [02. JobLauncher와 Job 실행 메커니즘](./batch-architecture/02-job-launcher-execution.md) | `SimpleJobLauncher.run()`이 `JobRepository`를 통해 `JobExecution`을 생성하고 `SimpleJob.execute()`를 호출하는 전 과정, 동기/비동기 `TaskExecutor` 선택에 따른 차이 |
| [03. JobRepository와 메타데이터 관리](./batch-architecture/03-job-repository-metadata.md) | `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION`, `BATCH_JOB_EXECUTION_CONTEXT` 6개 테이블의 관계, `JobRepository`가 상태를 저장하는 정확한 시점 |
| [04. JobInstance vs JobExecution vs StepExecution](./batch-architecture/04-job-instance-execution.md) | 같은 `Job`을 매일 실행할 때 `JobInstance`는 재사용되지 않는 이유, 실패한 `JobExecution`을 재시작하면 새 `JobExecution`이 생성되는 메커니즘, `COMPLETED` 상태인 `JobInstance`를 강제 재실행하는 방법 |
| [05. JobParameters와 실행 컨텍스트](./batch-architecture/05-job-parameters-context.md) | `JobParameters`가 `JobInstance`의 식별자 역할을 하는 원리, `ExecutionContext`의 Job-scope vs Step-scope 분리, `JobParametersIncrementer`로 매 실행마다 고유 파라미터를 생성하는 전략 |
| [06. @EnableBatchProcessing과 Batch 자동 구성](./batch-architecture/06-enable-batch-processing.md) | `@EnableBatchProcessing`이 등록하는 Bean 목록, Spring Boot 3.x에서 `@EnableBatchProcessing` 없이도 자동 구성되는 방식, `DataSource` 유무에 따른 `JobRepository` 구현체 선택 |

</details>

<br/>

### 🔹 Chapter 2: Chunk-Oriented Processing

> **핵심 질문:** `ChunkOrientedTasklet`은 Read → Process → Write를 어떻게 반복하며, Chunk 크기가 트랜잭션과 성능에 어떤 영향을 미치는가?

<details>
<summary><b>ItemReader·ItemProcessor·ItemWriter 내부 동작부터 Chunk Size 튜닝까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Chunk 처리 원리 (Read → Process → Write)](./chunk-processing/01-chunk-oriented-processing.md) | `ChunkOrientedTasklet.execute()`의 Read 루프 → `Chunk<I>` 누적 → `process()` → `write()` → 트랜잭션 커밋 전 과정, `ItemReader.read()`가 `null`을 반환할 때 루프가 종료되는 메커니즘 |
| [02. ItemReader 종류와 구현](./chunk-processing/02-item-reader-types.md) | `JpaPagingItemReader`와 `JdbcCursorItemReader`의 내부 차이(페이징 쿼리 재실행 vs 커서 유지), `FlatFileItemReader`의 라인 파싱 전략, Thread-safe 여부와 `SynchronizedItemStreamReader` 래핑 |
| [03. ItemProcessor 체인과 조건부 처리](./chunk-processing/03-item-processor-chain.md) | `CompositeItemProcessor`가 여러 Processor를 체인으로 연결하는 방식, Processor가 `null`을 반환하면 해당 아이템이 Write에서 제외되는 필터링 패턴, 조건부 변환 vs 조건부 Skip |
| [04. ItemWriter 구현 (JPA, JDBC, File)](./chunk-processing/04-item-writer-implementations.md) | `JpaItemWriter`가 `entityManager.merge()`를 Chunk 단위로 일괄 처리하는 방식, `JdbcBatchItemWriter`의 `batchUpdate()` 내부, `FlatFileItemWriter`의 `append` 모드와 재시작 안전성 |
| [05. Chunk Size 튜닝 (10 vs 100 vs 1000)](./chunk-processing/05-chunk-size-tuning.md) | Chunk Size가 트랜잭션 커밋 빈도·메모리 사용량·JDBC 배치 효율에 미치는 영향, 100만 건 기준 Chunk Size 10/100/1000/5000 처리 시간 실측 비교, OOM 발생 임계값 |
| [06. Cursor vs Paging 선택 기준](./chunk-processing/06-cursor-vs-paging.md) | DB 커넥션 점유 시간, 대용량에서 페이징 쿼리의 `OFFSET` 성능 저하 문제, MySQL `JdbcCursorItemReader` 스트리밍을 위한 `fetchSize` 설정, 멀티 쓰레드 환경에서 Paging이 더 안전한 이유 |
| [07. Custom ItemReader·ItemWriter 작성](./chunk-processing/07-custom-reader-writer.md) | `ItemStream` 인터페이스의 `open·update·close`를 구현해 재시작 가능한 Reader 만들기, `ExecutionContext`에 읽기 위치를 저장하는 패턴, Kafka Consumer 기반 Custom Reader 실전 예시 |

</details>

<br/>

### 🔹 Chapter 3: Job Execution & Flow Control

> **핵심 질문:** Step 실행 결과에 따라 분기하거나, 여러 Step을 병렬로 실행하려면 어떤 메커니즘이 동작하는가?

<details>
<summary><b>Sequential Flow부터 Parallel Split까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Sequential Flow (Step1 → Step2 → Step3)](./job-flow-control/01-sequential-flow.md) | `SimpleJob`이 `StepHandler`를 통해 Step을 순서대로 실행하는 내부 흐름, 앞 Step이 `FAILED`일 때 다음 Step으로 넘어가지 않는 기본 동작, `allowStartIfComplete(true)` 사용 시점 |
| [02. Conditional Flow (.on().to().from())](./job-flow-control/02-conditional-flow.md) | `FlowJob`의 전이 규칙(Transition)이 `ExitStatus` 와일드카드 패턴을 매칭하는 방식, `on("FAILED").to(errorStep)`처럼 실패 분기를 구성하는 전략, `UNKNOWN`·`NOOP` 상태 처리 |
| [03. JobExecutionDecider로 동적 분기](./job-flow-control/03-job-execution-decider.md) | `JobExecutionDecider.decide()`가 반환하는 `FlowExecutionStatus`로 분기를 동적으로 결정하는 패턴, 비즈니스 조건(처리 건수, 파일 존재 여부)에 따라 Step을 건너뛰는 전략 |
| [04. Parallel Steps (Split & Fork)](./job-flow-control/04-parallel-steps-split.md) | `FlowBuilder.split(TaskExecutor)`로 여러 Flow를 병렬 실행하는 방법, 각 Branch의 `ExitStatus`를 `FlowExecutionAggregator`가 집계하는 방식, 병렬 실행 중 한 Branch 실패 시 전체 Job 상태 |
| [05. Flow 외부화와 재사용](./job-flow-control/05-flow-externalization.md) | `Flow` Bean을 독립적으로 정의하고 여러 Job에서 재사용하는 패턴, `FlowStep`으로 Sub-flow를 하나의 Step처럼 취급하는 방법, 공통 에러 처리 Flow를 분리하는 실전 전략 |
| [06. Job·Step Listener 활용](./job-flow-control/06-job-step-listener.md) | `JobExecutionListener.beforeJob·afterJob`과 `StepExecutionListener.beforeStep·afterStep` 호출 시점, `@BeforeJob` 어노테이션 기반 등록 방식, Listener에서 `ExecutionContext`를 통해 Step 간 데이터를 전달하는 패턴 |

</details>

<br/>

### 🔹 Chapter 4: Error Handling & Recovery

> **핵심 질문:** 배치 처리 중 예외가 발생했을 때 Skip·Retry·Restart 중 무엇을 선택해야 하며, 각각 내부에서 어떻게 동작하는가?

<details>
<summary><b>Skip·Retry 전략부터 재시작 가능한 Job 설계까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Skip 전략 (SkipPolicy와 SkipLimit)](./error-recovery/01-skip-strategy.md) | `FaultTolerantChunkProcessor`가 Skip 발생 시 Chunk를 개별 아이템 단위로 재처리하는 "스캐터-개더" 패턴, `skipLimit`에 도달하면 `SkipLimitExceededException`으로 Step 실패 처리, Custom `SkipPolicy` 구현 |
| [02. Retry 전략 (RetryPolicy와 RetryTemplate)](./error-recovery/02-retry-strategy.md) | `RetryTemplate`이 `RetryPolicy`·`BackOffPolicy`·`RetryCallback`을 조합해 재시도하는 메커니즘, `SimpleRetryPolicy` vs `ExceptionClassifierRetryPolicy`, 지수 백오프(ExponentialBackOffPolicy)가 필요한 이유 |
| [03. Skip vs Retry 선택 기준](./error-recovery/03-skip-vs-retry.md) | 네트워크 오류(일시적) → Retry, 데이터 오염(영구적) → Skip으로 분류하는 기준, 두 전략을 동시에 적용할 때 실행 순서(Retry 먼저 → 소진 시 Skip), `noSkip·noRetry`로 특정 예외를 보호하는 방법 |
| [04. Job 재시작과 RestartabilityJobRestartException](./error-recovery/04-job-restart.md) | `JobRepository`가 `FAILED` 상태의 `JobExecution`을 찾아 재사용하는 방식, `restartable(false)` 설정 시 `JobRestartException` 발생 시점, 재시작 시 이미 `COMPLETED`된 Step을 건너뛰는 조건 |
| [05. ExecutionContext를 통한 상태 저장](./error-recovery/05-execution-context-state.md) | `ItemStream.update(ExecutionContext)`가 호출되는 시점(Chunk 커밋 직전), `ExecutionContext`가 `BATCH_STEP_EXECUTION_CONTEXT`에 직렬화되는 방식, 재시작 시 `open(ExecutionContext)`으로 이전 상태를 복원하는 패턴 |
| [06. Fault Tolerance 설정 완전 가이드](./error-recovery/06-fault-tolerance-config.md) | `faultTolerant()` 활성화 이후 `ChunkOrientedTasklet`이 `FaultTolerantChunkProvider`·`FaultTolerantChunkProcessor`로 교체되는 과정, `noRollback(Exception.class)`로 롤백 없이 처리하는 시나리오 |
| [07. Custom Skip·Retry Policy 구현](./error-recovery/07-custom-policy.md) | `SkipPolicy.shouldSkip(Throwable, long)`의 반환값이 Skip 여부를 결정하는 방법, 비즈니스 규칙(특정 필드 오염 여부)에 따라 Skip을 결정하는 Custom Policy 구현, 에러 로그와 알림을 함께 처리하는 `ItemSkipListener` 연동 |

</details>

<br/>

### 🔹 Chapter 5: Partitioning & Parallel Processing

> **핵심 질문:** 1,000만 건 데이터를 여러 쓰레드로 안전하게 분할해 처리하려면 Partitioning 내부에서 어떤 일이 일어나는가?

<details>
<summary><b>Partitioner 구현부터 Remote Partitioning까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Partitioning 개념 (Manager Step·Worker Steps)](./partitioning-parallel/01-partitioning-concept.md) | `PartitionStep`(Manager)이 `Partitioner.partition(gridSize)`을 호출해 Worker별 `ExecutionContext`를 생성하고, `PartitionHandler`가 Worker Step들을 분배·실행하는 전 과정 |
| [02. Partitioner 구현 (데이터 분할 전략)](./partitioning-parallel/02-partitioner-implementation.md) | ID 범위 기반(`RangePartitioner`), 파일 목록 기반, 날짜 기반 등 분할 전략 구현, 각 Worker의 `ExecutionContext`에 담기는 정보(`minId·maxId`)를 Worker Step의 `ItemReader`가 읽는 패턴 |
| [03. Grid Size와 Thread Pool 설정](./partitioning-parallel/03-grid-size-thread-pool.md) | `gridSize`와 실제 생성되는 Worker `StepExecution` 수의 관계, `TaskExecutorPartitionHandler`의 `TaskExecutor` 설정, 쓰레드 수 > Grid Size일 때와 작을 때의 차이, DB 커넥션 풀 고갈 방지 전략 |
| [04. Remote Partitioning (Master-Worker 분산)](./partitioning-parallel/04-remote-partitioning.md) | `MessageChannelPartitionHandler`가 메시지 큐(RabbitMQ·Kafka)를 통해 Worker에게 `StepExecutionRequest`를 전달하는 아키텍처, Worker 프로세스가 응답을 반환하는 방식, 네트워크 장애 시 재시도 전략 |
| [05. Partitioning 성능 튜닝](./partitioning-parallel/05-partitioning-performance.md) | 1,000만 건 기준 Grid Size 4/8/16/32 처리 시간 실측 비교, Worker별 `ItemReader` Paging 쿼리 최적화, `StepExecutionSplitter`의 `allowStartIfComplete` 설정이 재시작에 미치는 영향 |

</details>

<br/>

### 🔹 Chapter 6: Advanced Topics

> **핵심 질문:** Multi-threaded Step·Async ItemProcessor·JobScope는 어떻게 동작하며, 언제 선택해야 하는가?

<details>
<summary><b>비동기 처리부터 Spring Integration 연동까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Async ItemProcessor·ItemWriter](./advanced-topics/01-async-item-processor.md) | `AsyncItemProcessor`가 `Future<O>`를 반환해 Process 단계를 비동기로 오프로드하는 방식, `AsyncItemWriter`가 `Future`들을 모아 unwrap해 Write하는 패턴, Partitioning과의 차이점 |
| [02. Multi-threaded Step](./advanced-topics/02-multi-threaded-step.md) | `TaskExecutor`를 설정한 Step에서 여러 쓰레드가 동일한 `ItemReader`를 공유할 때 발생하는 Thread-safety 문제, `SynchronizedItemStreamReader`로 해결하는 방법, Multi-threaded Step vs Partitioning 선택 기준 |
| [03. JobScope와 StepScope](./advanced-topics/03-job-scope-step-scope.md) | `@JobScope`·`@StepScope` Bean이 Spring의 Proxy 기반 Scope로 생성되는 방식, `@Value("#{jobParameters['date']}")`이 런타임에 Late-binding되는 메커니즘, `@StepScope` 없이 `JobParameters`를 주입하려 할 때 NPE 발생 원인 |
| [04. Spring Batch Integration (메시지 기반 처리)](./advanced-topics/04-spring-batch-integration.md) | `JobLaunchingGateway`로 메시지 수신 시 Job을 자동 기동하는 패턴, `ChunkMessageChannelItemWriter`로 Write 단계를 메시지 큐에 위임하는 아키텍처, 배치와 이벤트 기반 시스템을 결합하는 전략 |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 "Spring Batch 면접 질문에 막힌다" — Chunk 처리 핵심 흐름 파악 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  Job·Step·Tasklet 관계와 구조
Day 2  Ch1-03  JobRepository와 메타데이터 관리
Day 3  Ch2-01  Chunk 처리 원리 (Read → Process → Write) ← 핵심
Day 4  Ch2-05  Chunk Size 튜닝
Day 5  Ch4-01  Skip 전략
Day 6  Ch4-02  Retry 전략
Day 7  Ch4-04  Job 재시작과 Restartability
```

</details>

<details>
<summary><b>🔵 "대용량 배치를 안정적으로 운영해야 한다" — 실무 오류 복구 전략 (1주)</b></summary>

<br/>

```
Day 1  Ch2-06  Cursor vs Paging 선택 기준
Day 2  Ch4-03  Skip vs Retry 선택 기준
Day 3  Ch4-05  ExecutionContext를 통한 상태 저장
Day 4  Ch4-06  Fault Tolerance 설정 완전 가이드
Day 5  Ch4-07  Custom Skip·Retry Policy 구현
Day 6  Ch3-02  Conditional Flow
Day 7  Ch3-06  Job·Step Listener 활용
```

</details>

<details>
<summary><b>🔴 "수천만 건 병렬 처리 아키텍처를 설계해야 한다" — Partitioning 완전 정복 (1주)</b></summary>

<br/>

```
Day 1  Ch5-01  Partitioning 개념 (Manager Step·Worker Steps)
Day 2  Ch5-02  Partitioner 구현 (데이터 분할 전략)
Day 3  Ch5-03  Grid Size와 Thread Pool 설정
Day 4  Ch6-01  Async ItemProcessor·ItemWriter
Day 5  Ch6-02  Multi-threaded Step
Day 6  Ch5-04  Remote Partitioning
Day 7  Ch5-05  Partitioning 성능 튜닝
```

</details>

<details>
<summary><b>⚫ "Spring Batch 소스코드를 직접 읽고 내부를 완전히 이해하고 싶다" — 전체 정복 (6주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — Batch Architecture 완전 분해
        → JobRepository DB 스키마를 직접 확인하며 실행 기록 추적

2주차  Chapter 2 전체 — Chunk-Oriented Processing 내부
        → ChunkOrientedTasklet.execute()에 브레이크포인트를 걸고 Read 루프 직접 확인

3주차  Chapter 3 전체 — Job Flow Control
        → FlowJob 전이 규칙을 다양한 ExitStatus 조합으로 실험

4주차  Chapter 4 전체 — Error Handling & Recovery
        → FaultTolerantChunkProcessor 소스로 Skip 스캐터-개더 패턴 직접 읽기

5주차  Chapter 5 전체 — Partitioning & Parallel Processing
        → 100만 건 데이터로 Grid Size별 처리 시간 직접 측정

6주차  Chapter 6 전체 — Advanced Topics
        → @StepScope Late-binding을 디버거로 Proxy 생성 시점 확인
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 패턴이 필요한가** | 문제 상황과 설계 배경 |
| 😱 **흔한 실수** | Before — 비효율적이거나 잘못된 배치 처리 방식 |
| ✨ **올바른 패턴** | After — 원리를 알고 난 후의 최적화된 접근 |
| 🔬 **내부 동작 원리** | Spring Batch 소스코드 직접 추적 + ASCII 구조도 |
| 💻 **실전 구현** | 실행 가능한 Job 코드 + JobRepository 메타데이터 확인 |
| 📊 **성능 비교** | 처리 시간, 메모리 사용량, TPS 측정 결과 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 방법을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🔬 핵심 분석 대상 — ChunkOrientedTasklet

이 레포의 모든 챕터는 아래 실행 흐름을 완전히 이해하는 것을 목표로 합니다.

```java
// JobLauncher → Job → Step → Tasklet 호출 체인
JobLauncher.run(job, jobParameters)
  → SimpleJob.execute(jobExecution)          // Ch1-02
    → SimpleStepHandler.handleStep()
      → AbstractStep.execute(stepExecution)  // Ch1-04
        → ChunkOrientedTasklet.execute()     // Ch2-01 ← 핵심

// ChunkOrientedTasklet.execute() 내부
while (true) {
    // ① Ch2-02: ItemReader가 null을 반환할 때까지 반복
    Chunk<I> inputs = chunkProvider.provide(contribution);
    if (inputs.isEnd()) break;

    // ② Ch2-03: ItemProcessor로 변환 (null 반환 시 필터링)
    // ③ Ch2-04: ItemWriter로 Chunk 단위 일괄 Write
    chunkProcessor.process(contribution, inputs);

    // ④ 트랜잭션 커밋 (Chunk 경계)
    // ⑤ Ch1-03: JobRepository에 StepExecution 상태 업데이트
}

// 실패 시 재시작 (Ch4-04, Ch4-05)
// BATCH_STEP_EXECUTION_CONTEXT에서 ExecutionContext 복원
// → ItemStream.open(executionContext)으로 이전 읽기 위치부터 재개
```

---

## 🧪 실험 환경

모든 성능 측정은 다음 환경에서 진행됩니다.

| 항목 | 설정 |
|------|------|
| DB | MySQL 8.0 (로컬), H2 (단위 테스트) |
| 데이터 규모 | 기본 100만 건, 대용량 실험 1,000만 건 |
| JVM | `-Xmx512m` 고정 (OOM 발생 조건 확인용) |
| Chunk Size | 10 / 100 / 1,000 / 5,000 비교 |
| Grid Size | 4 / 8 / 16 / 32 비교 (Partitioning 챕터) |
| 측정 지표 | 처리 시간(ms), 힙 사용량(MB), TPS |

---

## 🔗 선행 학습 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | IoC 컨테이너, DI, Bean 생명주기, Proxy | Ch1(Batch 컨텍스트 초기화), Ch6(@JobScope Proxy 메커니즘) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | JPA, 트랜잭션 관리, `@Transactional` 내부 | Ch2(Chunk 트랜잭션 경계), Ch4(Skip 시 롤백 범위) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration | Ch1(`BatchAutoConfiguration` 자동 구성) |

> 💡 **Transaction 개념은 필수입니다.** Chunk가 왜 그 경계에서 커밋되는지, Skip 시 왜 Chunk 전체가 롤백되지 않는지 이해하려면 `spring-data-transaction`의 트랜잭션 전파 챕터를 먼저 읽는 것을 강력히 권장합니다.

---

## 🙏 Reference

- [Spring Batch Reference Documentation](https://docs.spring.io/spring-batch/docs/current/reference/html/)
- [Spring Batch Source Code (GitHub)](https://github.com/spring-projects/spring-batch)
- [Spring Batch 5.x Migration Guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-5.0-Migration-Guide)
- [Baeldung Spring Batch Guides](https://www.baeldung.com/spring-batch)
- [Spring Batch — Job Repository Schema](https://docs.spring.io/spring-batch/docs/current/reference/html/schema-appendix.html)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"Spring Batch를 쓰는 것과, Chunk 트랜잭션이 왜 그 경계에서 커밋되는지 아는 것은 다르다"*

</div>
