# Git

## 一句話定位

Git 是本專案的 version control system，也是 Argo CD 的 desired-state source；Git commit 是 deployment workflow 的可追蹤變更單位。

## 官方文件

- [Pro Git book](https://git-scm.com/book/en/v2)
- [Git Basics](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository)
- [Git branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)

## 它解決什麼問題

Git 保存 manifests 的版本、作者、commit history 與 revert 能力。對 GitOps 而言，Git 不只是存檔位置，而是「希望 cluster 長什麼樣子」的可審查來源。

## 在本專案的價值

這個 repo 的 `app/hello` 是 Argo CD 的 source path。一次完整 demo 應該包含：

```text
edit YAML
  -> git diff
  -> git commit
  -> git push
  -> Argo CD OutOfSync
  -> Sync
```

故障演練時可以將錯誤 image tag commit，再透過修正 commit 或 `git revert` 恢復。這比只在 cluster 上手動改 resource 更容易解釋、稽核與重現。

## 常用操作

```bash
git status
git diff
git log --oneline
git add app/hello/deployment-v1.yaml
git commit -m "test: change hello v1"
git push
```

## 本次 lab 的邊界

不建立 CI pipeline、branch protection、signed commit、secret management 或正式 release strategy。這次先把 Git commit 與 Argo CD sync 的因果關係做清楚。
