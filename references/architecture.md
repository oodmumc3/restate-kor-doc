# Restate Architecture

Restate는 단일 바이너리로 모든 기능을 제공하므로 최소한의 설정으로도 매우 간단하게 시작할 수 있도록 설계되었습니다. 특히 단일 노드에서 Restate를 실행하여 시작할 때는 내부 아키텍처를 깊이 이해할 필요가 없습니다.

하지만 더 복잡한 배포 시나리오를 계획하기 시작하면, 확장 가능하고 복원력 있는 클러스터를 지원하기 위해 다양한 구성 요소가 어떻게 함께 작동하는지를 깊이 이해하는 것이 도움이 됩니다. 이 섹션의 목적은 서버 문서 전반에서 사용하는 용어를 소개하고, Restate 클러스터 구성 시 고려해야 할 선택 사항들을 안내하는 것입니다.

## Overview

Restate는 세 가지 계층 구조로 구성된 아키텍처를 기반으로 구현되어 있습니다: **Control Plane**, **Distributed Log**, 그리고 **Processors**입니다.

![Overview](https://docs.restate.dev/img/architecture/restate_layered_arch.png)

### Control plane

Control Plane은 서비스 배포와 관련된 모든 메타데이터(예: 어떤 서비스가 어떤 URL, 엔드포인트 주소, ARN 뒤에 위치하는지 등)를 저장합니다. 또한, 로그 파티션 및 프로세서에 대한 리더를 지정하는 역할도 담당합니다.

### Distributed Log a.k.a Bifrost

#### 초보자를 위한 설명

Restate에서 'Bifrost'는 모든 작업의 기록을 남기는 **이벤트 기록 시스템**입니다. 우리가 어떤 작업을 하든지 먼저 '기록하고' 그다음에 '처리'합니다. 예를 들어, 은행 앱에서 송금을 누르면 먼저 '누가 누구에게 얼마를 보냈다'는 기록을 남기고, 그 기록을 기준으로 실제 돈이 이동합니다. 이처럼 \*\*Bifrost는 일종의 시스템의 '기억장치'\*\*입니다.

왜 이렇게 하냐면, 문제가 생겨도 기록만 남아 있으면 다시 복구할 수 있기 때문입니다. 그리고 여러 대의 서버가 함께 동작할 때도 **하나의 정해진 기록**을 공유하면 혼란 없이 동기화할 수 있습니다.

#### 기존 설명

Restate는 **Bifrost**라 불리는 분산 로그를 사용하여 모든 이벤트를 처리하기 전에 안정적으로 기록합니다. 이는 데이터베이스 시스템의 Write-Ahead Log(WAL)와 유사한 역할을 수행합니다. Bifrost의 설계는 [Delos 논문](https://www.usenix.org/system/files/osdi20-balakrishnan.pdf)에서 설명된 **Virtual Consensus** 개념을 기반으로 하며, 핵심 합의 알고리즘은 **Flexible Paxos**를 따릅니다.

확장성 지원을 위해 Restate는 서비스 키-공간을 여러 파티션으로 나누며, 각각은 로그에 의해 지원됩니다. 현재는 하나의 파티션과 하나의 로그가 일대일로 매핑되지만, 이 관계는 향후 변경될 수 있습니다. 따라서 문맥에 따라 **partition ID**와 **log ID**가 모두 등장하는데, 이는 비슷한 값을 가지더라도 서로 다른 개념임을 인지해야 합니다.

각 Bifrost 로그는 **append-only segment**들의 체인으로 구성되며, Control Plane은 이전 segment를 닫고 새로운 segment를 연결하는 역할을 합니다. 각 segment는 **loglet**이라는 구성 요소에 의해 백업됩니다. 현재 Bifrost는 단일 노드 배포에 적합한 **로컬 loglet**, 클러스터 배포를 지원하는 **복제 loglet**을 함께 제공합니다.

### Processors

Processor(또는 Partition Processor)는 **호출 요청을 수신하고**, **지속 가능한 로그에서 이벤트를 처리하며**, **애플리케이션 또는 워크플로우 로직을 포함하는 서비스/함수와 상호작용**하는 역할을 합니다. 이들은 **내구성 있는 실행과 호출 수명 주기**를 관리하는 상태 머신을 캡슐화합니다.

Processor는 로그의 이벤트로부터 유도된 “세계의 상태(state of the world)”를 유지합니다. 이 상태는 다음과 같은 정보를 포함합니다:

- 현재 진행 중인 호출
- 각 호출의 journal
- future/promise
- 가상 객체의 실제 상태

이 상태는 **로그로부터 결정론적으로 유도된 materialized view**로 볼 수 있으며, RocksDB에 저장되고 \*\*오브젝트 저장소(object store)\*\*에 주기적으로 스냅샷됩니다. 이를 통해 장애 복구나 catch-up 상황에서 다시 적용해야 할 이벤트 수를 제한할 수 있습니다.

## Nodes and roles

![Nodes and roles](https://docs.restate.dev/img/architecture/node_internal_architecture.png)

Restate 문서에서는 'server'와 'node'라는 용어가 자주 등장합니다. 일반적으로 **server**는 `restate-server` 바이너리의 실행 인스턴스를 의미합니다. 이 하나의 바이너리는 여러 기능을 동시에 처리할 수 있습니다.

예를 들어, 단일 노드에서 로컬 개발이나 테스트 용도로 Restate 서버를 실행하는 경우, 하나의 프로세스 안에 다음과 같은 핵심 기능들이 함께 동작합니다:

- 외부 요청 수신
- 이벤트 영속적 기록
- 서비스 위임 처리 및 key-value 작업
- 시스템 내부 메타데이터 유지

### 클러스터 환경에서는?

여러 노드를 서로 다른 머신에서 실행하면서 각 노드가 역할을 분담하게 됩니다. 테스트 용도로는 단일 머신 내에 여러 노드를 띄울 수도 있습니다. 이때 각 노드는 Restate 서버의 **독립적인 인스턴스**입니다.

Restate 클러스터는 대규모 확장을 염두에 두고 설계되었기 때문에, 모든 기능을 모든 노드에 복제하는 것은 비효율적입니다. 따라서 **각 노드는 특정 역할(Role)** 만 수행하도록 구성할 수 있으며, 이를 통해 클러스터 내에서 역할별 전문화가 가능해집니다.

### 노드에서 실행 가능한 주요 역할



- **Metadata server**: 클러스터 전반의 정보를 관리하는 소스 오브 트루스
- **Ingress**: 외부 요청을 받아들이는 진입점
- **Log server**: 로그를 영속적으로 저장
- **Worker**: 파티션 프로세서를 실행하여 실제 작업 수행

## Metadata store

Metadata store는 Restate의 **Control Plane** 일부로, 클러스터 내의 노드 구성 및 역할 할당에 대한 **내부 소스 오브 트루스(source of truth, 시스템 내에서 최종적으로 믿을 수 있는 공식 데이터 저장소)** 역할을 합니다. 시스템의 정확성과 일관성을 위해 반드시 필요한 구성 요소입니다.

### 클러스터에서의 역할

클러스터 환경에서는 각 구성 요소의 설정에 대해 **분산 합의(distributed consensus)** 를 수행해야 하며, metadata store는 이 합의를 가능하게 해주는 핵심 컴포넌트입니다. Restate는 이를 위해 **Raft 기반의 메타데이터 저장소**를 내장하고 있으며, 이는 `metadata-server` 역할이 할당된 노드에 의해 호스팅됩니다.

### 특징

- 모든 노드는 metadata store에 접근할 수 있어야 합니다.
- 그러나 모든 노드가 metadata store를 호스팅할 필요는 없습니다.
- **읽기/쓰기 부하**는 비교적 낮지만, **무결성과 가용성**은 가장 높은 수준으로 보장됩니다.

## Ingress

외부의 요청은 Restate 클러스터의 **HTTP ingress 컴포넌트**를 통해 들어오며, 이는 `http-ingress` 역할이 부여된 노드에서 실행됩니다. 다른 역할들과 달리, ingress는 **장기적인 상태(state)를 유지하지 않기 때문에 유연하게 이동 가능**하며, 클라이언트와의 연결을 처리하는 데 집중합니다.

### 세분화된 ingress 역할 (Fine-grained ingress role)

Restate 1.2 버전부터는 `http-ingress` 역할이 세분화되어 독립적으로 구성될 수 있습니다. 
- 이전 버전과의 호환성을 위해, `worker` 역할을 수행하는 노드도 여전히 ingress 기능을 함께 실행합니다.
- 즉, 1.2에서는 `worker` 노드가 `http-ingress`를 **겸임**할 수 있지만, 향후에는 명확히 분리하여 운영할 수 있습니다.
## Log servers

`log-server` 역할을 수행하는 노드는 **로그를 영속적으로 저장하는 책임**을 집니다. 이 로그는 데이터베이스 시스템의 **WAL (Write-Ahead Log)** 과 같은 개념으로, Restate의 핵심적인 변경 이력을 안전하게 보존하는 역할을 합니다.

### 로그와 파티션 저장소의 관계
- 로그가 '기록'이라면, **파티션 저장소(partition store)** 는 해당 기록을 '빠르게 읽을 수 있는 형태로 만든 구조물'입니다.
- 즉, 파티션 저장소는 호출 저널(invocation journal)이나 key-value 데이터를 포함하여, 기록된 이벤트를 **효율적으로 읽기 위한 구현체**입니다.

### 로그 복제와 확장성
- Restate는 설정된 **로그 복제(log replication)** 수준에 따라 로그를 여러 `log-server` 노드에 복제합니다.
- 이 복제 메커니즘은 **유지보수**, **클러스터 확장/축소** 시에도 로그 데이터의 안정성과 가용성을 보장합니다.
## Workers

`worker` 역할이 할당된 노드는 **Partition Processor**를 실행합니다. 이 프로세서는 각 파티션의 상태를 관리하며, Restate 시스템 내에서 실제로 호출을 처리하는 핵심 컴포넌트입니다.

### 리더와 팔로워
- 각 파티션은 오직 하나의 **리더(leader)** 프로세서만 활성화될 수 있습니다.
- 리더는 서비스 호출을 실제로 처리하며, **단 하나의 진실된 실행자**입니다.
- 나머지 프로세서들은 **팔로워(follower)** 모드로 동작하며, 로그를 따라가되 실제 처리는 하지 않습니다. 대신 리더가 실패할 경우 즉시 대체할 수 있도록 준비되어 있습니다.
- **partition replication 설정**을 통해 각 파티션당 몇 개의 프로세서를 둘지 구성할 수 있습니다.

![Restate invocation flow](https://docs.restate.dev/img/architecture/invocation_flow.png)

### 상태 복제 및 장애 복구
- 각 프로세서는 자신이 속한 파티션의 로그를 읽고 적용함으로써 상태를 복제합니다.
- 예를 들어 유지보수로 잠시 멈췄다가 다시 시작하면, 빠진 로그를 다시 읽어와 상태를 복원합니다.
- 하지만 디스크 손실이나 클러스터 확장 등으로 인해 **로그 전체를 재적용하는 것보다**, 최근 스냅샷에서 **복사하는 것이 훨씬 효율적**입니다.

### 스냅샷과 로그 트리밍
- Restate는 외부 **오브젝트 스토어(object store)** 를 스냅샷 저장소로 활용할 수 있습니다.
- 이를 통해 빠르게 상태를 복원할 수 있으며, 불필요하게 커지는 로그를 **트리밍(trim)** 하는 것도 가능해집니다.
## Bifrost vs. Processors

Restate 아키텍처에서 자주 헷갈리는 두 구성 요소인 `Distributed Log (Bifrost)`와 `Processors`의 차이점을 정리한 섹션입니다.

### 📌 개념 요약

- **Bifrost (Distributed Log)**: 이벤트를 순서대로 기록하는 **내구성 있는 로그 시스템**입니다. 모든 작업은 우선 여기에 기록됩니다.
- **Processor**: 기록된 이벤트를 **읽고 처리하여 상태를 만드는 실행 단위**입니다. 이들은 실질적인 로직 수행자입니다.

### 🔍 차이점 비교

| 항목     | Bifrost (Distributed Log) | Processors               |
| ------ | ------------------------- | ------------------------ |
| 역할     | 이벤트를 기록                   | 이벤트를 처리                  |
| 기능     | 내구성 있는 로그 저장              | 상태(state) 생성 및 유지        |
| 형태     | 저장소                       | 실행 로직을 가진 컴포넌트           |
| 데이터 흐름 | 먼저 기록됨                    | 그 기록을 읽어서 처리             |
| 주요 기술  | Flexible Paxos, Loglet    | RocksDB, 상태 머신, Snapshot |

### 🔄 전체 흐름 요약

1. 요청이 들어오면 →
2. Bifrost에 이벤트로 기록 →
3. Processor가 해당 이벤트를 읽음 →
4. 처리 후 상태 저장 및 결과 반영