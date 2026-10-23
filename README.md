# MomFit Infrastructure

MomFit 프로젝트의 **K3s 배포 설정과 Flux CD GitOps 구성**을 관리하는 저장소입니다.

- 서비스: https://team3.bssbu.co.kr
- Frontend: https://github.com/samzo-3/momfit-frontend
- Backend: https://github.com/samzo-3/momfit-backend
- Infrastructure: https://github.com/samzo-3/momfit-infra
- Chatbot: https://github.com/samzo-3/momfit-chatbot

**운영 환경:** Gabia Cloud Gen2 / Ubuntu 24.04 LTS / K3s v1.36.5+k3s1

**최근 검증일:** 2026-10-23

## 1. 인프라 구조

```text
사용자
  │ HTTPS
  ▼
team3.bssbu.co.kr
  │
  ▼
Traefik Ingress + TLS (cert-manager)
  │
  ├── /
  │    └── Frontend Service :80
  │         └── React / Nginx Pod (t3-w1)
  │
  ├── /api/chat
  │    └── Chatbot Service :8000
  │         └── FastAPI Pod :8100 (t3-w1)
  │              │
  │              ├── Backend Service :8000
  │              │    └── JWT 인증 및 사용자 정보 조회
  │              │
  │              ├── Qdrant Service :6333
  │              │    └── Qdrant Pod (t3-w2)
  │              │         └── local-path PVC (5Gi)
  │              │
  │              ├── OpenAI API (Embedding)
  │              │
  │              └── Anthropic API (LLM)
  │
  └── /api
       └── Backend Service :8000
            └── FastAPI Pod (t3-w1)
                 │
                 ├── MySQL Service :3306
                 │    └── MySQL StatefulSet (t3-w2)
                 │         └── local-path PVC (10Gi)
                 │
                 └── AI Daily Report
                      └── Anthropic API (LLM)


[Monitoring]

Prometheus / Grafana / Alertmanager
  ├── Node Exporter
  │    └── Node CPU / Memory / Disk / Network
  │
  ├── kube-state-metrics
  │    └── Deployment / Pod 상태
  │
  ├── MySQL Exporter
  │    └── DB 연결 및 성능 지표
  │
  ├── Traefik Metrics
  │    └── HTTP 요청 및 오류율
  │
  └── Backend /metrics
       └── API 요청 수 및 응답시간


K3s Control Plane: t3-cp
관리용 Bastion Host: t3-jump
```

### 서버별 역할

| 서버 | 역할 |
| --- | --- |
| `t3-jump` | Bastion Host 및 SSH 접근 중계 |
| `t3-cp` | K3s Control Plane, Flux CD 관리 |
| `t3-w1` | Frontend / Backend / Chatbot 실행 |
| `t3-w2` | MySQL / Qdrant 및 영구 저장소 |

Frontend, Backend, Chatbot은 `t3-w1`에서 실행하며 MySQL과 Qdrant는 `t3-w2`에서 운영합니다.

Chatbot Deployment에는 다음 `nodeSelector`가 적용되어 있습니다.

```yaml
nodeSelector:
  kubernetes.io/hostname: t3-w1
```

해당 설정은 Flux CD GitOps에도 반영되었으며, Pod 재생성 및 GitOps 동기화 후에도 `t3-w1` 배치를 확인하였습니다.

단, 노드 고정 방식이므로 `t3-w1` 자체가 장애를 일으키면 다른 노드로 자동 이전되지 않습니다.

### 외부 서비스 라우팅

| 경로 | 서비스 | 상태 |
| --- | --- | --- |
| `/` | Frontend | HTTP 200 확인 |
| `/api` | Backend | HTTP 200 확인 |
| `/api/chat` | AI Chatbot | HTTP 200 확인 |

Chatbot은 Kubernetes 내부에서 `ClusterIP` Service로 운영하지만, Traefik Ingress에 `/api/chat` 경로를 추가하여 HTTPS 외부 접근이 가능하도록 구성하였습니다.

Chatbot의 준비 상태는 다음 주소로 확인할 수 있습니다.

```text
https://team3.bssbu.co.kr/api/chat/ready
```

2026년 10월 23일 최종 점검에서 HTTP 200 및 `{"status":"ready"}` 응답을 확인하였습니다.

### AI 서비스 구성

**AI Chatbot**

- FastAPI 기반 독립 Chatbot 서비스
- Backend API를 통한 JWT 인증 및 사용자 정보 확인
- Qdrant 기반 운동 가이드 벡터 검색
- OpenAI API를 활용한 Embedding
- Anthropic API를 활용한 LLM 응답 생성
- Kubernetes Secret `momfit-chatbot-env` 사용

**AI Daily Report**

- Backend에 통합된 AI 운동 리포트 기능
- API: `POST /api/agent/daily-report`
- MySQL 운동 기록을 조회하여 통계 계산
- Anthropic API를 활용한 리포트 문장 생성
- Kubernetes Secret `momfit-backend-env` 사용
- `agent_step_feedback`, `agent_daily_feedback`, `daily_feedback_sources` 테이블 활용

운영 데이터베이스와 Backend ORM 모델의 컬럼 불일치 문제를 수정하였으며, 최종 스키마 검사에서 누락 컬럼이 없음을 확인하였습니다.

### 데이터 저장소

| 항목 | MySQL | Qdrant |
| --- | --- | --- |
| 실행 노드 | t3-w2 | t3-w2 |
| 워크로드 | StatefulSet | Deployment |
| 버전 | MySQL 8.4 | Qdrant 1.16.3 |
| StorageClass | local-path | local-path |
| PVC | mysql-data-mysql-0 | momfit-qdrant-data |
| 용량 | 10Gi | 5Gi |
| 외부 직접 공개 | 없음 | 없음 |

Local Path 기반 저장소는 특정 노드의 로컬 디스크에 의존하므로, 노드 또는 디스크 손실에 대비해 별도의 백업이 필요합니다.

### 운영 모니터링

`monitoring` 네임스페이스에 Prometheus, Grafana, Alertmanager 및 Exporter를 구성하였습니다.

주요 모니터링 대상은 다음과 같습니다.

- Kubernetes 노드별 CPU, 메모리 및 디스크 사용량
- Frontend, Backend, Chatbot, MySQL, Qdrant Pod 자원 사용량
- Pod 상태 및 재시작 횟수
- HTTP 요청량, 응답시간 및 5xx 오류율
- MySQL 연결 상태 및 쿼리 통계
- Backend FastAPI `/metrics`
- 서비스 장애 알림 및 Discord 연동

Grafana는 외부 Ingress로 직접 공개하지 않고 Kubernetes Service와 Port Forward를 통해 관리합니다.

**Grafana Service:** `monitoring/monitoring-grafana`

**Prometheus Service:** `monitoring/monitoring-kube-prometheus-prometheus`

### 운영 테스트 결과

2026년 10월 23일 다음 항목을 검증하였습니다.

| 검증 항목 | 결과 |
| --- | --- |
| K3s 노드 상태 | 3개 노드 Ready |
| Frontend / Backend / Chatbot | HTTPS HTTP 200 |
| MySQL 백업 및 복원 | 18개 테이블 복원 성공 |
| Qdrant Snapshot 복원 | 31개 포인트 복원 성공 |
| Backend Pod 장애 복구 | 약 14초 관측 |
| Chatbot Pod 장애 복구 | 약 11초 관측 |
| Flux CD 배포 동기화 | READY=True |
| Chatbot 노드 고정 | t3-w1 적용 확인 |
| 백업 외부 보관 | 완료 |
| k6 성능 테스트 | 수행 완료 |

Pod 장애 복구 시험에서는 Kubernetes가 새 Pod를 자동 생성하는 것을 확인하였으나, 단일 Replica 구성으로 인해 복구 중 HTTP 503 응답이 발생하였습니다.

현재 구성은 Kubernetes 기반 자동 복구를 지원하지만, 무중단 고가용성(HA) 환경은 아닙니다.

## 2. GitOps 배포 흐름

### Frontend / Backend

```text
Frontend / Backend 소스 (develop)
       ↓
GitHub Actions (테스트 · Docker 빌드)
       ↓
GHCR Private Registry (develop-N 이미지)
       ↓
Flux ImageRepository → ImagePolicy
       ↓
Flux ImageUpdateAutomation
       ↓
momfit-infra (main) 배포 이미지 태그 갱신
       ↓
Flux Kustomization
       ↓
K3s Deployment 업데이트
       ↓
Frontend / Backend Pod 재배포
```

Frontend 및 Backend는 GitHub Actions와 Flux CD를 연계한 이미지 발행 및 GitOps 배포 구조로 관리합니다.

배포 이미지 버전은 `develop-N` 형식으로 관리하며, Kubernetes Deployment의 실제 이미지 태그를 확인하여 배포 상태를 검증합니다.

### Chatbot

```text
Chatbot 소스
       ↓
Docker 이미지 빌드
       ↓
GHCR Private Registry
(ghcr.io/samzo-3/momfit-chatbot:develop-1)
       ↓
momfit-infra (main)
       ↓
Flux Kustomization
       ↓
K3s Chatbot Deployment 업데이트
       ↓
nodeSelector: t3-w1
       ↓
Chatbot Service :8000 → Pod :8100
       ↓
Traefik Ingress /api/chat
       ↓
HTTPS 외부 접근
```

Chatbot은 현재 `develop-1` 이미지로 배포됩니다.

Frontend / Backend와 같은 Flux 이미지 자동 업데이트 정책은 Chatbot에 별도로 구성하지 않았습니다.

Chatbot의 이미지 버전, Secret 연결, 리소스 설정, 포트, 상태 검사 및 노드 고정 설정은 `apps/momfit/chatbot-deployment.yaml`에서 관리합니다.

### Qdrant

```text
Qdrant Docker Image
(qdrant/qdrant:v1.16.3)
       ↓
momfit-infra (main)
       ↓
Flux Kustomization
       ↓
K3s Qdrant Deployment
       ↓
Qdrant Service :6333
       ↓
local-path PVC (5Gi)
       ↓
exercise_guides Collection
```

Qdrant는 Chatbot에서 사용하는 운동 가이드 벡터 데이터를 저장합니다.

운영 중 `exercise_guides` 컬렉션을 생성하고 운동 가이드 문서를 색인하였습니다.

2026년 10월 23일 Snapshot 백업 및 격리된 테스트 Pod 복원 과정에서 31개 포인트가 정상 복원되는 것을 확인하였습니다.

### Flux CD 동기화

Flux CD는 `momfit-infra` 저장소의 `main` 브랜치를 기준으로 Kubernetes 리소스를 동기화합니다.

| 항목 | 설정 |
| --- | --- |
| GitRepository | `flux-system` |
| 애플리케이션 Kustomization | `momfit-app` |
| Namespace | `flux-system` |
| 동기화 경로 | `./apps/momfit` |
| 기준 브랜치 | `main` |
| 최근 확인 Revision | `3e120458` |

2026년 10월 23일 기준 `flux-system`, `momfit-app` Kustomization 모두 `READY=True` 상태를 확인하였습니다.

### 현재 배포 이미지

| 서비스 | 이미지 |
| --- | --- |
| Frontend | `ghcr.io/samzo-3/momfit-frontend:develop-13` |
| Backend | `ghcr.io/samzo-3/momfit-backend:develop-20` |
| Chatbot | `ghcr.io/samzo-3/momfit-chatbot:develop-1` |
| Qdrant | `qdrant/qdrant:v1.16.3` |

위 이미지 태그는 2026년 10월 23일 실제 Kubernetes Deployment에서 확인한 값입니다.

## 3. 저장소 디렉터리 구조

```text
momfit-infra/
├── README.md
├── apps/
│   └── momfit/
│       ├── kustomization.yaml
│       ├── frontend-deployment.yaml
│       ├── backend-deployment.yaml
│       ├── chatbot-deployment.yaml
│       ├── chatbot-service.yaml
│       └── qdrant.yaml
└── clusters/
    └── gabia/
        ├── kustomization.yaml
        ├── momfit-sync.yaml
        ├── image-repositories.yaml
        ├── image-policies.yaml
        ├── image-automation.yaml
        └── flux-system/
            └── ...                  # Flux 부트스트랩 생성 파일
```

현재 저장소는 애플리케이션 배포 선언과 GitOps 동기화 설정을 중심으로 구성합니다.

MySQL, 기존 Frontend/Backend Service, Ingress, Secret 및 모니터링 리소스 전체가 이 디렉터리 구조에 포함된 것은 아닙니다.

## 4. 디렉터리 및 파일 역할

| 경로 | 역할 |
| --- | --- |
| `apps/momfit/` | MomFit 애플리케이션의 Kubernetes 배포 선언 관리 |
| `apps/momfit/kustomization.yaml` | Frontend, Backend, Chatbot, Qdrant 리소스를 Kustomize로 묶음 |
| `apps/momfit/frontend-deployment.yaml` | Frontend 이미지, Replica, 리소스 및 배포 설정 |
| `apps/momfit/backend-deployment.yaml` | Backend 이미지, 환경변수 Secret, 리소스 및 배포 설정 |
| `apps/momfit/chatbot-deployment.yaml` | Chatbot 이미지, Secret 환경변수, 포트, 상태 검사 및 `t3-w1` 노드 고정 설정 |
| `apps/momfit/chatbot-service.yaml` | Chatbot 내부 Service 설정 (`8000` → `8100`) |
| `apps/momfit/qdrant.yaml` | Qdrant Deployment, Service (`6333`), PVC (`5Gi`) 설정 |
| `clusters/gabia/` | Gabia K3s 클러스터에 적용할 Flux 설정 관리 |
| `clusters/gabia/kustomization.yaml` | 클러스터에 동기화할 Flux 리소스 목록 정의 |
| `clusters/gabia/momfit-sync.yaml` | `apps/momfit` 경로를 클러스터에 동기화하는 Flux Kustomization 정의 |
| `clusters/gabia/image-repositories.yaml` | GHCR Frontend·Backend 이미지 태그 조회 설정 |
| `clusters/gabia/image-policies.yaml` | `develop-N` 이미지 태그의 최신 버전 선택 정책 |
| `clusters/gabia/image-automation.yaml` | 선택한 이미지 버전을 Git 저장소에 자동 반영하는 정책 |
| `clusters/gabia/flux-system/` | Flux 컨트롤러 설치 및 Git 동기화용 부트스트랩 설정 |

> **GitOps 관리 범위**
>
> 이 저장소에서 관리하는 주요 대상은 Frontend, Backend, Chatbot, Qdrant의 Deployment와 일부 Service·PVC 및 Flux GitOps 설정입니다.
>
> MySQL, 기존 Frontend/Backend Service, Traefik Ingress, TLS 인증서, Kubernetes Secret, Prometheus/Grafana 등 운영 중인 모든 리소스가 이 저장소에 포함된 것은 아닙니다.

> **Secret 및 인증 정보 관리**
>
> - Private GHCR 이미지 Pull: `ghcr-auth`
> - Chatbot 환경변수: `momfit-chatbot-env`
> - Backend 환경변수: `momfit-backend-env`
> - 사용자 인증: JWT Bearer Token
> - 외부 AI 서비스: OpenAI API, Anthropic API
>
> API Key, 데이터베이스 비밀번호, JWT Secret 및 GitHub Token은 Git 저장소에 직접 등록하지 않고 Kubernetes Secret 등 별도의 보안 수단으로 관리합니다.

> **운영 데이터 및 백업 관리**
>
> MySQL과 Qdrant는 `local-path` PVC를 사용합니다. Pod 재생성 시 기존 PVC가 유지된다면 데이터를 재사용할 수 있지만, 노드 또는 로컬 디스크 손실에 대비한 백업이 필요합니다.
>
> 2026년 10월 23일 MySQL SQL Dump와 Qdrant Snapshot을 생성하고 실제 복원 검증을 완료하였습니다.
>
> MySQL 복원은 운영 DB와 분리된 시험용 데이터베이스에서, Qdrant 복원은 임시 테스트 Pod에서 진행하여 운영 데이터를 덮어쓰지 않았습니다.
>
> 운영 데이터베이스 스키마 변경은 `Base.metadata.create_all()`만으로 자동 반영되지 않으므로, Backend 코드 변경 시 ORM 모델과 실제 DB 스키마 호환성을 확인하고 필요한 마이그레이션을 별도로 수행해야 합니다.
