# Istio / istioctl / Envoy

## 一句話定位

Istio 是 service mesh；它將 service-to-service traffic policy、routing 與 telemetry 從 application code 抽離。sidecar mode 下，每個被納管的 Pod 會多一個 Envoy proxy；`istioctl` 是安裝、診斷與檢查 Istio 的 CLI。

## 官方文件

- [Istio Overview](https://istio.io/latest/docs/overview/)
- [Istio Concepts](https://istio.io/latest/docs/concepts/)
- [Install with istioctl](https://istio.io/latest/docs/setup/install/istioctl/)
- [Using istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/)
- [istioctl analyze](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/)

## 三個角色

| 元件 | 角色 |
|---|---|
| Istio control plane / `istiod` | 接收 mesh configuration，將設定分發給 proxy |
| Envoy sidecar | 在 workload 旁攔截並轉送 service traffic |
| `istioctl` | 安裝 Istio、檢查 proxy status、分析 configuration |

## 它解決什麼問題

Kubernetes Service 能提供基本 service discovery，但不直接表達「帶有某個 header 的 request 要去 v2」。Istio 以 `VirtualService`、`DestinationRule` 等 CRD 提供 L7 traffic management，並能產生 service mesh telemetry。

## 在本專案的價值

`app/hello` 用最小規模展示 Istio traffic routing：

1. `namespace.yaml` 設定 `istio-injection=enabled`。
2. `deployment-v1.yaml` 與 `deployment-v2.yaml` 讓每個 hello Pod 被注入 `istio-proxy`。
3. `destination-rule.yaml` 建立 `v1`、`v2` subsets。
4. `virtual-service.yaml` 將 `x-version: v2` 導向 v2，其餘導向 v1。

```text
request without header  -> hello-v1
request x-version: v2   -> hello-v2
```

常用驗證與排障指令：

```bash
istioctl version
istioctl proxy-status
istioctl analyze -n demo
kubectl get virtualservice,destinationrule -n demo
```

## 它與 Prometheus 的關係

Istio/Envoy 可以產生 request metrics，但 Prometheus 必須有對應 scrape target 或 `ServiceMonitor` 才能收集。這個 lab 若未完成 Istio metrics scrape，仍先使用 Kubernetes metrics 完成 monitoring 成果，並將缺口記錄在 `monitoring/notes.md`。

## 本次 lab 的邊界

只展示 sidecar injection 與 header-based routing，不做 mTLS、authorization policy、gateway、multi-cluster、ambient mode 或 production traffic shifting。這些是後續學習方向，不是本 repo 的必要驗收項目。
