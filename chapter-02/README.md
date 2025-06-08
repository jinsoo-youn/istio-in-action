# Chapter 2 ― First Steps with Istio (정리본)

## 1 | 데모 설치로 Istio 맛보기

* `istioctl install --set profile=demo -y` 한 줄로 제어 플레인과 기본 게이트웨이를 설치한다.
  설치 결과

  ```
  istiod                (컨트롤 플레인)
  istio-ingressgateway  (인그레스)
  istio-egressgateway   (이그레스)
  ```
* `istioctl verify-install` 로 배포 상태를 신속히 검증할 수 있다.
* Grafana‧Prometheus‧Jaeger‧Kiali 애드온이 함께 설치돼 관찰 지표와 트레이싱을 바로 활용할 수 있다.

## 2 | 컨트롤 플레인 한눈에 보기

`istiod`(단일 Pod)이 담당하는 핵심 기능

1. **프록시 설정 전파** – xDS API
2. **보안** – SPIFFE ID 기반 X.509 발급·회전
3. **사이드카 자동 주입** – Mutating Webhook
4. **텔레메트리 수집 통합**

Ingress/Egress Gateway는 Envoy지만 애플리케이션과 분리된 프록시로 클러스터 경계를 다룬다.

## 3 | 데이터 플레인 – 사이드카 주입

* **자동 주입**: 네임스페이스에 `istio-injection=enabled` 라벨을 달면 Pod 생성 시 `istio-proxy`(Envoy)와 `istio-init`(iptables 설정) 컨테이너가 자동 삽입된다.
* **수동 주입**:

  ```bash
  istioctl kube-inject -f deployment.yaml | kubectl apply -f -
  ```
* iptables 규칙으로 모든 트래픽을 Envoy로 우회한다.

## 4 | 첫 애플리케이션 배포

1. 네임스페이스 생성 & 사이드카 자동 주입 라벨링

   ```bash
   kubectl create ns istioinaction
   kubectl label ns istioinaction istio-injection=enabled
   ```
2. `catalog` (v1)와 `webapp` 서비스를 배포한다. Pod당 2개 컨테이너(앱 + 프록시)가 떠야 정상.

## 5 | Gateway + VirtualService로 외부 노출

* `ingress-gateway.yaml` ‒ Ingress Gateway 리소스와 `VirtualService`를 정의해 `webapp`을 외부 80 / 443 포트로 노출.
* Docker Desktop + kind 예시에서는 `http://localhost` 로 바로 접속 가능(NodePort를 80·443에 매핑).

## 6 | 트래픽 제어 맛보기

* **재시도·타임아웃**
  `catalog-virtualservice.yaml`

  ```yaml
  http:
  - retries:
      attempts: 3
      perTryTimeout: 2s
  ```

  5xx 발생 시 최대 3 회 재시도.
* **헤더 기반 라우팅(다크런치)**
  `x-dark-launch: v2` 헤더가 있으면 `subset: version-v2` 로 전송하도록 `VirtualService` 에 조건부 매칭을 추가.

## 7 | 관찰성

* **Grafana** – 서비스별 RPS, 오류율, p50/p95/p99 지연.
* **Jaeger** – 자동으로 삽입되는 Trace 헤더 덕분에 서비스 호출 체인이 시각화된다.
* **istioctl proxy-config** – 각 Envoy의 라우팅·클러스터·리스너 구성을 CLI로 확인.

## 8 | 보안

* 설치 직후 워크로드 간 통신은 기본으로 **mTLS**.
* `istiod` 가 24 시간짜리 X.509 SVID 를 발급하며 애플리케이션 코드는 인증서 관리가 필요 없다.

## 9 | 핵심 요약

1. demo 프로필 설치만으로 Istio 주요 컴포넌트를 손쉽게 체험한다.
2. 자동 주입된 사이드카(Envoy)는 트래픽 프록싱, 보안, 관찰성을 코드 수정 없이 제공한다.
3. Gateway · VirtualService 조합으로 외부 노출과 세밀한 트래픽 정책을 선언적으로 관리한다.
4. Resiliency(재시도)·관찰성·mTLS 보안을 “YAML 수정”만으로 확보할 수 있음을 실습으로 확인했다.

---

## Self-Check Quiz (심화 학습용)

**Q1.** `istioctl install --set profile=demo` 명령이 설치하는 기본 컴포넌트 3가지는?
**Q2.** 사이드카 자동 주입을 활성화하는 네임스페이스 라벨은 무엇인가?
**Q3.** `retries.attempts: 3` 설정은 어떤 상황에서 효과가 있는가?
**Q4.** Jaeger UI에서 `webapp` 서비스 트레이스가 보이지 않을 때 첫 점검 항목은?
**Q5.** `x-dark-launch: v2` 헤더에 따라 버전 v2로 라우팅하려면 어떤 Istio 리소스와 필드를 수정해야 하나?
**Q6.** Istio에 mTLS가 적용되었는지 CLI에서 확인하는 명령은?
**Q7.** kind 클러스터에서 Ingress Gateway 없이 외부 테스트를 할 때 매핑해야 하는 NodePort는?
**Q8.** 사이드카 주입 실패로 Pod가 `Init:Error` 상태일 때 빠르게 우회할 수 있는 방법은?

---

### 정답

| 문제     | 정답                                                                                                           |
| ------ | ------------------------------------------------------------------------------------------------------------ |
| **A1** | `istiod`, `istio-ingressgateway`, `istio-egressgateway`                                                      |
| **A2** | `istio-injection=enabled`                                                                                    |
| **A3** | 애플리케이션이 5xx 오류(또는 지정 오류 코드)를 반환할 때 Envoy가 최대 3 회까지 재시도 후 응답한다.                                               |
| **A4** | 애플리케이션이 Trace 헤더(B3 또는 W3C) 전달/전파를 하고 있는지 확인한다.                                                              |
| **A5** | `VirtualService` 의 `http.match` 조건과 `route.subset` 필드(예: `subset: version-v2`)를 설정한다.                        |
| **A6** | `istioctl authn tls-check <pod-name> <namespace>`                                                            |
| **A7** | HTTP : 32080, HTTPS : 32443 (예시 kind 설정)                                                                     |
| **A8** | 네임스페이스 라벨을 제거하거나 Istio 설치 시 사이드카 주입 웹훅을 비활성화(`--set values.sidecarInjectorWebhook.enabled=false`)하고 다시 배포한다. |

---

> **Tip**: 위 요약과 문제·답안을 한 파일로 보관하면 Chapter 2 핵심을 빠르게 복습할 수 있습니다. 필요 시 추가 보충이나 다음 장(Chapter 3) 정리도 알려 주세요!
