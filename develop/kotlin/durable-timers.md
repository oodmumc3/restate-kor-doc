# Scheduling & Timers

## 개요
Restate SDK는 **내결함성 있는(durable) 타이머 기능**을 제공합니다. 이를 통해 핸들러가 지정된 시간 동안 "잠들게(sleep)" 하거나, 미래의 특정 시점에 핸들러를 다시 호출하도록 예약(scheduling)할 수 있습니다. 이 타이머는 장애나 재시작 상황에서도 정확하게 작동합니다.

## 목차
1. 타이머의 내결함성
2. 비동기 작업 예약
3. 내결함성 sleep (Durable sleep)
4. Virtual Object에서의 sleep 처리
5. Restate Server vs. SDK의 시간 동기화

---

## 1. 타이머의 내결함성
Restate는 타이머 정보를 내부에 저장하고 관리하므로, 장애(failure)나 시스템 재시작(restart) 이후에도 정확히 타이머를 다시 트리거합니다.

## 2. 비동기 작업 예약
핸들러를 미래 시점에 다시 호출하고 싶다면 **delayed calls** 기능을 사용하면 됩니다. 관련 기능은 Restate의 "delayed calls" 문서를 참고하십시오.

## 3. 내결함성 sleep (Durable sleep)
Restate 앱에서 일정 시간 동안 대기하려면 다음과 같이 작성합니다:

```kotlin
ctx.sleep(10.seconds)
```

핸들러는 sleep 중에 **중단(suspend)** 되며, 리소스를 점유하지 않으므로 AWS Lambda 같은 환경에서 비용 절감 효과가 있습니다.

## 4. Virtual Object에서의 sleep 처리
Virtual Object는 한 번에 하나의 호출만 처리하므로, sleep 중인 경우 해당 Virtual Object는 블로킹됩니다. 이를 고려한 설계가 필요합니다.

## 5. Restate Server vs. SDK의 시간 동기화
Restate Server와 SDK 간에는 시계(clock) 동기화가 필요할 수 있습니다. 정확한 동작 보장을 위해 시스템 간 시간 차이에 주의해야 합니다.