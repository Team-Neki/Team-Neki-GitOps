# gitops-k3s

k3s 클러스터의 Kubernetes 매니페스트를 GitOps 방식으로 관리하는 저장소입니다.

## 아키텍처 구조
![Infrastructure Architecture](neki_architecture.png)

## 디렉토리 구조

```
gitops-k3s/
├── cluster/                        # 클러스터 전체(네임스페이스 무관) 인프라 리소스
│   ├── cert-manager/
│   │   └── cluster-issuer.yaml     # Let's Encrypt ClusterIssuer
│   ├── coredns/
│   │   └── coredns-hairpin-nat.yaml # Hairpin NAT용 CoreDNS 커스텀 설정
│   └── traefik/
│       └── traefik-helmchart.yaml  # Traefik Ingress Controller HelmChart
└── overlays/                       # 환경별 Kustomize 오버레이
    ├── prod/                       # 프로덕션 환경 (namespace: prod)
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   ├── deployment.yaml         # neki-prod 애플리케이션
    │   ├── ingress.yaml            # HTTP (web entrypoint)
    │   ├── ingressroute-https.yaml # HTTPS (websecure entrypoint)
    │   ├── certificate.yaml        # cert-manager TLS 인증서 요청
    │   ├── monitoring.yaml         # Prometheus / Grafana / Loki / Promtail
    │   ├── monitoring-ingressroute.yaml # Grafana HTTPS IngressRoute
    │   ├── admin-web-deployment.yaml   # neki-admin-web (Deployment / Service / PVC)
    │   ├── admin-web-ingressroute-https.yaml # neki-admin-web HTTPS IngressRoute
    │   ├── admin-web-secret.example.yaml # neki-admin-web 환경변수 예시
    │   ├── admin-web-secret.yaml       # ⚠️ gitignore - 직접 관리 필요
    │   └── secret.yaml             # ⚠️ gitignore - 직접 관리 필요
    └── staging/                    # 스테이징 환경 (namespace: staging)
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── deployment.yaml         # yapp-dev 애플리케이션
        ├── ingress.yaml            # HTTP (web entrypoint)
        ├── ingressroute-https.yaml # HTTPS (websecure entrypoint)
        ├── certificate.yaml        # cert-manager TLS 인증서 요청
        └── secret.yaml             # ⚠️ gitignore - 직접 관리 필요
    └── prefect/                    # Prefect 워크플로 오케스트레이션 (namespace: prefect)
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── server.yaml             # Prefect Server API/UI (NodePort 30420)
        ├── worker.yaml             # Kubernetes work pool 워커 + RBAC
        ├── worker-base-job-template.json # work pool base job template (flow run Job 스펙)
        ├── secret.example.yaml     # DB 비밀번호 Secret 예시
        └── secret.yaml             # ⚠️ gitignore - 직접 관리 필요
```

## 클러스터 인프라 (`cluster/`)

### Traefik (`cluster/traefik/traefik-helmchart.yaml`)

k3s 내장 HelmChart CRD로 Traefik을 커스텀 설정으로 배포합니다.

| EntryPoint | 내부 포트 | 외부 노출 포트 |
|------------|----------|--------------|
| `web` (HTTP) | 5678 | 5678 |
| `websecure` (HTTPS) | 4641 | 4641 |

> 기존 nginx가 80/443 포트를 사용 중이므로 5678/4641 포트를 사용합니다. 공유기/방화벽에서 외부 5678→내부 5678, 외부 4641→내부 4641로 포트포워딩 설정이 필요합니다.

적용 방법:
```bash
kubectl apply -f cluster/traefik/traefik-helmchart.yaml
```

### cert-manager (`cluster/cert-manager/cluster-issuer.yaml`)

Let's Encrypt ACME HTTP-01 챌린지를 통해 TLS 인증서를 자동 발급/갱신합니다.

- Issuer: `letsencrypt-prod` (ClusterIssuer)
- HTTP-01 챌린지 solver: Traefik IngressClass 사용

**전제 조건:** cert-manager가 클러스터에 먼저 설치되어 있어야 합니다.

```bash
# cert-manager 설치 (아직 안 된 경우)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# ClusterIssuer 적용
kubectl apply -f cluster/cert-manager/cluster-issuer.yaml
```

### CoreDNS Hairpin NAT (`cluster/coredns/coredns-hairpin-nat.yaml`)

클러스터 내부에서 외부 도메인(suitestudy.com)으로 요청 시 공유기를 거치지 않고 직접 노드 IP로 라우팅되도록 CoreDNS를 커스텀 설정합니다.

- `yapp.suitestudy.com` → `192.168.219.106`
- `dev-yapp.suitestudy.com` → `192.168.219.106`
- `yapp-monitoring.suitestudy.com` → `192.168.219.106`

```bash
kubectl apply -f cluster/coredns/coredns-hairpin-nat.yaml
```

## 환경별 오버레이 (`overlays/`)

### 도메인 구성

| 환경 | 도메인 | 네임스페이스 |
|------|--------|------------|
| prod | `yapp.suitestudy.com` | prod |
| prod (모니터링) | `yapp-monitoring.suitestudy.com` | prod |
| prod (어드민) | `admin-web.suitestudy.com` | prod |
| staging | `dev-yapp.suitestudy.com` | staging |

### TLS 인증서

각 환경의 `certificate.yaml`이 cert-manager에게 인증서 발급을 요청합니다.
cert-manager가 Let's Encrypt에서 인증서를 발급받아 Secret으로 자동 저장합니다.

| 환경 | Certificate 이름 | 생성되는 Secret |
|------|----------------|---------------|
| prod | `neki-tls-cert` | `neki-tls-cert` |
| prod | `yapp-monitoring-tls-cert` | `yapp-monitoring-tls-cert` |
| prod | `admin-web-tls-cert` | `admin-web-tls-cert` |
| staging | `dev-yapp-tls-cert` | `dev-yapp-tls-cert` |

IngressRoute에서 `tls.secretName`으로 이 Secret을 참조합니다.

### 적용 방법

```bash
# prod 전체 적용
kubectl apply -k overlays/prod/

# staging 전체 적용
kubectl apply -k overlays/staging/
```

## Neki Admin (`overlays/prod/admin-web-*.yaml`)

[Team-Neki-Admin](https://github.com/Team-Neki/Team-Neki-Admin) 의 운영자 관리 페이지(Next.js
standalone) 입니다. `prod` 네임스페이스에 있으므로 기존 `neki-prod` ArgoCD Application 이
그대로 동기화합니다. 별도 Application 은 없습니다.

| 구성요소 | 리소스 | 비고 |
|---|---|---|
| neki-admin-web | Deployment (replicas 1) / Service (ClusterIP 80→3000) | `ghcr.io/team-neki/neki-admin-web` |
| (미사용) | PersistentVolumeClaim 1Gi (`local-path`) | `/app/.data` 마운트. 현재 앱이 쓰지 않음 |
| 환경변수 | Secret `neki-admin-web-secret` | ConfigMap 없이 전부 여기. ⚠️ gitignore - 직접 관리 |

접속: <https://admin-web.suitestudy.com:4641> (Traefik `websecure` 는 4641 포트)

### 이미지

태그는 `overlays/prod/admin-web-deployment.yaml` 의 `image:` 줄에 직접 적습니다.
`neki-prod` / `sprint` / `notification` 과 같은 방식이고, Team-Neki-Admin 의 빌드
워크플로가 그 줄을 갱신하면 ArgoCD 가 롤링합니다.

이미지는 한 곳에서만 참조합니다. `sprint-deployment.yaml` 처럼 같은 이미지를 여러
줄에 적으면 갱신에서 하나를 놓쳤을 때 컨테이너마다 버전이 갈리는데, 파드는 정상
기동하므로 드러나지 않습니다. 워크플로는 갱신 전후의 참조 개수를 비교해 이 경우를
막습니다.

GHCR 패키지는 `team-neki-workflow` 처럼 **공개(public)** 여야 합니다. 비공개로 두면
파드가 `ImagePullBackOff` 로 멈추므로, 그 경우에는 `dockerconfigjson` 타입 Secret 을
만들고 `admin-web-deployment.yaml` 에 `imagePullSecrets` 를 추가해야 합니다.

### PVC 와 `replicas: 1` / `Recreate`

원래는 Amplitude 일별 집계를 SQLite 파일 하나에 캐시했기 때문에 `replicas: 1` 고정,
`strategy: Recreate` 였습니다. `local-path` PVC 는 `ReadWriteOnce` 이고 노드 로컬이라
`RollingUpdate` 로 두면 새 파드와 기존 파드가 겹치는 순간 같은 볼륨을 동시에 붙잡기
때문입니다.

Team-Neki-Admin 이 `75cf4f7` 에서 로컬 저장소를 걷어내고 관리자 API 프록시로 바뀌면서
앱은 더 이상 `/app/.data` 에 아무것도 쓰지 않습니다. SQLite 의존성도 없습니다.
지금 PVC·`fsGroup`·`Recreate` 는 근거를 잃은 상태로 남아 있고, 걷어낼지는 아직
정하지 않았습니다. 그대로 둬도 동작에는 문제가 없습니다 (쓰지 않는 1Gi 볼륨이 붙을 뿐).

### Secret

`admin-web-secret.example.yaml` 을 `admin-web-secret.yaml` 로 복사해 값을 채우고 직접 apply 합니다.

```bash
kubectl apply -f overlays/prod/admin-web-secret.yaml
```

ConfigMap 은 두지 않습니다. 비민감 값(`TZ` 등)까지 이 Secret 하나에 모으고,
deployment 의 `envFrom` 도 `secretRef` 하나뿐입니다.

앱이 실제로 읽는 이름은 아래가 전부입니다. 이 목록에 없는 이름을 넣어도 파드는
정상 기동하고 화면만 비어서 원인이 드러나지 않으니, 값을 추가할 때는 코드에서
그 이름을 읽는지 먼저 확인하세요.

| 환경변수 | 읽는 곳 | 현재 |
|---|---|---|
| `NEKI_ADMIN_DASHBOARD_API_URL` | `app/api/amplitude/dashboard/route.ts` | 관리자 백엔드 미기동 - 넣지 않음 |
| `NEKI_ADMIN_ANALYTICS_API_URL` | `app/api/amplitude/metrics/route.ts` | 관리자 백엔드 미기동 - 넣지 않음 |
| `GROUP_ACCOUNT_DATA_MODE` | `app/api/group-account/group-account-server.ts` | 빈 값 (`mock` 은 개발 전용) |
| `OPENBANKING_BASE_URL` | 〃 | 빈 값 |
| `OPENBANKING_ACCESS_TOKEN` | 〃 | 빈 값 |
| `OPENBANKING_FINTECH_USE_NUM` | 〃 | 빈 값 |
| `OPENBANKING_BANK_TRAN_ID` | 〃 | 빈 값 |

관리자 백엔드 URL 두 개가 비어 있는 동안 `/api/amplitude/*` 는 503
`admin_api_not_configured` 로 응답하고 대시보드는 빈 채로 뜹니다. 백엔드가 뜨면
Secret 에 두 URL 을 넣고 `kubectl apply` 후 파드를 재시작하면 됩니다.

### 남은 작업

- **앱 인증 없음.** 현재 어드민에는 자체 로그인이 없어 도메인을 아는 누구나 접근할 수
  있습니다. 앱에 로그인을 붙이거나, Traefik `basicAuth` / `ipAllowList` 미들웨어를
  `admin-web-ingressroute-https.yaml` 에 추가해야 합니다.
- **health 엔드포인트 없음.** `/api/health` 가 없어 readiness 가 `/` 를 SSR 합니다.
  주기를 30s 로 늘려 부담을 줄였지만, 앱에 `/api/health` 가 추가되면 probe 를 그쪽으로
  옮기는 편이 낫습니다.

## Prefect (`overlays/prefect/`)

Prefect 3 셀프호스팅 서버와 Kubernetes 워커입니다. Helm 차트(prefect-helm)를 렌더링한 결과를 kustomize 용으로 정리해 관리하며, 다른 overlay 와 같은 방식으로 ArgoCD(`argocd/apps/prefect.yaml`)가 동기화합니다. 메타데이터 DB 는 클러스터 안에 두지 않고 노드(호스트)에 설치된 PostgreSQL 14 를 사용합니다.

| 구성요소 | 리소스 | 비고 |
|---|---|---|
| prefect-server | Deployment / Service(NodePort 30420) | `prefecthq/prefect:3.8.5-python3.11` |
| prefect-worker | Deployment / Role / RoleBinding | work pool `kubernetes-pool` 자동 생성, flow run 은 `prefect` 네임스페이스에 Job 으로 실행 |
| (DB) | 호스트 PostgreSQL 14 (`192.168.219.106:5432`, DB/role `prefect`) | 클러스터 리소스 없음. 파드 → 노드 IP 로 직접 접속 |

### 접속

외부(DNS/nginx/TLS) 노출 없이 SSH 터널로만 접근합니다. UI 가 API 를 `http://localhost:30420/api` 로 호출하도록 설정되어 있으므로 **로컬 포트도 30420** 으로 맞춰야 합니다.

```bash
ssh -L 30420:localhost:30420 -p <PORT> yapp@suitestudy.com
# 브라우저: http://localhost:30420
```

flow 코드에서 접속할 때(클러스터 내부): `PREFECT_API_URL=http://prefect-server.prefect.svc.cluster.local:4200/api`

### Secret

`overlays/prefect/secret.yaml` (gitignore) 에 호스트 DB `prefect` role 의 비밀번호가 있습니다(Secret `prefect-db`). `secret.example.yaml` 을 복사해 만들고 직접 apply 합니다. 호스트 DB 에 role/database 를 만드는 방법은 `secret.example.yaml` 주석을 참고하세요.

```bash
kubectl apply -f overlays/prefect/secret.yaml
kubectl apply -k overlays/prefect/
```

### 업그레이드

`server.yaml` / `worker.yaml` 의 이미지 태그를 같은 Prefect 버전으로 올립니다(워커는 `-kubernetes` 접미사 이미지). DB 마이그레이션은 서버 기동 시 자동 수행됩니다. work pool 의 base job template 을 바꾸려면 `worker-base-job-template.json` 을 수정하면 워커 재기동 시 initContainer 가 work pool 에 동기화합니다.

## gitignore 처리 대상

다음 파일들은 민감 정보를 포함하므로 Git에 올리지 않고 직접 관리합니다.

| 파일 | 내용 | 관리 방법 |
|------|------|---------|
| `overlays/prod/secret.yaml` | jasypt 암호화 키 | 서버에서 직접 `kubectl apply` |
| `overlays/staging/secret.yaml` | jasypt 암호화 키 | 서버에서 직접 `kubectl apply` |
| `overlays/prefect/secret.yaml` | Prefect DB 비밀번호 | 서버에서 직접 `kubectl apply` |
| `overlays/prod/admin-web-secret.yaml` | neki-admin-web 환경변수 전체 | 서버에서 직접 `kubectl apply` |

> TLS Secret(`tls-secret.yaml`)은 cert-manager의 Certificate로 대체되어 더 이상 사용하지 않습니다.

## 전체 초기 배포 순서

```bash
# 1. cert-manager 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=available deployment --all -n cert-manager --timeout=120s

# 2. 클러스터 인프라 적용
kubectl apply -f cluster/traefik/traefik-helmchart.yaml
kubectl apply -f cluster/coredns/coredns-hairpin-nat.yaml
kubectl apply -f cluster/cert-manager/cluster-issuer.yaml

# 3. App Secret 적용 (gitignore 파일 - 직접 관리)
kubectl apply -f overlays/prod/secret.yaml
kubectl apply -f overlays/staging/secret.yaml
kubectl apply -f overlays/prod/admin-web-secret.yaml

# 4. 환경별 리소스 적용
kubectl apply -k overlays/prod/
kubectl apply -k overlays/staging/
```
