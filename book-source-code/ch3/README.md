# Chapter 3 — Istio’s **Data Plane**: the Envoy Proxy

## 1 | Envoy가 무엇이고 왜 중요한가

Envoy는 Lyft가 만들고 CNCF가 보육-졸업(GA)한 **L3/L4+L7 프록시**다. 서비스 간 통신에서 **신뢰성(재시도·서킷브레이커)**, **가시성(메트릭·트레이스)**, **보안(mTLS)** 과 같은 교차 관심사를 애플리케이션 밖에서 일관되게 처리한다 .

* HTTP 1.1 / HTTP 2 / gRPC 네이티브 지원
* 풍부한 통계(counter·gauge·histogram) 및 OpenTracing 스팬 전파
* Connection / Request 레벨 **Circuit-Breaking & Outlier-Detection**
* **xDS**(LDS·RDS·CDS·EDS …) API 로 런타임 재구성 — Istio가 이 메커니즘을 사용해 사이드카를 실시간 제어한다 .

## 2 | Envoy 설정 방법 - Static vs Dynamic

### 2-1 Static YAML 예시

`simple.yaml`은 하나의 Listener(15001)와 Cluster(`httpbin_service`)를 가진 최소 구성이다 .

```yaml
listeners:
- name: httpbin-demo
  address: { socket_address: { address: 0.0.0.0, port_value: 15001 } }
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        route_config:
          virtual_hosts:
          - domains: ["*"]
            routes:
            - match: { prefix: "/" }
              route: { cluster: httpbin_service }
clusters:
- name: httpbin_service
  connect_timeout: 5s
  type: LOGICAL_DNS
  load_assignment:
    endpoints:
    - lb_endpoints:
      - endpoint: { address: { socket_address: { address: httpbin, port_value: 8000 } } }
```

컨테이너 실행:

```bash
docker run --name httpbin -d citizenstig/httpbin
docker run --link httpbin -p15001:15001 -v $(pwd)/simple.yaml:/etc/envoy.yaml \
       envoyproxy/envoy:v1.29-latest -c /etc/envoy.yaml
```

### 2-2 구성 변경 - Timeout & Retry

* **요청 타임아웃**을 1 초로 단축한 예: `simple_change_timeout.yaml`&#x20;
* **5xx 재시도 3회**를 추가한 예: `simple_retry.yaml`&#x20;

두 파일만 교체-재시작하면 실시간으로 동작 차이를 체험할 수 있다.

### 2-3 Dynamic(xDS) 구성

실무에서는 Envoy가 빈 “부트스트랩”만 갖고 띄워진 뒤 **ADS gRPC 스트림**으로 Listener·Cluster 등을 수신한다 . Istio의 `istiod` 가 바로 이 ADS 서버 역할이다.

추가 정리: [Envoy xDS](xDS.md)

## 3 | Envoy in Action — httpbin 연습

책은 Docker 로 **httpbin → Envoy → curl** 구조를 만들고,

* `/headers` 요청으로 프록시가 헤더를 어떻게 전달하는지
* `/delay/5` 호출 후 **timeout 1 s** 설정이 어떻게 실패로 바뀌는지
* `/status/503` 호출 후 **retry 3회** 설정이 성공률을 높이는지
  직접 확인하도록 안내한다 .

Admin API(`localhost:15000`)에서 `/stats`, `/clusters`, `/config_dump` 등을 열어보면 설정과 런타임 지표를 동시에 살필 수 있다 .

## 4 | Envoy와 Istio의 결합

1. **Sidecar**: Istio는 각 Pod 옆에 Envoy를 주입하여 데이터 플레인을 구성한다.
2. **istiod**: ADS 채널을 통해 Listener·Route·Cluster·Secret(인증서)을 푸시한다.
3. **mTLS & SPIFFE**: istiod 가 발급한 X.509 SVID를 Envoy 쌍이 교환하여 트래픽을 암호화하고 ID를 검증한다 .
4. **관찰성**: Envoy가 생성하는 메트릭·트레이스 정보는 Prometheus·Jaeger로 직접 전송되며 Istio 애드온이 시각화한다.

---

## Self-Check Quiz (심화 학습용)

**Q1.** Envoy가 “애플리케이션-레벨 프록시”로 다른 L4 프록시보다 제공하는 두 가지 주요 이점은?
**Q2.** `simple_change_timeout.yaml` 에서 요청이 2 초 지연되는 `/delay/2` 엔드포인트를 호출하면 어떤 결과가 나오나?
**Q3.** Envoy의 xDS 중 *EDS* 와 *RDS* 가 각각 전달하는 정보는 무엇인가?
**Q4.** Admin API에서 현재 연결 중인 모든 클러스터의 상태를 확인하려면 어떤 엔드포인트를 호출해야 하나?
**Q5.** Istio 사이드카가 동작 중인 Pod 안에서 **실제 적용된 Envoy 구성**을 확인할 수 있는 Istio CLI 명령은?
**Q6.** `simple_retry.yaml` 의 `retry_policy` 가 동작하려면 어떤 응답 코드 범주가 조건에 포함돼야 하는가?
**Q7.** Envoy가 수집하는 지표 중 **circuit-breaker 임계 초과 횟수**를 알려주는 카운터 이름은?
**Q8.** Istio가 Envoy에 **X.509 인증서**를 제공하는 메커니즘은 무엇이라 부르나?

### 정답

* **A1.** 세밀한 L7 라우팅(헤더·메서드·경로 기반) + 풍부한 관찰성(메트릭·트레이스 자동 수집).
* **A2.** Listener-Route 설정에 `timeout: 1s` 가 있으므로 1 초 뒤 504 Gateway Timeout(혹은 503) 으로 실패한다.
* **A3.** EDS: 엔드포인트(IP·포트) 목록, RDS: HTTP 라우팅 규칙(가상호스트·경로 매칭).
* **A4.** `http://localhost:15000/clusters`
* **A5.** `istioctl proxy-config listeners|routes|clusters <pod-name>`
* **A6.** `5xx` (예: 500–599) — `retry_on: 5xx` 로 지정.
* **A7.** `cluster.<name>.upstream_cx_overflow` 카운터.
* **A8.** **SDS(Secret Discovery Service)** 를 통한 동적 비밀 배포.
