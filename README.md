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
| staging | `dev-yapp.suitestudy.com` | staging |

### TLS 인증서

각 환경의 `certificate.yaml`이 cert-manager에게 인증서 발급을 요청합니다.
cert-manager가 Let's Encrypt에서 인증서를 발급받아 Secret으로 자동 저장합니다.

| 환경 | Certificate 이름 | 생성되는 Secret |
|------|----------------|---------------|
| prod | `neki-tls-cert` | `neki-tls-cert` |
| prod | `yapp-monitoring-tls-cert` | `yapp-monitoring-tls-cert` |
| staging | `dev-yapp-tls-cert` | `dev-yapp-tls-cert` |

IngressRoute에서 `tls.secretName`으로 이 Secret을 참조합니다.

### 적용 방법

```bash
# prod 전체 적용
kubectl apply -k overlays/prod/

# staging 전체 적용
kubectl apply -k overlays/staging/
```

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
# 접속 정보(사용자, 호스트, SSH 포트)는 팀 내부 채널을 참고한다. 이 레포는 public 이다.
ssh -L 30420:localhost:30420 -p <포트> <사용자>@<호스트>
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

# 4. 환경별 리소스 적용
kubectl apply -k overlays/prod/
kubectl apply -k overlays/staging/
```
