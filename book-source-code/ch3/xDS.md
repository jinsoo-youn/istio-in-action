# \[보충] Envoy **xDS** 실습 가이드

*(Chapter 3 부록 – “실제 손으로 해 보는 Dynamic Config”)*

> **목표** – 로컬 Mac M1 + Docker Desktop 환경에서 **ADS(완전 동적 모드)** 를 사용해 Envoy 구성을 실시간으로 바꿔 본다.
> 학습 포인트
>
> 1. xDS API(LDS‧RDS‧CDS‧EDS‧SDS)의 역할
> 2. 최소한의 **Control Plane**(go-control-plane) 만들기
> 3. Envoy가 **ADS 스트림**을 통해 설정 스냅샷을 받는 흐름 체험

---

## 1. xDS 아주 짧은 개념 복습

| 축약      | 풀네임 / 내용                                                 |
| ------- | -------------------------------------------------------- |
| **LDS** | Listener Discovery – 어떤 포트를 열 것인가                        |
| **RDS** | Route Discovery – HTTP 라우팅 규칙                            |
| **CDS** | Cluster Discovery – 백엔드(풀) 이름·LB 정책                      |
| **EDS** | Endpoint Discovery – 각 Cluster가 가진 개별 IP/Port            |
| **SDS** | Secret Discovery – TLS 인증서·키 등 보안 비밀                     |
| **ADS** | Aggregated Discovery – 위 ①\~⑤를 **하나의 gRPC 스트림**으로 묶어서 전송 |

Istio의 `istiod`는 ADS 서버이지만, 이번 실습에서는 **경량 예제**로 자체 Control-Plane 을 띄운다.

---

## 2. 실습 환경 준비

### 2-1 Docker 이미지 받기

```bash
docker pull envoyproxy/envoy:v1.29-latest
docker pull ghcr.io/karimra/go-control-plane-sample:latest   # 30 MB 미만
```

*(go-control-plane 샘플: 위 이미지에는 Envoy 공식 go-control-plane 예제 코드가 빌드돼 있다.)*

### 2-2 작업 폴더 구조

```
xds-demo/
 ├─ bootstrap.yaml        # Envoy 부트스트랩
 ├─ config-sample.json    # Control-plane 이 읽을 스냅샷
 └─ docker-compose.yml
```

---

## 3. 파일 작성

### 3-1 bootstrap.yaml (Envoy)

```yaml
static_resources:
  admins:
  - address:
      socket_address: { address: 0.0.0.0, port_value: 9901 }

dynamic_resources:
  ads_config:
    transport_api_version: V3
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: xds_cluster

  cds_config: { ads: {} }
  lds_config: { ads: {} }

static_resources:
  clusters:
  - name: xds_cluster
    connect_timeout: 1s
    type: STATIC
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: controlplane, port_value: 18000 }
```

포인트

* Envoy는 **ADS(gRPC)** 로 Listener·Cluster·Route 를 모두 받는다.
* `xds_cluster` → Docker Compose 서비스명 `controlplane` 과 매핑.

### 3-2 config-sample.json (스냅샷)

```json
{
  "listeners": [
    {
      "name": "demo-listener",
      "address": {
        "socket_address": { "address": "0.0.0.0", "port_value": 10000 }
      },
      "filter_chains": [
        {
          "filters": [
            {
              "name": "envoy.filters.network.http_connection_manager",
              "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                "stat_prefix": "ingress",
                "route_config": {
                  "name": "local_route",
                  "virtual_hosts": [
                    {
                      "name": "demo",
                      "domains": ["*"],
                      "routes": [
                        {
                          "match": { "prefix": "/" },
                          "route": { "cluster": "httpbin_cluster" }
                        }
                      ]
                    }
                  ]
                },
                "http_filters": [
                  { "name": "envoy.filters.http.router" }
                ]
              }
            }
          ]
        }
      ]
    }
  ],
  "clusters": [
    {
      "name": "httpbin_cluster",
      "connect_timeout": "2s",
      "type": "LOGICAL_DNS",
      "load_assignment": {
        "cluster_name": "httpbin_cluster",
        "endpoints": [
          {
            "lb_endpoints": [
              {
                "endpoint": {
                  "address": {
                    "socket_address": { "address": "httpbin", "port_value": 80 }
                  }
                }
              }
            ]
          }
        ]
      }
    }
  ],
  "endpoints": [],
  "routes": [],
  "secrets": []
}
```

### 3-3 docker-compose.yml

```yaml
version: "3.8"
services:
  httpbin:
    image: kennethreitz/httpbin
    restart: unless-stopped

  controlplane:
    image: ghcr.io/karimra/go-control-plane-sample:latest
    environment:
      - GCPS_SNAPSHOT_FILE=/data/config-sample.json
      - GCPS_PORT=18000
    volumes:
      - ./config-sample.json:/data/config-sample.json

  envoy:
    image: envoyproxy/envoy:v1.29-latest
    command: ["envoy", "-c", "/etc/envoy/bootstrap.yaml", "-l", "info"]
    ports:
      - "10000:10000"   # 애플리케이션 포트
      - "9901:9901"     # Admin API
    volumes:
      - ./bootstrap.yaml:/etc/envoy/bootstrap.yaml
    depends_on:
      - controlplane
```

---

## 4. 실행 & 검증

```bash
cd xds-demo
docker compose up -d
sleep 5
curl -I http://localhost:10000/get
```

**HTTP/1.1 200 OK** 가 나오면 Listener ➜ Route ➜ Cluster ➜ httpbin 흐름이 정상.

### 4-1 Admin API 확인

```bash
curl localhost:9901/config_dump | jq '.configs[] | select(.@type=="type.googleapis.com/envoy.admin.v3.BootstrapConfigDump")'
```

`dynamic_resources` 섹션에 control-plane 로부터 받은 구성(LDS/CDS 등)이 보인다.

---

## 5. 실시간 구성 변경 시연

1. `config-sample.json` 을 열어 **`connect_timeout`** 값을 2s → 500ms 로 수정.

2. control-plane 컨테이너 재시작

   ```bash
   docker compose restart controlplane
   ```

3. Envoy 로그에서 다음과 같이 **ADS 업데이트** 를 수신한 메시지를 볼 수 있다.

   ```
   [info] ads stream PAUSED
   [info] CDS: add 0 update 1 remove 0
   ```

4. 다시 `/delay/1` 호출 → 500 ms 타임아웃으로 504 Gateway Timeout 발생.

---

## 6. Istio와 매칭해 보기

| 실습 요소                  | Istio 대응                                          |
| ---------------------- | ------------------------------------------------- |
| go-control-plane 샘플    | `istiod`                                          |
| `config-sample.json`   | Pilot 내부 **xDS 스냅샷** (Sidecar·Gateway 리소스 기반)     |
| docker-compose `envoy` | Pod 의 **istio-proxy** 사이드카                        |
| 파일 수정 + 재시작            | `VirtualService`·`DestinationRule` 변경 후 **즉시 반영** |

실제 Istio에서는 YAML 을 `kubectl apply` 하는 순간 Pilot 이 새 스냅샷을 만들어 ADS 로 푸시한다. 위 미니 실습으로 “Envoy가 재시작 없이 실시간 업데이트”되는 원리를 직접 확인한 셈이다.

---

## 7. 추가 미션 (자율 과제)

1. **Route 추가**

    * `/status/418` 경로를 매칭해 httpbin `/status/418` 로 프록시하도록 RDS(virtual host) 수정
2. **TLS 적용**

    * Secrets 섹션에 자가 서명 인증서를 넣고 Listener → `transport_socket` 설정(SDS)으로 HTTPS 8443 포트 노출
3. **Outlier Detection**

    * Cluster 설정에 `outlier_detection` 추가하고 httpbin 컨테이너를 의도적으로 500 에러 내도록 해본 뒤 ejection 카운터 확인

---

### 마무리

이 실습으로

* **xDS(특히 ADS)** 흐름
* **Control-Plane ↔ Envoy** 실시간 구성 교체
* Istio가 “쿠버네티스 리소스 → Envoy 스냅샷” 으로 변환해 배포한다는 개념

을 구체적인 손동작으로 확인할 수 있다. 위 과제를 끝내면 Chapter 3 의 ‘데이터 플레인 & Envoy 내부’ 이해가 한층 깊어진다.

필요하면 다음 장(트래픽 관리 심화)이나 다른 xDS 예제(Incremental xDS)도 정리해 드리니 언제든 말씀하세요!
