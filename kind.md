# kind

## 一句話定位

kind（Kubernetes IN Docker）使用 Docker containers 作為 Kubernetes nodes，快速建立 disposable local cluster。

## 官方文件

- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kind configuration](https://kind.sigs.k8s.io/docs/user/configuration/)
- [kind with Docker Desktop](https://kind.sigs.k8s.io/docs/user/quick-start/#settings-for-docker-desktop)

## 它解決什麼問題

要學 Kubernetes、Istio、Argo CD 與 monitoring，首先需要一個可重建的 cluster。kind 把 Kubernetes control plane 與 node 放進 Docker，讓 cluster 可以快速建立、刪除與重來。

```bash
kind create cluster --name cloud-native-lab
kubectl config use-context kind-cloud-native-lab
kubectl get nodes
```

## 在本專案的價值

kind 是這個 repo 的 local infrastructure layer：

1. 提供 Kubernetes API server，讓 `kubectl`、Helm、Istio 與 Argo CD 有共同目標。
2. 讓 Istio sidecar、Argo CD、Prometheus、Grafana 在同一個 cluster 中互動。
3. 讓整個 lab 能在 demo 後刪除，不污染正式環境。
4. 與 Docker Desktop 整合，適合本機快速重現。

## 為什麼不用 Minikube

Minikube 同樣是合理的 local Kubernetes 選項，也能使用 Docker driver；但本 repo 已選定 kind，避免同一份操作文件維護兩條 cluster lifecycle。重點不是 kind 比 Minikube「更 production」，而是固定一個可重現的執行環境。

參考：[Minikube Docker driver](https://minikube.sigs.k8s.io/docs/drivers/docker/)

## 本次 lab 的邊界

kind 是 local cluster，不代表 managed Kubernetes、GKE、EKS 或 production multi-node topology。它適合展示 Kubernetes resource 與 operator workflow，不適合推導 production capacity 或 availability 結論。
