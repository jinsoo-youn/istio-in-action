# Chapter 5 정리 ― **Traffic control: fine-grained routing**

Istio에서는 **DestinationRule**(버전 정의) + **VirtualService**(라우팅 정책) 두 리소스로
서비스 버전(v1, v2, …) 간 트래픽을 자유롭게 분배‧미러링‧A/B 테스트할 수 있다.

---

## 1. 핵심 리소스 역할

| 리소스                 | 핵심 필드                                       | 기능                                  |
| ------------------- | ------------------------------------------- | ----------------------------------- |
| **DestinationRule** | `subsets` (name + labels)                   | 같은 Service 안에서 **버전(Subset) 그룹** 정의 |
| **VirtualService**  | `http / tls / tcp`·`match`·`route`·`mirror` | 요청을 어떤 Subset으로 보낼지 **정책** 기술       |

> → **순서**: ① `DestinationRule`로 서브셋 선언 → ② `VirtualService`에서 `destination.subset`으로 참조.

---

## 2. 기본 예제

### 2-1 Subset 정의 (DestinationRule)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
  namespace: istioinaction
spec:
  host: catalog.istioinaction.svc.cluster.local
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
      version: v2
```

### 2-2 Ingress 전용 라우트(모두 v1)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs
  namespace: istioinaction
spec:
  hosts:
    - catalog.istioinaction.io
  gateways:
    - catalog-gateway
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
        port:
          number: 8080
```

> *Mesh 내부 호출도 함께* 제어하려면 `gateways: ["mesh"]` 를 포함한 VS 를 별도로 만든다.

---

## 3. 트래픽-분배 패턴

### 3-1 가중치 카나리 (v2 10 %)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs-10v2
  namespace: istioinaction
spec:
  hosts:
    - catalog.istioinaction.io
  gateways:
    - catalog-gateway
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
        port:
          number: 8080
      weight: 90
    - destination:
        host: catalog
        subset: version-v2
        port:
          number: 8080
      weight: 10
```

### 3-2 헤더 기반 A/B

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs-header
  namespace: istioinaction
spec:
  hosts:
    - catalog.istioinaction.io
  gateways:
    - catalog-gateway
  http:
  - match:
    - headers:
        x-dark-launch:
          exact: v2                 # 조건 (= 내부 테스터)
    route:
    - destination:
        host: catalog
        subset: version-v2
        port:
          number: 8080
  - route:
    - destination:
        host: catalog
        subset: version-v1
        port:
          number: 8080
```

### 3-3 트래픽 미러링(dark-launch)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs-mirror
  namespace: istioinaction
spec:
  hosts:
    - catalog.istioinaction.io
  gateways:
    - catalog-gateway
  http:
  - route:                       # 클라이언트 응답은 v1
    - destination:
        host: catalog
        subset: version-v1
        port:
          number: 8080
      weight: 100
    mirror:                      # 요청 사본을 v2 로 전송
      host: catalog
      subset: version-v2
      port:
        number: 8080
```

---

## 4. SNI Passthrough 예 (TLS 그대로 전달)

### Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: tls-passthrough-gw
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: tls
      protocol: TLS
    tls:
      mode: PASSTHROUGH
    hosts:
      - mysql.istioinaction.io
```

### VirtualService

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: mysql-vs
  namespace: istioinaction
spec:
  hosts:
    - mysql.istioinaction.io
  gateways:
    - tls-passthrough-gw
  tls:
  - match:
    - port: 443
      sniHosts:
        - mysql.istioinaction.io
    route:
    - destination:
        host: mysql
        port:
          number: 3306
```

---

## 5. 모니터링 & 검증

* **curl / hey** — 헤더·쿠키를 바꿔 실제 버전 분기 확인
* `istioctl proxy-config routes <pod>` — Envoy 라우팅 테이블 확인
* Prometheus `istio_requests_total{destination_version="v2"}` — 가중치 반영 비율 모니터링
* Grafana·Kiali — 버전별 RPS·지연·에러율 시각화

---

## 6. 운영 팁

1. Subset 이름과 Deployment `version` 라벨을 반드시 일치시켜야 엔드포인트 누락을 방지.
2. Weight 변경은 `kubectl apply` 만으로 *무중단* 수행(Envoy RDS 교체).
3. **OutlierDetection** 을 DestinationRule 에 추가해 오류 Pod 자동 배제.
4. Gateway VS 와 Mesh VS 를 분리 배포하면 “내부 안착 → 외부 노출” 2-단계 롤아웃이 쉬워진다.

---

## 7. 복습 퀴즈

| 질문                                                           | 답                                                  |
| ------------------------------------------------------------ | -------------------------------------------------- |
| **Q1** DestinationRule 에서 v2 서브셋을 정의할 때 필수 필드 두 가지는?         | `name: version-v2`, `labels.version: v2`           |
| **Q2** 80/20 가중치 전환 중 v2 5xx 폭증 시, 다운타임 없이 즉시 0 %로 되돌리는 방법은? | `VirtualService` 가중치 값 재조정 후 `kubectl apply`       |
| **Q3** 헤더 기반 라우팅을 Mesh 내부에도 동일 적용하려면 VS 의 어떤 필드를 어떻게 설정?     | `gateways` 에 `"mesh"` 추가 (또는 mesh-전용 VS 작성)        |
| **Q4** 미러링 실험에서 v2 응답이 없는데 Grafana RPS 는 증가, 정상일까? 이유는?      | 정상. 미러링은 요청 복사만 보내고 응답은 버려서 로그는 없지만 RPS 메트릭은 집계된다. |
