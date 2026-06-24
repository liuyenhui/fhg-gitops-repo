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
- ecommerce dev HTTP/HTTPS FRPS 公网入口：ai-agent `47.113.204.168`
- SSH tcpmux FRPS 公网入口：cmdz `ssh.fcihome.com` / `39.97.193.74`
- `bpg-frpc` 固定调度到 `agent-0`，避免 master 在国内网络下重新拉取 `snowdreamtech/frpc` 镜像卡住。
- ecommerce dev 域名的公网 HTTPS 由 ai-agent Traefik 终止 TLS，再转发到 frps HTTP vhost；`bpg-frpc` 只注册 `cs-agent-dev-http`，不要再启用 frpc `type=https`。
- `bpg-frpc` Deployment 内有两个 frpc 容器：`frpc` 连接 ai-agent，`frpc-ssh` 连接 cmdz。不要把整个 Deployment 的 `FRP_SERVER_ADDR` 改到 `ssh.fcihome.com`，否则 ecommerce HTTP 域名会从 ai-agent 断开。
- SSH 公网入口复用 cmdz `/root/wangdian/docker-compose.yml` 管理的 `frps-server`，端口 `7002` 用于 `tcpmux/httpconnect`。不要把 SSH tcpmux 加到 ai-agent frps。
- `frpc-auth` Secret 中 `token` 必须匹配 ai-agent FRPS，`cmdz-token` 必须匹配 cmdz `/root/wangdian/frps/frps.toml` 中的 FRPS token。核对时只比较 hash，不打印 token 明文。
- SSH 代理域名：
  - `m-master.ssh.fcihome.com` -> 既有 cmdz proxy，当前本机 `remote-m-master` 使用 `ssh.fcihome.com:7002`
  - `m-agent1.ssh.fcihome.com` -> 既有 cmdz proxy，当前本机 `remote-m-agent1` 使用 `ssh.fcihome.com:7002`
  - `mac-intel.ssh.fcihome.com` -> 本仓库 `bpg-frpc` 的 `frpc-ssh` 容器，转发到 `192.168.1.76:22`
- FRP 已预留域名：
  - `api.ecommerce-cs-agent-dev.fcihome.com`
  - `admin.ecommerce-cs-agent-dev.fcihome.com`
  - `system-admin.ecommerce-cs-agent-dev.fcihome.com`

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

Grafana / Prometheus 这类 monitoring HelmRepository 仍依赖外部 chart index，属于 Flux source-controller 出网问题；它们已在 `infrastructure/controllers/monitoring/repositories.yaml` 显式设置 `timeout: 5m`。若仍因中国网络失败，优先处理集群侧代理、OCI 镜像源或内部缓存，不要把 monitoring source 同步失败和 `ecommerce-cs-agent` 应用 release gate 混为同一类失败。

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
