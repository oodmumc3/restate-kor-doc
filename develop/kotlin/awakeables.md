# Restate - Awakeables

## 목차
1. 개요
2. 주요 개념 정리
3. Awakeable 생성 및 사용 흐름
4. Awakeable 완료 처리
5. 비용 최적화 관점
6. Virtual Object와의 관계

---

## 1. 개요

Awakeable은 특정 작업의 완료를 기다리는 동안 Restate 핸들러의 실행을 일시 중단시키는 메커니즘입니다. 이는 외부 서비스 또는 비동기 처리 결과를 기다려야 할 때 유용하며, 흔히 "callback pattern" 혹은 "task token pattern"으로도 알려져 있습니다.

이를 통해 핸들러는 외부에서 작업을 수행하도록 위임하고, 작업이 완료되면 결과를 받아 재개할 수 있습니다. 특히 FaaS(Function-as-a-Service) 환경에서 비용을 절감할 수 있는 중요한 패턴입니다.

---

## 2. 주요 개념 정리

| 용어 | 설명 |
|------|------|
| **Awakeable** | 외부 작업 완료를 기다리기 위해 생성된 객체. 고유 ID와 `DurableFuture`를 포함함 |
| **awakeable.id** | 외부 작업에서 사용하기 위해 전달되는 고유 식별자 |
| **ctx.runBlock** | 핸들러 재시도 시 작업이 중복 실행되지 않도록 보장하는 블록 |
| **awakeable.await()** | 외부 작업의 완료를 기다리며 핸들러를 일시 중단 |
| **ctx.awakeableHandle(id)** | 외부에서 해당 ID의 awakeable을 참조하여 resolve 또는 reject 수행 |

---

## 3. Awakeable 생성 및 사용 흐름

1. Awakeable 생성
```kotlin
val awakeable = ctx.awakeable<String>()
val awakeableId: String = awakeable.id
```

2. 외부 작업 트리거 및 awakeable ID 전달
```kotlin
ctx.runBlock {
    triggerTaskAndDeliverId(awakeableId) // Kafka, HTTP 등 사용 가능
}
```

3. 결과 대기
```kotlin
val payload: String = awakeable.await()
```

---

## 4. Awakeable 완료 처리

Awakeable은 두 가지 방식으로 완료할 수 있습니다:

### 1. 성공 (resolve)
- HTTP:
```bash
curl localhost:8080/restate/awakeables/<id>/resolve \
     --json '{"hello": "world"}'
```
- SDK:
```kotlin
ctx.awakeableHandle(awakeableId).resolve("hello")
```

### 2. 실패 (reject)
- HTTP:
```bash
curl localhost:8080/restate/awakeables/<id>/reject \
     --json 'Very bad error!'
```
- SDK:
```kotlin
ctx.awakeableHandle(awakeableId).reject("my error reason")
```

실패 시 대기 중인 핸들러에서는 terminal error가 발생합니다.

---

## 5. 비용 최적화 관점 (FaaS)

AWS Lambda와 같은 Function-as-a-Service 플랫폼에서는 awakeable이 완료될 때까지 핸들러 실행을 일시 중단하므로, 과금이 발생하지 않습니다. 이는 Restate의 큰 장점 중 하나로, 비동기 외부 작업을 많이 처리하는 서비스에서 비용을 대폭 절감할 수 있습니다.

---

## 6. Virtual Object와의 관계

Virtual Object는 동시에 하나의 요청만 처리할 수 있으므로, awakeable 대기 상태에서는 해당 Virtual Object 전체가 블로킹됩니다. 이는 설계 시 고려가 필요한 요소입니다.

---

## 참고 링크
- [에러 핸들링 문서](https://docs.restate.dev/develop/java/error-handling)
- [직렬화 문서](https://docs.restate.dev/develop/java/serialization)

