### todo-vue-chart

这个仓库用于存放并发布 Helm Charts（包含 Kubernetes 与 OpenShift 兼容的示例）。当前包含多个 Chart，并通过 GitHub Actions 将 Chart 以 OCI 形式推送到 GitHub Container Registry（GHCR）。

### 仓库结构
- **`charts/`**：Helm Chart 源码目录（每个子目录一个 Chart）
- **`.github/workflows/ci-helm-oci.yml`**：检测变更、渲染校验、lint、打包并推送到 GHCR 的工作流

### 前置条件
- **Helm**：建议 Helm 3（工作流里使用 `v3.14.4`）
- **OpenShift（可选）**：如果你希望使用 `Route`（Chart 内置 `route.openshift.io/v1`）
- **权限**：发布到 GHCR 需要仓库具备 `packages: write` 权限（工作流已配置）

### Charts
- **`todo-vue`**：主要示例 Chart
  - **包含资源**：`Deployment`、`Service`、可选 `Ingress`、可选 OpenShift `Route`、可选 `ServiceAccount`
  - **默认端口**：`8080`
- **`todo-vue-test`**：用于验证 workflow 多 Chart 构建/发布的测试拷贝
  - 逻辑与 `todo-vue` 类似，但 Chart 名不同，避免 OCI 推送时冲突

### 本地安装/升级（从源码）
- **安装 `todo-vue`**：
  - `helm upgrade --install todo-vue charts/todo-vue -n <namespace> --create-namespace`
- **安装 `todo-vue-test`**：
  - `helm upgrade --install todo-vue-test charts/todo-vue-test -n <namespace> --create-namespace`

- **Kubernetes 对外暴露（可选）**：
  - **Service**：`--set service.type=NodePort` 或 `--set service.type=LoadBalancer`
  - **Ingress**：`--set ingress.enabled=true`（并配置 `ingress.hosts`/`ingress.tls`）

- **OpenShift Route（可选）**：
  - 默认启用；非 OpenShift 集群请关闭：`--set route.enabled=false`

### 常用配置（values）
以 `charts/todo-vue/values.yaml` 为例：
- **`image.repository` / `image.tag`**：应用镜像
  - 默认：`ghcr.io/xdevops-2026/todo-vue:develop`
- **`image.pullPolicy`**：镜像拉取策略
  - `develop` 是 floating tag，默认使用 `Always`
- **`serviceAccount.create` / `serviceAccount.name`**：ServiceAccount 行为
  - 默认不创建 SA，使用命名空间的 `default` ServiceAccount
- **`automountServiceAccountToken`**：是否挂载 SA token
  - 简单 Web 应用默认 `false`（降低暴露面）
- **`probes.*`**：健康检查
  - 默认启用 `livenessProbe`/`readinessProbe`，确保 `path` 返回 HTTP 200
- **`resources.requests` / `resources.limits`**：资源请求/限制
  - 默认提供一组适合 Node.js Web 应用的基线值，可按负载调整
- **`service.type`**：`ClusterIP`/`NodePort`/`LoadBalancer`
- **`service.port`**：服务端口（默认 `8080`）
- **`ingress.enabled`**：是否创建 Kubernetes `Ingress`（默认 `false`）
- **`route.enabled`**：是否创建 OpenShift `Route`（默认 `true`）

### 发布到 GHCR（OCI）
工作流：`Publish Helm Charts (OCI to GHCR)`（见 `.github/workflows/ci-helm-oci.yml`）

- **触发方式**：
  - **push 到 `main`**：仅发布有变更的 Chart（基于 `charts/**` diff 计算）
  - **打 tag（`v*`）**：发布全部 Chart
  - **手动触发**：`workflow_dispatch` 可指定单个 Chart（例如 `todo-vue`）或 `all`

- **push 前检查（不连接集群）**：
  - `helm lint`
  - `helm template` 输出通过 `kubeconform` 做结构/schema 校验（CRD 如 OpenShift `Route` 会被忽略 schema）

- **发布产物路径（全部小写）**：
  - `oci://ghcr.io/<owner>/<repo>/<chart>`
  - 例如：`oci://ghcr.io/xdevops-2026/todo-vue-chart/todo-vue`

说明：在 OCI 模式下，`helm push <chart.tgz> oci://ghcr.io/<owner>/<repo>` 会自动在 `<repo>` 下创建 `<chart>` 这一层，所以 workflow 里不会把 `<chart>` 拼进 push 目标。

### 从 GHCR 安装（OCI）
- **登录 GHCR**：
  - `helm registry login ghcr.io`
- **安装 `todo-vue`**：
  - `helm install todo-vue oci://ghcr.io/xdevops-2026/todo-vue-chart/todo-vue --version 0.1.2 -n <namespace> --create-namespace`
- **安装 `todo-vue-test`**：
  - `helm install todo-vue-test oci://ghcr.io/xdevops-2026/todo-vue-chart/todo-vue-test --version 0.1.2 -n <namespace> --create-namespace`

### 版本管理
- **`charts/<chart>/Chart.yaml`**：
  - **`version`**：Chart 版本（打包/发布使用它；Chart 有变更时应 bump）
  - **`appVersion`**：应用版本（用于展示；通常填发布版本号，而不是 floating tag）
