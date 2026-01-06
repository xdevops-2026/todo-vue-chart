### todo-vue-chart

这个仓库用于存放并发布 Helm Charts（偏 OpenShift 场景）。当前包含一个示例 Chart：`todo-vue`，并通过 GitHub Actions 将 Chart 以 OCI 形式推送到 GitHub Container Registry（GHCR）。

### 仓库结构
- **`charts/`**：Helm Chart 源码目录（每个子目录一个 Chart）
- **`.github/workflows/ci-helm-oci.yml`**：Lint/打包并推送到 GHCR 的工作流

### 前置条件
- **Helm**：建议 Helm 3（工作流里使用 `v3.14.4`）
- **OpenShift**：如果你希望使用 `Route`（Chart 内置 `route.openshift.io/v1`）
- **权限**：发布到 GHCR 需要仓库具备 `packages: write` 权限（工作流已配置）

### Charts
- **`todo-vue`**：
  - **包含资源**：`Deployment`、`Service`、可选 `Ingress`、可选 OpenShift `Route`、可选 `ServiceAccount`
  - **默认端口**：`8080`

### 本地安装/升级（从源码）
- **安装**：
  - `helm upgrade --install todo-vue charts/todo-vue -n <namespace> --create-namespace`
- **Kubernetes 对外暴露（可选）**：
  - **Service**：`--set service.type=NodePort` 或 `--set service.type=LoadBalancer`
  - **Ingress**：`--set ingress.enabled=true`（并配置 `ingress.hosts`/`ingress.tls`）
- **OpenShift Route（可选）**：
  - 默认启用；非 OpenShift 集群请关闭：`--set route.enabled=false`

### 常用配置（values）
位于 `charts/todo-vue/values.yaml`：
- **`image.repository` / `image.tag`**：应用镜像（默认是可在 OpenShift 运行的示例镜像）
- **`service.type`**：`ClusterIP`/`NodePort`/`LoadBalancer`
- **`service.port`**：服务端口（默认 `8080`）
- **`service.nodePort`**：当 `service.type=NodePort` 时可指定端口
- **`ingress.enabled`**：是否创建 Kubernetes `Ingress`（默认 `false`）
- **`ingress.className`** / **`ingress.hosts`** / **`ingress.tls`**：Ingress 相关配置
- **`route.enabled`**：是否创建 OpenShift `Route`（默认 `true`）
- **`route.host`**：指定 Route Host（默认空，由 OpenShift 自动分配）
- **`route.tls.enabled`**：是否启用 TLS（默认 `true`）
- **`route.tls.termination`**：TLS 终止方式（默认 `edge`）

### 发布到 GHCR（OCI）
工作流：`Publish Helm Charts (OCI to GHCR)`（见 `.github/workflows/ci-helm-oci.yml`）

- **触发方式**：
  - **push 到 `main`**：仅发布有变更的 Chart（基于 `charts/**` diff 计算）
  - **打 tag**：匹配 `v*` 或 `*-v*` 时发布全部 Chart
  - **手动触发**：`workflow_dispatch` 可指定单个 Chart（例如 `todo-vue`）或 `all`

- **发布产物路径（全部小写）**：
  - `oci://ghcr.io/<owner>/<repo>/<chart>`
  - 例如：`oci://ghcr.io/xdevops-2026/todo-vue-chart/todo-vue`

说明：在 OCI 模式下，`helm push <chart.tgz> oci://ghcr.io/<owner>/<repo>` 会自动在 `<repo>` 下创建 `<chart>` 这一层，所以 workflow 里不会把 `<chart>` 拼进 push 目标。

### 从 GHCR 安装（OCI）
- **登录 GHCR**：
  - `helm registry login ghcr.io`
- **安装**：
  - `helm install todo-vue oci://ghcr.io/xdevops-2026/todo-vue-chart/todo-vue --version 0.1.0 -n <namespace> --create-namespace`

### 版本管理
- **`charts/<chart>/Chart.yaml`**：
  - **`version`**：Chart 版本（打包/发布使用它）
  - **`appVersion`**：应用版本（用于展示）
