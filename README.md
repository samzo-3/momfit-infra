# MomFit Infrastructure

MomFit 프로젝트의 **K3s 배포 설정과 Flux CD GitOps 구성**을 관리하는 저장소입니다.

- 서비스: https://team3.bssbu.co.kr
- Frontend: https://github.com/samzo-3/momfit-frontend
- Backend: https://github.com/samzo-3/momfit-backend
- Infrastructure: https://github.com/samzo-3/momfit-infra
- chatbot: https://github.com/samzo-3/momfit-chatbot

## 1. 인프라 구조

```text
사용자
  │ HTTPS
  ▼
team3.bssbu.co.kr
  │
  ▼
Traefik Ingress + TLS (cert-manager)
  ├── /      → Frontend Service :80   → React / Nginx Pod (t3-w1)
  └── /api   → Backend Service :8000  → FastAPI Pod (t3-w1)
                                            │
                                            ▼
                                    MySQL Service :3306
                                            │
                                            ▼
                                    MySQL StatefulSet (t3-w2)
                                            │
                                    local-path PVC (10Gi)


[AI Chatbot 내부 서비스]

Chatbot Service :8000 → FastAPI Pod :8100 (t3-w1)
                            │
                            ├── Backend Service :8000
                            │      └── JWT 인증 및 사용자 정보 조회
                            │
                            ├── Qdrant Service :6333
                            │      └── Qdrant Pod (t3-w2)
                            │             └── local-path PVC (5Gi)
                            │
                            ├── OpenAI API (Embedding)
                            │
                            └── Anthropic API (LLM)


K3s Control Plane: t3-cp
관리용 Bastion Host: t3-jump
```

Chatbot은 현재 Kubernetes 내부 `ClusterIP` Service로 배포되어 있습니다. Frontend에서 사용하기 위한 외부 API 라우팅 또는 Backend 프록시 연결은 별도 구성이 필요합니다.

| 서버 | 역할 |
| --- | --- |
| `t3-jump` | Bastion Host 및 SSH 접근 중계 |
| `t3-cp` | K3s Control Plane, Flux CD 관리 |
| `t3-w1` | Frontend / Backend / Chatbot 실행 |
| `t3-w2` | MySQL / Qdrant 및 영구 저장소 |

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
Flux Kustomization → K3s Deployment 업데이트
```

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
Chatbot Service :8000 → Pod :8100
```

Chatbot은 현재 `develop-1` 이미지로 배포되며, Frontend / Backend와 같은 Flux 이미지 자동 업데이트 정책은 아직 별도로 구성하지 않았습니다.

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
```

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

## 4. 디렉터리 및 파일 역할

| 경로 | 역할 |
| --- | --- |
| `apps/momfit/` | MomFit 애플리케이션의 Kubernetes 배포 선언 관리 |
| `apps/momfit/kustomization.yaml` | Frontend, Backend, Chatbot, Qdrant 리소스를 Kustomize로 묶음 |
| `apps/momfit/frontend-deployment.yaml` | Frontend 컨테이너 이미지 및 Deployment 설정 |
| `apps/momfit/backend-deployment.yaml` | Backend 컨테이너 이미지 및 Deployment 설정 |
| `apps/momfit/chatbot-deployment.yaml` | Chatbot 컨테이너 이미지, Secret 환경변수, 포트 및 상태 검사 설정 |
| `apps/momfit/chatbot-service.yaml` | Chatbot 내부 Service 설정 (`8000` → `8100`) |
| `apps/momfit/qdrant.yaml` | Qdrant Deployment, Service (`6333`), PVC (`5Gi`) 설정 |
| `clusters/gabia/` | Gabia K3s 클러스터에 적용할 Flux 설정 관리 |
| `clusters/gabia/kustomization.yaml` | 클러스터에 동기화할 Flux 리소스 목록 정의 |
| `clusters/gabia/momfit-sync.yaml` | `apps/momfit` 경로를 클러스터에 동기화하는 Flux Kustomization 정의 |
| `clusters/gabia/image-repositories.yaml` | GHCR Frontend·Backend 이미지 태그 조회 설정 |
| `clusters/gabia/image-policies.yaml` | `develop-N` 이미지 태그의 최신 버전 선택 정책 |
| `clusters/gabia/image-automation.yaml` | 선택한 이미지 버전을 Git 저장소에 자동 반영하는 정책 |
| `clusters/gabia/flux-system/` | Flux 컨트롤러 설치 및 Git 동기화용 부트스트랩 설정 |

> 이 저장소에서 관리하는 주요 대상은 Frontend, Backend, Chatbot, Qdrant의 Deployment와 일부 Service·PVC 및 Flux GitOps 설정입니다. MySQL, 기존 Frontend/Backend Service, Ingress, Secret 등 운영 중인 모든 리소스가 이 저장소에 포함된 것은 아닙니다.
>
> Chatbot API 인증에는 JWT Bearer Token을 사용하며, OpenAI·Anthropic API Key는 Git 저장소가 아닌 Kubernetes Secret(`momfit-chatbot-env`)으로 관리합니다.
