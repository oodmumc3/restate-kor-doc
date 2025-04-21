# Restate SDK Client 사용 가이드

## 목차

1. Restate SDK Client 개요
2. UI Playground를 활용한 코드 생성
3. Restate Context가 있는 경우의 권장 방식
4. SDK Client로 핸들러 호출하기
5. Idempotent 호출 구현하기
6. 호출 결과 조회하기

## 내용 섹션 (대단원 제목 정리)

### 1. Restate SDK Client 개요

- Restate SDK 클라이언트를 사용하면 Restate 핸들러를 애플리케이션 어디서든 호출할 수 있음
- Restate Context에 접근할 수 없는 비-Restate 서비스에서만 사용 권장

### 2. UI Playground를 활용한 코드 생성

- [Restate UI](https://docs.restate.dev/develop/local_dev#restate-ui) 통해 서비스 등록 및 코드 스니펫 복사 가능
- 포트 9070에서 UI 실행 후 서비스 등록 및 Playground 접근
- Playground를 통해 원하는 언어로 서비스 호출 코드를 복사 가능

### 3. Restate Context가 있는 경우의 권장 방식

- Restate Context에 접근 가능한 경우에는 항상 Context를 통해 핸들러 호출
- 호출 정보를 상위 호출과 연결하여 트레이싱 가능
- 이는 분산 트레이싱 및 서비스 호출 체계 유지에 필수적임

### 4. SDK Client로 핸들러 호출하기

- 의존성 추가 (Kotlin/Java)

```kotlin
implementation("dev.restate:client-kotlin:2.0.0")
```

- 서비스 등록 후 클라이언트로 핸들러 호출
- 클라이언트 생성:

```kotlin
val rs = Client.connect("http://localhost:8080")
```

#### 📌 Request-response 방식

핸들러의 응답을 기다리는 방식입니다.

```kotlin
val greet: String = GreeterServiceClient.fromClient(rs).greet("Hi")
val count: Int = GreetCounterObjectClient.fromClient(rs, "Mary").greet("Hi")
```

#### 📌 One-way 방식

응답을 기다리지 않고 메시지를 보내는 방식입니다.

```kotlin
GreeterServiceClient.fromClient(rs)
    .send()
    .greet("Hi")

GreetCounterObjectClient.fromClient(rs, "Mary")
    .send()
    .greet("Hi")
```

#### 📌 Delayed 방식

호출을 특정 시간 이후로 예약하는 방식입니다.

```kotlin
GreeterServiceClient.fromClient(rs)
    .send()
    .greet("Hi", 1.seconds)

GreetCounterObjectClient.fromClient(rs, "Mary")
    .send()
    .greet("Hi", 1.seconds)
```

> 워크플로우 호출 방식은 다르므로 [워크플로우 문서](https://docs.restate.dev/develop/java/workflows#submitting-workflows-with-sdk-clients)를 참고하세요.

### 5. Idempotent 호출 구현하기

- idempotency key를 통해 중복 호출 방지 가능
- 동일 키로 24시간 이내 재호출 시 동일 응답 반환
- header 설정을 통해 다른 옵션 추가 가능

#### 📌 예제 코드 (Kotlin)

```kotlin
val rs = Client.connect("http://localhost:8080")

GreetCounterObjectClient.fromClient(rs, "Mary")
    .send()
    .greet("Hi") {
        idempotencyKey = "abcde"
    }
```

- 호출 완료 후 Restate는 결과를 24시간 동안 보존
- 동일 idempotency key로 다시 호출하면, 서비스 재실행 없이 기존 응답 반환
- 별도의 로직 없이 모든 호출을 안전하게 idempotent하게 만들 수 있음

#### 📌 헤더 추가 및 보존 기간 조정

- `idempotencyKey` 설정 외에도 요청에 헤더 추가 가능
- 보존 기간(retention time)은 Admin API로 서비스 단위 조정 가능

```bash
curl -X PATCH localhost:9070/services/MyService \
    --json '{"idempotency_retention": "2days"}'
```

- 시간 포맷은 [humantime format](https://docs.rs/humantime/latest/humantime/) 사용

- idempotency key를 통해 중복 호출 방지

- 동일 키로 24시간 이내 재호출 시 동일 응답 반환

- header 설정을 통해 다른 옵션 추가 가능

- Restate는 응답을 24시간 보존하며 동일 키에 대해 동일 응답을 반환함

### 6. 호출 결과 조회하기

- idempotency key 또는 workflow를 기반으로 결과 조회 가능
- 다음 두 가지 방식 제공:
  - attach: 실행 중인 호출의 결과를 나중에 가져옴
  - peek: 호출이 완료되었는지 확인 후 결과 조회

#### 📌 예제 코드 (Kotlin)

```kotlin
val rs = Client.connect("http://localhost:8080")

val sendResponse =
    GreeterServiceClient.fromClient(rs)
        .send()
        .greet("Hi") {
            idempotencyKey = "abcde"
        }

// ... 다른 작업 수행 ...

// 옵션 1: attach로 결과 조회
val greeting: String = sendResponse.attachSuspend().response

// 옵션 2: peek으로 완료 여부 확인 및 결과 조회
val output: Output<String> = sendResponse.getOutputSuspend().response
if (output.isReady) {
    val result = output.value
}
```

> 워크플로우에 attach하는 방법은 [워크플로우 문서](https://docs.restate.dev/develop/java/workflows#submitting-workflows-with-sdk-clients)를 참고하세요.

