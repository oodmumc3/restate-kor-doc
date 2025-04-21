# Restate Java/Kotlin SDK 개요 위키

## 목차

1. 개요
2. Services
3. Virtual Objects
4. Workflows
5. 주요 비교
6. 참고 자료

---

## 1. 개요

Restate Java/Kotlin SDK는 서버리스 환경에서 durable한 서비스 로직을 구현할 수 있도록 도와주는 오픈소스 프레임워크입니다. Java 및 Kotlin 언어로 빠르게 설정하고 실행할 수 있으며, 핸들러 기반의 구조를 통해 서비스, 상태 보존 객체(Virtual Objects), 워크플로우(Workflows)를 구성합니다.

- GitHub: [https://github.com/restatedev/sdk-java](https://github.com/restatedev/sdk-java)
- SDK는 JDK 17 이상 필요
- 서비스 단위로 핸들러를 정의하고, `Context`를 통해 Restate Runtime과 상호작용

---

## 2. Services

### 정의

`@Service` 어노테이션을 사용하여 정의되는 기본 서비스 단위입니다. 핸들러는 외부 요청을 처리하고, 필요 시 상태 없이 작업을 수행합니다.

### 특징

- `@Service`와 `@Handler`로 구성
- 핸들러는 `Context`를 첫 번째 인자로 받으며, Restate와의 상호작용은 이 객체를 통해 이루어집니다.
- 핸들러 내에서 수행하는 작업은 모두 Restate Journal에 durable하게 기록되어, 중단이나 장애 발생 시에도 복구가 가능합니다.
- 입력 파라미터는 최대 하나까지만 받을 수 있으며, 입력값과 반환값은 어떤 타입이든 가능합니다. (자세한 직렬화 방식은 [Serialization 문서](https://docs.restate.dev/develop/java/serialization) 참고)
- 서비스 이름은 기본적으로 클래스 이름(`MyService`)이 되며, 필요 시 `@Name` 어노테이션을 통해 커스터마이징할 수 있습니다.
- `RestateHttpServer.listen(...)`을 통해 서비스 인스턴스를 Restate 엔드포인트에 바인딩하며, 기본 포트는 `9080`입니다.

### 예시

```kotlin
@Service
class MyService {
  @Handler
  suspend fun myHandler(ctx: Context, greeting: String): String {
    return "$greeting!"
  }
}

fun main() {
  RestateHttpServer.listen(endpoint { bind(MyService()) })
}
```

---

## 3. Virtual Objects

### 정의

Virtual Object는 키 기반 상태를 가지며, `@VirtualObject` 어노테이션을 통해 정의됩니다. 각 객체는 고유 key를 기반으로 식별되며 상태를 저장합니다.

### 특징

- `@VirtualObject` 어노테이션을 클래스에 부여하여 Virtual Object로 정의합니다.
- 핸들러는 첫 번째 인자로 반드시 `ObjectContext`를 받아야 하며, 이를 통해 Restate의 K/V 상태 저장소에 쓰기 작업이 가능합니다.
- 동일한 Virtual Object(Key 기준)에 대해 하나의 핸들러만 동시에 실행될 수 있어 상태 일관성이 보장됩니다.
- `ctx.key()`를 사용하면 해당 객체의 고유 키를 조회할 수 있습니다.
- 동시 실행이 필요한 경우, `@Shared` 어노테이션과 함께 `SharedObjectContext`를 사용하여 읽기 전용 핸들러를 정의할 수 있습니다.
  - 예: 외부에 상태를 노출하거나, 블로킹 핸들러의 완료를 기다리는 `awakeable` 처리 등
- Virtual Object는 키 기반 상태를 안전하게 처리하면서도, 병렬 처리와 외부 상호작용이 가능한 유연한 아키텍처를 제공합니다.

### 예시

```kotlin
@VirtualObject
class MyVirtualObject {

  @Handler
  suspend fun myHandler(ctx: ObjectContext, greeting: String): String {
    val objectKey = ctx.key()

    return "$greeting $objectKey!"
  }

  @Shared
  suspend fun myConcurrentHandler(ctx: SharedObjectContext, input: String): String {
    return "my-output"
  }
}

fun main() {
  RestateHttpServer.listen(endpoint { bind(MyVirtualObject()) })
}
```

---

## 4. Workflows

### 정의

워크플로우는 하나의 실행 흐름(run)을 가지며, Durable하게 복원 가능한 상태 머신입니다. Virtual Object의 특성을 가지면서 실행 순서와 결과를 저장합니다.

### 특징

- `@Workflow` 어노테이션을 클래스와 실행 메서드에 사용하여 워크플로우 정의
- 모든 워크플로우 클래스에는 단 하나의 `@Workflow` 핸들러가 필요하며, 이 핸들러는 `WorkflowContext`를 통해 Restate SDK와 상호작용합니다. ([JavaDocs](https://docs.restate.dev/javadocs/dev/restate/sdk/WorkflowContext) / [KotlinDocs](https://docs.restate.dev/ktdocs/sdk-api-kotlin/dev.restate.sdk.kotlin/-workflow-context/))
- `WorkflowContext`는 워크플로우의 실행 ID를 `ctx.key()`로 조회할 수 있으며, durable한 블록 실행, 슬립, 핸들러 호출 등 다양한 단계 정의가 가능합니다. 예: `ctx.run { ... }`, `ctx.sleep(...)`, `ctx.call(...)`
- 워크플로우 핸들러는 워크플로우 실행마다 **정확히 한 번만 실행**되며, 실행 도중 실패하거나 재시작되더라도 같은 로직을 정확히 복원 가능
- 워크플로우 실행이 완료된 이후에도 signal/query를 위한 외부 핸들러(`@Handler` + `SharedWorkflowContext`)를 통해 상호작용할 수 있습니다. ([JavaDocs](https://docs.restate.dev/javadocs/dev/restate/sdk/SharedWorkflowContext) / [KotlinDocs](https://docs.restate.dev/ktdocs/sdk-api-kotlin/dev.restate.sdk.kotlin/-shared-workflow-context/))
- signal/query 핸들러는 워크플로우와 동시에 병렬 실행될 수 있으며, 실행 완료 이후에도 접근 가능합니다
- 복잡한 흐름을 안전하게 처리할 수 있으며, 내부 단계는 Journal에 저장되어 안정성과 복원력을 갖춘 워크플로우 처리를 구현할 수 있습니다

### 예시

```kotlin
@Workflow
class MyWorkflow {

  @Workflow
  suspend fun run(ctx: WorkflowContext, input: String): String {
    // implement workflow logic here

    return "success"
  }

  @Handler
  suspend fun interactWithWorkflow(ctx: SharedWorkflowContext, input: String) {
    // implement interaction logic here
  }
}

fun main() {
  RestateHttpServer.listen(endpoint { bind(MyWorkflow()) })
}
```

---

## 5. 주요 비교

| 항목    | Service       | Virtual Object              | Workflow                  |
| ----- | ------------- | --------------------------- | ------------------------- |
| 상태 보존 | X             | O (K/V 상태 저장소)              | O + 실행 단계 보존              |
| 키 기반  | X             | O (`ctx.key()`)             | O (`ctx.key()`)           |
| 동시성   | 자유로움          | 키별 순차 실행 (`@Shared`는 병렬 허용) | run 1회 + signal/query는 병렬 |
| 주요 용도 | Stateless API | 상태 기반 API                   | 복잡한 흐름 / 단계 기반 프로세스       |

---

## 5.1 인터페이스에 어노테이션 적용

Restate는 클래스뿐 아니라 **인터페이스에도 어노테이션을 적용**할 수 있습니다.
이 기능은 서비스 정의와 구현을 서로 다른 패키지에 분리하고자 할 때 유용합니다.

예: 인터페이스에 `@Service` 및 `@Handler` 어노테이션을 지정하고, 구현은 다른 모듈/패키지에서 작성

---

## 5.2 수동 프로젝트 설정 및 의존성 구성

### Gradle 기반 Kotlin 프로젝트 설정 예시

```bash
gradle init --type kotlin-application
```

### plugins 설정
```kotlin
plugins {
    kotlin("plugin.serialization") version "2.0.0"
    id("com.google.devtools.ksp") version "2.0.0-1.0.21"
}
```

### HTTP 서비스로 배포할 경우
```kotlin
ksp("dev.restate:sdk-api-kotlin-gen:2.0.0")
implementation("dev.restate:sdk-kotlin-http:2.0.0")
```

### Lambda 함수로 배포할 경우
```kotlin
ksp("dev.restate:sdk-api-kotlin-gen:2.0.0")
implementation("dev.restate:sdk-kotlin-lambda:2.0.0")
```

---

## 5.3 어노테이션 프로세싱 없이 수동으로 서비스 정의하기

어노테이션 기반 자동 생성 방식을 사용하지 않으려면, 아래 클래스를 직접 사용할 수 있습니다:

- [`ServiceDefinition`](https://docs.restate.dev/javadocs/dev/restate/sdk/endpoint/definition/ServiceDefinition.html)
- [`HandlerRunner` (Java)](https://docs.restate.dev/javadocs/dev/restate/sdk/HandlerRunner.html)
- [`HandlerRunner` (Kotlin)](https://docs.restate.dev/ktdocs/sdk-api-kotlin/dev.restate.sdk.kotlin/-handler-runner)

이 방식을 사용하면, 서비스 바인딩과 핸들러 실행 로직을 코드로 수동 정의할 수 있습니다.

---

## 6. 참고 자료

- [Restate 공식 문서](https://docs.restate.dev)
- [Java SDK GitHub](https://github.com/restatedev/sdk-java)
- [Services 개요](https://docs.restate.dev/develop/java/overview#services)
- [Virtual Objects 개요](https://docs.restate.dev/develop/java/overview#virtual-objects)
- [Workflows 개요](https://docs.restate.dev/develop/java/overview#workflows)

