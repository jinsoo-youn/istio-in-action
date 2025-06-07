
### 2.4.3 Istio for traffic routing

version에 따라 routing을 조절할 수 있음
예)  catalog service V1 와 V2  
V1 of the catalog service
```json
{
 "id": 1,
  "color": "amber",
  "price": "10.99"
}
```

V2 of the catalog service
```json
{
 "id": 1,
  "color": "amber",
  "price": "10.99",
  "imageUrl": "https://example.com/image.jpg"
}
```

위 경우 DestinationRule을 사용하여 V1과 V2를 구분할 수 있음
```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: catalog
spec:
  host: catalog
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
      version: v2
```

```bash
$ kubectl apply -f ch2/catalog-destinationrule.yaml
```

그다음 VirtualService를 사용하여 라우팅을 설정할 수 있음
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
```
이제 virtual service를 적용하면 catalog 서비스에 대한 요청은 V1로 라우팅됨
```bash
$ kubectl apply -f ch2/catalog-virtualservice-all-v1.yaml
```

Now, if we send traffic to our webapp endpoint, we see only v1 responses:
```bash
$ while true; do curl http://localhost/api/catalog; sleep .5; done
```

이제 특정 유저는 v2 catalog 서비스로 라우팅되도록 설정할 수 있음
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v2
    match:
    - headers:
        x-dark-launch:
          exact: "v2"
  - route:
    - destination:
        host: catalog
        subset: version-v1
```

Let's create this new routing rule in our VirtualService:
```bash
$ kubectl apply -f ch2/catalog-virtualservice-dark-launch.yaml
```

let's call endpoint with our special header `x-dark-launch`
```bash
$ curl http://localhost/api/catalog -H "x-dark-launch: v2"
[
  {"id":1,"color":"amber","price":"10.99","imageUrl":"https://example.com/image.jpg"}
]
```

# Summary
* We can use `istioctl` to install Istio and `istioctl x precheck` to verify that Istio can be installed in a cluster.
* Istio's configuration is implemented as Kubernetes custom resources.
* To configure proxies, we describe the intent in YAML (according to the Istio custom resources) and apply it to the cluster. 
* The control plane watches for Istio resources, converts them to Envoy configuration, and uses the xDS API to dynamically update Envoy proxies.
* Inbound and outbound traffic to and from the mesh is managed by ingress and egress gateways. 
* The sidecar proxy can be injected manually into YAML using `istioctl kube-inject` 
* In maespaces labeled with `istio-injection=enabled`, the proxies are automatically injected into newly created Pods. 
* We can use the `VirtualService` API to manipulate application network traffic, such as implementing retries on failed requests. 


istioctl docs link: https://istio.io/latest/docs/reference/commands/istioctl/