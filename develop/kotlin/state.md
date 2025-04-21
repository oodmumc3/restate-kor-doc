# Restate 상태(State) 관리

## 목차
1. 개요
2. 상태 저장의 적용 범위
3. 상태 작업 API
   - 3.1 상태 키 조회
   - 3.2 상태 조회
   - 3.3 상태 설정
   - 3.4 특정 상태 삭제
   - 3.5 전체 상태 삭제
4. 커맨드라인 상태 인스펙션
5. 관련 링크

## 1. 개요
Restate는 Virtual Object와 Workflow에서 key-value 형태의 상태(state)를 저장하고, 코드 실행과의 정합성을 유지하도록 보장합니다.

## 2. 상태 저장의 적용 범위
- **Virtual Object**: 객체별로 상태가 격리되며, 객체 생애주기 동안 상태가 유지됩니다.
- **Workflow**: 각 실행 단위가 독립된 객체처럼 취급되며, 상태는 실행 단위별로 격리됩니다. `run` 핸들러만 상태를 변경할 수 있고, 다른 핸들러는 읽기만 가능합니다.

## 3. 상태 작업 API

### 3.1 상태 키 조회
- `ctx.stateKeys()`를 통해 해당 Virtual Object에 저장된 모든 상태 키를 조회할 수 있습니다.

### 3.2 상태 조회
```kotlin
val STRING_STATE_KEY = stateKey<String>("my-key")
val stringState: String? = ctx.get(STRING_STATE_KEY)
```
- 상태 키를 먼저 정의한 후, `ctx.get`으로 해당 키의 값을 조회합니다.

### 3.3 상태 설정
```kotlin
val STRING_STATE_KEY = stateKey<String>("my-key")
ctx.set(STRING_STATE_KEY, "my-new-value")
```
- 정의된 상태 키에 대해 `ctx.set`으로 값을 설정합니다.

### 3.4 특정 상태 삭제
```kotlin
val STRING_STATE_KEY = stateKey<String>("my-key")
ctx.clear(STRING_STATE_KEY)
```
- 정의된 상태 키에 대해 `ctx.clear`를 호출하여 해당 상태만 삭제합니다.

### 3.5 전체 상태 삭제
```kotlin
ctx.clearAll()
```
- 현재 Virtual Object에 저장된 모든 상태를 삭제합니다.

## 4. 커맨드라인 상태 인스펙션
- `psql` 및 Restate CLI를 이용하여 상태를 조회하거나 수정할 수 있습니다.
- 자세한 사용 방법은 [Introspection 문서](https://docs.restate.dev/operate/introspection#inspecting-application-state)를 참고하세요.

## 5. 관련 링크
- [상태 직렬화 타입 정의](https://docs.restate.dev/develop/java/serialization)
- [Introspection 문서](https://docs.restate.dev/operate/introspection#inspecting-application-state)

