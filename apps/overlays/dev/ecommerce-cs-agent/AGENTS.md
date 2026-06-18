# AGENTS.md

本目录定义 `ecommerce-cs-agent-dev` 的 Flux overlay。当前目标是基础运行环境加业务 Agent API/Admin Web 的 dev 发布目标状态。

## 环境入口

- Namespace：`ecommerce-cs-agent-dev`
- Flux Kustomization：`flux-system/ecommerce-cs-agent-dev`
- Overlay path：`apps/overlays/dev/ecommerce-cs-agent`
- Base path：`apps/base/ecommerce-cs-agent`
- 公网 kubeconfig：`~/.kube/bpg-debian12-master-public.yaml`

## 当前应存在的资源

正常状态下本 namespace 应有基础设施资源和业务 Helm release：

- `StatefulSet/postgres`
- `StatefulSet/minio`
- `Service/postgres`
- `Service/minio`
- `CronJob/postgres-backup`
- `ConfigMap/ecommerce-postgres-initdb`
- `ConfigMap/ecommerce-cs-agent-ops-runbook`
- `GitRepository/ecommerce-cs-agent-app`
- `HelmRelease/ecommerce-cs-agent`
- `Deployment/ecommerce-cs-agent-api`
- `Deployment/ecommerce-cs-agent-admin`
- `Service/ecommerce-cs-agent-api`
- `Service/ecommerce-cs-agent-admin`
- API/Admin Ingress
- 相关 PVC 和 NetworkPolicy

业务 API/Admin 由 `app-source.yaml` 和 `app-release.yaml` 接管，chart 来源是应用仓库 `liuyenhui/ecommerce-cs-agent` 的 `deploy/helm/ecommerce-cs-agent`。不要再用手工 `helm upgrade`、`kubectl set image` 或旧的临时 FastAPI ConfigMap 作为常规发布路径。

`HelmRelease.spec.chart.spec.reconcileStrategy` 必须保持 `Revision`。应用 chart 直接来自 GitRepository，同一 chart version 下也可能有模板、hook、安全上下文或探针变更；按 source revision reconcile 才能保证 GitOps 发布不仅更新 image tag，也会同步 chart 内容。

## PostgreSQL

- Service：`postgres.ecommerce-cs-agent-dev.svc.cluster.local:5432`
- namespace 内别名：`postgres:5432`
- 镜像：`pgvector/pgvector:pg16`
- 已验证版本：PostgreSQL `16.14`
- 已验证扩展：`pgcrypto`、`vector`
- 连接 Secret：`ecommerce-postgres-auth`
  - `database`
  - `username`
  - `password`
- 应用使用的 `DATABASE_URL` 放在 `ecommerce-cs-agent-runtime/DATABASE_URL`
- 集群内 SSL：`sslmode=disable`

验证：

```bash
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev exec postgres-0 -- sh -ec 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Atc "select current_setting('"'server_version'"'); select extname from pg_extension where extname in ('"'pgcrypto'"','"'vector'"') order by extname;"'
```

## MinIO / 对象存储

- Service：`http://minio.ecommerce-cs-agent-dev.svc.cluster.local:9000`
- namespace 内别名：`http://minio:9000`
- Bucket：`ecommerce-cs-agent-dev`
- Region：`us-east-1`
- Path style：true
- Secret：`ecommerce-minio-auth`
  - `root-user`
  - `root-password`
  - `bucket`
  - `access-key`
  - `secret-key`

注意：`ecommerce-minio-auth` 里的应用 access key 必须在 MinIO 内创建为用户并绑定 `readwrite`。如果备份或应用上传报 `Access Key Id ... does not exist`，用 root secret 在 `minio-0` 内创建用户和策略，不要改 Git 提交密钥。

## Runtime Secret

`ecommerce-cs-agent-runtime` 预留这些 key：

- `DATABASE_URL`
- `OBJECT_STORAGE_ENDPOINT`
- `OBJECT_STORAGE_BUCKET`
- `OBJECT_STORAGE_REGION`
- `OBJECT_STORAGE_ACCESS_KEY_ID`
- `OBJECT_STORAGE_SECRET_ACCESS_KEY`
- `LLM_API_KEY`
- `LLM_BASE_URL`
- `LLM_MODEL`
- `SESSION_SECRET`
- `JWT_SECRET`
- `AGENT_API_TOKEN`
- `ADMIN_INITIAL_EMAIL`
- `ADMIN_INITIAL_PASSWORD_HASH`

旧 key `agent-api-token`、`llm-provider-api-key` 可能仍存在，仅为兼容历史临时服务；新业务优先使用大写 key。

当前 `LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MODEL`、`ADMIN_INITIAL_EMAIL`、`ADMIN_INITIAL_PASSWORD_HASH`
允许为空，等待业务方通过安全渠道写入真实值。更新时保留已有 key，不要用空值覆盖非空值。

## 应用部署约定

业务应用由应用仓库 chart 创建 Deployment、Service、Ingress 和 migration hook；本 overlay 保存 dev values 和 Flux 目标状态。

镜像命名：

- Agent API：`registry.cn-beijing.aliyuncs.com/threepeople/ecommerce-cs-agent-api:<tag>`
- Admin Web：`registry.cn-beijing.aliyuncs.com/threepeople/ecommerce-cs-agent-admin:<tag>`
- GHCR 备份：`ghcr.io/liuyenhui/ecommerce-cs-agent-api:<tag>`、`ghcr.io/liuyenhui/ecommerce-cs-agent-admin:<tag>`

Pod pull secret：

- `imagePullSecrets`
  - `name: aliyun-registry-auth`
  - `name: ghcr-auth`

建议 chart values：

```yaml
api:
  image:
    repository: registry.cn-beijing.aliyuncs.com/threepeople/ecommerce-cs-agent-api
    tag: <git-sha-or-semver>
imagePullSecrets:
  - name: aliyun-registry-auth
  - name: ghcr-auth
envFromSecret: ecommerce-cs-agent-runtime
proxy:
  enabled: true
  httpProxy: http://192.168.1.198:1087
  httpsProxy: http://192.168.1.198:1087
  noProxy: localhost,127.0.0.1,.svc,.cluster.local,postgres.ecommerce-cs-agent-dev.svc.cluster.local,minio.ecommerce-cs-agent-dev.svc.cluster.local
ingress:
  enabled: true
  className: traefik
  clusterIssuer: letsencrypt-http01
```

Admin Web 同样使用阿里云 Registry 主镜像，并保留 GHCR pull secret 作为备用。

## 业务 Pod 出网代理

Agent API 后续会调用 LLM Provider。当前 Flux controller 已配置代理；业务 Pod 是否必须走代理取决于 LLM Provider 直连稳定性。为降低首次部署风险，建议 API Deployment 默认注入：

- `HTTP_PROXY=http://192.168.1.198:1087`
- `HTTPS_PROXY=http://192.168.1.198:1087`
- `NO_PROXY=localhost,127.0.0.1,.svc,.cluster.local,postgres.ecommerce-cs-agent-dev.svc.cluster.local,minio.ecommerce-cs-agent-dev.svc.cluster.local`

如果后续验证直连 LLM 稳定，可以在应用 chart values 中关闭代理。

## 节点镜像拉取代理 / 中国网络

不要把业务 Pod 代理和节点镜像拉取代理混为一类问题。`HTTP_PROXY`/`HTTPS_PROXY` 注入到 API Deployment 只保证应用调用 LLM Provider 时走代理；kubelet/containerd 拉取 `ghcr.io`、`docker.io`、`gcr.io` 镜像需要 k3s systemd 服务代理。

当前已在节点侧永久配置：

- master：`/etc/systemd/system/k3s.service.d/http-proxy.conf`
- agent：`/etc/systemd/system/k3s-agent.service.d/http-proxy.conf`
- `HTTP_PROXY=http://192.168.1.198:1087`
- `HTTPS_PROXY=http://192.168.1.198:1087`
- `NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,10.42.0.0/16,10.43.0.0/16,192.168.1.0/24,.svc,.svc.,.cluster.local,.cluster.local.,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster.local,192.168.1.200,192.168.1.198,postgres.ecommerce-cs-agent-dev.svc.cluster.local,minio.ecommerce-cs-agent-dev.svc.cluster.local`

同时已整理 `/etc/rancher/k3s/registries.yaml`，保留 `docker.io`、`ghcr.io`、`gcr.io` 官方 endpoint，以及 `192.168.1.198:5000` / `localhost:5000` 内网 registry mirror 入口。修改这些文件后必须 `systemctl daemon-reload` 并重启对应 k3s 服务。

验证节点代理时优先在节点 namespace 内测：

```bash
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl get nodes -o wide
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev get pods -o wide
```

已知现象：短请求访问 registry API 可通，但大镜像 layer 拉取在当前国内网络下仍可能超时。用户明确要求“不要本机 docker push”时，不要从本机向远端 registry 推镜像；正式发布应由 CI/GitHub Actions 推送到 GHCR，临时部署可将镜像导入目标节点 containerd。

## 备份 CronJob

- CronJob：`postgres-backup`
- Schedule：`17 18 * * *` UTC
- 备份路径：`s3://ecommerce-cs-agent-dev/postgres/`
- initContainer 使用 `minio/minio:latest` 复制 `/usr/bin/mc`，避免 `minio/mc` 镜像拉取被 Docker Hub 镜像代理 429 限制。

手动验证：

```bash
ns=ecommerce-cs-agent-dev
job=postgres-backup-manual-$(date +%H%M%S)
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n "$ns" create job --from=cronjob/postgres-backup "$job"
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n "$ns" wait --for=condition=complete "job/$job" --timeout=180s
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n "$ns" logs "job/$job" --tail=40
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n "$ns" delete job "$job"
```

已验证成功日志包含：

```text
uploaded minio/ecommerce-cs-agent-dev/postgres/cs-agent-cs_agent-*.dump
```

## Ingress / 域名

业务 Ingress 由应用 chart 创建：

- API：`https://api.ecommerce-cs-agent-dev.fcihome.com`
- Admin：`https://admin.ecommerce-cs-agent-dev.fcihome.com`

应使用：

- `ingressClassName: traefik`
- annotation：`cert-manager.io/cluster-issuer: letsencrypt-http01`

建议 service 名称：

- Agent API service：`ecommerce-cs-agent-api`
- Admin Web service：`ecommerce-cs-agent-admin`

如果 `Certificate/cs-agent-dev-tls` 一直是 `READY=False`，先查 DNS。2026-06-16 部署业务 chart 后，Ingress 规则经 Traefik service port-forward 验证可路由到 API/Admin `/health`，但公网域名 `api.ecommerce-cs-agent-dev.fcihome.com` 和 `admin.ecommerce-cs-agent-dev.fcihome.com` 在 Codex 本机解析为空，cert-manager HTTP-01 challenge 报 `no such host`。这两个域名应解析到 ai-agent/frps 公网入口 `47.113.204.168`；解析生效后等待 cert-manager 重新 self-check 和签发证书。

## 评测工具

当前 `TARGET_BASE_URL`：

- `TARGET_BASE_URL=https://api.ecommerce-cs-agent-dev.fcihome.com`
- 鉴权建议：`Authorization: Bearer <token>`
- token 来源：`ecommerce-cs-agent-runtime/AGENT_API_TOKEN`

## 常用排查命令

```bash
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev get deploy,svc,ingress,sts,cronjob,pods,pvc --ignore-not-found
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n flux-system get kustomization ecommerce-cs-agent-dev -o wide
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev get secret ecommerce-cs-agent-runtime -o jsonpath='{.data}' | jq 'keys'
```
