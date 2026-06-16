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
- FRPS 公网入口：ai-agent `47.113.204.168`
- FRP 已预留域名：
  - `api.ecommerce-cs-agent-dev.fcihome.com`
  - `admin.ecommerce-cs-agent-dev.fcihome.com`

## 中国网络 / 节点镜像拉取代理

业务 Pod 内的 `HTTP_PROXY`/`HTTPS_PROXY` 只影响应用运行时出网，不影响 kubelet/containerd 拉取镜像。排查 `ImagePullBackOff`、Kaniko/BuildKit 镜像长时间 `Pulling` 或 `crictl pull` 超时时，先检查 k3s 节点侧代理：

- master：`/etc/systemd/system/k3s.service.d/http-proxy.conf`
- agent：`/etc/systemd/system/k3s-agent.service.d/http-proxy.conf`
- 当前代理：`HTTP_PROXY=http://192.168.1.198:1087`、`HTTPS_PROXY=http://192.168.1.198:1087`
- `NO_PROXY` 至少包含：`localhost,127.0.0.1,10.0.0.0/8,10.42.0.0/16,10.43.0.0/16,192.168.1.0/24,.svc,.svc.,.cluster.local,.cluster.local.,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster.local,192.168.1.200,192.168.1.198`
- registry 配置：`/etc/rancher/k3s/registries.yaml`，当前保留 `docker.io`、`ghcr.io`、`gcr.io` 官方 endpoint，并保留 `192.168.1.198:5000` / `localhost:5000` 内网 registry mirror 入口。

修改后需要在对应节点执行：

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s        # master
sudo systemctl restart k3s-agent  # agent
```

验证：

```bash
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl get nodes -o wide
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev get pods -o wide
```

注意：已验证节点通过代理访问 registry API 可通，但大镜像 layer 拉取在当前国内网络下仍可能超时。用户明确要求“不要本机 docker push”时，不要从本机向远端 registry 推镜像；优先使用 CI/GitHub Actions 发布到 GHCR，或临时把已构建镜像导入目标节点 containerd 作为应急部署手段。

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
