# Prometheus

## 一句話定位

Prometheus 是 metrics monitoring and alerting toolkit，以 time series、labels 與 PromQL 儲存及查詢數值型監控資料。

## 官方文件

- [Prometheus Overview](https://prometheus.io/docs/introduction/overview/)
- [Prometheus data model](https://prometheus.io/docs/concepts/data_model/)
- [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)

## 它解決什麼問題

Logs 適合描述單次事件；metrics 適合觀察一段時間的數值趨勢，例如 request rate、error rate、latency、Pod restart 與 resource usage。Prometheus 以 pull/scrape model 從 targets 收集 metrics，並以 labels 形成可查詢的 dimensions。

## `kube-prometheus-stack` 是什麼

### Why：為什麼需要這個 stack

只安裝 Prometheus 本身，還需要另外處理 Kubernetes resource discovery、exporter、告警管理、dashboard 與相關設定。`prometheus-community/kube-prometheus-stack` 將這些常見元件用一個 Helm chart 組合起來，降低安裝與版本相依的複雜度。

它不是一個單獨的 Prometheus process，而是一組互相配合的 Kubernetes observability 元件。

### What：stack 裡面有什麼

本 lab 使用的 `kube-prometheus-stack` 主要包含：

| 元件 | 職責 | 本 lab 對應資源 |
|---|---|---|
| Prometheus Operator | 透過 Kubernetes CRD 管理 Prometheus 與 Alertmanager | `monitoring-kube-prometheus-operator` |
| Prometheus | scrape、儲存、查詢 metrics，並評估 alert rules | `monitoring-kube-prometheus-prometheus` |
| Alertmanager | 接收 Prometheus alerts，分組、去重、靜音與發送通知 | `monitoring-kube-prometheus-alertmanager` |
| Grafana | 連接 Prometheus，提供 dashboard 與視覺化分析 | `monitoring-grafana` |
| Node Exporter | 暴露 node 作業系統與硬體相關 metrics | `monitoring-prometheus-node-exporter` |
| kube-state-metrics | 將 Kubernetes object 狀態轉成 metrics | `monitoring-kube-state-metrics` |
| CRDs | 提供 `ServiceMonitor`、`PodMonitor`、`PrometheusRule` 等宣告式設定 | `kubectl get crd` 可查看 |

實際資料與告警流程：

```text
Kubernetes / application / exporter
        -> Prometheus scrape
        -> PromQL 查詢或 PrometheusRule 評估
        -> Alertmanager 管理並發送告警
        -> Grafana dashboard 呈現 metrics
```

要特別區分：Prometheus 負責「發現問題」，Alertmanager 負責「管理與通知問題」，Grafana 負責「呈現與分析問題」。

### How：如何安裝與使用

本 lab 透過 Helm 安裝：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f monitoring/values.yaml
```

確認 stack 內有哪些元件：

```bash
kubectl get pods,svc -n monitoring
kubectl get prometheus,alertmanager -n monitoring
kubectl get servicemonitor,podmonitor,prometheusrule -n monitoring
```

本 lab 常用的存取方式：

```bash
# Prometheus UI
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus 9090:9090

# Grafana UI
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Alertmanager UI
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

使用時的基本順序是：先確認 exporter 或 Kubernetes target 有 metrics，再用 PromQL 查詢；需要告警時建立 `PrometheusRule`，由 Prometheus 評估後交給 Alertmanager 通知。

### When：什麼時候使用

適合使用 `kube-prometheus-stack` 的情境：

- 需要在 Kubernetes 中快速建立一套完整的 metrics、dashboard 與 alerting 基礎設施。
- 希望用標準的 `ServiceMonitor`、`PodMonitor` 與 `PrometheusRule` 管理監控設定。
- 需要同時觀察 node、Pod、Kubernetes object 與 application metrics。
- Local lab、測試環境或初期 production observability，希望先使用社群維護的整合方案。

不一定需要它的情境：

- 已經使用雲端託管 Prometheus、Grafana 與告警服務。
- 只需要一個非常小的 Prometheus instance，不需要 Kubernetes operator 與 exporters。
- 組織已有完整且標準化的 observability platform，不希望引入另一套元件生命週期。

因此，本 lab 使用這個 stack 是為了用一個 chart 同時學習 Prometheus、Grafana、Alertmanager、exporters 與 Kubernetes monitoring integration；不是因為這些元件都是 Prometheus 內建功能。

## 在本專案的價值

本 repo 使用 `kube-prometheus-stack` 安裝 Prometheus，至少完成一個 Kubernetes metric 的 PromQL：

```promql
sum by (namespace) (kube_pod_info)
```

若 Istio metrics 已成功 scrape，可再查：

```promql
sum(rate(istio_requests_total[5m]))
```

要記錄的不是只有 query 結果，也要理解：

1. metric 從哪個 target 來。
2. label 代表什麼，是否造成 high cardinality。
3. scrape target 為什麼是 UP 或 DOWN。
4. query 如何支援故障判斷，而不是只做漂亮圖表。

## 它與 Grafana 的關係

Prometheus 負責收集、儲存與查詢 metrics；Grafana 連到 Prometheus datasource，負責 dashboard 與視覺化。Grafana 不會取代 Prometheus 的 time-series storage。

## 本次 lab 的邊界

不做 long-term remote storage、HA Prometheus、Alertmanager routing、recording rules 或完整 SLO/SLI 設計。若 Istio metrics 尚未 scrape，先完成 Kubernetes metrics，並將 scrape configuration 缺口記錄於 `monitoring/notes.md`。
