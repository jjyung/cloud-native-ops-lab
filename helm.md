# Helm

## 一句話定位

Helm 是 Kubernetes package manager；Helm chart 將一組相關 Kubernetes manifests、預設值與版本包成可安裝、升級與 rollback 的 release。

## 官方文件

- [Helm Quickstart](https://helm.sh/docs/intro/quickstart/)
- [Introduction to Helm](https://helm.sh/docs/intro/)
- [Helm Charts](https://helm.sh/docs/topics/charts/)

## 它解決什麼問題

Prometheus、Grafana、Alertmanager、node exporter 與 Kubernetes exporters 彼此有許多設定與相依關係。逐一撰寫 manifests 會增加安裝與版本管理成本；Helm chart 提供一個有版本的 package 與 values 入口。

## 在本專案的價值

本 repo 用 Prometheus Community 的 `kube-prometheus-stack` chart 安裝 monitoring stack：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f monitoring/values.yaml
```

其中：

- `monitoring/values.yaml`：保存本次 local lab 的非敏感設定。
- `helm upgrade --install`：同一個指令可處理首次安裝與後續升級。
- release name `monitoring`：讓 Helm 管理這組 resources 的生命週期。
- 這個 chart 不只安裝 Prometheus，也包含 Prometheus Operator、Alertmanager、Grafana、Node Exporter 與 kube-state-metrics；完整分工見 [`prometheus.md`](./prometheus.md) 的 `kube-prometheus-stack` 章節。

## 它與 Argo CD 的關係

本 lab 直接由 Helm CLI 安裝 platform monitoring stack；Argo CD 只管理 repo 內的 `app/hello`。這是刻意縮小範圍，避免把 Istio、Argo CD 與 monitoring 全部再包成 bootstrap repo。

## 本次 lab 的邊界

Helm 是本次 lab 才接觸的工具，不代表既有 production Helm 經驗。這次不建立自己的 chart，也不討論 chart repository governance、signing、dependency lock 或 production values 分層。
