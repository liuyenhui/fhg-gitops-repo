# AGENTS.md

本目录定义 `ecommerce-cs-agent-dev` 的 Flux overlay。当前目标是基础运行环境，不部署业务 Agent API、Admin Web 或评测 worker 镜像。

## 环境入口

- Namespace：`ecommerce-cs-agent-dev`
- Flux Kustomization：`flux-system/ecommerce-cs-agent-dev`
- Overlay path：`apps/overlays/dev/ecommerce-cs-agent`
- Base path：`apps/base/ecommerce-cs-agent`
- 公网 kubeconfig：`~/.kube/bpg-debian12-master-public.yaml`

## 当前应存在的资源

正常状态下本 namespace 只应有基础设施资源：

- `StatefulSet/postgres`
- `StatefulSet/minio`
- `Service/postgres`
- `Service/minio`
- `CronJob/postgres-backup`
- `ConfigMap/ecommerce-postgres-initdb`
- `ConfigMap/ecommerce-cs-agent-ops-runbook`
- 相关 PVC 和 NetworkPolicy

当前不应由本 overlay 部署：

- `Deployment/ecommerce-cs-agent-api`
- `Service/ecommerce-cs-agent-api`
- API/Admin Ingress

如果这些业务资源重新出现，先确认是否有新 chart/overlay 接管；不要恢复旧的临时 FastAPI ConfigMap 业务镜像。

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

当前不创建业务 Ingress。等业务 service 存在后再创建：

- API：`https://api.ecommerce-cs-agent-dev.fcihome.com`
- Admin：`https://admin.ecommerce-cs-agent-dev.fcihome.com`

应使用：

- `ingressClassName: traefik`
- annotation：`cert-manager.io/cluster-issuer: letsencrypt-http01`

## 评测工具

当前 `TARGET_BASE_URL` 暂不可用，因为没有部署 Agent API。后续部署 API 后使用：

- `TARGET_BASE_URL=https://api.ecommerce-cs-agent-dev.fcihome.com`
- 鉴权建议：`Authorization: Bearer <token>`
- token 来源：`ecommerce-cs-agent-runtime/AGENT_API_TOKEN`

## 常用排查命令

```bash
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev get deploy,svc,ingress,sts,cronjob,pods,pvc --ignore-not-found
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n flux-system get kustomization ecommerce-cs-agent-dev -o wide
KUBECONFIG=$HOME/.kube/bpg-debian12-master-public.yaml kubectl -n ecommerce-cs-agent-dev get secret ecommerce-cs-agent-runtime -o jsonpath='{.data}' | jq 'keys'
```
