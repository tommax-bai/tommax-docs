# Tommax 开发规范（04）：编码规范与命名规范

> 原则：**规范能用工具强制的，一律进 CI，不靠自觉**。本文件每条规范都标注执行方式：〔lint〕自动检查 /〔CI〕流水线卡点 /〔review〕人工评审关注。

---

## 1. 命名规范（全局字典）

### 1.1 仓库与服务

| 对象 | 规则 | 示例 |
|---|---|---|
| 服务仓库 | `tommax-<domain>-svc`，domain 单数、kebab-case | `tommax-canvas-svc`、`tommax-model-adapter-svc` |
| 前端仓库 | `tommax-web` / `tommax-admin-web` | |
| 共享仓库 | `tommax-proto` / `tommax-go-kit` / `tommax-infra` / `tommax-svc-template` | |
| K8s Deployment/Service | 去前缀：`<domain>-svc` | `canvas-svc` |
| K8s Namespace | `tommax-<env>` | `tommax-prod` / `tommax-staging` / `tommax-dev` |
| 镜像 | `registry.example.com/tommax/<domain>-svc:<git-sha>` | 不用 latest 部署 |

### 1.2 API（REST 对外）

- 路径：`/v1/<资源复数-kebab-case>`；层级 ≤3；动作用子资源或标准方法表达，实在需要动词用 `:verb` 后缀。〔review〕
  - `POST /v1/generations`、`GET /v1/generations/{id}`、`POST /v1/generations/{id}:cancel`
  - `POST /v1/canvases/{id}/nodes`、`POST /v1/canvases/{id}:fork`
- 管理端统一前缀 `/admin/v1/...`；供应商回调 `/callback/{provider}`（独立 ingress）。
- JSON 字段 **lowerCamelCase**（与 proto3 JSON 映射、TS 前端天然一致）。〔lint: buf + openapi diff〕
- 分页统一：`pageSize`（≤100）+ `pageToken`/`nextPageToken`（游标式）；列表响应 `items`。
- 时间一律 RFC3339 UTC 字符串；ID 一律字符串。
- 统一响应包（HTTP 状态码 + 业务码双轨）：
  ```json
  { "code": 0, "message": "OK", "data": { ... }, "requestId": "..." }
  ```

### 1.3 gRPC / proto（tommax-proto 仓）

- package：`tommax.<domain>.v1`；目录 `tommax/<domain>/v1/`。〔lint: buf〕
- Service：`<Domain>Service`；RPC 动宾式：`CreateCanvas` / `SubmitGeneration` / `FreezeCredits`。
- message：请求/响应成对 `XxxRequest` / `XxxResponse`；枚举值 `SCREAMING_SNAKE` 且 0 值为 `_UNSPECIFIED`。
- 破坏性变更靠 `buf breaking` 在 CI 卡死；需要不兼容变更时升 `v2` 包并双跑。〔CI〕

### 1.4 数据库（PostgreSQL）

| 对象 | 规则 | 示例 |
|---|---|---|
| 库名 | `tommax_<domain>` | `tommax_canvas` |
| 表名 | snake_case 复数 | `canvas_nodes`、`wallet_ledgers` |
| 主键 | `id BIGINT`（雪花 ID，服务内生成），对外暴露转字符串 | |
| 外键列 | `<entity>_id`（应用层维护，禁用 DB 级联删除） | `canvas_id` |
| 通用列 | `created_at` / `updated_at`（timestamptz）/ 软删 `deleted_at` | |
| 枚举/状态列 | `status`、`<x>_type`，存 SCREAMING_SNAKE 字符串 | `status='RUNNING'` |
| 索引 | `idx_<表>_<列...>` / 唯一 `uk_<表>_<列...>` | `idx_generation_tasks_user_id_created_at` |
| 迁移文件 | `NNNNNN_动作.up/down.sql`，只增不改已合并迁移 | 〔CI〕 |
| JSONB | 仅用于真正无固定 schema 的 payload（节点数据、任务参数），需在 proto 中定义 schema 化的镜像类型 | 〔review〕 |

### 1.5 Redis Key 与 Kafka Topic

- Redis：`tommax:{svc}:{entity}:{id}[:{field}]`，TTL 必设（除长驻房间态），示例：
  `tommax:collab:lock:{canvas_id}:{node_id}`、`tommax:generation:quota:{user_id}`
- Kafka topic：`tommax.<domain>.<event>`，事件名**过去式/完成态** kebab-case：
  `tommax.generation.task-succeeded`、`tommax.moderation.result-updated`
- 任务队列型 topic：`tommax.<domain>.<queue>-queue`：`tommax.generation.dispatch-queue`、`tommax.media.render-queue`
- 消费组：`tommax-<svc>-<用途>`：`tommax-notify-task-events`
- 事件 payload：CloudEvents 风格信封（`id/type/source/time/data`），data 的 schema 注册在 tommax-proto 的 `events/` 目录。〔CI: schema 校验〕

### 1.6 错误码

- 格式：8 位数字 `AABBCCC`：`AA` 可映射 HTTP 大类（40/50），`BB` 域编号，`CCC` 序号；同时给字符串 reason（SCREAMING_SNAKE）。
- 域编号注册表（tommax-proto/errors 统一登记，禁止私设）：
  `01` 通用 · `11` user · `12` billing · `13` generation · `14` model-adapter · `15` canvas · `16` collab · `17` asset · `18` media · `19` community · `20` moderation · `21` assistant · `22` notify · `23` admin
- 示例：`40130001 GENERATION_MODEL_OFFLINE`、`40212002 CREDITS_INSUFFICIENT`（HTTP 402）、`40320003 PORTRAIT_CHECK_REQUIRED`。
- 客户端只依赖 `code` 与 reason 做逻辑分支，`message` 仅做展示，允许随时改文案。

### 1.7 对象存储路径

```
{bucket}/{env}/{domain}/{yyyy}/{mm}/{dd}/{ulid}.{ext}
  例: tommax-media/prod/generation/2026/08/18/01J5...X.mp4
私有资产走 CDN 签名 URL（TTL 30min），公开作品走公共 CDN 域名
```

### 1.8 环境变量与配置

- 环境变量：`TOMMAX_<SVC>_<KEY>`（`TOMMAX_CANVAS_DB_DSN`）；密钥只从 K8s Secret/KMS 注入，禁止入库入仓。〔CI: gitleaks〕
- 配置优先级：默认值 < config.yaml < 环境变量；所有配置项必须出现在 `internal/conf` 结构体中。

### 1.9 Git

- 分支：trunk-based。`main` 保护分支，功能分支 `feat/<issue>-<slug>`、修复 `fix/...`；发布打 tag `v1.4.0`（语义化版本）。
- Commit：Conventional Commits（`feat: / fix: / refactor: / chore: / docs:`），正文中文可。〔CI: commitlint〕
- 合并：squash merge；MR 必须绿 CI + ≥1 reviewer 批准；禁止直接 push main。〔CI〕

---

## 2. Go 编码规范

工具链（全部仓库统一，配置从 tommax-go-kit 同步）：`gofmt + goimports + golangci-lint`（启用 govet、staticcheck、errcheck、revive、gocritic、gosec、ineffassign、misspell、depguard）。〔lint〕

### 2.1 结构与依赖

- 分层依赖方向见 03 文档；`depguard` 配置强制：`domain` 包禁止 import `repo/`、驱动、proto。〔lint〕
- 包名：小写单词、单数、不含下划线（`repo`、`assembler`）；文件名 snake_case（`canvas_service.go`）。
- 接口定义在**消费方**（domain 定义 Repo 接口）；接口保持小（1–4 个方法），不搞万能接口。
- 禁止 `util`/`common`/`helper` 垃圾抽屉包；工具函数按主题归包（`timex`、`idgen`）。〔review〕

### 2.2 错误处理

- 透传加语境：`fmt.Errorf("load canvas %d: %w", id, err)`；判断用 `errors.Is/As`，不比对字符串。
- 领域错误用 sentinel + 错误码包装（go-kit `errs.New(code, reason)`），handler 层由统一中间件转响应；**业务代码不吞错、不重复打日志**（日志只在最外层打一次，带 requestId/userId/taskId）。
- 禁止 `panic` 做流程控制；worker 内 recover 只用于隔离单任务崩溃。

### 2.3 并发与上下文

- `context.Context` 必须是第一参数，贯穿到 repo/外呼；所有外呼必设超时（gRPC 客户端默认 3s，供应商调用按任务型放宽）。
- goroutine 必须有归属：用 `errgroup`/worker 池，禁止裸 `go func()` 无生命周期管理；channel 关闭权归发送方。
- 共享状态优先消息传递或加锁小临界区；`-race` 在 CI 单测中常开。〔CI〕

### 2.4 数据与事务

- SQL 一律走 sqlc/squirrel 参数化，禁止手拼字符串。〔lint: gosec〕
- 事务只在 service 层开启（TxManager），事务内禁止外呼 RPC/HTTP；跨服务一致性用 outbox + 事件 + 幂等消费（消费者按 `event_id` 去重）。
- 幂等：所有写型对外接口支持幂等键（任务提交用客户端 `requestId`，计费用 `task_id`）。

### 2.5 测试

- 单测：表驱动 + testify；domain/service 覆盖率 ≥70%（CI 卡点），repo 用 testcontainers 跑真 PG。〔CI〕
- mock 只 mock 自己定义的接口（gomock），不 mock 第三方 SDK；对 model-adapter 的 provider 写契约测试（录制回放）。
- 集成冒烟：每服务提供 `make e2e`，compose 起依赖跑核心链路。

### 2.6 日志与可观测

- 结构化日志（slog）：`level/ts/svc/traceId/requestId/userId/msg` 固定字段；禁止 `fmt.Println`。〔lint〕
- 指标命名：`tommax_<svc>_<subject>_<unit>`（`tommax_generation_task_duration_seconds`）；每个服务必带 RED 指标 + 队列深度。
- 生成链路 trace 必须贯穿：HTTP → Kafka（header 透传）→ worker → 供应商调用。

### 2.7 注释与文档

- 导出符号必须有 doc comment（revive 强制）；注释写"为什么/约束"，不写"做了什么"。〔lint+review〕
- 每服务 README 维持三段：一句话职责、本地启动、负责人；接口文档以 proto 注释为唯一事实源。

---

## 3. TypeScript / React 编码规范

工具链：ESLint（typescript-eslint strict + react-hooks + import 边界规则）+ Prettier + tsc `strict: true`。〔lint〕

### 3.1 类型

- `strict` 全开；禁止 `any`（必要处 `unknown` + 收窄）；对外数据一律用生成类型（openapi-typescript），禁止手写接口响应类型。〔lint〕
- 联合字面量优先于 enum；工具类型放 `shared/lib/types.ts`。

### 3.2 组件与状态

- 函数组件 + hooks；组件文件 `PascalCase.tsx`，一个文件一个导出组件；hooks `useXxx.ts`；其余文件 kebab-case。
- 组件 props 类型 `XxxProps`；回调 prop `onXxx`，处理函数 `handleXxx`。
- 服务端状态只走 TanStack Query（key 规范：`[domain, resource, params]`）；本地交互态 zustand，store 放 entity/feature 内部，禁止全局大 store。
- 层级 import 边界（eslint-plugin-boundaries）：`pages → features → entities → shared` 单向。〔lint〕
- 画布/时间线等高频渲染区：节点组件必须 memo 化、拖拽用 rAF 节流、大列表虚拟化。〔review〕

### 3.3 样式与资源

- CSS：Tailwind + CSS Modules（复杂编辑器区域）；设计 token（色彩/间距/圆角）集中在 `shared/ui/tokens`，禁止散落魔法色值。
- 文案集中管理（为后续 i18n 预留），组件内不写死长文案。

### 3.4 WS/SSE 客户端

- 所有长连接经 `shared/ws` 的连接管理器：指数退避重连、心跳、页面隐藏降频；断线期间的 op 本地排队重放（协作场景）。

---

## 4. CI 卡点清单（每个服务仓一致）

| 阶段 | 卡点 |
|---|---|
| pre-commit | gofmt/goimports、eslint --fix、commitlint |
| MR CI | lint（golangci / eslint+tsc）→ unit test（-race，覆盖率阈值）→ buf lint & breaking → migration dry-run → gitleaks → docker build |
| merge 后 | 镜像推送 → dev 环境 ArgoCD 自动同步 → 冒烟 e2e |
| 发布 | tag → staging 验证 → prod 灰度（网关按 header/比例）|

---

## 5. 规范落地顺序建议

1. 先立 `tommax-proto`、`tommax-go-kit`、`tommax-svc-template` 三仓，把中间件链、错误码、日志、Makefile、CI 模板固化；
2. 用模板仓生成 user-svc / generation-svc / model-adapter-svc 三个首批服务，跑通「登录 → 提交文生图 → 适配一个供应商 → 回调 → 推送」纵向切片；
3. 其余服务按 02 文档清单以同一模板铺开。任何偏离本规范的例外，必须在服务 README 的「例外登记」一节写明原因。
