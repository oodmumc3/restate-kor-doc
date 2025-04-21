# Restate 한국어 위키 문서 모음 📘

이 저장소는 [Restate](https://docs.restate.dev)의 개념, 실행 모델, 개발 방법 등을 한국어로 정리한 문서 모음입니다. Restate는 서버리스 환경에서 **내결함성(fault-tolerance)**과 **durable execution**을 지원하는 런타임 프레임워크로, 복잡한 워크플로우 및 상태 기반 처리를 쉽게 구현할 수 있도록 도와줍니다.

## 📁 디렉토리 구성

```
concepts/
  durable_building_blocks.md      # Durable Execution의 핵심 개념 및 예시
  durable_execution.md            # 전체 실행 흐름 및 시퀀스 다이어그램
  invocations.md                  # Invocation 방식 및 멱등 호출 등
  services.md                     # Services, Virtual Objects, Workflows 개요

develop/kotlin/
  awakeables.md                   # Awakeable (외부 완료 이벤트 대기)
  clients.md                      # SDK Client 사용법
  durable-timers.md               # Durable Timer 및 sleep
  error-handling.md              # 에러 처리 전략 및 TerminalException
  journaling-results.md          # Journaled Actions 및 결정론적 실행
  overview.md                     # Kotlin SDK 개요
  service-communication.md       # 서비스 간 통신 방식
  serving.md                      # 실행 방식 (HTTP / Lambda)
  state.md                        # Virtual Object의 상태 저장 및 관리
  workflows.md                    # Workflows 개념 및 실행 흐름

deploy/
  cluster.md                      # 클러스터 배포 및 스냅샷 구성 방법

references/
  architecture.md                 # Restate 아키텍처 (Bifrost, Processors 등)

README.md                         # 이 파일
```

## 📚 주요 문서 요약

### ✅ Durable Execution
Restate의 핵심 기능으로, 실행 중간 상태를 **journal**에 저장하여 실패 시에도 **정확한 지점에서 재실행**할 수 있도록 합니다. PlantUML 시퀀스 다이어그램 및 코드 예시가 포함되어 있습니다.

### ✅ Invocation 방식
- 요청-응답, 단방향, 지연 호출
- 멱등 호출 (Idempotency key)
- 호출 추적 (UI, CLI, OpenTelemetry)

### ✅ Services / Virtual Objects / Workflows
- Service: 상태 없는 기본 핸들러
- Virtual Object: 상태(K/V)를 저장하는 키 기반 객체
- Workflow: 복잡한 흐름을 관리하는 단일 실행 단위 (Promise 사용 가능)

### ✅ Kotlin SDK 가이드
- SDK Client 생성 및 핸들러 호출
- Durable Timer 및 sleep 기능
- Awakeable을 통한 외부 비동기 작업 연계
- DurableFuture 조합 및 재시도 정책 설정

### ✅ 클러스터 구성
- Docker 기반 3노드 클러스터 예시
- MinIO 기반 스냅샷 저장소 설정
- replicated metadata/loglet 구성 및 마이그레이션 전략

## 🔗 참고 링크

- [공식 문서](https://docs.restate.dev)
- [Kotlin SDK GitHub](https://github.com/restatedev/sdk-java)
- [Durable Execution 개념](https://docs.restate.dev/concepts/durable_execution)
- [워크플로우 예제](https://docs.restate.dev/develop/java/workflows)

## 📄 라이선스

MIT

---

> 본 문서는 AI 분석을 기반으로 자동 생성된 내용이며, 최신 Restate 문서와의 정합성을 유지하기 위해 공식 문서와 병행하여 확인하는 것을 권장합니다.
