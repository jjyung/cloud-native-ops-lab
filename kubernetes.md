# Kubernetes

## 一句話定位

Kubernetes 是負責管理 containerized workloads 與 services 的 orchestration platform；本專案的所有 Deployment、Service、Namespace 與 CRD 都由 Kubernetes API 管理。

## 官方文件

- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
- [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)

## 它解決什麼問題

Kubernetes 把「我希望 cluster 最後長什麼樣子」描述成 declarative resources，然後由 control plane 持續讓實際狀態接近 desired state。核心概念包括：

- **Pod**：最小可部署單位。
- **Deployment**：維持 Pod replicas，並管理 rollout 與 self-healing。
- **Service**：提供穩定的 service discovery 與流量入口。
- **Namespace**：隔離與組織資源。
- **CRD**：讓 Istio 與 Argo CD 擴充 Kubernetes API。

## 在本專案的價值

Kubernetes 是所有工具的共同平台：

| Kubernetes resource | 本專案用途 |
|---|---|
| `Namespace/demo` | 放置 hello application，並開啟 Istio sidecar injection |
| `Deployment/hello-v1` | 維持 v1 Pod |
| `Deployment/hello-v2` | 維持 v2 Pod |
| `Service/hello` | 提供統一的 `hello:8080` service endpoint |
| `DestinationRule` / `VirtualService` | 由 Istio 擴充的流量規則 |
| `Application` | 由 Argo CD 擴充的 GitOps 管理物件 |

## 這個 lab 展示的 Kubernetes 能力

1. Deployment 建立 Pod 並在 Pod 被刪除後 self-healing。
2. Service 將 v1/v2 Pod 組成一個可被 discovery 的服務。
3. `kubectl describe`、`logs`、`events`、`rollout status` 形成基本排障路徑。
4. Argo CD 將 Git desired state 套用到 Kubernetes cluster。

## 本次 lab 的邊界

這不是 production-grade cluster：只有 local kind node，不做 HA control plane、multi-cluster、TLS PKI 或 persistent storage 設計。重點是先理解 workload、networking、delivery、observability 的關係。
