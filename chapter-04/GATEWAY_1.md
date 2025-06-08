# Gateway Deep-Dive for Real-World Istio Clusters

*(production patterns, multi-Ingress, multi-Egress, fixed-IP routing)*

---

## 1. Production-grade Ingress Gateway patterns

| 목적                      | 실무 적용 기법                                                                                                                                           |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **퍼블릭 / 프라이빗 분리**       | - `public-gw` (LB public) + `internal-gw` (LB internal) <br>- 서로 다른 `selector` 라벨 → VirtualService 마다 명시                                           |
| **팀별 게이트웨이**            | - `sidecar.istio.io/inject:"true", inject.istio.io/templates: gateway` 을 팀 Deployment에 사용 → 자체 Envoy Gateway 생성 <br>- 네임스페이스 HPA, NetworkPolicy 분리 |
| **카나리 업그레이드**           | - 게이트웨이를 **revision** 별로 두고 `revisionTags`로 트래픽 분할 → 1% → 10% → 100%                                                                               |
| **WAF / Rate-Limit 통합** | - Envoy WASM or `ext_authz` → 公司 ▪︎ 모듈(CloudArmor, ModSecurity) 삽입 <br>- Layer7 DDoS 대비는 LB 앞단에서, L7 OWASP 룰은 Gateway 안에서                          |
| **Access Log & 모니터링**   | - Gate­way 전용 Telemetry CR → 로그 ON, 사이드카에는 OFF <br>- Prometheus `istio_requests_total{reporter="gateway"}` 로 대시보드 분리                               |
| **리소스 최적화**             | - `PILOT_FILTER_GATEWAY_CLUSTER_CONFIG=true` + `filterGatewayRouteTable=true` 로 Envoy CDS/RDS 슬림 <br>- Pod 2 vCPU, 512 MiB → 2000 RPS 기준           |
| **TLS 운영**              | - SIMPLE 모드에 cert-manager SDS로 자동 교체 <br>- 클라우드 L7 LB가 TLS 종료, Gateway는 mTLS 내부 TLS 재종단                                                            |

---

## 2. 여러 Ingress 게이트웨이 생성 (예: 퍼블릭 + 프라이빗)

```yaml
# istio-gateways.yaml (IstioOperator, single CR)
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata: {name: multi-gw, namespace: istio-system}
spec:
  profile: empty            # control-plane는 이미 설치돼 있다고 가정
  components:
    ingressGateways:
    - name: public-gw
      enabled: true
      label: {istio: public-gw}
      k8s:
        service: {type: LoadBalancer}
    - name: internal-gw
      enabled: true
      label: {istio: internal-gw}
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
  name: public-gw
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

* **VirtualService 구분**

  ```yaml
  gateways: ["public-gw"]        # 외부용
  ...
  gateways: ["corp-gw"]      # 사내망용
  ```

* **로그·HPA 분리**

  ```bash
  kubectl autoscale deploy public-gw     --cpu-percent=60 --min=2 --max=10 -n istio-system
  kubectl autoscale deploy internal-gw   --cpu-percent=50 --min=1 --max=4  -n istio-system
  ```

---

## 3. Egress Gateway 활용 & 고정 IP 라우팅

### 3-1 기본 배선

```yaml
# ① Egress Gateway (IstioOperator)
components:
  egressGateways:
  - name: egress-fixed
    enabled: true
    label: {istio: egress-fixed}
    k8s:
      service: {type: LoadBalancer}   # 예약된 퍼블릭 IP
```

```yaml
# ② ServiceEntry (외부 호스트 선언)
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata: {name: stripe-api}
spec:
  hosts: ["api.stripe.com"]
  ports: [{number: 443, name: https, protocol: TLS}]
  resolution: DNS
  location: MESH_EXTERNAL
```

```yaml
# ③ VirtualService (Egress 라우팅)
kind: VirtualService
metadata: {name: stripe-vs}
spec:
  hosts: ["api.stripe.com"]
  gateways: ["mesh","egress-fixed"]     # 내부 → egress-fixed → Internet
  tls:
  - match: [{port: 443, sniHosts: ["api.stripe.com"]}]
    route: [{destination: {host: api.stripe.com}}]
```

> **결과** : Stripe 호출은 반드시 `egress-fixed` 의 **LoadBalancer IP** 로 나간다.

### 3-2 여러 Egress Gateway 중 “서비스별” 고정 IP 선택

| 서비스           | 요구 조건           | 게이트웨이              |
| ------------- | --------------- | ------------------ |
| payments-svc  | 카드사 화이트리스트 IP A | `egress-card`      |
| analytics-svc | SaaS 연계 IP B    | `egress-analytics` |

1. 두 Egress Deployment 라벨 `istio: egress-card / egress-analytics`
2. 두 VirtualService : 각 서비스 Domain → 해당 egress 게이트웨이
3. (옵션) `Sidecar` CR로 팀 네임스페이스가 **다른 egress로 나가는** 것을 근본 차단

---

## 4. 현장에서 자주 쓰는 Gateway & Egress ‘패턴 카드’

| 패턴 이름                       | 요약                                                                                                         | 핵심 리소스·옵션                    |
| --------------------------- | ---------------------------------------------------------------------------------------------------------- | ---------------------------- |
| **Blue/Green Gateway**      | 새 버전 Envoy Gateway 를 `revision: gw-v2` 로 배포 → external DNS 레코드 CNAME을 순차 전환                                | 두 Revision -- `revisionTags` |
| **SNI Multiplexer**         | 443 단일 포트, PASSTHROUGH 모드. SNI별 VirtualService 분기 → 내부 TLS 재종단                                             | `tls.mode: PASSTHROUGH`      |
| **Edge mTLS Offload**       | 외부는 HTTPS(SIMPLE), Gateway\~Pod 사이는 mTLS(ISTIO\_MUTUAL). `Gateway SIMPLE` + `DestinationRule ISTIO_MUTUAL` | 보안/관찰성 유지                    |
| **Partner-API mTLS**        | Gateway `MUTUAL` + CA bundle Secret. Partner CertificateRequest 요구. RateLimit + JWT 인증 WASM 필터             | WAF·ExtAuthz 플러그인            |
| **Data-Egress NAT**         | 하나의 Egress Gateway, NodeSelector=egress-nodes, Service type=NodePort → 전용 NAT GW IP 풀                      | 오프라인 클러스터                    |
| **Hub-Spoke Multi-cluster** | 각 클러스터 eastwest-gw (Ingress+Egress) → SPIFFE Federation, ServiceEntry `*.global`                           | multi-network mesh           |

---

## 5. 베스트 프랙티스 체크리스트

* [ ] **Gateway 로그**: Telemetry CR (workloadSelector) 로 필요한 GW만 ON
* [ ] **HPA**: 게이트웨이별 CPU/QPS 수치에 맞춰 독립 오토스케일
* [ ] **Config Slim**: `PILOT_FILTER_GATEWAY_CLUSTER_CONFIG=true` 활성화
* [ ] **정책 ↔ 라우팅 분리**: L7 Authz (RequestAuthentication + AuthorizationPolicy) 로 게이트웨이에 과부하 필터를 넣지 않는다.
* [ ] **CI 테스트**: `istioctl analyze -A` + `kube-rbac-proxy` e2e 테스트로 Gateway 변경 시 회귀 방지
* [ ] **Secret SDS**: cert-manager or Vault → SDS for rotation (재시작 無)

---

### 빠른 복습 퀴즈 (답은 스크롤 아래)

1. SNI Passthrough 모드에서 게이트웨이가 *하지 않는* TLS 단계는?
2. `VirtualService` 에서 인그레스·사이드카 트래픽을 동시에 핸들링하려면 `gateways` 필드에 어떤 값을 포함해야 하나?
3. Egress Gateway 를 통해서만 외부 도메인을 호출하게 만들 때 **DestinationRule**에 자주 추가하는 TLS 모드는?
4. 다중 게이트웨이 설치 시 Listener에 **불필요한 Cluster 정보**가 실리지 않도록 줄이는 Pilot 환경 변수는?

<details><summary>정답</summary>

1. TLS 핸드셰이크(Decryption) – 게이트웨이는 SNI만 읽고 암/복호화하지 않는다.
2. `"mesh"` (클러스터 내부 호출용 예약 값)
3. `ISTIO_MUTUAL` 또는 `PASSTHROUGH` — 외부 TLS 를 게이트웨이까지 유지하려면 PASSTHROUGH, 게이트웨이에서 다시 암호화하려면 ISTIO\_MUTUAL
4. `PILOT_FILTER_GATEWAY_CLUSTER_CONFIG=true`

</details>

