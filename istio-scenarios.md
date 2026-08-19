# Istio 使用情境與 Traffic Routing

這份筆記用實際案例理解 Istio traffic management。重點不是背 YAML，而是先判斷「要根據什麼條件分流」，再選擇 header、path、host、weight 或 locality。

## 一句話記憶

> `DestinationRule` 定義「有哪些版本或流量目的地」；`VirtualService` 定義「request 要去哪裡」。

```text
Kubernetes Service
  -> 找到符合 selector 的所有 Pod
DestinationRule
  -> 把 Pod 分成 v1、v2、canary、stable 等 subsets
VirtualService
  -> 根據 header、path、host 或 weight 選擇 subset
```

## 目前 lab 的 routing

目前 `app/hello` 有兩個 Deployment：

```text
hello-v1 -> version: v1
hello-v2 -> version: v2
```

`DestinationRule` 定義兩個 subsets：

```yaml
subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

`VirtualService` 定義：

```text
沒有 x-version header -> v1
x-version: v2          -> v2
```

測試：

```bash
curl http://hello.demo.svc.cluster.local:8080
curl -H "x-version: v2" http://hello.demo.svc.cluster.local:8080
```

## 情境一：Header-based routing

### 什麼時候使用

- 讓開發或 QA 測試指定版本。
- 讓內部員工先使用 beta 功能。
- Debug 某一個版本，而不影響其他使用者。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello
  namespace: demo
spec:
  hosts:
    - hello.demo.svc.cluster.local
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: hello.demo.svc.cluster.local
            subset: v2
            port:
              number: 8080
    - route:
        - destination:
            host: hello.demo.svc.cluster.local
            subset: v1
            port:
              number: 8080
```

效果：`x-canary: true` 進入 v2，其他 request 進入 v1。

如果 header 是 client 自己提供，不要把它當成可信的身分判斷。正式環境通常由 API Gateway、authentication proxy 或 trusted service 注入與驗證。

## 情境二：Canary release / weighted routing

### 什麼時候使用

- 新版先承受少量 production 流量。
- 觀察新版的 error rate、latency、CPU 與 memory。
- 確認穩定後逐步提高流量，出問題時快速降回舊版。

```yaml
http:
  - route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v1
          port:
            number: 8080
        weight: 95
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v2
          port:
            number: 8080
        weight: 5
```

這代表大約 95% 流量到 v1、5% 到 v2。常見 rollout：

```text
100% v1 / 0% v2
  -> 95% v1 / 5% v2
  -> 90% v1 / 10% v2
  -> 50% v1 / 50% v2
  -> 0% v1 / 100% v2
```

權重是 traffic policy，不代表 Pod 數量比例；即使 v1 有 10 個 Pod、v2 只有 1 個 Pod，也可以設定 95/5。

## 情境三：Header override + Canary

一般使用者走權重分流，指定測試者強制進入新版：

```yaml
http:
  # 具體條件放前面
  - match:
      - headers:
          x-canary:
            exact: "true"
    route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v2
        weight: 100

  # 一般流量走 canary weight
  - route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v1
        weight: 90
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v2
        weight: 10
```

```text
x-canary: true -> 100% v2
其他 request   -> 90% v1、10% v2
```

## 情境四：Blue-Green deployment

適合新版本先完整部署與驗證，但希望一次切換全部流量，並能快速回滾：

```text
Blue  = 目前穩定版本
Green = 已部署但尚未接正式流量的新版本
```

切換前後只需改 route：

```yaml
# before
route:
  - destination:
      host: hello.demo.svc.cluster.local
      subset: blue
    weight: 100

# after
route:
  - destination:
      host: hello.demo.svc.cluster.local
      subset: green
    weight: 100
```

| 方法 | 流量變化 | 主要優點 |
|---|---|---|
| Canary | 逐步增加新版比例 | 風險較小，可觀察真實流量 |
| Blue-Green | 通常一次切換全部流量 | 切換與回滾簡單、版本邊界清楚 |

## 情境五：依 URL path 分流

適合 API v1/v2 共存、新功能使用新 path，或不同業務路徑由不同 workload 處理：

```yaml
http:
  - match:
      - uri:
          prefix: /api/v2
    route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v2

  - match:
      - uri:
          prefix: /api/v1
    route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v1

  # default route 放最後
  - route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v1
```

HTTP route 通常由上往下匹配；越具體的 rule 應該越前面，fallback/default route 放最後。

## 情境六：依 Host 或 domain 分流

適合 `api.example.com` 提供 stable、`beta.example.com` 提供 beta，或多個 domain 共用 ingress gateway：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello-public
  namespace: demo
spec:
  hosts:
    - beta.example.com
  gateways:
    - public-gateway
  http:
    - route:
        - destination:
            host: hello.demo.svc.cluster.local
            subset: v2
```

外部流量通常需要同時設定 `Gateway`；目前 lab 只展示 cluster 內部 service-to-service routing。

## 情境七：依 tenant 或客戶分流

適合特定客戶先使用新版本，或 Enterprise customer 使用專用 workload：

```yaml
http:
  - match:
      - headers:
          x-tenant:
            exact: enterprise-a
    route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v2
  - route:
      - destination:
          host: hello.demo.svc.cluster.local
          subset: v1
```

不要只相信使用者自己傳入的 `x-tenant`；正式環境應由已驗證的 gateway 或 trusted upstream 產生這個 routing attribute。

## 情境八：Locality-aware routing 與 failover

適合優先使用同 zone Pod、控制跨 zone/region 成本，或 region 故障時切換到備援 region。通常會搭配：

- Kubernetes topology labels。
- Istio locality load balancing。
- `DestinationRule` 的 traffic policy。
- Multi-cluster mesh 或外部 global load balancer。

這是從單一 cluster routing 走向 platform-level traffic management 的進階情境，不一定只靠 `VirtualService` 完成。

## 情境九：故障演練與 resilience policy

這不完全是分流，但常和 routing 一起使用：timeout、retry、circuit breaker 與 fault injection。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello-resilience
  namespace: demo
spec:
  hosts:
    - hello.demo.svc.cluster.local
  http:
    - timeout: 3s
      retries:
        attempts: 2
        perTryTimeout: 1s
        retryOn: 5xx,connect-failure,reset
      route:
        - destination:
            host: hello.demo.svc.cluster.local
            subset: v1
```

Retry 要小心使用。付款、建立訂單等非 idempotent request 不應盲目重試，否則可能造成重複操作。

## Routing rule 實務慣例

### 1. Subset label 必須一致

`DestinationRule` 的 subset label 必須對得上 Deployment/Pod：

```yaml
metadata:
  labels:
    app: hello
    version: v2
```

如果 label 對不上，YAML 可能沒有語法錯誤，但 route 實際上沒有可用 endpoint。

### 2. Default route 永遠放最後

推薦順序：

```text
特定 header
  -> 特定 path
  -> 特定 host
  -> weighted split
  -> default route
```

### 3. 同一個 host 儘量由單一 VirtualService 管理

多個團隊各自建立同一 host 的 VirtualService，會增加規則衝突與排錯成本。大型組織通常會定義 ownership、命名與審查規則。

### 4. Production 使用完整 service FQDN

推薦使用 `hello.demo.svc.cluster.local`，而不是只寫 `hello`，降低 namespace 或 search domain 造成誤判的機率。

### 5. Weight 必須搭配觀測資料

每次提高 canary 流量前，至少觀察 request rate、error rate、latency、Pod restart 與 CPU/memory。沒有 metrics 的 weighted routing 只是盲目分流。

### 6. 每個 routing strategy 都要有回滾方式

```text
Canary      -> 把新版 weight 調回 0
Blue-Green  -> route 切回 blue
Header      -> 移除 override 或停用 header
Path        -> 把 path route 指回 stable
```

### 7. 驗證 Kubernetes resource 與 Envoy 實際狀態

```bash
kubectl get virtualservice,destinationrule -n demo
istioctl analyze -n demo
istioctl proxy-status
istioctl proxy-config routes <pod-name> -n demo
```

設定存在不代表 proxy 已收到正確 configuration；排障時要同時看 Kubernetes resource、Envoy config 與實際 response。

## Lab 練習順序

### 練習一：Header routing

```bash
curl http://hello.demo.svc.cluster.local:8080
curl -H "x-version: v2" http://hello.demo.svc.cluster.local:8080
```

### 練習二：90/10 Canary

把 `VirtualService` 改成 90% v1、10% v2，再送出多次 request：

```bash
for i in $(seq 1 20); do
  curl -s http://hello.demo.svc.cluster.local:8080
done
```

### 練習三：Header override + Canary

讓一般流量走 90/10，但 `x-canary: true` 永遠走 v2，觀察 rule 順序與 fallback 的關係。

### 練習四：搭配 Grafana

逐步提高 v2 流量，同時觀察 v2 error rate、latency、request volume 與 Pod restart。

## 何時不需要 Istio

以下情況可能只用 Kubernetes Service、Ingress 或 API Gateway 就足夠：

- 只有簡單的 north-south ingress routing。
- 不需要 service-to-service 的 L7 policy。
- 沒有 canary、mTLS、細緻 telemetry 或 resilience policy 需求。
- 團隊尚未準備好維護 sidecar、control plane 與 traffic configuration 的複雜度。

引入 Istio 的判斷標準不應是「大家都在用」，而是確實需要它提供的 routing、security、telemetry 或 resilience 能力。
