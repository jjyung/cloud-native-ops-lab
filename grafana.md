# Grafana

## 一句話定位

Grafana 是 observability visualization layer，將 Prometheus 等 datasource 的查詢結果組成 dashboard panels，讓 operator 能快速讀取系統狀態與趨勢。

## 官方文件

- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Build your first dashboard](https://grafana.com/docs/grafana/latest/fundamentals/getting-started/first-dashboards/)
- [Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

## 它解決什麼問題

PromQL 可以回答問題，但故障演練時需要一個可快速閱讀、可重複使用的畫面。Grafana 將 query、時間範圍、panel visualization 與 dashboard 組合起來，讓 request rate、error rate、latency 或 Pod restart 趨勢可以被展示。

## 在本專案的價值

本 repo 用 Helm 安裝的 Grafana 連到同一個 monitoring stack 裡的 Prometheus，完成一個可讀的 panel：

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

瀏覽 `http://localhost:3000` 後：

1. 確認 Prometheus datasource。
2. 建立一個 panel，使用 Kubernetes metrics 或 Istio request metrics。
3. 產生幾次 hello request，確認 panel 會變化。
4. 將 dashboard 截圖放入 `evidence/04-grafana-dashboard.png`。

## 它與 Prometheus 的關係

```text
Kubernetes / Istio metrics
          -> Prometheus scrape & query
          -> Grafana datasource
          -> dashboard panel
```

Grafana 是呈現與分析介面，不是 metrics collector，也不是 alert policy 的完整替代品。面試時應能說明 datasource、query、time range、panel 與 alert/runbook 的關係。

## 本次 lab 的邊界

不做 dashboard-as-code、Grafana provisioning、SSO、HA、通知渠道或完整 on-call runbook。這次成果是能展示一個 metrics panel，並說明它如何協助觀察故障演練。
