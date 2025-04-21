# Restate Workflows 개요 문서

## 목차
1. 개요
2. Workflow 정의 및 실행 방식
3. Workflow 상태 접근 (쿼리/시그널)
4. Workflow 구현 예시 (Java/Kotlin)
5. Workflow 배포 및 등록
6. Workflow 실행 방법 (SDK, HTTP, 서비스 내부 등)
7. Workflow 결과 확인 및 점검
8. 보존 기간 및 설정

## 1. 개요
Restate의 Workflow는 **내구성 있는 단계 시퀀스를 실행하는 특수한 Virtual Object**로, 한 번의 실행 단위를 기준으로 상태를 관리하고, 이벤트를 처리할 수 있도록 설계되었습니다.

- 각 워크플로우 정의는 워크플로우 로직을 수행하는 `run handler`를 가짐
- `run handler`는 워크플로우 인스턴스(오브젝트/키)당 정확히 한 번 실행됨
- 실행 내용은 다음 두 가지 활동으로 구성:
  - **Inline 활동**: 예를 들어 `run block`, `sleep` 등
  - **외부 핸들러 호출**: 별도로 정의된 활동 핸들러 호출
- 워크플로우는 SDK, Restate 서비스, HTTP, Kafka 등으로 제출 가능

## 2. Workflow 정의 및 실행 방식
- 워크플로우 정의 내에 다중 핸들러를 구현할 수 있으며, 이를 통해 워크플로우와 상호작용 가능
- 핸들러를 통해 워크플로우의 상태를 쿼리하거나, 워크플로우가 대기 중인 프라미스를 해소하여 시그널을 보낼 수 있음
- `WorkflowContext`와 `SharedWorkflowContext`를 활용해 Durable Promise 등 고급 기능 사용 가능
- 워크플로우의 K/V 상태는 실행 중 워크플로우에 국한되며, `run handler`만이 이를 변경 가능함

## 3. Workflow 상태 접근 (쿼리/시그널)
- **쿼리(Query)**: 워크플로우의 상태(K/V)를 외부에서 조회할 수 있으며, 이는 워크플로우 정의에 포함된 다른 핸들러를 통해 노출됩니다. 예를 들어 `getStatus()` 같은 핸들러를 통해 외부 클라이언트는 상태를 조회할 수 있습니다.
  - 각 워크플로우 실행은 고유한 인스턴스로 간주되며, 그에 따른 상태는 해당 실행에만 격리됨
  - 상태(K/V)는 오직 `run handler`만이 변경할 수 있으며, 다른 핸들러는 읽기만 가능
  ```kotlin
  @Shared
    suspend fun getStatus(ctx: SharedWorkflowContext): String? {
      return ctx.get(STATUS)
    }
  ```

- **시그널(Signal)**: 워크플로우가 외부 이벤트를 기다리거나 해당 이벤트를 처리할 수 있도록 Durable Promise를 활용합니다.
  - Durable Promise는 **분산 환경에서 안전하고 장애 복구가 가능**한 약속(Promise) 구조로, 워크플로우의 어떤 핸들러든 이를 해결(resolve)하거나 거부(reject)할 수 있음
  ```kotlin
  @Workflow
  suspend fun run(ctx: WorkflowContext, email: Email): Boolean {
    ...
    val clickSecret = ctx.promise(EMAIL_CLICKED).future().await()
    ...
  }

  @Shared
  suspend fun click(ctx: SharedWorkflowContext, secret: String) {
    ctx.promiseHandle(EMAIL_CLICKED).resolve(secret)
  }
  ```
### Durable Promise 사용 패턴:
1. `run handler`에서 Durable Promise 생성
2. 다른 핸들러에서 해당 Promise를 **한 번만** resolve 또는 reject
3. 또는 그 반대로 `run handler`가 워크플로우 진행 중 특정 Promise를 resolve하여 다른 핸들러가 그 상태를 확인 가능

이러한 구조는 워크플로우가 외부 입력을 기다리는 동안 안정적으로 블로킹되도록 하고, 이벤트 기반의 유연한 설계를 가능하게 합니다.

## 4. Workflow 구현 예시 (Java/Kotlin)
워크플로우는 다음과 같은 구조로 구현됩니다:

```kotlin
@Workflow
class SignupWorkflow {

  companion object {
    private val EMAIL_CLICKED = durablePromiseKey<String>("email_clicked")
    private val STATUS = stateKey<String>("status")
  }

  @Workflow
  suspend fun run(ctx: WorkflowContext, email: Email): Boolean {
    val secret = ctx.random().nextUUID().toString()
    ctx.set(STATUS, "Generated secret")

    ctx.runBlock("send email") { sendEmailWithLink(email, secret) }

    val clickSecret = ctx.promise(EMAIL_CLICKED).future().await()
    ctx.set(STATUS, "Clicked email")

    return clickSecret == secret
  }

  @Shared
  suspend fun click(ctx: SharedWorkflowContext, secret: String) {
    ctx.promiseHandle(EMAIL_CLICKED).resolve(secret)
  }

  @Shared
  suspend fun getStatus(ctx: SharedWorkflowContext): String? {
    return ctx.get(STATUS)
  }
}

fun main() {
  RestateHttpServer.listen(endpoint { bind(SignupWorkflow()) })
}
```

- `run` 핸들러는 이메일 발송 이후 사용자의 클릭을 기다리는 흐름을 Durable Promise로 구성
- `click` 핸들러는 외부에서 secret을 전달받아 Promise를 해소함
- `getStatus`는 현재 워크플로우 상태를 반환함

## 5. Workflow 배포 및 등록
- Service 및 Virtual Object와 동일하게 HTTP endpoint 또는 AWS Lambda로 등록 가능
- Restate에 사전 등록 필요

## 6. Workflow 실행 방법 (SDK, HTTP, 서비스 내부 등)
### SDK 클라이언트를 통한 실행
- 워크플로우는 생성된 SDK 클라이언트를 통해 **제출(submit)**, **쿼리(query)**, **시그널(signal)** 가능
- `submit()`은 워크플로우 ID에 대해 한 번만 호출 가능하며, 실행 핸들러를 등록하고 핸들을 반환함

```kotlin
val restateClient = Client.connect("http://localhost:8080")
val handle = SignupWorkflowClient.fromClient(restateClient, "someone").submit(email)
```

### 워크플로우 상태 조회 / 시그널 전달
- `.getStatus()`로 상태를 조회하거나, `.click()`으로 시그널 전달 가능

```kotlin
val status = SignupWorkflowClient.fromClient(restateClient, "someone").getStatus()
```

### attach / peek
- 워크플로우 실행 결과를 기다리거나 상태만 확인

```kotlin
// 결과가 준비될 때까지 대기
val result = SignupWorkflowClient.fromClient(restateClient, "someone")
    .workflowHandle()
    .attach()
    .response()

// 상태만 확인
val peekOutput = SignupWorkflowClient.fromClient(restateClient, "someone")
    .workflowHandle()
    .getOutput()
    .response()

if (peekOutput.isReady()) {
  val result2 = peekOutput.getValue()
}
```

### Restate 서비스 내에서 워크플로우 실행하기
- 워크플로우는 서비스 핸들러 내부에서도 실행 가능하며, `.run()` 호출을 통해 워크플로우 실행 결과를 `await()`로 받아올 수 있음

```java
@Handler
public void setup(ObjectContext ctx, Email email) {
  boolean result = SignupWorkflowClient.fromContext(ctx, "someone").run(email).await();
}

@Handler
public void queryStatus(ObjectContext ctx) {
  String status = SignupWorkflowClient.fromContext(ctx, "someone").getStatus().await();
}
```

### HTTP로 워크플로우 실행 및 결과 확인
- 워크플로우의 모든 핸들러는 HTTP를 통해 호출 가능하며, `/send`를 붙이면 one-way 호출 가능
- `run` 핸들러는 워크플로우 ID당 한 번만 실행 가능함

```bash
curl localhost:8080/SignupWorkflow/someone/run --json '"someone@restate.dev"'
```

- 워크플로우 결과 조회:

```bash
curl localhost:8080/restate/workflow/SignupWorkflow/someone/attach
curl localhost:8080/restate/workflow/SignupWorkflow/someone/output
```

## 7. Workflow 결과 확인 및 점검
- **UI 또는 CLI**를 통해 다음 항목을 확인할 수 있음:
  - **실행 이력**: Invocations 탭에서 워크플로우 실행 저널(Journal) 확인
  - **상태 조회**: State 탭에서 K/V 상태 정보 확인

## 8. 보존 기간 및 설정
- 기본 보존 시간: run handler 완료 후 **24시간**
- 보존 시간 초과 시:
  - K/V 상태 삭제
  - 공유 핸들러 호출 불가
  - Durable Promise 폐기
- UI 또는 Admin API에서 `workflow_completion_retention` 설정으로 조정 가능

