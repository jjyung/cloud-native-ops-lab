# Kustomize

## 一句話定位

Kustomize 是用 `kustomization.yaml` 組合與客製化 Kubernetes manifests 的 declarative configuration tool；`kubectl` 已內建 `kustomize` 與 `apply -k` 支援。

## 官方文件

- [Declarative Management with Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [kubectl kustomize reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/)

## 它解決什麼問題

當一個 app 有多個 YAML resource 時，Kustomize 用一個入口描述 resource 組合，並可在不修改 upstream YAML 的情況下加入 namespace、labels、名前前綴或環境差異。

## 在本專案的價值

`app/hello/kustomization.yaml` 是 hello app 的 manifest entrypoint：

```yaml
resources:
  - namespace.yaml
  - deployment-v1.yaml
  - deployment-v2.yaml
  - service.yaml
  - destination-rule.yaml
  - virtual-service.yaml
```

它帶來兩種使用方式：

```bash
# 預覽 render 結果
kubectl kustomize app/hello

# 直接套用，適合初次驗證
kubectl apply -k app/hello
```

在主要 demo 中，Argo CD 會把 `app/hello` 當成 source path，並由 Kustomize render 後套用到 cluster。因此 Kustomize 負責 manifest composition，Argo CD 負責 GitOps delivery；兩者不是互相取代。

## 本次 lab 的邊界

這裡只展示單一 app 的 resource composition，不建立 base/overlay、多環境 patch、secret generator 或複雜 template。需要大量參數化與 chart lifecycle 時，才交給 Helm。
