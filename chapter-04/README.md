# Chapter 4 요약 – Istio Gateways: Getting Traffic into a Cluster

## 4.1 Ingress 개념 정리

* **Ingress Point** : 외부에서 들어오는 트래픽을 받아들이는 ‘게이트키퍼’ 역할. 허용되지 않은 요청은 차단하고, 허용된 요청은 내부 서비스로 프록시한다.&#x20;
* **가상 IP(virtual IP) + Reverse Proxy** 패턴으로 고가용성 제공. 도메인은 고정 IP가 아닌 프록시 앞의 VIP로만 매핑한다.&#x20;

## 4.2 Ingress Gateway 기본 흐름

```text
         ┌────────────┐
 client ─▶  Gateway   │ ① L4/L5 매칭(Host·Port·TLS)
         └────┬───────┘
              │
              ▼
        ┌────────────┐
        │VirtualSvc  │ ② L7 라우팅(경로·헤더·가중치)
        └────┬───────┘
              ▼
        ┌────────────┐
        │   Service  │ ③ K8s 서비스 → Pod
        └────────────┘
```

## 4.2 Istio Ingress Gateway 기본

### 4.2.1 Gateway 리소스

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata: {name: coolstore-gateway}
spec:
  selector: {istio: ingressgateway}
  servers:
  - port: {number: 80, name: http, protocol: HTTP}
    hosts: ["webapp.istioinaction.io"]
```

*포트·프로토콜·가상호스트* 만 정의한다. Envoy Listener(8080)로 매핑된 것을 `istioctl proxy-config listener` 로 확인할 수 있다.&#x20;

### 4.2.2 VirtualService 리소스

Gateway 로 들어온 요청을 실제 서비스로 보낸다.

```yaml
kind: VirtualService
spec:
  hosts: ["webapp.istioinaction.io"]
  gateways: ["coolstore-gateway"]
  http:
  - route:
    - destination: {host: webapp, port: {number: 8080}}
```

*`gateways` 필드가 **edge 트래픽**만 매칭하도록 한다.*&#x20;

### 4.2.3 요청 테스트

Host 헤더를 `webapp.istioinaction.io` 로 지정해야 200 OK 를 받는다. 그렇지 않으면 Envoy Blackhole(404)로 라우트된다.&#x20;

## 4.3 TLS 옵션

| 모드              | 특징                                   | 사용 예                     |   |
| --------------- | ------------------------------------ | ------------------------ | - |
| **SIMPLE**      | 게이트웨이에서 TLS 종료, 서버 인증서 필요            | 일반 HTTPS 웹사이트            |   |
| **MUTUAL**      | 클라이언트 인증서까지 요구(mTLS)                 | B2B 파트너 API              |   |
| **PASSTHROUGH** | 게이트웨이는 SNI만 보고 라우팅, TLS 세션은 백엔드에서 종료 | DB / 메시지큐 등 TCP over TLS |   |

## 4.4 TCP 및 SNI Passthrough

* **TCP Gateway** : `protocol: TCP` 로 비-HTTP 서비스도 노출 가능. 예제에서는 Telnet echo 서비스를 31400 포트로 노출해 연결을 시연한다.&#x20;
* **SNI Passthrough** : 동일 포트(31400)에서 SNI 값으로 여러 TLS 서비스 분기. Gateway TLS mode `PASSTHROUGH` + VirtualService `tls.match.sniHosts` 로 구현한다.&#x20;

## 4.5 Operational Tips

### 4.5.1 Ingress Gateway 분리

IstioOperator 로 **여러 게이트웨이**를 정의해 팀·도메인·컴플라이언스 별로 트래픽을 격리할 수 있다.&#x20;

### 4.5.2 Gateway Injection

Deployment stub + annotation (`sidecar.istio.io/inject: "true"`, `inject.istio.io/templates: gateway`) 만으로 팀이 자체 게이트웨이를 생성하도록 지원한다.&#x20;

### 4.5.3 Access Logs

Demo 프로필은 기본 활성화, 프로덕션 기본은 비활성. `meshConfig.accessLogFile=/dev/stdout` 또는 Telemetry CR로 특정 게이트웨이만 켤 수 있다.&#x20;

### 4.5.4 Gateway Config Slimming

`PILOT_FILTER_GATEWAY_CLUSTER_CONFIG=true` 플래그로 게이트웨이에 **필요한 Cluster만** 전달해 메모리·제어 부하를 줄인다.&#x20;

## 4.6 Egress Gateway(미리보기)

게이트웨이 구조는 동일하되 **클러스터 밖으로 나가는** 트래픽을 제어한다. (자세한 구성은 이후 장에서 다룸)&#x20;

---

### Chapter 4 핵심

* Gateway는 **L4/L5**(포트·프로토콜·호스트) 설정, VirtualService는 **L7**(경로·헤더) 라우팅을 담당한다.
* TLS 3 모드(SIMPLE, MUTUAL, PASSTHROUGH) 로 HTTPS, mTLS, SNI Passthrough 시나리오를 모두 수용한다.
* TCP 서비스도 동일 패턴으로 노출할 수 있으며, SNI Passthrough 로 하나의 포트에 여러 TLS 서비스를 묶을 수 있다.
* 다중 게이트웨이·Gateway Injection·Config Slimming 등 운영 기법으로 대규모 환경에서도 유연하게 관리한다.
