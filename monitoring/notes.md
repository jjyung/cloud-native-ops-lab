# Monitoring notes

## Installation

This lab uses the Prometheus Community `kube-prometheus-stack` Helm chart. Record the chart version, installation command, and the actual Service names here after installation.

## Prometheus

Record at least one successful PromQL query in `evidence/03-prometheus-query.txt`. Suggested starting query:

```promql
sum by (namespace) (kube_pod_info)
```

If Istio metrics are available, also try:

```promql
sum(rate(istio_requests_total[5m]))
```

If Istio metrics are not available, record which scrape target or `ServiceMonitor` configuration is missing. Keep the Kubernetes metrics dashboard as the completed fallback.

## Grafana

Record the dashboard URL or panel title and place the final screenshot in `evidence/04-grafana-dashboard.png`. Never commit Grafana passwords or other credentials.
