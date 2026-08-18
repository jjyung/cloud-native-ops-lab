# kubectl

## 一句話定位

`kubectl` 是 Kubernetes command-line client，透過 kubeconfig 與 Kubernetes API server 溝通。

## 官方文件

- [kubectl overview](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl command reference](https://kubernetes.io/docs/reference/kubectl/generated/)
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/)

## 它解決什麼問題

`kubectl` 是操作與觀察 cluster 的主要入口。它不建立 cluster，也不負責 GitOps；它只是把 operator 的查詢、套用、排障操作送到 Kubernetes API。

## 在本專案的價值

本 repo 用 `kubectl` 串起安裝、驗證與故障排查：

| 操作 | 目的 |
|---|---|
| `kubectl config use-context` | 切換到 kind cluster |
| `kubectl apply -k app/hello` | 不經 Argo CD 時直接套用 manifests |
| `kubectl get pods` | 觀察 Pod 是否 ready，以及是否有 `istio-proxy` |
| `kubectl describe` / `logs` | 找出 image、probe、mount 或 application 問題 |
| `kubectl get events` | 觀察 scheduler、pull image、container lifecycle 事件 |
| `kubectl rollout status` | 確認 Deployment rollout 是否完成 |
| `kubectl port-forward` | 從本機存取 Prometheus、Grafana 或 Argo CD |
| `kubectl kustomize` | 在套用前預覽 Kustomize render 結果 |

## 建議的排查順序

```bash
kubectl get pods -n demo -o wide
kubectl describe pod -n demo <pod-name>
kubectl logs -n demo <pod-name> -c <container-name>
kubectl get events -n demo --sort-by=.lastTimestamp
kubectl rollout status deployment/<name> -n demo
```

## 本次 lab 的邊界

熟悉 `kubectl` 不代表熟悉 Kubernetes production operations。這次只需要能看懂 resource state、events、logs、rollout 與基本 network resources，先建立可解釋的排障習慣。
