# Chapter 4 — *Istio gateways: getting traffic into a cluster*

> **목표** : 외부 트래픽이 **어떻게 Istio Ingress Gateway에 도착하여 내부 서비스까지 전달되는지**를 단계별로 익히고,
> 실무에서 흔히 쓰는 HTTP·TLS·TCP·SNI, 다중 Gateway·Access Log·IstioOperator 패턴을 한눈에 정리한다.

---

## 1. Gateway가 해결하는 문제

| 전통 L-B                    | Istio Ingress Gateway                           |
| ------------------------- | ----------------------------------------------- |
| L4/L7 옵션·TLS 종료만 제공       | **L4\~L7 + mTLS + 라우팅 + 리트라이 + 로깅**을 Envoy 한곳에서 |
| HA Proxy 또는 클라우드 LB → 서비스 | *Gateway* → *VirtualService* → 서비스 (리소스 분리)     |
| 도메인·팀·보안 경계 분리가 번거롭다      | **게이트웨이별 디플로이먼트**·라벨·HPA·보안 정책을 손쉽게 분리          |

---

## 2. Gateway 리소스 ⚙️ (포트·호스트·TLS)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: web-gw
spec:
  selector: {istio: ingressgateway}     # 연결할 Envoy Deployment 라벨
  servers:
  # ──────────── 1. HTTP  ────────────
  - port:   {number: 80, name: http,  protocol: HTTP}
    hosts:  ["web.example.com"]
  # ──────────── 2. SIMPLE TLS  ───────
  - port:   {number: 443, name: https, protocol: HTTPS}
    tls:
      mode: SIMPLE                     # 게이트웨이에서 TLS 종료
      credentialName: web-tls-secret   # 서버 인증서 Secret
    hosts:  ["web.example.com"]
  # ──────────── 3. PASSTHROUGH TLS ───
  - port:   {number: 443, name: passthrough, protocol: TLS}
    tls:  {mode: PASSTHROUGH}          # SNI 값만 확인, TLS는 백엔드에서 종료
    hosts: ["db.example.com","*.mq.internal"]
```

* **protocol** — `HTTP`, `HTTPS`, `TLS`, `TCP` (HTTP / TLS ⇒ L7 라우팅 지원, TCP ⇒ L4 only)
* **tls.mode**

    * `SIMPLE` : 서버 인증서만, HTTPS 종료
    * `MUTUAL` : 클라이언트 + 서버 mTLS
    * `PASSTHROUGH` : SNI Switch, TLS 암·복호화 없음

---

## 3. VirtualService 리소스 🎯 (애플리케이션 라우팅)

```yaml
kind: VirtualService
metadata: {name: web-vs}
spec:
  hosts: ["web.example.com"]
  gateways: ["web-gw"]                 # 관문 지정
  http:
  - match: [{uri: {prefix: "/api"}}]    # /api → v2 70%, v1 30%
    route:
    - destination: {host: api, subset: v2, port: {number: 8080}}
      weight: 70
    - destination: {host: api, subset: v1, port: {number: 8080}}
      weight: 30
  - route: [{destination: {host: web-frontend, port: {number: 80}}}]
```

* `gateways: ["mesh"]` 를 같이 넣으면 **사이드카 트래픽과 Ingress 트래픽**을 동시에 매칭.

---

## 4. TCP / SNI Passthrough 사용법

### 4-1 순수 TCP

```yaml
servers:
- port: {number: 31400, name: tcp-echo, protocol: TCP}
  hosts: ["*"]
```

VirtualService:

```yaml
tcp:
- match: [{port: 31400}]
  route: [{destination: {host: echo-tcp, port: {number: 9000}}}]
```

### 4-2 SNI 분기 (TLS Passthrough)

```yaml
servers:
- port: {number: 443, name: tls-sni, protocol: TLS}
  tls:  {mode: PASSTHROUGH}
  hosts: ["mysql.prod.local","kafka.prod.local"]
```

VirtualService:

```yaml
tls:
- match:
  - port: 443
    sniHosts: ["mysql.prod.local"]
  route:
  - destination: {host: mysql, port: {number: 3306}}
- match:
  - port: 443
    sniHosts: ["kafka.prod.local"]
  route:
  - destination: {host: kafka, port: {number: 9093}}
```

---

## 5. **IstioOperator** 로 다중 게이트웨이 운용 🚀

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata: {name: multi-gw}
spec:
  profile: empty
  components:
    ingressGateways:
    - name: public-gw
      label: {istio: public-gw}
      enabled: true
      k8s:
        service: {type: LoadBalancer}
    - name: partner-gw
      label: {istio: partner-gw}
      enabled: true
      k8s:
        nodeSelector: {role: secure-edge: "true"}   # 전용 노드
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

* **VirtualService** 에서 `gateways: ["partner-gw"]` 처럼 명시적으로 라우팅.
* 게이트웨이별 **HPA**·**NetworkPolicy**·**PodSecurity** 를 독립 설정.
* 업그레이드는 `revision` 태그로 롤링 교체(예 `public-gw-v2`).

---

## 6. 게이트웨이 Access Log 설정 📝

| 범위          | 방법                                                                                                                                |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------- |
| *mesh-wide* | `istioctl install --set meshConfig.accessLogFile=/dev/stdout`                                                                     |
| *게이트웨이만*    | Telemetry CR + workloadSelector <br>`kind: Telemetry → selector: {app: istio-ingressgateway}` <br>`accessLogging.disabled: false` |

> 프로덕션 기본값은 *OFF*. 필요한 게이트웨이만 켜야 로그 폭주를 막을 수 있다.

---

## 7. **Egress Gateway** — 고정 IP & 서비스별 라우팅

1. **Egress Deployment/Service**

   ```yaml
   components.egressGateways:
   - name: egress-fixed
     label: {istio: egress-fixed}
     enabled: true
     k8s: {service: {type: LoadBalancer}}
   ```
2. **ServiceEntry** : 외부 호스트(예 `api.stripe.com`) 선언
3. **Gateway** (egress listener, `tls.mode: PASSTHROUGH`)
4. **VirtualService** : `gateways: ["mesh","egress-fixed"]` 로 강제
5. (옵션) **DestinationRule** → `tls.mode: PASSTHROUGH` / `ISTIO_MUTUAL`

> Stripe 호출은 **egress-fixed LB IP** 한 개로만 나가며 방화벽·파트너 화이트리스트 설정이 단순해진다.

---

## 8. 운영 베스트 프랙티스 ✔️

| 카테고리    | 팁                                                                                     |
| ------- | ------------------------------------------------------------------------------------- |
| 리소스     | 게이트웨이마다 HPA, Pod Requests/Limits 를 따로 튜닝.                                             |
| 성능      | `PILOT_FILTER_GATEWAY_CLUSTER_CONFIG=true` 로 CDS 축소, 메모리 ↓·xDS 대역폭 ↓                  |
| 보안      | Gateway Deployment 자체에 `securityContext.dropCapabilities: ALL` / `runAsNonRoot: true` |
| TLS 인증서 | cert-manager or Vault → SDS API 로 핫스왑 (재시작 없음).                                       |
| 관찰성     | 게이트웨이 전용 Grafana 대시보드 + Loki/ELK 로그 파이프라인 분리                                          |

---

## 9. 핵심 기억 포인트

1. **Gateway(L4/5) + VirtualService(L7)** 2-단계 모델이 Istio Ingress의 본질.
2. HTTP / TLS(SIMPLE·MUTUAL) / TCP / SNI Passthrough 까지 **모든 프로토콜 케이스**를 처리.
3. IstioOperator 로 **다중 게이트웨이 배포** → 라벨 매칭만으로 도메인·팀·보안 경계 분리.
4. Egress Gateway 로 **고정 IP** / 특정 SaaS 전용 경로를 구성, 레거시 방화벽·파트너 연동을 해결.
5. 로그·HPA·정책을 “게이트웨이 단위”로 분리 운영해야 대규모 환경에서 견고하다.

---

### 짧은 확인 퀴즈

| Q                                                           | 답 (한 줄)                                 |
| ----------------------------------------------------------- | --------------------------------------- |
| 1. Gateway 리소스에서 `tls.mode: PASSTHROUGH` 는 어떤 기능을 하지 *않는다*? | TLS 암·복호화(핸드셰이크).                       |
| 2. 다중 게이트웨이 환경에서 VirtualService 에 반드시 지정하는 필드는?             | `gateways: ["…"]`                       |
| 3. Egress Gateway 를 통해서만 나가게 하는 핵심 리소스 조합 3가지는?             | ServiceEntry + Gateway + VirtualService |
