# AGENTS.md

本仓库是 FluxCD GitOps 集群状态仓库。排查问题时优先确认实际集群状态，再判断 Git 清单是否需要调整。

## 集群访问

- 当前可用公网 kubeconfig：`~/.kube/bpg-debian12-master-public.yaml`
- 使用方式：`KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl ...`
- context：`default`
- 集群：k3s，已验证节点 `master`、`agent-0` Ready
- 旧/内网 kubeconfig 可能指向 `https://192.168.1.202:6443` 并在 Codex 本机超时；遇到 kubectl 卡住时先确认是否误用了内网 kubeconfig

## FluxCD

- GitRepository：`flux-system/flux-system`
- 根 Kustomization：`flux-system/flux-system`
- dev 环境入口：`clusters/dev/kustomization.yaml`
- ecommerce-cs-agent dev 入口：`clusters/dev/kustomizations/ecommerce-cs-agent-dev.yaml`
- 手动触发：

```bash
ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n flux-system annotate gitrepository flux-system reconcile.fluxcd.io/requestedAt=$ts --overwrite
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n flux-system annotate kustomization ecommerce-cs-agent-dev reconcile.fluxcd.io/requestedAt=$ts --overwrite
```

## 已知基础设施状态

- Ingress controller：Traefik
- TLS：cert-manager `letsencrypt-http01`
- StorageClass：`local-path`
- FRP Deployment：`frp-system/bpg-frpc`
- FRP 已预留域名：
  - `api.ecommerce-cs-agent-dev.fcihome.com`
  - `admin.ecommerce-cs-agent-dev.fcihome.com`

## Secret 规则

- 不要把 Secret 明文或 base64 后的明文提交到 Git。
- 本仓库当前没有 SOPS、External Secrets 或 SealedSecrets 配置；Secret 需要通过 kubeconfig 或 CI/CD 安全渠道预置/轮换。
- 输出排障结果时只列 Secret 名称和 key，不能返回值。

## 验证命令

```bash
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n flux-system get kustomization,gitrepository
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl get nodes -o wide
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl get ingressclass,storageclass,clusterissuer
```
