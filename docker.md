# Docker / Docker Desktop

## 一句話定位

Docker 是本機的 container runtime 與映像檔工具；Docker Desktop 則是在 macOS、Windows、Linux 上提供 Docker daemon、CLI 與相關工具的整合環境。

## 官方文件

- [What is Docker?](https://docs.docker.com/get-started/docker-overview/)
- [Install Docker Desktop](https://docs.docker.com/get-started/get-docker/)

## 它解決什麼問題

Docker 把應用程式與 runtime 依賴包成 image，再以 container 執行。這讓本機可以用一致的方式啟動服務，也讓 `kind` 能把 Kubernetes node 執行在 Docker containers 裡。

Docker 本身不是 Kubernetes。它不提供 Kubernetes API、Deployment、Service、scheduler 或 GitOps；它只是這個 lab 最底層的執行環境。

## 在本專案的價值

```text
Docker Desktop
└─ kind node containers
   └─ Kubernetes cluster
      ├─ Istio / Envoy
      ├─ Argo CD
      └─ Prometheus / Grafana
```

本 repo 不需要自己撰寫 Dockerfile，`hello-v1` 和 `hello-v2` 使用公開的 `hashicorp/http-echo` image。Docker 的主要價值是承載整個 local Kubernetes lab，並提供映像檔拉取、container inspection 與資源限制。

## 本 repo 的相關操作

```bash
docker version
docker info
docker ps
```

建立 kind cluster 後，也可以用下列指令看到 kind 的 Kubernetes node container：

```bash
docker ps --filter name=cloud-native-lab
```

## 本次 lab 的邊界

Docker Desktop 的本機環境方便重現，但不等於 production 的 container runtime、registry、networking 或 storage 設計。這個 lab 用它來學習 Kubernetes 元件關係，不用它模擬完整 production infrastructure。
