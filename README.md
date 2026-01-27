# fhg-gitops-repo

这是 `flux-helm-github` 项目的 **GitOps 配置仓库**。它通过 FluxCD 控制集群的理想状态。

## 核心职责

1.  **定义外部源**: 声明 Helm 仓库 (OCI) 和 Git 仓库地址。
2.  **管理应用发布**: 定义 `HelmRelease` 资源，指定 Chart 版本及引用的配置。
3.  **多环境管理**: 通过基准配置 (`base`) 与环境差异配置 (`overlays`) 实现多集群管理。

## 目录结构

```text
fhg-gitops-repo/
├── apps/
│   ├── base/           # 共享的基础定义 (Frontend, Backend)
│   │   ├── frontend/   # 包含 repo.yaml (源) 和 release.yaml (发布)
│   │   └── backend/    # 包含 repo.yaml (源) 和 release.yaml (发布)
│   └── overlays/       # 针对特定环境 (如 dev) 的配置补丁
└── clusters/
    └── dev/            # dev 集群的入口，通过 Kustomize 组合应用
```

## 自动化验证流程

1.  **代码提交**: 开发者修改 `apps/` 下的配置并推送到此仓库。
2.  **Flux 轮询**: 集群内的 `source-controller` 检测到 Git 变更。
3.  **状态对齐**: `kustomize-controller` 和 `helm-controller` 自动应用变更，保持集群 Pod 与此仓库定义一致。

## 注意事项

- **Secret 管理**: 这里不存储敏感明文，所有 Secret (如 `ghcr-auth`) 通常手动创建或通过外部 Secret 管理器加载。
- **镜像版本**: 具体的镜像 Tag 通常记录在 `values.yaml` 中，由 `fhg-charts` 自动化流程自动更新。
