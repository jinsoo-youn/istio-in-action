아래 예제는 **『Istio in Action』 4장**(Gateway)에서 다루는 핵심 구성을 “그대로 실행 → 검증”할 수 있도록 재구성한 실습 세트입니다.
(쿠버네티스 클러스터는 *kind* · minikube · EKS 등 어떤 환경이든 동일하게 동작합니다)

---

## 0. 전제 & 공통 준비

```bash
# ❶ 네임스페이스 준비 + 사이드카 자동 주입
kubectl create ns istioinaction
kubectl label ns istioinaction istio-injection=enabled

# ❷ 샘플 앱 배포 (webapp:80, mysql:3306, echo-tcp:9000)
kubectl -n istioinaction run webapp      --image=nginxdemos/hello   --port 80
kubectl -n istioinaction expose pod webapp      --port 80

kubectl -n istioinaction run echo-tcp    --image=ealen/echo-server --port 9000 --env TCP_PORT=9000
kubectl -n istioinaction expose pod echo-tcp    --port 9000

kubectl -n istioinaction run mysql       --image=mysql:8 --port 3306 \
        --env MYSQL_ROOT_PASSWORD=rootpw
kubectl -n istioinaction expose pod mysql        --port 3306
```

> **Ingress IP 확인**
> *LoadBalancer 클러스터*
>
> ```bash
> export GW_IP=$(kubectl -n istio-system get svc istio-ingressgateway \
>                -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
> ```
>
> *kind/minikube* (NodePort > 32080/32443)
>
> ```bash
> export GW_IP=127.0.0.1
> ```

---

## 1. HTTP + HTTPS Gateway

### 1-1 gateway-http-tls.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gw
  namespace: istio-system
spec:
  selector: {istio: ingressgateway}
  servers:
  # HTTP
  - port: {number: 80, name: http, protocol: HTTP}
    hosts: ["webapp.istioinaction.io"]
  # HTTPS (서버 TLS 종료)
  - port: {number: 443, name: https, protocol: HTTPS}
    tls:
      mode: SIMPLE
      credentialName: webapp-tls        # kubernetes.io/tls Secret
    hosts: ["webapp.istioinaction.io"]
```

(테스트용이면 `credentialName` 대신 `tls: {mode: PASSTHROUGH}` 로 일단 생략 가능)

### 1-2 webapp-virtualservice.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: webapp-vs
  namespace: istioinaction
spec:
  hosts: ["webapp.istioinaction.io"]
  gateways: ["istio-system/public-gw"]
  http:
  - route:
    - destination:
        host: webapp.istioinaction.svc.cluster.local
        port: {number: 80}
```

```bash
kubectl apply -f gateway-http-tls.yaml
kubectl apply -f webapp-virtualservice.yaml
```

### 1-3 검증

```bash
curl -H "Host: webapp.istioinaction.io" http://$GW_IP   | grep "<title>"
# HTTPS (자체 서명이라면 -k)
curl -k --resolve webapp.istioinaction.io:443:$GW_IP https://webapp.istioinaction.io
```

---

## 2. SNI Passthrough (TLS 그대로 내보내기)

### 2-1 gateway-sni.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata: {name: tls-passthrough-gw, namespace: istio-system}
spec:
  selector: {istio: ingressgateway}
  servers:
  - port: {number: 443, name: tls, protocol: TLS}
    tls:  {mode: PASSTHROUGH}
    hosts: ["mysql.istioinaction.io"]
```

### 2-2 mysql-vs.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata: {name: mysql-vs, namespace: istioinaction}
spec:
  hosts: ["mysql.istioinaction.io"]
  gateways: ["istio-system/tls-passthrough-gw"]
  tls:
  - match:
    - port: 443
      sniHosts: ["mysql.istioinaction.io"]
    route:
    - destination:
        host: mysql.istioinaction.svc.cluster.local
        port: {number: 3306}
```

```bash
kubectl apply -f gateway-sni.yaml
kubectl apply -f mysql-vs.yaml
```

### 2-3 검증

```bash
openssl s_client -connect ${GW_IP}:443 -servername mysql.istioinaction.io \
        -quiet 2>/dev/null | head
# TCP 접속 확인
```

---

## 3. 순수 TCP (예: Telnet-Echo)

### 3-1 gateway-tcp.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata: {name: tcp-gw, namespace: istio-system}
spec:
  selector: {istio: ingressgateway}
  servers:
  - port: {number: 31400, name: tcp-echo, protocol: TCP}
    hosts: ["*"]        # TCP에는 호스트 개념 X
```

### 3-2 echo-vs.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata: {name: echo-vs, namespace: istioinaction}
spec:
  gateways: ["istio-system/tcp-gw"]
  hosts: ["*"]
  tcp:
  - match: [{port: 31400}]
    route:
    - destination:
        host: echo-tcp.istioinaction.svc.cluster.local
        port: {number: 9000}
```

```bash
kubectl apply -f gateway-tcp.yaml
kubectl apply -f echo-vs.yaml
```

### 3-3 검증

```bash
nc $GW_IP 31400
# → 입력한 문자열이 그대로 echo 되면 성공
```

---

## 4. 게이트웨이 Access Log 켜기 (선택)

```yaml
# istio-system/gw-telemetry.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata: {name: ingress-logs, namespace: istio-system}
spec:
  selector:
    matchLabels:
      istio: ingressgateway        # 해당 게이트웨이 Pod 라벨
  accessLogging:
  - providers:
    - name: envoy                 # 기본 로그 포맷
      disabled: false
```

```bash
kubectl apply -f gw-telemetry.yaml
kubectl -n istio-system logs deploy/istio-ingressgateway -f
```

---

## 5. 다중 게이트웨이 운영(요약)

| 게이트웨이 Pod     | Gateway 리소스                      | VirtualService              |
| ------------- | -------------------------------- | --------------------------- |
| `public-gw`   | `selector: {istio: public-gw}`   | `gateways: ["public-gw"]`   |
| `internal-gw` | `selector: {istio: internal-gw}` | `gateways: ["internal-gw"]` |

IstioOperator 한 파일에서 `components.ingressGateways` 배열로 여러 Deployment/Service를 동시에 선언하면 클러스터 외부 IP·HPA·PodSecurity를 완벽히 분리해 운영할 수 있다.
