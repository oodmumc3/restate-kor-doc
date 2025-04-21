# Restate Journaling Results 문서 정리

## 목차
1. 개요
2. 핵심 개념 정리
    - Journaled Actions
    - DurableFuture 조합 도구
    - 결정론적 Random 생성기
3. 실패 처리 및 재시도 정책
4. 관련 문서 링크
5. 개념 간 관계 정리 (텍스트 및 PlantUML)

---

## 1. 개요
Restate는 실패나 중단 이후의 재실행(replay)을 위해 실행 로그(execution log)를 사용합니다. 따라서 데이터베이스 응답이나 UUID 생성과 같은 **비결정적(non-deterministic)** 결과는 반드시 실행 로그에 저장되어야 합니다.

이를 지원하기 위해 Restate SDK는 다음과 같은 기능을 제공합니다:

- **Journaled actions**: 코드 블록을 실행하고 그 결과를 저장합니다. 이후 재시도 시에는 코드를 재실행하는 대신 저장된 결과를 재생합니다.
- **DurableFuture 조합 도구**: 여러 `DurableFuture`의 완료/실패 순서를 기록하여 재실행 시에도 동일한 순서를 보장합니다.
- **Random generators**: 안정적으로 UUID와 난수를 생성하는 헬퍼 기능을 제공합니다. 이는 재시도 시에도 동일한 값을 반환합니다.

## 2. 핵심 개념 정리
### Journaled Actions

**Journaled Actions**는 비결정적인 작업(DB 요청, 외부 API 호출 등)의 실행 결과를 Restate의 실행 로그에 저장하는 기능입니다. 이렇게 저장된 결과는 재시도 시에 동일하게 재생되어 **결정론적 재실행(deterministic replay)** 을 가능하게 합니다.

#### 사용 예시 (Kotlin)
```kotlin
val output: String = ctx.runBlock { doDbRequest() }
```

`doDbRequest()`가 DB에 요청을 보내는 비결정적 작업이라면, 이 결과가 실행 로그에 저장되어 재시도 시에는 다시 DB에 요청하지 않고 이전 결과를 재생합니다.

#### 제약 사항
- `ctx.run` 내부에서는 Restate의 컨텍스트 기능(예: 상태 접근, 다른 서비스 호출, 중첩된 journaled action 등)을 사용할 수 없습니다.

#### 실패 처리
- `ctx.run` 내부에서의 실패는 일반 핸들러 코드의 실패와 동일한 처리 방식이 적용됩니다.
- 기본적으로 무한 재시도가 설정되어 있으며, `RetryPolicy`로 재시도 정책을 커스터마이징할 수 있습니다.
- 재시도 횟수를 초과하면 `TerminalException`이 발생하며, 이 예외를 통해 롤백이나 오류 전파 처리를 수행할 수 있습니다.

#### 관련 문서
- [RetryPolicy 설정](https://docs.restate.dev/develop/java/journaling-results#run-block-retry-policies)
- [에러 처리 및 Sagas](https://docs.restate.dev/guides/sagas)

### DurableFuture 조합 도구

`ctx.run`, 서비스 호출, `sleep`, `awakeables` 등의 작업은 모두 `DurableFuture`를 반환합니다. Restate SDK는 이 `DurableFuture`들을 결합하거나 대기할 수 있는 기능을 제공하며, **해당 작업들의 완료 순서를 실행 로그에 저장**하여 재시도 시에도 결정론적인 결과를 보장합니다.

#### awaitAll
모든 `DurableFuture`가 완료될 때까지 기다립니다. `CompletableFuture.allOf()`와 유사한 기능이지만, 결과가 실행 로그에 저장되어 재시도 시에도 동일한 동작을 보장합니다.

```kotlin
listOf(a1, a2, a3).awaitAll()
```

#### select (first-complete)
여러 `DurableFuture` 중 가장 먼저 완료되는 결과를 선택합니다. `CompletableFuture.anyOf()`와 유사하며, 완료 순서 역시 실행 로그에 저장됩니다.

```kotlin
val resSelect =
    select {
        a1.onAwait { it }
        a2.onAwait { it }
        a3.onAwait { it }
    }.await()
```

#### 요약
- `awaitAll()`: 모든 작업이 완료될 때까지 기다림
- `select {}`: 가장 먼저 완료된 작업의 결과를 사용
- 재시도 시에도 동일한 순서로 재실행되도록 보장함

### 결정론적 Random 생성기

Restate SDK는 **재시도 시에도 동일한 값을 반환하는** UUID 및 난수 생성 도구를 제공합니다. 내부적으로는 요청 단위로 고정된 시드를 사용하여 동일한 값이 생성되도록 보장합니다.

#### UUID 생성
`ctx.random().nextUUID()`를 사용하면 **결정론적인 UUID**를 생성할 수 있습니다. 이는 idempotency key 생성 등에 유용합니다.

```kotlin
val uuid: UUID = ctx.random().nextUUID()
```

> 참고: 보안이나 암호화 용도에는 적합하지 않습니다.

#### 난수 생성
`ctx.random()`은 `java.util.Random`과 유사한 API를 제공합니다. 다음은 정수 난수를 생성하는 예입니다:

```kotlin
val value: Int = ctx.random().nextInt()
```

- `nextBoolean()`
- `nextLong()`
- `nextFloat()`
- `nextDouble()` 등 다양한 메서드를 사용할 수 있습니다.

이러한 값들은 실행 로그에 기록되므로, 재실행 시에도 동일한 값이 재사용됩니다.

## 3. 실패 처리 및 재시도 정책

`ctx.runBlock`은 실패 시 기본적으로 **무한 재시도**를 수행합니다. 하지만 `RetryPolicy`를 통해 재시도 동작을 세밀하게 제어할 수 있습니다.

### RetryPolicy 구성 항목
- `initialDelay`: 첫 재시도 전 대기 시간 (예: `5.seconds`)
- `exponentiationFactor`: 지수 백오프 계수 (예: `2.0f`)
- `maxDelay`: 재시도 간 최대 대기 시간 (예: `60.seconds`)
- `maxAttempts`: 최대 재시도 횟수 (예: `10`)
- `maxDuration`: 전체 재시도 최대 지속 시간 (예: `5.minutes`)

### 예제 (Kotlin)
```kotlin
try {
    val myRunRetryPolicy = RetryPolicy(
        initialDelay = 5.seconds,
        exponentiationFactor = 2.0f,
        maxDelay = 60.seconds,
        maxAttempts = 10,
        maxDuration = 5.minutes
    )

    ctx.runBlock("write", myRunRetryPolicy) {
        writeToOtherSystem()
    }

} catch (e: TerminalException) {
    // 모든 재시도 실패 후의 처리 로직
    // 예: Sagas 패턴을 통한 롤백 처리 및 오류 전파
    throw e
}
```

### 실패 시 동작
- 지정된 최대 시도 횟수를 초과하면 `TerminalException`이 발생합니다.
- `TerminalException`을 catch하여 적절한 후속 조치를 취할 수 있습니다.

### 관련 문서
- [RetryPolicy KotlinDoc](https://docs.restate.dev/ktdocs/sdk-api-kotlin/dev.restate.sdk.kotlin/-retry-policy/index.html)
- [RetryPolicy JavaDoc](https://docs.restate.dev/javadocs/dev/restate/sdk/common/RetryPolicy.html)
- [에러 처리 가이드](https://docs.restate.dev/develop/java/error-handling)
- [Sagas 가이드](https://docs.restate.dev/guides/sagas)

## 5. 관련 문서 링크
- [Serialization](https://docs.restate.dev/develop/java/serialization)
- [RetryPolicy KotlinDoc](https://docs.restate.dev/ktdocs/sdk-api-kotlin/dev.restate.sdk.kotlin/-retry-policy/index.html)
- [RetryPolicy JavaDoc](https://docs.restate.dev/javadocs/dev/restate/sdk/common/RetryPolicy.html)
- [Error Handling](https://docs.restate.dev/develop/java/error-handling)
- [Sagas Guide](https://docs.restate.dev/guides/sagas)



