## Chapter 4 Self-Check Quiz

*Istio Gateways: Getting Traffic Into a Cluster*

## Self-Check Quiz

1. Gateway와 VirtualService가 각기 담당하는 계층은 무엇인가?
2. SIMPLE TLS와 PASSTHROUGH TLS의 차이를 한 줄로 설명하라.
3. 다중 Ingress Gateway를 IstioOperator로 배포할 때 VirtualService에서 반드시 지정해야 하는 필드는?
4. Egress Gateway를 통해 특정 외부 도메인만 고정 IP로 나가게 하려면 필요한 리소스 3가지는?
5. `PILOT_FILTER_GATEWAY_CLUSTER_CONFIG` 설정을 사용하면 얻는 두 가지 이점은?
6. Gateway Injection을 활성화하기 위해 Deployment 메타데이터에 들어가는 필수 annotation 두 개는?
7. mTLS 환경에서 Ingress Gateway를 MUTUAL TLS로 설정했을 때 클라이언트가 제공해야 하는 것은?

---

### 정답

**A1.** Gateway = L4/L5 포트·호스트·TLS, VirtualService = L7 경로·헤더·가중치.
**A2.** SIMPLE은 게이트웨이가 TLS를 종료, PASSTHROUGH는 SNI만 보고 TLS는 백엔드에서 종료.
**A3.** `selector` 라벨(`gateways: ["public-gw"]` 식) – 어떤 게이트웨이에 적용할지 명시.
**A4.** ① Egress Gateway Deployment/Service, ② ServiceEntry(외부 호스트), ③ VirtualService(egress gateway 지정).
**A5.** Envoy에 필요한 클러스터만 내려 **메모리·CPU 절감** / xDS 패킷 수 감소로 **컨트롤-플레인 부하 감소**.
**A6.** `sidecar.istio.io/inject: "true"`, `inject.istio.io/templates: gateway`.
**A7.** 클라이언트 X.509 인증서 + 프라이빗 키(또는 SDS 제공 토큰).


---

### 문제 (스스로 풀어보세요)

1. **게이트웨이(Gateway) 리소스**와 **VirtualService 리소스**가 각각 담당하는 네트워크 계층(L4–L7)은 무엇인가?
2. Gateway `servers[].tls.mode` 에서 선택할 수 있는 세 가지 값(SIMPLE, MUTUAL, PASSTHROUGH)이 서로 다른 점을 한 줄씩 설명하라.
3. 책의 예제에서 `curl` 호출 시 `Host: webapp.istioinaction.io` 헤더를 반드시 넣어야 하는 이유는?
4. 하나의 443 포트에서 두 개의 서로 다른 도메인(예: *webapp* 와 *catalog*)을 호스팅하려면 Gateway 와 VirtualService 를 어떻게 설정해야 하는가? 핵심 키워드 두 개만 적어라.
5. TCP 기반 MySQL 서비스(3306/TLS)를 **백엔드에서 직접 TLS 종료**하도록 게이트웨이를 설계하려면 `protocol` 과 `mode` 를 각각 무엇으로 설정해야 하나?
6. 다중 인그레스 게이트웨이를 IstioOperator 로 배포할 때 VirtualService 마다 **반드시** 지정해야 하는 필드는?
7. Deployment 에 아래 annotation 을 추가해 주면 무엇이 생성되는가?

   ```yaml
   sidecar.istio.io/inject: "true"
   inject.istio.io/templates: gateway
   ```
8. `meshConfig.accessLogFile` 대신 **Telemetry API** 를 이용해 특정 Gateway 의 Access Log 만 켤 수 있는 이유는? 핵심 키워드(스코프)를 써라.
9. `PILOT_FILTER_GATEWAY_CLUSTER_CONFIG=true` 플래그를 켜면 게이트웨이에 어떤 두 가지 이점이 생기는가?
10. Egress Gateway 로 특정 외부 도메인(`api.partner.com`)만 고정 IP 로 나가게 하려면 ServiceEntry 외에 꼭 작성해야 하는 Istio 리소스는 무엇인가?

---

### 정답

1. Gateway = L4/L5(포트·프로토콜·호스트), VirtualService = L7(경로·헤더·가중치)
2. * **SIMPLE** : 게이트웨이에서 서버 TLS 종료(HTTPS)
* **MUTUAL** : 게이트웨이에서 mTLS, 클라이언트 인증서 필요
* **PASSTHROUGH** : 게이트웨이는 SNI만 보고 라우팅, TLS 세션은 백엔드에서 종료
3. Listener 가 `hosts: ["webapp.istioinaction.io"]` 로 제한돼 있어 Host 헤더가 일치하지 않으면 Envoy 가 “Blackhole” 라우트로 보내기 때문
4. **SNI** 와 **PASSTHROUGH** (Gateway `tls.mode: PASSTHROUGH`, VirtualService `tls.match.sniHosts`)
5. `protocol: TLS`, `tls.mode: PASSTHROUGH`
6. `gateways:` 필드에 자신이 사용할 게이트웨이 라벨(예 `["public-gw"]`)을 지정
7. 애플리케이션 Pod 대신 **Envoy Gateway Deployment**(gateway 템플릿) 가 자동 생성됨
8. Telemetry API 는 **workload-selector 스코프** 로 게이트웨이 Pod 만 골라 Access Log 설정을 오버라이드할 수 있다.
9. (1) 필요 없는 Cluster 설정을 제거해 **Gateway 메모리·CPU 감소**, (2) xDS 전달 트래픽이 줄어 **Pilot 부하 감소**
10. **VirtualService** – `gateways: ["mesh", "<egress-gw>"]` 로 지정해 내부 트래픽을 해당 Egress Gateway 로 강제 라우팅해야 한다.
