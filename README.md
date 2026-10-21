# MomFit Infrastructure

MomFit 프로젝트의 **K3s 배포 설정과 Flux CD GitOps 구성**을 관리하는 저장소입니다.

- 서비스: https://team3.bssbu.co.kr
- Frontend: https://github.com/samzo-3/momfit-frontend
- Backend: https://github.com/samzo-3/momfit-backend

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

K3s Control Plane: t3-cp
관리용 Bastion Host: t3-jump
```

| 서버 | 역할 |
| --- | --- |
| `t3-jump` | Bastion Host 및 SSH 접근 중계 |
| `t3-cp` | K3s Control Plane, Flux CD 관리 |
| `t3-w1` | Frontend / Backend 실행 |
| `t3-w2` | MySQL 및 영구 저장소 |

## 2. GitOps 배포 흐름

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

## 3. 저장소 디렉터리 구조

```text
momfit-infra/
├── README.md
├── apps/
│   └── momfit/
│       ├── kustomization.yaml
│       ├── frontend-deployment.yaml
│       └── backend-deployment.yaml
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
| `apps/momfit/kustomization.yaml` | Frontend·Backend Deployment 파일을 Kustomize 리소스로 묶음 |
| `apps/momfit/frontend-deployment.yaml` | Frontend 컨테이너 이미지 및 Deployment 설정 |
| `apps/momfit/backend-deployment.yaml` | Backend 컨테이너 이미지 및 Deployment 설정 |
| `clusters/gabia/` | Gabia K3s 클러스터에 적용할 Flux 설정 관리 |
| `clusters/gabia/kustomization.yaml` | 클러스터에 동기화할 Flux 리소스 목록 정의 |
| `clusters/gabia/momfit-sync.yaml` | `apps/momfit` 경로를 클러스터에 동기화하는 Flux Kustomization 정의 |
| `clusters/gabia/image-repositories.yaml` | GHCR Frontend·Backend 이미지 태그 조회 설정 |
| `clusters/gabia/image-policies.yaml` | `develop-N` 이미지 태그의 최신 버전 선택 정책 |
| `clusters/gabia/image-automation.yaml` | 선택한 이미지 버전을 Git 저장소에 자동 반영하는 정책 |
| `clusters/gabia/flux-system/` | Flux 컨트롤러 설치 및 Git 동기화용 부트스트랩 설정 |

> 이 저장소에서 확인된 주요 관리 대상은 Frontend·Backend Deployment와 Flux GitOps 설정입니다. MySQL, Service, Ingress, Secret 등 운영 중인 모든 리소스가 이 저장소에 포함된 것은 아닙니다.
