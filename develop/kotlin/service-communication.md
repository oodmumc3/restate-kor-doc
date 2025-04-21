# 📘 Restate Java 서비스 통신 위키

## 목차

- [📘 Restate Java 서비스 통신 위키](#-restate-java-서비스-통신-위키)
  - [목차](#목차)
  - [문서 개요](#문서-개요)
  - [해석 기능 요약](#해석-기능-요약)
  - [핸들러 호출 방식](#핸들러-호출-방식)
    - [요청-응답 호출 (Request-response)](#요청-응답-호출-request-response)
    - [메시지 전송 (Fire-and-forget)](#메시지-전송-fire-and-forget)
    - [지연 호출 (Delayed call)](#지연-호출-delayed-call)
  - [Virtual Object와 호출 순서](#virtual-object와-호출-순서)
    - [Deadlock 발생 사례](#deadlock-발생-사례)
    - [해결 방법](#해결-방법)
  - [도칭 반복 검증: Idempotency Key](#도칭-반복-검증-idempotency-key)
  - [호출 재접속 및 취소](#호출-재접속-및-취소)
    - [호출 취소 (Cancel an invocation)](#호출-취소-cancel-an-invocation)

---

## 문서 개요

Restate Java SDK의 “서비스 간 통신”기능을 설명한 문서입니다. 

관련 데이터 및 요청을 관련 핸들러 사이에서 연결할 때 여러 가지 방식과 또한 고급 기능들을 제공합니다.

---

## 해석 기능 요약

| 기능 | 설명 |
|--------|--------|
| **Request-response call** | 응답을 기다린 호출, `await()` 사용 |
| **Fire-and-forget call** | 응답 없이 메시지 전송, `.send()` 사용 |
| **Delayed Call** | 결과를 지역 호출, Duration 값 제공 |
| **Virtual Object** | 순차적 호출, 데드런스 주의 |
| **Workflow 호출** | `run()`, custom handler 호출 가능 |
| **Idempotency Key** | 중복 호출 방지를 위해 key 제공 |
| **InvocationHandle** | 호출에 재접속 또는 취소가능 |

---

## 핸들러 호출 방식

### 요청-응답 호출 (Request-response)

- 한 핸들러가 다른 핸들러의 호출을 실행한 후, 응답을 `await()`해서 기다리는 그런 전환 형식
- Restate Java SDK는 하나한 서비스에 대한 타입에 안전한 클라이언트를 자동으로 생성
- 예)
  ```kotlin
  val response = MyServiceClient.fromContext(ctx).myHandler(request).await()
  ```
- Virtual Object가 대상이면:
  ```kotlin
  val response = MyVirtualObjectClient.fromContext(ctx, objectKey).myHandler(request).await()
  ```
- Workflow가 대상이면:
  ```kotlin
  val response = MyWorkflowClient.fromContext(ctx, workflowId).run(request).await()
  ```
- 또는 `interactWithWorkflow()` handler 호출 가능
- 가능하면 타입이 없는 generic client 방식으로 도 호출 가능:
  ```kotlin
  val target = Target.service("MyService", "myHandler")
  val response = ctx.call(Request.of(target, typeTag<String>(), typeTag<String>(), request)).await()
  ```
- **이 호출은 전달처리 및 성공 역시가 journal에 로그되며**, Restate가 자동으로 재시도 구할
- 호출의 부정가 발생할 경우 복구되고, 발생한 오류는 tracing을 통해 조회 가능

### 메시지 전송 (Fire-and-forget)

- 핸들러가 응답을 기다리지 않고 메시지를 전송하는 방식 (일명 'one-way call' 또는 'fire-and-forget')
- 예시:
  ```kotlin
  MyServiceClient.fromContext(ctx).send().myHandler(request)
  ```
- generic 클라이언트 사용도 가능:
  ```kotlin
  val target = Target.service("MyService", "myHandler")
  ctx.send(Request.of(target, typeTag<String>(), typeTag<String>(), request))
  ```
- **Restate는 메시지를 durable하게 journal에 기록하므로, 외부 MQ(메시지 큐) 없이도 신뢰성 있는 전송이 가능**
- 메시지는 서비스가 장애에서 복구된 후에도 반드시 실행됨을 보장함
- 단, `send()`는 응답을 기다리지 않기 때문에 결과를 얻거나 에러를 캐치하고 싶다면 `InvocationHandle`을 사용하거나 idempotency key와 함께 조합하여 사용 가능

### 지연 호출 (Delayed call)

- `send()` 호출 시 `Duration` 값을 두 번째 인자로 넘겨 호출을 일정 시간 뒤에 실행되도록 예약
- 예시:
  ```kotlin
  MyServiceClient.fromContext(ctx).send().myHandler(request, 5.days)
  ```
- generic 클라이언트를 사용할 수도 있음:
  ```kotlin
  val target = Target.service("MyService", "myHandler")
  Request.of(target, typeTag<String>(), typeTag<String>(), request).send(ctx, 5.days)
  ```
- **지연 호출은 일반적인 메시지 전송 방식(fire-and-forget)과 동일하게 durable 하며**, 지정된 시간 이후에 Restate가 자동 실행을 보장함
- **활용 예시:**
  - 특정 시간 이후 실행되는 백그라운드 작업 예약
  - 지연된 리마인더 알림 발송
  - 워크플로우 내에서 타임아웃 기반 처리 예약
- **스케줄링된 작업의 안정성 보장:**
  - Restate는 요청을 로그에 기록하고, 지정된 시간 후 자동 실행되도록 관리
  - 외부 스케줄러나 큐 시스템 불필요

---

## Virtual Object와 호출 순서

- Virtual Object는 호출이 변경 없이 **순차적**으로 진행
- `call A` 후 `call B` 를 실행해도 순서 보정
- 다른 핸들러에서 돌아오는 호출은 가상 가중
- 동일 키로 연결시 Deadlock 가능 → 수동 취소 필요

### Deadlock 발생 사례

Request-response 방식으로 Virtual Object를 호출할 경우 **데드락(deadlock)** 이 발생할 수 있습니다. 이 경우 해당 Virtual Object는 잠긴 상태가 되어 추가 요청을 처리하지 못하게 됩니다. 다음과 같은 경우 주의가 필요합니다:

- **교차 데드락 (Cross deadlock):**
  - Virtual Object A가 B를 호출하고, B도 다시 A를 호출하는 경우
  - 두 호출 모두 같은 키를 사용하는 경우 발생 가능
- **순환 데드락 (Cyclical deadlock):**
  - A → B → C → A처럼 순환 구조로 연결된 호출 체인에서 발생

### 해결 방법

- 데드락이 발생한 Virtual Object는 **자동으로 해제되지 않기 때문에**, [수동으로 호출 취소](https://docs.restate.dev/operate/invocation#cancelling-invocations)를 통해 잠금을 해제해야 합니다.
- 이때 `InvocationHandle.cancel()`을 사용해 강제 취소 처리할 수 있습니다.

---

## 도칭 반복 검증: Idempotency Key

- 동일 요청이 두 번 이상 전달되는 상황을 방지하기 위한 고유 키(idempotency key)를 설정 가능
- 예시:
  ```kotlin
  MyServiceClient.fromContext(ctx).myHandler(request) {
    idempotencyKey = "order-123"
  }
  ```
- fire-and-forget 메시지 전송에도 설정 가능:
  ```kotlin
  MyServiceClient.fromContext(ctx).send().myHandler(request) {
    idempotencyKey = "order-123"
  }
  ```
- **효과:**
  - 동일 요청이 반복되어도 단 한 번만 실행되도록 보장
  - 호출 결과는 일정 기간 동안 Restate가 저장하며, 재접속 시 재사용 가능
- **주의사항:**
  - idempotency key는 호출 내에서 고유해야 하며, 동일 서비스/핸들러 조합에서 중복 사용 시 이전 결과가 반환됨
  - key가 없으면 중복 호출 여부를 판단할 수 없어 동일 요청이 여러 번 처리될 수 있음
- **활용 예시:**
  - 외부 클라이언트에서 동일한 요청을 네트워크 재시도 등으로 여러 번 보내는 경우
  - 워크플로우 내에서 같은 작업이 중복 실행되지 않도록 제어할 때

---

## 호출 재접속 및 취소

- `InvocationHandle` 을 통해 fire-and-forget 또는 식별 가능한 호출에 재접속하거나 취소할 수 있음
- 예시:
  ```kotlin
  val handle = MyServiceClient.fromContext(ctx).send().myHandler(request) {
    idempotencyKey = "task-123"
  }
  val response = handle.attach().await()
  ```
- `attach()`는 해당 요청이 완료될 때까지 대기하고, 결과가 있으면 반환함
- **주의:** `idempotencyKey` 없이 보낸 요청에는 attach만 가능하고, 결과(response)는 얻을 수 없음
- **활용 예시:**
  - 한 번 전송된 메시지의 결과를 나중에 확인하고 싶을 때
  - 호출 상태를 외부에서 관리하고 재확인하거나 복구 처리하고자 할 때

### 호출 취소 (Cancel an invocation)

- `InvocationHandle` 은 실행 중인 요청을 취소할 수도 있음
- 예시:
  ```kotlin
  val handle = MyServiceClient.fromContext(ctx).send().myHandler(request)
  handle.cancel()
  ```
- 취소된 요청은 실행되지 않으며, 상태는 `Cancelled`로 표시됨
- **활용 예시:**
  - 사용자 인터페이스에서 '취소' 버튼 처리 시
  - 워크플로우에서 특정 분기나 작업을 강제로 종료할 필요가 있을 때
  - Virtual Object 간 순환 호출로 인해 데드락이 발생했을 경우 이를 해소할 때
- 더 자세한 사례는 [Sagas 가이드](https://docs.restate.dev/guides/sagas)에서 확인 가능