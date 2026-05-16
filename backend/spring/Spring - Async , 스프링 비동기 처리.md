

## 주제: 알림 처리 구조와 @Async 동작 원리

월간 큐레이션 알림 처리에서 Spring Batch 도입 여부를 고민하다가, 현재 `@Async` 기반 이벤트 처리 구조의 내부 동작 공부를 해봤다. 

---

## 1. 작업 분리의 층위

세 가지는 같은 층위가 아니라 각자 다른 역할.

- `@Scheduled`: **언제** 돌릴지 (트리거)
- `@Async` / `ThreadPoolTaskExecutor`: **얼마나 동시에** 돌릴지 (실행 동시성)
- Spring Batch: **대량 데이터를 어떻게 안전하게** 처리할지 (처리 모델 + 복구 인프라)

`@Async`를 Batch가 대체하는 게 아니라, Batch 내부에서도 TaskExecutor를 씀.

## 2. 큐레이션 + 알림 분리 전략(이미 적용 완료)

생성과 알림은 성질이 다른 작업 (DB 트랜잭션 vs 외부 호출) → 분리 필요.

- **Step 분리** (Batch 도입 시): Reader 쿼리로 자연스러운 멱등성
- **트랜잭션 이벤트** (`@TransactionalEventListener(AFTER_COMMIT)` + `@Async`): 가볍지만 누락 시 복구 불가
- **Outbox 패턴**: DB 트랜잭션이 메시지 영속성 보장, 가장 견고

현재 구조는 트랜잭션 이벤트 방식. happy path엔 좋지만 **JVM 죽으면 인플라이트 이벤트 증발**하는 한계.

## 3. @Async 내부 동작 

**AOP 프록시 + TaskExecutor의 조합**.

- 프록시란 런타임에 만들어지는 "원본을 감싼 가짜 객체". CGLIB(상속 기반)이 Spring Boot 2.x부터 기본값이라 **인터페이스/Impl 따로 안 만들어도 됨**.
- 프록시는 **빈 주입을 통해야만 동작**. 직접 `new`, `this.method()` self-invocation은 우회됨.
- 프록시는 DispatcherServlet과 별개 층위. 빈 생성 시점에 컨테이너 안에 만들어져서 빈 호출을 가로챔.

호출 흐름: 호출자 → 프록시 → `TaskExecutor.submit(Runnable)` → 즉시 return → 워커 스레드가 큐에서 꺼내 실행.

## 4. TaskExecutor 함정 (가장 중요)

**기본값 절대 신뢰 금지**.

- `SimpleAsyncTaskExecutor` 기본값은 호출 수만큼 `new Thread()` → JVM 폭발
- `ThreadPoolTaskExecutor` 명시 등록 필수

**ThreadPoolExecutor 처리 순서

1. 스레드 수 < core → 새 스레드
2. 큐에 자리 있음 → 큐에 적재 (스레드 추가 안 함)
3. 큐 꽉 참 → 그때서야 max까지 스레드 늘림
4. max도 꽉 참 → RejectedExecutionHandler

**즉 큐가 안 차면 max는 영원히 도달 못 함**. core != max로 두고 큐 크게 잡으면 max는 사실상 장식.

## 5. 내 config 진단

```java
// defaultTaskExecutor: core 6, max 12, queue 10000
// fcmTaskExecutor: core 2, max 4, queue 500
```

둘 다 **core != max + 큰 queue** 함정. max는 거의 안 쓰임.

서버 다운의 직접 원인이 큐 10000인지는 단정 못 함. **큐에 쌓인 Runnable이 캡처한 객체들**(User, Curation, 영속성 컨텍스트)이 메모리를 잡고 있던 게 진짜 원인일 가능성. heap dump 확인 필요.

**`defaultTaskExecutor`에 graceful shutdown 누락**도 시급한 문제 — 배포 시 큐 안 작업 증발.

개선 방향:

- core = max로 통일 (I/O 작업 기준 12~20)
- 큐는 작게 (1000 정도), CallerRunsPolicy로 백프레셔
- `WaitForTasksToCompleteOnShutdown` + `AwaitTerminationSeconds` 필수
- FCM은 MulticastMessage(500개 묶음 발송) 도입 검토

## 6. 비동기와 동시성

비동기 = "호출자가 안 기다림" ≠ "전부 동시 실행". **풀의 가용 스레드 수가 진짜 동시성 한도**.

- 풀 5개에 20개 호출 → 5개 실행, 15개 큐 대기, 워커 비면 큐에서 꺼냄
- `void` 반환 = fire-and-forget, 호출자는 결과 못 받음
- 결과 기다리려면 `CompletableFuture` 반환 + `allOf().join()`

## 7. 누락 절대 안 되는 알림

`@Async`로는 보장 불가. 큐는 메모리라 JVM 죽으면 증발.

**Outbox 패턴**이 답:

- 큐레이션 생성과 같은 트랜잭션에서 `notification_outbox` 테이블에 적재
- 별도 스케줄러가 PENDING 상태 폴링해서 발송
- 실패 시 재시도 카운트, 임계치 넘으면 FAILED

`@Async`는 happy path 최적화, Outbox는 안전망. **대체 관계가 아니라 조합**.

---

## 면접용 한 줄 요약

> "월간 큐레이션 알림을 `@TransactionalEventListener` + `@Async`로 처리하다가 서버 다운을 겪고, `@Async` 내부 동작(AOP 프록시 + ThreadPoolExecutor)을 다시 들여다봤다. core != max + 큰 queue 설정 때문에 max 풀이 도달 못 하는 구조였고, 큐에 캡처된 객체들이 메모리 압박을 만들었다. 단기적으론 풀 설정 재조정 + graceful shutdown 추가, 중장기적으론 알림 누락 방지를 위한 outbox 패턴 또는 Spring Batch Step 분리를 검토 중."