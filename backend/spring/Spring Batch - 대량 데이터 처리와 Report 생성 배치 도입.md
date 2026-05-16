# Spring Batch - 대량 데이터 처리와 Report 생성 배치 도입

매월 1일에 생성되는 `Report`에서 OOM 원인을 분석하고, 스프링 배치 도입을 고려한다.

```java
// 유저의 Report를 자동 생성
@Override
@Transactional
public void generateMonthlyReportForAllUsers() {
    YearMonth prevMonth = YearMonth.now().minusMonths(1);
    String month = prevMonth.toString();
    String thumbnailUrl = thumbnailUrlProvider.getUrlForMonth("report", month);

    List<Users> users = userRepository.findAll();
    for (Users user : users) {
        if (reportRepository.existsByUserAndMonth(user, month)) continue;

        Report report = Report.builder()
                .user(user)
                .month(month)
                .thumbnailUrl(thumbnailUrl)
                .build();

        reportRepository.save(report);
        reportTopLogService.calculateAndSaveTopLogs(user.getId(), report);

        // 커밋 이후에 외부 추천 비동기 실행 (레이스 방지)
        Long rid = report.getReportId();
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                externalRecommendMaterializer.generateAndStoreExternalAsync(rid);
            }
        });
    }
}
```

## 영속성 컨텍스트 비대화

하나의 트랜잭션으로 묶여 있는 로직 안에서 `findAll`로 유저를 한 번에 다 가지고 오고 있다.
그리고 그 안에서 `Report` 존재 여부 조회, `Report` 저장, `Report` 계산까지 하나에서 다 하고 있다.
이걸 하나의 트랜잭션에서 다 하게 되면 1차 캐시에 `Report` 엔티티가 다 누적되어 메모리를 잡아먹을 가능성이 높다.

해결 방안은 다음과 같다.

- 유저별로 별도의 트랜잭션으로 분리한다.
- 페이징을 해서 유저를 가져온다. 한 번에 `findAll`로 가져오면 모든 유저가 한 번에 메모리에 올라가게 된다.
- `afterCommit` 방식이 너무 옛날 방식이라서 listener 방식으로 구현한다. listener를 생성하고, 외부 추천은 비동기로 `Report` 저장이 제대로 되었을 때 진행한다.

그런데 Spring Batch가 이 모든 것을 해결해 준다.

```text
Job
 └─ Step: generateReportStep
       Reader: JpaPagingItemReader<Users>     (페이징 자동)
       Processor: User → Report               (existsByUserAndMonth 체크)
       Writer: chunk 단위 저장 + 이벤트 발행
```

## Spring Batch

대량 데이터를 끊어서 안전하게 처리하는 프레임워크다.

- 중간에 실패하게 되면, `JobRepository` 메타테이블에 자동 기록된다.
- 일반적으로 운영 환경에서 이 테이블을 그라파나 대시보드에 연결하고, 실패 시 알림을 전송한다.

### 기본 개념 간단 정리

`Job`: 배치 실행 단위  
`Step`: `Job`을 구성하는 단계  
`chunk`: 읽기, 처리, 저장 후 커밋하는 묶음 단위  
`reader`: 데이터를 읽는 컴포넌트  
`processor`: 읽은 데이터를 가공하거나 필터링하는 컴포넌트  
`writer`: 처리 결과를 저장하는 컴포넌트

| 테이블 | 역할 |
|---|---|
| `BATCH_JOB_INSTANCE` | "월간 Report + 2026년 4월" 같은 논리적 실행 단위 |
| `BATCH_JOB_EXECUTION` | 실제 실행 시도. 재시작하면 새 row가 추가된다. |
| `BATCH_JOB_EXECUTION_PARAMS` | `Job` 실행 시 넘긴 파라미터 |
| `BATCH_STEP_EXECUTION` | `Step`별 처리 건수, 성공, 실패 |
| `BATCH_JOB_EXECUTION_CONTEXT` | `Job` 단위 임시 저장소 |
| `BATCH_STEP_EXECUTION_CONTEXT` | `Step` 단위 임시 저장소. 어디까지 읽었는지 등을 저장한다. |

### Chunk size 튜닝 방법론

- 기본은 동기 방식으로 청크 1, 2, 3 이런 식의 단일 스레드 순차 방식이다.
- 청크는 적절히 튜닝하는 게 중요하다.
- 청크가 너무 작으면 청크마다 커밋 오버헤드가 발생하고 DB 왕복이 많아진다.
- 청크가 너무 크면 한 청크의 메모리 점유가 커지고, 실패 시 재처리 비용도 커진다.

관련 튜닝 방법들은 다음과 같다.

- `메모리`: `chunk` 안의 아이템이 차지하는 메모리 * `chunk size` * 스레드 수 < 가용 힙
- `트랜잭션 시간`: `chunk` 처리 시간이 DB 트랜잭션 타임아웃보다 짧아야 한다.
- `재처리 비용`: 실패 시 `chunk` 통째로 다시 처리하므로 너무 크면 손해다.
- `JDBC batch size`: `hibernate.jdbc.batch_size`와 `chunk size`를 맞춰야 효과가 있다. 둘이 다르면 batch insert가 안 된다.
- `외부 시스템 한도`: FCM처럼 한 호출에 `N`개 제한이 있으면 그게 상한이다.

### Reader 캐시, 청크에 모이는 것의 차이

- Reader의 내부 캐시에서는 `JpaPagingItemReader`가 효율을 위해 내부적으로 페이지 단위로 미리 가져온다.
- Reader를 호출할 때는 한 개씩 받는 것처럼 보이고, 처음 호출할 때만 한 번씩 가져오고, 필요 시 1개씩 캐시에서 꺼내서 반환한다.
- 100개까지 Reader 캐싱되도록 설정해놨다고 치면, 101번째에서는 100개씩 가져오는 쿼리를 실행한다.
- Reader 페이지 캐시는 Reader 내부 최적화 방식이고, DB 쿼리를 줄이기 위함이다.

```text
pageSize = 100 설정
  ↓
read() 첫 호출 시 → DB에 "SELECT ... LIMIT 100 OFFSET 0" 쿼리 1번
  → 결과 100개를 Reader 내부 List에 저장 (이게 캐시)
  → 그 중 1개 반환

read() 2번째 호출 → 캐시에서 꺼냄 (쿼리 안 함)
read() 3번째 호출 → 캐시에서 꺼냄
...
read() 101번째 호출 → 캐시 다 떨어짐 → "SELECT ... LIMIT 100 OFFSET 100" 쿼리
```

- 청크는 `Step`에서의 처리 단위이고, Writer가 한 번에 받는 묶음이며, 트랜잭션 단위다.
- 청크와 Reader의 `pageSize`는 다른 개념이고 보통 같게 맞추는 편이다.

### Processor

- Processor에서는 Reader가 읽어준 것을 입력으로 받아서 Writer로 넘긴다.
- 즉 유저 조회, 유저와 관련된 `Report` 저장이 있다고 하면, 유저 조회 결과를 받아서 만들지 말지를 판단한다.
- 조금 더 효율적인 조회가 필요한데, 현재 로직은 유저 조회 → 존재 여부 조회 → 생성 순서로 되어 있다. 이렇게 하면 존재 여부 조회에서 `N`번 쿼리가 실행되기 때문에, 유저를 조회할 때 존재 여부까지 같이 조회해서 중복되지 않게 `Set`을 들고 있게 한다.
- 즉 프로세서의 역할은 "만들지 말지" 결정이다. 존재 여부까지 프로세서가 실행하게 된다면 비효율적이기 때문에, Reader에서 이를 같이 처리하도록 한다.

```java
ItemProcessor<Users, Report> processor = user -> {
    // input: Reader가 읽어준 User 한 명

    // 처리 로직
    if (reportRepository.existsByUserAndMonth(user, month)) {
        return null;  // null 반환 = 이 아이템 skip
    }

    Report report = Report.builder()
            .user(user)
            .month(month)
            .thumbnailUrl(thumbnailUrl)
            .build();

    // output: Writer에 넘길 Report 한 개
    return report;
};
```

### Writer

- 청크 단위로 묶어서 동작한다.
- `JpaItemWriter`에서는 내부적으로 100개를 영속화만 한다.

```java
ItemWriter<Report> writer = chunk -> {
    // chunk = Report 100개가 담긴 묶음
    // 100개를 한 번에 저장
    for (Report report : chunk) {
        entityManager.persist(report);  // 영속화만 함, 아직 DB 안 감
    }
    entityManager.flush();  // 그제서야 한 번에 DB로
};
```

### 단일 vs 멀티 차이

단일과 멀티 방식의 차이는 `taskExecutor`를 주입한다는 것이다.
기본은 단일 스레드로, 청크를 하나씩 차례대로 처리한다.

대부분의 경우 `단일 스레드 + chunk size 튜닝`으로 충분하다.
멀티스레드는 정말 처리량이 부족할 때만 사용한다.

- `chunk`를 여러 스레드가 병렬로 처리한다.
- Reader는 thread-safe해야 한다.
- `JpaPagingItemReader`는 thread-safe가 아니므로, `SynchronizedItemStreamReader`로 감싸야 한다.

```java
// 단일 (기본)
return new StepBuilder("step", jobRepository)
        .<Users, Report>chunk(100, txManager)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .build();

// 멀티 (taskExecutor 추가만)
return new StepBuilder("step", jobRepository)
        .<Users, Report>chunk(100, txManager)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .taskExecutor(taskExecutor)     // taskExecutor를 주입받음
        .build();
```

### FaultTolerant에 대해서

종류는 다음과 같다.

- `retry`: 같은 아이템 다시 시도한다. 3번까지 같은 방식으로 일시적 네트워크 오류에 유효하다.
- `skip`: 그 아이템만 건너뛰고 계속 진행한다. 100건까지 같은 방식으로 잘못된 데이터에 유효하다.
- `fail`: 즉시 `Job` 실패다.

이걸 적용해보면 다음과 같다.
알림 전송 배치 작업 로직이다.

```java
return new StepBuilder("sendNotificationStep", jobRepository)
        .<Report, Report>chunk(100, txManager)
        .reader(reportReader)
        .writer(fcmWriter)
        .faultTolerant()
        .retry(FcmServerException.class).retryLimit(3)       // FCM 5xx → 3번 재시도
        .skip(InvalidTokenException.class).skipLimit(1000)   // 만료 토큰 → 1000건까지 skip
        .build();
```

동작은 다음과 같다.

- `chunk` 처리 중 `FcmServerException`이 발생하면 같은 `chunk`를 다시 시도한다. 최대 3번이다.
- 3번 다 실패하면 `skip` 정책을 확인한다.
- `InvalidTokenException`이 발생하면 그 아이템을 `skip`하고 카운터를 1 증가시킨다.
- `skip` 카운터가 1000을 넘으면 `Step`이 실패한다.
- 그 외 예외는 즉시 실패한다.

### Tasklet

단일 작업을 수행하는 `Step`의 다른 형태다.
배치에서 일반적인 Reader-Processor-Writer 사이클 없이, 메서드를 하나 실행하고 끝이다.

```java
@Bean
public Tasklet cleanupTasklet() {
    return (contribution, chunkContext) -> {
        log.info("임시 파일 정리 시작");
        fileService.cleanupTempFiles();
        log.info("완료");
        return RepeatStatus.FINISHED;
    };
}

@Bean
public Step cleanupStep(Tasklet cleanupTasklet) {
    return new StepBuilder("cleanupStep", jobRepository)
            .tasklet(cleanupTasklet, txManager)
            .build();
}
```

용도는 다음과 같다.

- 작업 시작 전 준비: 임시 테이블 `truncate`, 디렉토리 생성
- 작업 끝난 후 정리: 임시 파일 삭제, 캐시 무효화
- 단일 외부 호출: 이번 달 환율 한 번 가져와서 저장
- 알림 1회: `Job` 완료됐다고 Slack에 한 번 메시지
