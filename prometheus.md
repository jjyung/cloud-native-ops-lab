# Prometheus

## 一句話定位

Prometheus 是 metrics monitoring and alerting toolkit，以 time series、labels 與 PromQL 儲存及查詢數值型監控資料。

## 官方文件

- [Prometheus Overview](https://prometheus.io/docs/introduction/overview/)
- [Prometheus data model](https://prometheus.io/docs/concepts/data_model/)
- [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)

## 它解決什麼問題

Logs 適合描述單次事件；metrics 適合觀察一段時間的數值趨勢，例如 request rate、error rate、latency、Pod restart 與 resource usage。Prometheus 以 pull/scrape model 從 targets 收集 metrics，並以 labels 形成可查詢的 dimensions。

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
