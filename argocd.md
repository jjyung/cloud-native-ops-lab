# Argo CD

## 一句話定位

Argo CD 是 Kubernetes 的 declarative GitOps continuous delivery tool，將 Git repository 的 desired state 與 cluster live state 做比對並同步。

## 官方文件

- [Argo CD official site](https://argo-cd.readthedocs.io/en/stable/)
- [Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Declarative Setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)

## 它解決什麼問題

傳統部署常由 operator 在 terminal 直接執行 `kubectl apply`，但 cluster 最後為什麼是這個狀態、誰改的、如何恢復，可能不容易追蹤。Argo CD 將 Git commit 當成可審查的 desired state，持續比較並回報差異。

## 它和 Jenkins、GitLab 的角色差異

Argo CD、Jenkins 和 GitLab 不完全是相同角色：

| 工具 | 主要角色 | 主要工作 |
|---|---|---|
| Jenkins | CI/CD 平台 | build、test，以及依 pipeline 執行部署 |
| GitLab CI/CD | CI/CD 平台 | 整合 Git repository、Merge Request、build、test、deploy |
| Argo CD | Kubernetes continuous delivery / GitOps 工具 | 持續讓 Kubernetes 的 live state 符合 Git 裡的 desired state |

Argo CD 通常不是 Jenkins 或 GitLab CI 的直接替代品，而是補足「建置完成後，如何持續且可稽核地管理 Kubernetes 部署」這個缺口。

常見的分工如下：

```text
GitLab / GitHub
    -> GitLab CI 或 Jenkins：build、test、建立 container image
    -> 更新 Git 裡的 Kubernetes manifest
    -> Argo CD：從 Git 讀取設定並同步到 Kubernetes
```

一句話記憶：

> Jenkins / GitLab CI 負責「把東西做出來」；Argo CD 負責「確保 Kubernetes 一直是 Git 定義的樣子」。

## Argo CD 補足的缺口

- **避免環境漂移（configuration drift）**：有人直接修改 cluster 時，Argo CD 能偵測 live state 與 Git 的差異。
- **Git 作為唯一真相來源**：部署版本與設定有 commit 紀錄，可以 review、稽核與追蹤。
- **持續同步而非只部署一次**：Argo CD 會持續 reconcile，確保實際狀態回到 desired state。
- **降低 CI 對 production cluster 的權限**：CI 可以只負責更新 Git，不必直接操作 production cluster。
- **容易回滾**：回退 Git commit，再由 Argo CD 同步回先前版本。
- **支援多環境、多叢集管理**：可以讓不同 Application 對應 dev、staging、production 或不同 Kubernetes cluster。

## 監控與資安的邊界

Argo CD 確實有一些監控與資安治理的意味，但它監控的核心是**部署狀態與設定一致性**，不是完整的監控或資安平台。

它可以觀察或協助：

- Git desired state 與 Kubernetes live state 是否一致。
- Application 是否 `Synced`、`OutOfSync`、`Healthy`。
- 部署變更的 Git commit 與操作紀錄。
- 透過 RBAC 限制誰可以管理哪些 Application 或環境。
- 搭配 policy、image signing、漏洞掃描等工具，阻擋不合規的部署。

它本身不是：

- Prometheus / Grafana 這類效能與系統監控工具。
- SIEM、SOC 或 runtime security 工具。
- SAST、DAST 或 container vulnerability scanner。

因此，Argo CD 的資安價值主要來自 **Git 稽核、權限治理、減少直接修改 production，以及維持部署設定一致性**。

## 容易混淆的 Argo 元件

本文件討論的是 **Argo CD**。Argo 是一組 Kubernetes 原生工具，不是只有 Argo CD：

- **Argo CD**：GitOps continuous delivery，負責 Kubernetes 部署與狀態同步。
- **Argo Workflows**：Kubernetes 工作流引擎，可執行多步驟 pipeline 或批次任務。
- **Argo Events**：事件觸發器，監聽 webhook、Git 或訊息事件後觸發工作流。

所以「Argo 可以取代 Jenkins 嗎？」沒有單一答案：Argo CD 主要取代的是部署管理部分；Argo Workflows 才可能和 Jenkins pipeline 的部分工作重疊。

## Architecture

![alt text](./assets/argo.png)

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

## 常用指令

### 本 lab 的操作方式

本機目前沒有安裝 `argocd` CLI，因此本 lab 使用：

```text
kubectl -> 安裝 Argo CD、建立 Application、查看 Kubernetes resources
Argo CD UI -> 登入、查看 Application、執行 Refresh / Sync
```

`argocd` CLI 是可選的 client，不是 Argo CD server 運作的必要元件。

### 透過 UI 登入 Argo CD

先將 Argo CD server 暴露到本機：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

瀏覽器開啟：

```text
https://localhost:8080
```

使用者名稱通常是 `admin`。Local lab 的初始 password 可透過 Kubernetes Secret 查詢：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### 用 kubectl 查看 Application

```bash
# 列出所有 Application
kubectl get application -n argocd

# 查看 hello 的 spec、sync status 與 health status
kubectl get application hello -n argocd -o yaml

# 查看 Application events 與 conditions
kubectl describe application hello -n argocd
```

### Refresh 與 Sync

不使用 CLI 時，推薦透過 Argo CD UI 執行：

```text
Applications -> hello -> Refresh
Applications -> hello -> Sync
```

也可以用 annotation 觸發 Application refresh：

```bash
# 重新檢查 Git revision 與 cluster 狀態
kubectl annotate application hello -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

### Diff、Sync 與等待 rollout

在 UI 中可以查看：

```text
hello -> APP DIFF：查看 Git desired state 與 live state 差異
hello -> SYNC：將 Git desired state 同步到 Kubernetes
```

Sync 後用 `kubectl` 確認 workload：

```bash
kubectl rollout status deployment/hello-v1 -n demo
kubectl rollout status deployment/hello-v2 -n demo
kubectl get deployment,pod -n demo -o wide
```

`kubectl rollout status` 觀察的是 Kubernetes rollout，不是 Argo CD CLI；Argo CD 會讀取結果並更新 Application health。

### 可選：安裝 Argo CD CLI

如果日後需要 terminal workflow，可以另外安裝 `argocd` CLI：

```bash
brew install argocd
argocd login localhost:8080 --insecure
```

安裝後可使用：

```bash
argocd app list
argocd app get hello
argocd app refresh hello
argocd app diff hello
argocd app sync hello
argocd app wait hello --sync --health
```

CLI 是操作 Argo CD 的便利介面；沒有安裝 CLI 不會影響 Argo CD controller 在 cluster 中運作。

### 使用 CLI 時的手動 sync 流程

若日後安裝 CLI，這個 lab 的手動 sync 流程是：

```text
修改 app/hello
  -> git commit / push
  -> argocd app refresh hello
  -> argocd app diff hello
  -> argocd app sync hello
  -> argocd app wait hello --sync --health
```

目前沒有 CLI 時，將 refresh / diff / sync 步驟改由 Argo CD UI 執行。

### 查看受 Argo 管理的 resources

```bash
kubectl get all -n demo
kubectl get virtualservice,destinationrule -n demo
```

### History 與回滾

Argo CD UI 的 `History and Rollback` 頁面可查看同步歷史。

GitOps lab 優先使用 Git revert 回滾：

```bash
git revert <bad-commit>
git push
```

Push 後，在 Argo CD UI 執行 `Refresh`，確認 diff 後執行 `Sync`。如果日後安裝 CLI，也可以使用 `argocd app refresh hello` 與 `argocd app sync hello`。

這樣 Git、Argo CD 與 cluster 最終狀態保持一致。直接用 `kubectl edit` 或只在 cluster 中回滾，會造成 Git 與 live state 不一致。

### 設定與事件排障

```bash
# 檢查 Application、Pod 與最近事件
kubectl get application hello -n argocd -o yaml
kubectl get pods -n argocd
kubectl get events -n argocd --sort-by=.lastTimestamp
kubectl get events -n demo --sort-by=.lastTimestamp

# 分析 Istio 或其他 Kubernetes 設定問題
kubectl get virtualservice,destinationrule -n demo
```

如果 Application 是 `OutOfSync`，先在 UI 查看 `APP DIFF`；如果日後安裝 CLI，也可以執行 `argocd app diff hello`。如果是 `Degraded` 或 rollout 未完成，再檢查 `kubectl describe pod`、Deployment events 與 container logs。

## 本次 lab 的邊界

不做 automated sync policy、multi-cluster、SSO/RBAC、ApplicationSet、progressive delivery 或 rollback pipeline。這次只需要完成一個 Application 的 sync、drift 觀察與 Git revert 恢復。
