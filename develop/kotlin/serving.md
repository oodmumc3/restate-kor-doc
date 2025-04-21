# Restate 서비스 실행 가이드

## 목차
1. 개요
2. 실행 방식
   - HTTP 엔드포인트 실행
   - AWS Lambda 핸들러 실행
3. Java 21 Virtual Threads 지원
4. 요청 검증: Request Identity

---

## 1. 개요
Restate 서비스는 다음 두 가지 방식으로 실행할 수 있습니다:
- HTTP 엔드포인트로 실행
- AWS Lambda 함수로 실행

두 방식 모두 동일한 서비스 구현체를 사용할 수 있으며, 배포 대상 환경에 따라 실행 방식만 변경하면 됩니다.


## 2. 실행 방식

### 2.1 HTTP 엔드포인트 실행

#### 2.1.1 의존성
- Kotlin: `dev.restate:sdk-kotlin-http:2.0.0`
- Java: `dev.restate:sdk-java-http:2.0.0`

#### 2.1.2 기본 구조 (Kotlin 예시)
```kotlin
fun main() {
  RestateHttpServer.listen(
      endpoint {
        bind(MyService())
        bind(MyVirtualObject())
        bind(MyWorkflow())
      })
}
```
- 기본 포트: 9080
- 여러 서비스 바인딩 가능


### 2.2 AWS Lambda 핸들러 실행

#### 2.2.1 의존성
- Kotlin: `dev.restate:sdk-kotlin-lambda:2.0.0`
- Java: `dev.restate:sdk-java-lambda:2.0.0`

#### 2.2.2 기본 구조 (Kotlin 예시)
```kotlin
import dev.restate.sdk.endpoint.Endpoint
import dev.restate.sdk.lambda.BaseRestateLambdaHandler

class MyLambdaHandler : BaseRestateLambdaHandler() {
  override fun register(builder: Endpoint.Builder) {
    builder.bind(MyService()).bind(MyVirtualObject())
  }
}
```

#### 2.2.3 특징
- 서비스 구현은 HTTP 실행 방식과 동일하게 유지 가능
- 자세한 배포 방식은 "Deployment" 섹션 참고


## 3. Java 21 Virtual Threads
Java 21에서 도입된 Virtual Threads를 사용할 수 있으며, 고성능 요청 처리에 적합합니다. (자세한 내용은 문서에 별도로 기술되어야 함)


## 4. 요청 검증: Request Identity

Restate SDK는 요청의 출처를 검증할 수 있는 기능을 제공합니다.

#### 4.1 의존성
```kotlin
dependencies {
  implementation("dev.restate:sdk-request-identity:2.0.0")
}
```

#### 4.2 예시 코드
```kotlin
import dev.restate.sdk.auth.signing.RestateRequestIdentityVerifier
import dev.restate.sdk.http.vertx.RestateHttpServer
import dev.restate.sdk.kotlin.endpoint.endpoint

fun main() {
  RestateHttpServer.listen(
      endpoint {
        bind(MyService())
        requestIdentityVerifier =
            RestateRequestIdentityVerifier.fromKeys(
                "publickeyv1_w7YHemBctH5Ck2nQRQ47iBBqhNHy4FV7t2Usbye2A6f",
            )
      })
}
```

- 위 코드를 통해 특정 Public Key를 통해 요청의 신뢰성을 검증할 수 있습니다.
- 보안 관련 세부사항은 Security 문서 참고