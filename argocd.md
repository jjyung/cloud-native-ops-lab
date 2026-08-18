# Argo CD

## 一句話定位

Argo CD 是 Kubernetes 的 declarative GitOps continuous delivery tool，將 Git repository 的 desired state 與 cluster live state 做比對並同步。

## 官方文件

- [Argo CD official site](https://argo-cd.readthedocs.io/en/stable/)
- [Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Declarative Setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)

## 它解決什麼問題

傳統部署常由 operator 在 terminal 直接執行 `kubectl apply`，但 cluster 最後為什麼是這個狀態、誰改的、如何恢復，可能不容易追蹤。Argo CD 將 Git commit 當成可審查的 desired state，持續比較並回報差異。

## 在本專案的價值

`argocd/application.yaml` 定義：

- source repo：這個 Git repository。
- source path：`app/hello`。
- target：目前 kind cluster 的 `demo` namespace。
- sync policy：手動 sync，方便觀察 `OutOfSync` 與 `Synced` 轉換。

實際流程：

```text
修改 app/hello manifest
  -> git commit / push
  -> Argo CD 偵測 OutOfSync
  -> 手動 Sync
  -> Kubernetes resource 更新
  -> Application 變成 Synced / Healthy
```

## 這裡要觀察的狀態

| 狀態 | 意義 |
|---|---|
| `Synced` | live state 與 Git desired state 一致 |
| `OutOfSync` | cluster 與 Git 有差異，需要 sync 或判斷是否為 drift |
| `Healthy` | Argo CD 判斷 resource health 正常 |

## 它與 Kustomize、kubectl 的關係

- Kustomize：render `app/hello` 的 manifests。
- Argo CD：從 Git 取得 source、比較 state、協調 sync。
- kubectl：安裝 Argo CD、建立 Application，以及直接觀察 cluster。

三者分工不同；`kubectl apply -k app/hello` 是 fallback 或初次驗證，不是 GitOps demo 的主要路徑。

## 本次 lab 的邊界

不做 automated sync policy、multi-cluster、SSO/RBAC、ApplicationSet、progressive delivery 或 rollback pipeline。這次只需要完成一個 Application 的 sync、drift 觀察與 Git revert 恢復。
