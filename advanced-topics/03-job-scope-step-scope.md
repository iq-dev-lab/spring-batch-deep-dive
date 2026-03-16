# JobScope와 StepScope — Proxy 기반 Late Binding의 동작 원리

---

## 🎯 핵심 질문

- `@JobScope`와 `@StepScope` Bean이 Spring Proxy 기반 Scope로 생성되는 원리는?
- `@Value("#{jobParameters['date']}")`이 런타임에 Late Binding되는 정확한 시점은?
- `@StepScope` 없이 `jobParameters`를 주입하려 할 때 왜 `null`이 되는가?
- `@JobScope`와 `@StepScope`의 라이프사이클 차이는 무엇인가?
- Partitioning에서 Worker가 `stepExecutionContext`를 읽는 시점은?

---

## 🔍 왜 Late Binding이 필요한가

### 문제: Bean 생성 시점과 JobParameters 존재 시점의 불일치

```
Spring Context 초기화 순서:

  1. ApplicationContext 초기화 시작
  2. @Bean 메서드 호출 → Bean 생성
  3. ApplicationContext 초기화 완료
  4. ... (나중에)
  5. JobLauncher.run(job, jobParameters) → Job 실행 시작

JobParameters는 5번 시점에 존재
@Bean은 2번 시점에 생성

문제:
  @Bean
  public ItemReader<Order> reader(
          @Value("#{jobParameters['date']}") String date) {
      // 2번 시점 호출 → jobParameters 없음 → null!
  }

해결: @StepScope
  Bean 생성을 Step 시작 시점(5번 이후)으로 지연
  → jobParameters 존재 → SpEL 정상 평가
```

---

## 😱 흔한 실수

### Before: @StepScope 없이 jobParameters를 주입받아 null을 얻는다

```java
// ❌ @StepScope 없음 → null
@Bean
// @StepScope 없음!
public JpaPagingItemReader<Order> reader(
        EntityManagerFactory emf,
        @Value("#{jobParameters['targetDate']}") String targetDate) {
    // Context 초기화 시 호출 → jobParameters 없음 → targetDate = null
    // NullPointerException 또는 잘못된 쿼리 실행

    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("WHERE date = '" + targetDate + "'")  // null!
        .build();
}

// ✅ @StepScope 추가 → Step 시작 시 Bean 생성 → jobParameters 존재
@Bean
@StepScope   // ← 이 한 줄
public JpaPagingItemReader<Order> reader(
        EntityManagerFactory emf,
        @Value("#{jobParameters['targetDate']}") String targetDate) {
    // Step 시작 시 호출 → targetDate = "2024-01-01"
    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("WHERE date = :date")
        .parameterValues(Map.of("date", LocalDate.parse(targetDate)))
        .build();
}
```

### Before: @StepScope Bean을 Step 외부에서 직접 주입받으려 한다

```java
// ❌ @StepScope Bean을 Controller에서 직접 사용
@RestController
public class BatchController {

    @Autowired
    private JpaPagingItemReader<Order> reader;
    // @StepScope Bean → Step 외부에서 주입받으면 Proxy만 반환
    // Proxy를 통해 read()를 호출하면:
    // "No StepContext currently active" → IllegalStateException

    @GetMapping("/preview")
    public List<Order> preview() {
        return reader.read();  // 예외!
    }
}
// @StepScope Bean은 Step 실행 컨텍스트 내에서만 정상 동작
```

---

## ✨ @JobScope vs @StepScope 라이프사이클

```
@JobScope:
  Bean 생성: Job 실행 시작 (JobLauncher.run() → AbstractJob.execute())
  Bean 소멸: Job 실행 종료 (Job COMPLETED/FAILED/STOPPED)
  범위:      하나의 JobExecution 내
  사용 가능: jobParameters, jobExecutionContext

@StepScope:
  Bean 생성: Step 실행 시작 (AbstractStep.execute())
  Bean 소멸: Step 실행 종료
  범위:      하나의 StepExecution 내
  사용 가능: jobParameters, jobExecutionContext, stepExecutionContext

타임라인:
  Job 시작
    └─ @JobScope Bean 생성 (jobParameters 접근 가능)
       │
       Step 1 시작
         └─ @StepScope Bean 생성 (stepExecutionContext 접근 가능)
            │ ...
         └─ @StepScope Bean 소멸
       Step 1 종료
       │
       Step 2 시작
         └─ @StepScope Bean 생성 (새 인스턴스!)
            │ ...
         └─ @StepScope Bean 소멸
       Step 2 종료
    └─ @JobScope Bean 소멸
  Job 종료
```

---

## 🔬 내부 동작 원리

### 1. @StepScope Proxy 생성 과정

```java
// @StepScope는 커스텀 Spring Scope 구현체
// → BeanFactoryPostProcessor가 @StepScope 붙은 Bean 정의를 수정

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(value = "step", proxyMode = ScopedProxyMode.TARGET_CLASS)
public @interface StepScope {
}
// proxyMode = TARGET_CLASS → CGLIB 기반 Proxy 생성

// 동작:
// 1. Context 초기화 시: CGLIB Proxy Bean 생성 (실제 Bean 아님)
// 2. Step 실행 시: StepContext 활성화
// 3. Proxy.method() 호출 시:
//    → StepSynchronizationManager.getContext()로 현재 StepContext 조회
//    → StepContext에서 실제 Bean 조회 or 생성
//    → 실제 Bean의 method() 호출
```

### 2. StepScope.get() — 실제 Bean 생성 시점

```java
// StepScope.java
public class StepScope implements BeanFactoryPostProcessor, Scope {

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        // ① 현재 활성화된 StepContext 가져오기
        StepContext context = StepSynchronizationManager.getContext();

        if (context == null) {
            throw new IllegalStateException(
                "No StepContext currently active! " +
                "@StepScope Bean은 Step 실행 컨텍스트 내에서만 사용 가능.");
        }

        // ② StepContext에서 이 Bean 인스턴스 조회
        Object scopedObject = context.getAttribute(name);

        if (scopedObject == null) {
            // ③ 없으면 새로 생성 (SpEL 평가 포함)
            scopedObject = objectFactory.getObject();
            // ← 이 시점에 @Value SpEL이 평가됨
            // ← jobParameters['date'] → StepExecution.jobParameters에서 조회
            context.setAttribute(name, scopedObject);
        }

        return scopedObject;
    }
}
```

### 3. SpEL Late Binding 경로

```java
// @Value("#{jobParameters['date']}") 평가 경로:

// 1. BeanExpressionContext 생성
//    → root object: StepScope 또는 JobScope 컨텍스트

// 2. StandardEvaluationContext에 변수 등록:
//    jobParameters → stepExecution.getJobParameters()
//    jobExecutionContext → stepExecution.getJobExecution().getExecutionContext()
//    stepExecutionContext → stepExecution.getExecutionContext()

// 3. SpEL 평가:
//    jobParameters['date'] → jobParameters.getString("date") → "2024-01-01"
//    stepExecutionContext['minId'] → EC.getLong("minId") → 1L

// 변수명 목록:
// jobParameters          → JobParameters 객체
// jobExecutionContext    → Job 수준 ExecutionContext
// stepExecutionContext   → Step 수준 ExecutionContext (Partitioning Worker)
// stepExecution          → StepExecution 객체 전체
// jobExecution           → JobExecution 객체 전체
```

### 4. Partitioning에서 stepExecutionContext 사용

```java
// Worker Step에서 stepExecutionContext에서 범위 읽기
// Partitioner가 설정한 EC가 Worker의 StepExecution.executionContext에 있음

@Bean
@StepScope
public JpaPagingItemReader<Order> workerReader(
        EntityManagerFactory emf,
        @Value("#{stepExecutionContext['minId']}") Long minId,   // Worker의 EC
        @Value("#{stepExecutionContext['maxId']}") Long maxId) { // Worker의 EC

    // AbstractStep.execute()에서:
    // StepSynchronizationManager.register(workerStepExecution)
    //   → workerStepExecution.executionContext = {minId=1, maxId=2500000}
    // @StepScope Bean 생성 시 SpEL 평가
    //   → stepExecutionContext['minId'] = 1L

    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("WHERE id BETWEEN :min AND :max")
        .parameterValues(Map.of("min", minId, "max", maxId))
        .build();
}
```

---

## 💻 실전 구현

### @JobScope와 @StepScope 적절한 사용 패턴

```java
@Configuration
public class ScopeUsageConfig {

    // @JobScope: Job 전체에서 공유되는 설정 (각 Step에서 재사용)
    @Bean
    @JobScope
    public ExchangeRateService exchangeRateService(
            @Value("#{jobParameters['currency']}") String currency) {
        // Job 시작 시 환율 서비스 초기화 (API 호출 1회)
        // Job 내 모든 Step이 이 인스턴스를 공유
        ExchangeRateService service = new ExchangeRateService();
        service.setCurrency(currency);
        service.loadRates();  // 외부 API 1회 호출
        return service;
    }

    // @StepScope: Step마다 독립 인스턴스가 필요한 Reader
    @Bean
    @StepScope
    public JdbcPagingItemReader<Order> orderReader(
            DataSource dataSource,
            @Value("#{jobParameters['targetDate']}") String targetDate,
            @Value("#{jobParameters['batchSize']:'1000'}") Integer batchSize) {
        // Step마다 새 Reader 인스턴스 생성
        // targetDate: 날짜별 쿼리 조건
        // batchSize: 기본값 1000 (파라미터 없으면 기본값 사용)
        return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderReader")
            .dataSource(dataSource)
            // ...
            .pageSize(batchSize)
            .build();
    }

    // @StepScope: 재시작 시 특정 Step의 이전 처리 결과 참조
    @Bean
    @StepScope
    public Tasklet reportTasklet(
            @Value("#{jobExecutionContext['step1.processedCount']}") Long count,
            @Value("#{stepExecutionContext['reportType']:'DAILY'}") String reportType) {
        // jobExecutionContext: 이전 Step이 저장한 값 (Step 간 데이터 전달)
        // stepExecutionContext: 이 Step의 EC (Partitioning Worker 범위 등)
        return (contribution, chunkContext) -> {
            log.info("리포트 생성: type={}, count={}", reportType, count);
            return RepeatStatus.FINISHED;
        };
    }
}
```

### 단위 테스트에서 @StepScope Bean 테스트

```java
@SpringBatchTest
@SpringBootTest
class StepScopeBeanTest {

    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;

    @Autowired
    private JobRepositoryTestUtils jobRepositoryTestUtils;

    @AfterEach
    void cleanUp() {
        jobRepositoryTestUtils.removeJobExecutions();
    }

    @Test
    void stepscopeReader_targetDate_주입_테스트() throws Exception {
        // @SpringBatchTest가 StepScope 지원을 자동 설정
        // → @StepScope Bean을 테스트 메서드 내에서 사용 가능

        JobParameters params = new JobParametersBuilder()
            .addString("targetDate", "2024-01-01")
            .toJobParameters();

        JobExecution execution = jobLauncherTestUtils.launchStep(
            "orderProcessingStep", params);

        assertThat(execution.getStatus()).isEqualTo(BatchStatus.COMPLETED);
    }

    // 또는 StepScopeTestExecutionListener로 직접 테스트
    @TestExecutionListeners(
        listeners = {StepScopeTestExecutionListener.class},
        mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS)
    static class DirectBeanTest {

        // 테스트 메서드에서 StepExecution을 반환하면 StepContext 자동 활성화
        public StepExecution getStepExecution() {
            StepExecution stepExecution = MetaDataInstanceFactory.createStepExecution();
            stepExecution.getJobExecution().getJobParameters();
            // JobParameters 설정
            return MetaDataInstanceFactory.createStepExecution(
                new JobParametersBuilder()
                    .addString("targetDate", "2024-01-01")
                    .toJobParameters());
        }
    }
}
```

---

## 📊 @Scope 선택 가이드

```
┌──────────────────────────────────┬────────────┬────────────┬──────────────────────────────┐
│ 상황                              │ @JobScope  │ @StepScope │ 이유                          │
├──────────────────────────────────┼────────────┼────────────┼──────────────────────────────┤
│ JobParameters 주입                │ ✅         │ ✅         │ 둘 다 접근 가능                 │
│ stepExecutionContext 주입         │ ❌         │ ✅         │ StepExecution 필요            │
│ Job 전체에서 공유할 서비스            │ ✅          │ ❌         │ Job 내 재사용                  │
│ Step마다 독립 Reader               │ ❌         │ ✅         │ Step마다 새 인스턴스             │
│ Partitioning Worker 범위          │ ❌         │ ✅         │ stepExecutionContext 필요     │
│ 여러 Step에서 공유 캐시              │ ✅         │ ❌         │ Job 수명 동안 유지              │
│ FlatFileItemWriter (파일 1개)     │ ❌         │ ✅         │ Step마다 파일 다를 수 있음        │
└──────────────────────────────────┴────────────┴────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
@StepScope:

  장점
    Step 실행 시 Bean 생성 → JobParameters, EC 정상 접근
    Partitioning Worker마다 독립 인스턴스 → 범위 격리
    Step 종료 시 자동 소멸 → 메모리 효율

  단점
    CGLIB Proxy 생성 → 클래스가 final이면 안 됨
    Step 외부에서 호출 시 IllegalStateException
    Spring Context 초기화 시 Proxy만 생성 → 초기화 오류 늦게 발견

@JobScope:

  장점
    Job 내 모든 Step이 같은 Bean 공유
    Job 시작 시 한 번만 초기화 (외부 API 호출 등)

  단점
    동시에 여러 Job 실행 시 Bean 인스턴스 분리됨
    (JobExecution당 1개 → 의도대로 동작)
```

---

## 📌 핵심 정리

```
@StepScope Late Binding 동작 순서

  1. Context 초기화 시: CGLIB Proxy Bean 등록 (실제 Bean 아님)
  2. Step 실행 시: StepSynchronizationManager.register(stepExecution)
  3. Proxy 메서드 호출 시: StepScope.get() 호출
  4. 실제 Bean 생성: objectFactory.getObject()
  5. SpEL 평가: jobParameters['date'] → StepExecution.jobParameters에서 조회
  6. 인스턴스 StepContext에 캐싱 → 동일 Step 내 재사용

@StepScope vs @JobScope 선택

  jobParameters, jobExecutionContext → 둘 다 사용 가능
  stepExecutionContext (Partitioning 범위) → @StepScope 필수
  Job 전체 공유 서비스/캐시 → @JobScope
  Step마다 독립 Reader/Writer → @StepScope

null이 되는 조건

  @StepScope 없이 @Value SpEL 사용
  → Context 초기화 시 SpEL 평가 → jobParameters 없음 → null
  → 해결: @StepScope 또는 @JobScope 추가
```

---

## 🤔 생각해볼 문제

**Q1.** `@StepScope` Bean이 CGLIB Proxy로 생성됩니다. `final` 클래스를 `@StepScope`로 등록하면 어떻게 되는가?

**Q2.** `@JobScope`와 `@StepScope`를 모두 붙이면 어떻게 되는가? 어떤 Scope가 적용되는가?

**Q3.** Partitioning에서 4개 Worker가 동시에 실행됩니다. 각 Worker가 동일한 `@StepScope` Bean 이름(예: `workerReader`)을 참조합니다. 각 Worker는 독립된 인스턴스를 갖는가, 아니면 공유하는가?

> 💡 **해설**
>
> **Q1.** CGLIB는 서브클래스를 생성해 Proxy를 만듭니다. `final` 클래스는 상속이 불가하므로 `Cannot subclass final class` 오류가 발생합니다. 해결: (1) `final` 수식자 제거. (2) `@StepScope` 대신 인터페이스 기반 Proxy(`proxyMode = ScopedProxyMode.INTERFACES`) 사용 — 단, 클래스가 인터페이스를 구현해야 합니다. (3) Kotlin의 경우 `open` 키워드 추가 또는 `kotlin-allopen` 플러그인 사용.
>
> **Q2.** `@JobScope`와 `@StepScope`를 모두 붙이면 마지막으로 처리된 것이 적용됩니다. Spring `@Scope`는 하나의 Bean에 하나만 적용되므로, 실제로는 `@StepScope`가 덮어씁니다(`@StepScope`가 `@JobScope`보다 더 구체적인 Scope). 하지만 이런 설정은 의도가 불명확하므로 사용하지 않는 것이 좋습니다.
>
> **Q3.** 각 Worker는 독립된 인스턴스를 갖습니다. `StepScope.get()`은 `StepSynchronizationManager.getContext()`로 현재 쓰레드의 `StepContext`를 조회합니다. 각 Worker는 별도 쓰레드에서 실행되며 각자의 `StepExecution`을 등록합니다. 따라서 쓰레드 1의 `StepContext`와 쓰레드 2의 `StepContext`는 다르며, 각자 독립된 Bean 인스턴스를 가집니다. 이것이 Partitioning에서 Worker마다 다른 `minId`, `maxId`를 주입받을 수 있는 이유입니다.

---

<div align="center">

**[⬅️ 이전: Multi-threaded Step](./02-multi-threaded-step.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Batch Integration ➡️](./04-spring-batch-integration.md)**

</div>
