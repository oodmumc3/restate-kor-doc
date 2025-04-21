## Restate Cluster Deployment

### 목차

1. Quickstart using Docker
2. Migrating an existing single-node deployment
3. Configuration
4. Cluster provisioning

---

### 1. Quickstart using Docker

이 단원은 Docker 및 Docker Compose를 사용하여 3개의 노드로 구성된 분산형 Restate 클러스터를 배포하는 방법을 설명합니다.

#### 사전 조건

- Docker
- Docker Compose

#### 클러스터 배포

[docker-compose.yml](https://docs.restate.dev/guides/cluster/) 파일을 작성하고 `docker compose up` 명령어를 실행하여 클러스터를 시작합니다. 예제 구성은 다음과 같습니다:

- `RESTATE_CLUSTER_NAME`: 모든 노드에서 동일한 클러스터 이름 사용
- `RESTATE_NODE_NAME`: 노드마다 고유해야 함
- `RESTATE_ADVERTISED_ADDRESS`: 노드 간 통신 주소
- `RESTATE_AUTO_PROVISION`: 하나의 노드만 true 설정
- 로그 및 메타데이터 서버는 모두 `replicated` 설정
- Snapshot 설정은 MinIO를 통해 객체 저장소에 저장됨

3개의 노드가 메타데이터 서버 역할을 수행하며, Bifrost 로그 제공자는 replicated 모드로 설정됩니다. Replication factor가 2이므로 1개의 노드 장애를 견딜 수 있습니다.

MinIO는 snapshot 저장소 역할을 합니다.

#### 클러스터 상태 확인

```bash
docker compose exec restate-1 restatectl status
```

#### 서비스 시작 및 등록

```bash
restate dp register http://host.docker.internal:9080
```

또는 브라우저에서 [http://localhost:9080](http://localhost:9080) 열기

#### 서비스 호출 예시

```bash
curl localhost:8080/Greeter/greet --json '"Sarah"'
curl localhost:28080/Greeter/greet --json '"Bob"'
curl localhost:38080/Greeter/greet --json '"Eve"'
```

#### 서버 중단 및 재시작 테스트

```bash
docker compose kill restate-1
sleep 5
docker compose up -d restate-1
```

#### 스냅샷 생성

```bash
docker compose exec restate-1 restatectl snapshot create
```

MinIO 콘솔: [http://localhost:9000](http://localhost:9000) (기본 계정: minioadmin/minioadmin)

🎉 첫 분산 클러스터 실행과 장애 복구 시뮬레이션 성공!

**다음 단계 제안:**

- 5 노드 클러스터 구성 및 2 노드 장애 허용 테스트
- 로그 자동 정리 설정
- Kubernetes를 이용한 3 노드 클러스터 배포

---

### 2. Migrating an existing single-node deployment

이 단원에서는 기존의 단일 노드 Restate 배포를 데이터 손실 없이 다중 노드 클러스터로 마이그레이션하는 방법을 설명합니다.

기본적으로 v1.2 이하 버전의 Restate 서버는 **로컬 로그 저장소(local loglet)** 및 **로컬 메타데이터 서버(local metadata server)** 를 사용합니다. 이는 개발 환경과 단일 노드 배포에 적합하지만, 운영 환경에서는 **replicated loglet**과 **replicated metadata server** 사용이 권장됩니다.

이 가이드는 로컬 로그 및 메타데이터 서버를 사용하는 단일 노드 환경에서 출발하여 다중 노드로 확장하는 과정을 단계별로 안내합니다.

#### 1. 메타데이터 서버를 replicated로 변경

`restate.toml`

```toml
[metadata-server]
type = "replicated"
```

해당 설정으로 서버를 재시작하면 로컬 메타데이터가 자동으로 replicated 서버로 마이그레이션됩니다.

또한, 다중 노드 구성을 위한 **스냅샷 저장소** 설정도 필요합니다:

```toml
[worker.snapshots]
destination = "s3://snapshots-bucket/cluster-prefix"
```

#### 2. 로그 제공자를 replicated로 변경

`restate.toml`에서 다음과 같이 역할을 지정:

```toml
roles = [
  "worker",
  "admin",
  "metadata-server",
  "log-server"
]
```

그리고 `restatectl`을 통해 로그 제공자를 변경:

```bash
restatectl config set --log-provider replicated --log-replication 1
```

단일 노드에서는 replication을 **1**로 설정해야 서버가 정상 작동합니다.

#### 3. 변경 사항 확인

```bash
restatectl status
```

#### 4. 스냅샷 생성

다른 노드가 클러스터에 참여할 수 있도록 모든 파티션에 대해 스냅샷을 생성해야 합니다:

```bash
restatectl snapshots create
```

#### 5. 다중 노드로 확장

신규 노드를 추가할 때는 다음 설정을 사용해야 합니다:

```toml
cluster-name = "my-cluster"

[metadata-client]
addresses = ["ADVERTISED_ADDRESS_OF_EXISTING_NODE"]

[metadata-server]
type = "replicated"
```

#### 6. 클러스터 상태 확인

모든 노드가 정상적으로 구성되었다면, 다음 명령으로 상태를 확인할 수 있습니다:

```bash
restatectl status
```

🎉 기존 단일 노드 Restate 서버를 멀티 노드 클러스터로 성공적으로 마이그레이션 완료!

### 3. Configuration

이 단원에서는 외부 종속성 없이 분산형 Restate 클러스터를 배포하기 위한 핵심 설정 항목을 설명합니다. 모든 설정은 `restate.toml` 파일에서 정의되며, 클러스터를 구성하는 각 노드가 특정 요건을 충족해야 합니다.

#### 아키텍처 개요 참고

이 페이지에서 사용하는 용어에 익숙하지 않다면, 먼저 [아키텍처 레퍼런스](https://docs.restate.dev/references/architecture)를 참고하세요.

#### 스냅샷 중요성

- 스냅샷은 **로그 트리밍을 안전하게 수행**하고, **빠른 파티션 장애 전환(fail-over)** 및 **노드 추가를 가능**하게 만듭니다.
- 분산 클러스터에서 스냅샷 기능은 필수이며, 미리 활성화하는 것이 좋습니다.

#### 기본 설정 예시 (`restate.toml`)

```toml
# 노드별로 고유한 이름 필요
node-name = "UNIQUE_NODE_NAME"

# 모든 노드가 동일한 클러스터 이름 사용
cluster-name = "CLUSTER_NAME"

# 노드 간 충돌이 없는 광고 주소
advertised-address = "ADVERTISED_ADDRESS"

# 하나의 노드만 auto-provision 가능
auto-provision = false

# 로그 및 파티션 복제 수 설정 (예: 2)
default-replication = 2

[bifrost]
# 분산형 배포에서는 반드시 replicated 사용
default-provider = "replicated"

[metadata-server]
# 고가용성을 위한 replicated 설정
type = "replicated"

[metadata-client]
# metadata-server 역할을 가진 노드의 주소 목록
addresses = ["ADVERTISED_ADDRESS", "ADVERTISED_ADDRESS_NODE_X"]

[admin]
bind-address = "ADMIN_BIND_ADDRESS"

[ingress]
bind-address = "INGRESS_BIND_ADDRESS"

[admin.query-engine]
pgsql-bind-address = "PGSQL_BIND_ADDRESS"
```

#### 설정 요약

- **node-name**: 각 노드에 고유하게 지정해야 함
- **cluster-name**: 클러스터 내 모든 노드가 동일한 이름 사용
- **auto-provision**: 한 노드만 true로 설정 가능, 없을 경우 수동으로 프로비저닝 필요
- **default-replication**: 복제 수를 지정하며, `2 * replication - 1` 개의 노드가 필요함
- **metadata-server**: `replicated`로 설정하여 장애 허용
- **metadata-client**: 메타데이터 노드 주소를 모두 포함
- **worker 역할**을 가진 노드는 ingress 서버도 함께 실행되어 요청을 수신함
- **같은 머신에서 실행되는 노드** 간에는 포트 충돌이 없도록 주의해야 함

### 4. Cluster provisioning

이 단원에서는 클러스터 프로비저닝(초기화) 과정을 설명합니다.

#### 자동 프로비저닝

`auto-provision = true`로 설정된 노드를 시작하면, 해당 노드는 다음 작업을 수행하여 클러스터를 자동으로 프로비저닝합니다:

- 메타데이터 저장소 초기화
- 초기 `NodesConfiguration`을 작성하고 저장소에 기록
- 이후 다른 노드들이 클러스터에 참여할 수 있도록 준비됨

> 클러스터에 오직 하나의 노드만 `auto-provision = true`로 설정할 수 있습니다.

#### 수동 프로비저닝

모든 노드에서 `auto-provision = false`인 경우, 다음 명령어를 사용하여 수동으로 클러스터를 프로비저닝해야 합니다:

```bash
restatectl provision --address <SERVER_TO_PROVISION> --yes
```

이 명령은 서버 설정 파일에 지정된 기본 설정을 사용하여 클러스터를 초기화합니다.

