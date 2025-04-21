# Restate 실행 모델 및 구성요소 위키

> [docs.restate.dev](https://docs.restate.dev/concepts/services)

## 📚 목차
1. Restate 개요
2. Restate의 3가지 주요 구성요소
3. Restate의 3가지 실행 모델
   - Services (plain)
   - Virtual Objects
   - Workflows
4. 실행 모델 비교 요약

---

## 1. Restate 개요
Restate는 실패 복구, 상태 저장, 비동기 로직 실행을 쉽게 구현할 수 있도록 도와주는 **durable execution 엔진**입니다. 핸들러 기반의 서비스 구조로, 복잡한 워크플로우도 안정적으로 처리할 수 있습니다.

---

## 2. Restate의 3가지 주요 구성요소

| 구성요소 | 설명 |
|----------|------|
| **Services** | Restate SDK를 사용하는 사용자 비즈니스 로직 단위로, 핸들러의 모음입니다. |
| **Restate Server** | 요청을 수신하고, 적절한 서비스 핸들러로 전달하며 실행을 관리하는 중앙 제어 컴포넌트입니다. |
| **Invocation** | 특정 핸들러를 실행하라는 호출 요청으로, 외부 API 호출이든 내부 호출이든 모두 invocation 단위로 처리됩니다. |

---

## 3. Restate의 3가지 실행 모델

### ✅ Services (plain)
- **핸들러들의 모음**으로 구성된 기본적인 실행 단위입니다.
- 상태 저장은 없으며, 실행 중인 핸들러가 실패하더라도 중단 지점부터 **자동 복구**되어 실행을 끝마칩니다.
- 병렬 실행 가능하며, HTTP 기반으로 호출됩니다.

**예제 (Kotlin):**
```kotlin
@Service
class SubscriptionService {
  @Handler
  suspend fun add(ctx: Context, req: SubscriptionRequest) {
    // 고유한 결제 ID를 생성함 (idempotency 보장 목적)
    val paymentId = ctx.random().nextUUID().toString()

    // 외부 결제 생성 API 호출 - durable하게 실행됨
    val payRef = ctx.runBlock { createRecurringPayment(req.creditCard, paymentId) }

    // 사용자의 구독 항목을 하나씩 처리하며 각각의 구독 생성 API 호출
    for (subscription in req.subscriptions) {
      ctx.runBlock { createSubscription(req.userId, subscription, payRef) }
    }
  }
}

fun main() {
  // Restate 서버에 SubscriptionService를 HTTP 핸들러로 바인딩하여 실행
  RestateHttpServer.listen(endpoint { bind(SubscriptionService()) })
}
```

---

### ✅ Virtual Objects (VO)
- 고유한 key로 식별되며, 오브젝트별로 **격리된 K/V 상태 저장소**를 가집니다.
- 핸들러는 해당 오브젝트의 상태에 접근해 읽고 쓸 수 있습니다.
- 쓰기 핸들러는 동시 실행 불가 (하나씩 큐잉), 읽기 핸들러는 `@Shared`로 병렬 실행 가능.

**예제 (Kotlin):**
```kotlin
@VirtualObject
class GreeterObject {
  companion object {
    // greet 호출 횟수를 저장하는 상태 키 정의 (타입: Int)
    private val COUNT = stateKey<Int>("greet-count")
  }

  @Handler
  suspend fun greet(ctx: ObjectContext, greeting: String): String {
    val name = ctx.key() // 현재 오브젝트의 고유 키 (예: 사용자 이름)
    val count = ctx.get(COUNT) ?: 0 // greet 횟수를 읽음 (기본값 0)
    val newCount = count + 1
    ctx.set(COUNT, newCount) // greet 횟수를 1 증가시켜 저장
    return "$greeting $name, for the $newCount-th time" // 메시지 반환
  }

  @Handler
  suspend fun ungreet(ctx: ObjectContext): String {
    val name = ctx.key()
    val count = ctx.get(COUNT) ?: 0
    if (count > 0) ctx.set(COUNT, count - 1) // greet 횟수를 1 감소
    return "Dear $name, taking one greeting back: $count" // 메시지 반환
  }

  @Shared
  suspend fun getGreetCount(ctx: SharedObjectContext): Int {
    return ctx.get(COUNT) ?: 0 // 현재 greet 횟수를 반환 (읽기 전용)
  }
}

fun main() {
  // GreeterObject를 Restate 서버에 바인딩하여 실행
  RestateHttpServer.listen(endpoint { bind(GreeterObject()) })
}
```

---

### ✅ Workflows

- Virtual Object의 확장 버전으로, **각 워크플로우 ID(오브젝트)당 한 번만 실행되는 `run` 핸들러**를 통해 워크플로우 로직을 정의합니다.
- 외부에서 워크플로우에 signal을 보내거나 query로 정보를 조회할 수 있습니다.
- 복잡한 승인, 사용자 개입, 장기 실행, 타임아웃 등에 적합합니다.

#### 🔁 실행 흐름 예시
1. `run()` 핸들러에서 `sendEmailWithLink(...)`를 실행해 사용자에게 이메일을 전송합니다.
2. 이후 워크플로우는 `ctx.promise(...).await()` 상태에서 외부 입력을 기다리며 **durably paused** 상태로 진입합니다.
3. 사용자가 이메일 링크를 클릭하면, 외부에서 `click()` 핸들러가 호출되고 해당 promise를 `resolve()` 합니다.
4. 대기 중이던 `await()`는 완료되고 워크플로우가 이어서 실행되며, secret을 비교하고 최종 결과를 반환합니다.

이 구조를 통해 사용자 입력/승인 등을 기다리는 **비동기 + 지속 가능한 워크플로우**를 쉽게 구성할 수 있습니다.

**예제 (Kotlin):**
```kotlin
@Workflow
class SignupWorkflow {
  companion object {
    // 사용자 이메일 클릭 여부를 추적할 Promise 키 정의
    private val LINK_CLICKED = durablePromiseKey<String>("link_clicked")
  }

  @Workflow
  suspend fun run(ctx: WorkflowContext, user: User): Boolean {
    val userId = ctx.key() // 워크플로우의 고유 ID로 사용자 ID를 사용함

    // 사용자 정보를 기반으로 초기 엔트리 생성 (예: DB에 유저 등록)
    ctx.runBlock { createUserEntry(user) }

    // 비밀 코드 생성 (클릭 유효성 확인용)
    val secret = ctx.random().nextUUID().toString()

    // 사용자에게 이메일 전송 (비밀 코드 포함)
    ctx.runBlock { sendEmailWithLink(userId, user, secret) }

    // 사용자가 링크를 클릭할 때까지 대기 (promise 비동기 대기)
    val clickSecret = ctx.promise(LINK_CLICKED).future().await()

    // 클릭 시 전달받은 비밀 코드와 비교하여 일치 여부 반환
    return clickSecret == secret
  }

  @Shared
  suspend fun click(ctx: SharedWorkflowContext, secret: String) {
    // 외부에서 링크 클릭 시 promise를 resolve하여 워크플로우 재개
    ctx.promiseHandle(LINK_CLICKED).resolve(secret)
  }
}

fun main() {
  // Restate 서버에 SignupWorkflow를 바인딩하여 실행
  RestateHttpServer.listen(endpoint { bind(SignupWorkflow()) })
}
```

---

## 4. 실행 모델 비교 요약

| 항목 | Services (plain) | Virtual Objects | Workflows |
|------|------------------|------------------|-----------|
| 상태 저장 | ❌ 없음 | ✔️ 객체별 K/V | ✔️ 실행당 K/V |
| 실행 보장 | ✔️ 실행 끝까지 보장 | ✔️ | ✔️ |
| 동시성 | ✔️ 병렬 실행 가능 | ❌ (쓰기 핸들러는 순차) | ✔️ run 외 병렬 가능 |
| 사용자 개입 | ❌ 별도 구현 필요 | ❌ | ✔️ promise, signal 활용 |
| 적합한 시나리오 | 멱등 API, 병렬 호출 | 상태 머신, 엔티티 모델 | 가입/결제/승인 워크플로우 |


