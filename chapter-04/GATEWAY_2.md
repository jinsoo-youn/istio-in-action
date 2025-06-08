아래 내용만 **추가로 붙여 넣으면** 기존 정리본이 “IstioOperator → Gateway 리소스 생성” 단계까지 완전해집니다.
Egress Gateway 챕터도 **구체적 YAML**·**실행 흐름**까지 확장했으니 그대로 이어서 사용하세요.

---

## ✚ 보강 ① IstioOperator로 Gateway 만들기 → Gateway 리소스 배선

### 1. IstioOperator—게이트웨이 Pod/Service 배포

```yaml
# gateways-prod.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: gateways-prod
  namespace: istio-system
spec:
  profile: empty            # 제어 플레인은 이미 설치돼 있다고 가정
  components:
    ingressGateways:
    - name: public-gw
      label: {istio: public-gw}
      enabled: true
      k8s:
        service: {type: LoadBalancer}
    - name: internal-gw
      label: {istio: internal-gw}
      enabled: true
      k8s:
        service:
          type: LoadBalancer
          annotations:
            service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

```bash
istioctl install -f gateways-prod.yaml
```

*결과*

```
NAME                 TYPE           EXTERNAL-IP
public-gw            LoadBalancer   34.120.x.x
internal-gw          LoadBalancer   10.0.1.27
```

### 2. Gateway 리소스—포트·TLS·호스트 매핑

```yaml
# public-http-gw.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-http
  namespace: istio-system
spec:
  selector: {istio: public-gw}          # IstioOperator에서 만든 label
  servers:
  - port: {number: 80, name: http, protocol: HTTP}
    hosts: ["shop.example.com"]
```

```yaml
# internal-gw.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: corp-gw
  namespace: istio-system
spec:
  selector: {istio: internal-gw}
  servers:
  - port: {number: 8080, name: http, protocol: HTTP}
    hosts: ["*.corp.internal"]
```

> **정리:** IstioOperator로 **배포 대상(Envoy Pod+Service)** 를 만든 뒤, `Gateway` 리소스로 **리스너 구성**을 선언하는 2-단계 구조다.

---

## ✚ 보강 ② Egress Gateway 더 깊이 보기

### A. ‘고정 IP’ egress 흐름 전체 예

| 단계                               | 리소스                                                                 | 핵심 설정                                |
| -------------------------------- | ------------------------------------------------------------------- | ------------------------------------ |
| ① **Egress Gateway Pod/Service** | IstioOperator (egress-fixed)                                        | `service.type: LoadBalancer` + 고정 IP |
| ② **ServiceEntry**               | `hosts: ["api.stripe.com"]`                                         | 외부 서비스 DNS 등록                        |
| ③ **Gateway**                    | selector: `istio: egress-fixed` + `servers[].tls.mode: PASSTHROUGH` | egress-fixed 가 443/TLS 받음            |
| ④ **VirtualService**             | `gateways: ["mesh","egress-fixed"]` + `tls.match.sniHosts`          | “내부 → egress-fixed → stripe”         |
| ⑤ **DestinationRule**            | `trafficPolicy.tls.mode: PASSTHROUGH`                               | 게이트웨이\~Stripe TLS 유지                 |

#### 전체 YAML 묶음

```yaml
# 1) Egress Gateway 정의(이미 배포됨 가정)

# 2) ServiceEntry
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata: {name: stripe-api}
spec:
  hosts: ["api.stripe.com"]
  ports: [{number: 443, name: https, protocol: TLS}]
  resolution: DNS
  location: MESH_EXTERNAL
---
# 3) Gateway (egress listener)
kind: Gateway
metadata: {name: egress-fixed-gw, namespace: istio-system}
spec:
  selector: {istio: egress-fixed}
  servers:
  - port: {number: 443, name: tls, protocol: TLS}
    tls: {mode: PASSTHROUGH}
    hosts: ["api.stripe.com"]
---
# 4) VirtualService (라우팅)
kind: VirtualService
metadata: {name: stripe-vs}
spec:
  hosts: ["api.stripe.com"]
  gateways: ["mesh","istio-system/egress-fixed-gw"]
  tls:
  - match:
    - port: 443
      sniHosts: ["api.stripe.com"]
    route:
    - destination: {host: api.stripe.com}
---
# 5) DestinationRule (옵션)
kind: DestinationRule
metadata: {name: stripe-dr}
spec:
  host: api.stripe.com
  trafficPolicy:
    tls: {mode: PASSTHROUGH}
```

> **검증**
>
> ```bash
> curl https://api.stripe.com --resolve api.stripe.com:443:<EGRESS_LB_IP>
> ```
>
> 외부 방화벽 / 파트너 화이트리스트에서 `EGRESS_LB_IP`만 열어두면 된다.

### B. **여러 Egress Gateway** 중 서비스별로 선택

1. `egress-card`, `egress-analytics` 두 Gateway Deployment
2. 두 Gateway 리소스 (`selector: istio: egress-card …`)
3. VirtualService 들을 **각각 필요한 egress 게이트웨이**에만 매핑

   ```yaml
   gateways: ["mesh","egress-card"]
   ```

### C. Egress + **Outbound mTLS 재종단**

* 외부 SaaS 가 자체 CA 로 mTLS 요구 → egress Gateway 에 SIMPLE TLS(서버), DestinationRule `ISTIO_MUTUAL` 로 **내부→Egress** 구간 재암호화
* SDS Secret 이름(credentialName) = 사내 RootCA bundle

### D. 네임스페이스 단위 차단/허용

`Sidecar` CRD 로 egress hosts 제한

```yaml
egress:
- hosts:
  - ./internal/*             # 같은 ns 서비스만
  - istio-system/*           # 모니터링
```

일반 Pod 는 외부 호출 불가 → 특정 서비스만 VirtualService + Egress Gateway 로 “허가 티켓” 발급.

---

### 실무 체크리스트 : Egress Gateway

* **IP 고정** — 클라우드 LB IP 예약, GCP NAT 게이트웨이 풀 or AWS EIP
* **중앙 로깅** — `egressRequestTotal`·`egressRequestDuration` 전용 Prometheus 룰
* **대역폭 제한** — Envoy WASM Pacing Filter, QoS 클래스로 트래픽 톤다운
* **Fail-open vs Fail-close** — Egress Gateway 비가용 시 대체 경로(mesh 직접) 허용 여부 결정
* **릴리스 전략** — 새 Egress Gateway 를 `revision=egress-v2` 로 배포 → `VirtualService` TrafficShift 라벨 단계적 교체

---

이 추가 파트를 **기존 문서의 “Ingress/TLS/운영” 뒤에 붙여** 사용하면,

> ① IstioOperator → Gateway 리소스 흐름 **완결**,
> ② Egress Gateway “고정 IP + 서비스별 분기” **실전 예시**
> 를 모두 갖춘 Chapter 4 실무 가이드가 완성됩니다.
