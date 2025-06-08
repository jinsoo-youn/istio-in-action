### 왜 `istioctl profile` 명령이 사라졌나?

Istio 1.24(2024-11-07 발표)부터 **`istioctl profile …` 하위 명령군이 완전히 제거**되었습니다. 같은 릴리스 노트에 다음과 같이 명시돼 있습니다.

> “Removed `istioctl profile` command. The same information can be found in Istio documentation.” ([istio.io][1])

따라서 1.24 이후 버전(현재 사용 중인 1.26.1 포함)의 `istioctl --help` 출력에서는 `profile` 명령이 보이지 않습니다.

---

### 대체 방법

| 이전 사용법                                                              | 1.24+에서의 대안                                            | 설명                                                                         |
| ------------------------------------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------- |
| `istioctl profile list`                                             | **문서 확인**<br>혹은<br>`ls $ISTIO_HOME/manifests/profiles` | 기본 프로파일·플랫폼 프로파일 목록을 확인                                                    |
| `istioctl profile dump demo`                                        | `istioctl manifest generate --set profile=demo`        | 선택한 프로파일이 생성하는 **모든 K8s YAML** 출력                                          |
| `istioctl profile diff default demo`                                | 두 개의 `manifest generate` 결과를 diff 도구로 비교               | 내장 diff 기능이 없어져 일반 YAML 비교 도구 사용                                           |
| `istioctl install -p demo` 또는 `istioctl install --set profile=demo` | **변경 없음**                                              | `install/upgrade/manifest generate` 명령에서 `--set profile=<name>` 방식은 그대로 지원 |

> **TIP** YAML만 보고 싶으면 --dry-run 옵션을 함께 쓰면 클러스터에 적용되지 않습니다.
>
> ```bash
> istioctl manifest generate --set profile=minimal --dry-run > minimal.yaml
> ```

---

### 내 환경에서 프로파일 확인·사용 예시

```bash
# 1) 어떤 프로파일로 설치했는지 기억이 안 날 때
#    (IstioOperator CRD가 있다면 annotation에 남아 있지 않으므로
#     실제 설치 YAML을 확보해 diff 하는 편이 가장 정확합니다)
istioctl manifest generate --set profile=default --dry-run > guess.yaml
kubectl get all -n istio-system -oyaml > live.yaml
diff -u live.yaml guess.yaml | less

# 2) 제어 플레인만 먼저 설치하고 게이트웨이는 따로 두고 싶을 때
istioctl install --set profile=minimal -y
istioctl install \
  --set profile=default \
  --set components.ingressGateways[0].name=istio-ingressgateway \
  --revision=prod-gw
```

---

### 정리

* **사라진 것이 아니라 공식적으로 삭제**되었습니다.
* 프로파일 지정 자체(`--set profile=`)는 계속 지원되며,
  CLI에 내장돼 있던 *list/dump/diff* 기능만 빠졌습니다.
* 동일 기능은
  *Istio 문서, `manifests/profiles/` 디렉터리, `manifest generate` + diff* 로 대체합니다.

궁금한 점이나 추가 예제가 필요하면 말씀해 주세요!

[1]: https://istio.io/latest/news/releases/1.24.x/announcing-1.24/change-notes/ "Istio / Istio 1.24.0 Change Notes"
