# Cloud Native Ops Lab

一個可重現、可解釋、可展示的 local Kubernetes lab，串起 Istio、Argo CD、Prometheus 與 Grafana 的最小實作流程。

這個 repo 的目標是驗證基本操作與故障排查方向，不是建立 production-grade 平台。Lab 使用 Docker Desktop、kind Kubernetes cluster，以及 Helm 安裝 `kube-prometheus-stack`。

## Lab 架構

```text
Docker Desktop
└─ kind cluster
   ├─ Istio / Envoy sidecar
   ├─ demo namespace
   │  ├─ hello-v1
   │  └─ hello-v2
   ├─ Argo CD
   └─ monitoring namespace
      ├─ Prometheus
      └─ Grafana
```

## 目錄結構

```text
cloud-native-ops-lab/
├─ README.md
├─ app/
│  └─ hello/
│     ├─ namespace.yaml
│     ├─ deployment-v1.yaml
│     ├─ deployment-v2.yaml
│     ├─ service.yaml
│     ├─ destination-rule.yaml
│     ├─ virtual-service.yaml
│     └─ kustomization.yaml
├─ argocd/
│  └─ application.yaml
├─ monitoring/
│  ├─ values.yaml
│  └─ notes.md
└─ evidence/
   ├─ 01-istio-routing.txt
   ├─ 02-argocd-sync.txt
   ├─ 03-prometheus-query.txt
   └─ 04-grafana-dashboard.png
```

`evidence/` 內的檔案是操作紀錄佔位檔，完成 lab 後再填入實際輸出與截圖。

## Prerequisites

請先確認以下工具可執行：

```bash
docker version
kubectl version --client
kind version
helm version
git --version
```

Docker Desktop 建議至少配置 8 GB memory 與 4 CPUs。不同版本的 Istio、Argo CD 與 Helm chart 可能產生不同的 Pod 或 Service 名稱；操作時以實際指令輸出為準。

## Quick start

### 1. 建立 kind cluster

```bash
kind create cluster --name cloud-native-lab
kubectl config use-context kind-cloud-native-lab
kubectl get nodes
```

### 2. 安裝 Istio

請依 [Istio Getting Started](https://istio.io/latest/docs/setup/getting-started/) 安裝符合本機平台的 `istioctl`，再執行：

```bash
istioctl install --set profile=demo -y
kubectl get pods -n istio-system
```

### 3. 由 Argo CD 管理 hello app

先確認 `argocd/application.yaml` 的 `spec.source.repoURL` 已改成這個 repo 的 Git remote URL。接著安裝 Argo CD：

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

建立 Application：

```bash
kubectl apply -f argocd/application.yaml
kubectl get application -n argocd hello
```

用 Argo CD UI 或 CLI 手動 sync `hello` Application。此 repo 的 `app/hello` 會建立 `demo` namespace、兩個版本的 Deployment、Service，以及 Istio routing resources。

若只想先驗證 Kubernetes manifests，也可以直接執行：

```bash
kubectl apply -k app/hello
```

這個方式適合初次驗證；正式 demo 建議使用 Argo CD，保留 Git as source of truth 的流程。

### 4. 驗證 Istio routing

確認 application Pod 都有 application container 與 `istio-proxy`：

```bash
kubectl get pods -n demo
kubectl get pod -n demo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
```

從 cluster 內建立暫時的 curl client，測試預設路由與 header-based 路由：

```bash
kubectl run curl -n demo --rm -it --restart=Never \
  --image=curlimages/curl -- sh
```

在 client shell 中執行：

```sh
for i in 1 2 3; do curl -s http://hello.demo.svc.cluster.local:8080; echo; done
curl -s -H 'x-version: v2' http://hello.demo.svc.cluster.local:8080; echo
```

預期結果：不帶 header 的 request 回傳 `hello-v1`，帶有 `x-version: v2` 的 request 回傳 `hello-v2`。完成後將實際輸出記錄到 `evidence/01-istio-routing.txt`。

### 5. 安裝 Prometheus 與 Grafana

使用 Prometheus Community 的 `kube-prometheus-stack` chart：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f monitoring/values.yaml
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

開啟 Prometheus 與 Grafana：

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Service 名稱若因 chart 版本不同而改變，請先用 `kubectl get svc -n monitoring` 查詢實際名稱。

Grafana 預設帳號通常是 `admin`；密碼請依目前 chart 的 Secret 取值，不要把密碼提交到 repo：

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

至少完成一項 PromQL 查詢，例如：

```promql
sum by (namespace) (kube_pod_info)
```

再於 Grafana 建立一個 panel，顯示 Pod、request rate、error rate 或 resource usage。若 Istio metrics 尚未被自動 scrape，先保留 Kubernetes metrics，並把缺少 `ServiceMonitor` 或 scrape config 的狀況記錄在 `monitoring/notes.md`。

## GitOps demo 流程

建議用以下順序展示：

```text
Git commit
  -> Argo CD 偵測 OutOfSync
  -> 手動 Sync
  -> Kubernetes Deployment / Pod 更新
  -> Istio route v1/v2
  -> Prometheus query
  -> Grafana panel
```

可用下列方式做一次 drift 與恢復演練：

```bash
# 修改 app/hello/deployment-v1.yaml 的 replicas 或 image tag
git add app/hello/deployment-v1.yaml
git commit -m "test: change hello v1"
git push

# 在 Argo CD 觀察 OutOfSync，完成 sync 後再將變更 revert 並重新 sync
kubectl rollout status deployment/hello-v1 -n demo
```

## 故障排查順序

```bash
kubectl get pods -n demo -o wide
kubectl describe pod -n demo <pod-name>
kubectl logs -n demo <pod-name> -c <container-name>
kubectl get events -n demo --sort-by=.lastTimestamp
kubectl rollout status deployment/<deployment-name> -n demo
kubectl get virtualservice,destinationrule -n demo
istioctl analyze -n demo
```

建議至少完成兩個故障演練：刪除一個 hello Pod，觀察 Deployment self-healing；以及套用錯誤 image tag 或 manifest，觀察 Argo CD / Kubernetes 狀態後用 Git revert 或修正 commit 恢復。

## Lab scope

本 repo 聚焦於四個可展示成果：

| 元件 | 成果 |
|---|---|
| Istio | 以 `x-version: v2` 將流量導向 v2，其餘導向 v1 |
| Argo CD | Git commit 後偵測差異並 sync `app/hello` |
| Prometheus | 完成至少一項 Kubernetes 或 Istio metrics 的 PromQL |
| Grafana | 建立一個可讀的 metrics dashboard panel |

本次不包含 production HA、TLS 憑證、multi-cluster Argo CD、完整 canary pipeline、Terraform、OpenTelemetry trace、ELK migration 或真實 GCP/GKE deployment。

這是 local lab hands-on，不代表 production 經驗。適合的經驗描述是：過去正式使用較多 ELK 與 GCP Solution；本次透過 Docker、kind、Istio、Argo CD、Prometheus、Grafana 與 Helm，實際理解元件關係、基本操作與故障排查方向。

## Evidence checklist

- [ ] `evidence/01-istio-routing.txt`：預設 v1 與 `x-version: v2` 的實際輸出
- [ ] `evidence/02-argocd-sync.txt`：`Synced`、`Healthy`、`OutOfSync` 狀態
- [ ] `evidence/03-prometheus-query.txt`：PromQL、查詢時間與結果摘要
- [ ] `evidence/04-grafana-dashboard.png`：完成後放入 dashboard 截圖

## Cleanup

完成 lab 後可刪除整個 kind cluster：

```bash
kind delete cluster --name cloud-native-lab
```
