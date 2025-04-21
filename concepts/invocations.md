# Restate 문서 정리: Invocations

> [docs.restate.dev](https://docs.restate.dev/concepts/invocations)

## 📚 목차
1. 개요
2. Invocation 방식
3. Invocation 유형
4. 멱등 호출 (Idempotent Invocations)
5. 호출 추적 및 관찰 (Inspecting Invocations)
6. 호출 취소 및 강제 종료
7. 개념 간 관계도 (서술형)

---

## 1. 개요

Restate의 "Invocations" 문서는 Restate 서비스 내의 핸들러를 **어떻게 호출하고 관리할 수 있는지**에 대한 개념을 설명합니다. 모든 호출은 **Restate Server**를 통해 **프록시 및 추적**되며, 각 호출에는 **고유한 ID**가 부여되어 **상태 추적 및 로깅**이 가능합니다.

---

## 2. Invocation 방식

Restate에서 핸들러는 세 가지 방식으로 호출할 수 있습니다:

### ✅ HTTP 요청
- `localhost:8080/{HandlerName}` 경로로 요청
- 일반적인 REST 방식처럼 사용

### ✅ SDK 호출
- Java, TypeScript 등 다양한 SDK 제공
- 핸들러 내부 또는 외부 애플리케이션에서 호출 가능

### ✅ Kafka 이벤트
- Restate가 특정 Kafka 토픽을 구독
- 메시지가 오면 지정된 핸들러가 자동 호출됨

---

## 3. Invocation 유형

핸들러를 호출하는 방식은 동기/비동기, 지연 여부에 따라 다음과 같이 나뉩니다:

### Request-response (요청-응답 호출)
- 동기 호출 방식
- 결과를 기다렸다가 응답받음
- 예: 사용자 정보 조회, 결제 승인 등

**예시 (Java):**
```java
String result = GreeterServiceClient.fromContext(ctx)
    .greet("Hello");
System.out.println("응답: " + result);
```

### One-way (단방향 호출)
- 비동기 호출 방식
- 응답을 기다리지 않으며 Invocation ID만 반환
- 예: 이벤트 트리거, 로그 저장 등

**예시 (Java):**
```java
GreeterServiceClient.fromContext(ctx)
    .send()
    .greet("Hello");
```

### Delayed (지연 호출)
- 미래 시점에 호출 예약
- 예: 리마인더, 타임아웃 로직 등

**예시 (Java, 가상의 API 사용):**
```java
ctx.schedule(Duration.ofMinutes(10),
    () -> GreeterServiceClient.fromContext(ctx)
              .send()
              .greet("Hello in 10 minutes"));
```

| 호출 유형 | 동기 여부 | 반환 내용 | 대표 용도 |
|------------|-----------|------------|------------|
| Request-response | ✅ 동기 | 결과 응답 | 데이터 조회, 검증 |
| One-way | ❌ 비동기 | Invocation ID | 비동기 트리거 |
| Delayed | ❌ 예약 | 예약된 호출 실행 | 알림, 타이머 기반 로직 |

---

## 4. 멱등 호출 (Idempotent Invocations)

### 🧩 정의
동일한 요청이 여러 번 들어와도, **한 번만 처리된 것처럼 보장하는 호출 방식**입니다.

### 🛠️ 사용 방법
- HTTP: `idempotency-key` 헤더 추가
```bash
curl localhost:8080/GreeterService/greet \
  -H 'idempotency-key: ad5472esg4dsg525dssdfa5loi' \
  --json '"Hi"'
```
- Java SDK: `.withIdempotencyKey("...")` 사용

### 💡 특징
- 같은 키가 재사용되면 첫 호출 결과 재사용
- 아직 처리 중이면 기존 실행에 "latch-on"
- 네트워크 재시도, 웹훅 중복 요청 등에 유용

---

## 5. 호출 추적 및 관찰 (Inspecting Invocations)

### 🔍 방법 1: Restate UI
- 호출 흐름과 결과를 시각적으로 추적 가능
- 호출 인자, 결과, 상태, 실행 시간 확인 가능

### 🔍 방법 2: CLI 명령어
```bash
restate services list
restate services describe CartObject
restate invocations list
restate invocations describe inv_xxx
```

### 🔍 방법 3: OpenTelemetry 연동
- Jaeger, Grafana, Honeycomb 등과 연계 가능
- 호출 트레이스 시각화, 성능 모니터링에 활용

---

## 6. 호출 취소 및 강제 종료

### 🚫 취소 (Cancel)
- 핸들러를 **정상적으로 종료**하며 보상 로직 실행

### 🛑 강제 종료 (Kill)
- 핸들러를 **즉시 종료**하며 보상 로직 생략

### 예시 (CLI)
```bash
restate invocations cancel --kill inv_xxx
```
