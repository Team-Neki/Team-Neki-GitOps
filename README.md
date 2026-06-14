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

## gitignore 처리 대상

다음 파일들은 민감 정보를 포함하므로 Git에 올리지 않고 직접 관리합니다.

| 파일 | 내용 | 관리 방법 |
|------|------|---------|
| `overlays/prod/secret.yaml` | jasypt 암호화 키 | 서버에서 직접 `kubectl apply` |
| `overlays/staging/secret.yaml` | jasypt 암호화 키 | 서버에서 직접 `kubectl apply` |

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
