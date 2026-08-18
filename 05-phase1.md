# Tommax 初版范围（05）：Phase 1 裁剪决策与开源选型

> 01–04 文档保持为**终态蓝图**不变；本文档定义 Phase 1 实际开工范围：哪些不做、砍掉之后架构怎么简化、引入哪些开源框架和工具来减少自研并提高标准化。

---

## 1. Phase 1 范围裁剪

### 1.1 明确不做的七块（用户拍板 2026-08-18）

| # | 砍掉的能力 | 对应终态服务 | 砍掉后的替代/影响 |
|---|---|---|---|
| 1 | 我的资产（资产页/收藏）、积分账户（计价/充值/明细） | asset-svc、billing-svc | 生成不计费不扣积分；**上传与媒资元数据是硬依赖砍不掉**，降级为薄模块并入 media-svc（见 1.3） |
| 2 | 协作与分享（多人实时、分享链接、公开画布） | collab-svc | 画布仅单人编辑，前端直接 REST 持久化（终态设计里单人本就走 REST，协作层是纯增量） |
| 3 | 社区内容（Feed/模板广场/资源广场/一键引用） | community-svc | 首页只剩模型展示区；ES 一并不引入 |
| 4 | 审核合规（人像合规检测、机审/人审） | moderation-svc | ⚠️ 风险登记：若供应商（如 Seedance 系列）强制要求人像预检，接入该系列时需补一个最小预检调用，放在 model-adapter 的 provider 内部处理 |
| 5 | AI 小助手 | assistant-svc | 不做；LLM 通道仅保留给分镜脚本生成（script.* 任务走 model-adapter） |
| 6 | 通知推送 | notify-svc | 任务进度改为**前端轮询**（TanStack Query，2s 间隔退避）；generation-svc 预留一个 SSE 进度端点作为体验优化项 |
| 7 | 运营配置后台 | admin-svc、tommax-admin-web | 模型目录改为**配置文件管理**（generation-svc 的 model_catalog.yaml，随发布生效） |

3D 导演台维持暂缓（此前已定）。

### 1.2 Phase 1 保留的产品主干

登录（薄）→ 快速生成（文生图/参考生图/文生视频/首尾帧/音乐 + 摄像机控制 + 常用提示词）→ 单人画布（全部节点工具：扩图/重绘/擦除/打光/多角度/高清/裁剪/标记/宫格拆分）→ 分镜脚本（文本生成/视频反推/批量出图/分镜图生视频）→ 时间线剪辑与导出（MP4/WebM）→ 历史生成记录（"历史创作引用"依赖它，由 generation-svc 提供）。

### 1.3 Phase 1 服务清单（13 → 5）

| 服务 | Phase 1 职责 | 相对终态的变化 |
|---|---|---|
| `user-svc` | 登录/JWT/用户资料（可先账号密码或白名单手机号，短信后接） | 保留但最薄 |
| `generation-svc` | 任务状态机/调度/历史/提示词模板/**模型目录（配置文件）**/SSE 进度（可选） | 去掉积分冻结与合规前置两步编排 |
| `model-adapter-svc` | 供应商适配插件/密钥/限流熔断/成本流水 | 不变（这是不能省的自研核心） |
| `canvas-svc` | 单人画布文档 CRUD/节点/连线/组/标签/检索/智能整理 | 去掉协作者、分享、公开、op log 简化为直接写 |
| `media-svc` | **上传 STS + 媒资元数据（吸收原 asset 最小集）** + 转码/截帧/缩略图 + 时间线导出渲染 | 吸收 asset 薄化后的职责 |

基建仓不变：`tommax-proto` / `tommax-go-kit` / `tommax-svc-template` / `tommax-infra` / `tommax-web`。

**基础设施同步简化**：Kafka 和 Elasticsearch Phase 1 不引入 —— 任务队列与事件用 Redis（asynq，见下文），全套只需 **PostgreSQL + Redis + OSS**。服务边界与 proto 契约仍按终态设计定义，后续把砍掉的服务按 02 文档补回来即可，不需要返工。

### 1.4 简化后的生成链路

```
前端 → gateway → generation-svc（校验 → 建任务 → asynq 入队）
  worker → model-adapter → 供应商 → 回调/轮询归一化
  → media-svc 转存 OSS + 登记元数据 → 任务置 succeeded
前端轮询 GET /v1/generations/{id}（或订阅 SSE）→ 画布节点/生成面板刷新
```

---

## 2. 开源框架与工具选型

原则：**每引入一项必须替代掉一块本要自研的代码，或把一条规范变成工具强制**。按层列出，标注"替代了什么"。

### 2.1 Go 后端

| 工具 | 用途 | 替代/收益 |
|---|---|---|
| **Kratos v2** | 微服务框架：gRPC+HTTP 双协议、中间件链、config | 替代自研脚手架；proto 定义 API 同源生成 HTTP 路由 |
| **wire** | 编译期依赖注入 | 替代手写装配，分层依赖显式化 |
| **buf + protovalidate** | proto 管理、lint、breaking 检查、请求校验 | 契约治理标准化；handler 校验代码几乎归零 |
| **ent**（+ Atlas 迁移） | ORM：schema-as-code、类型安全查询、代码生成 | canvas/media 大量 CRUD 不手写；迁移文件自动生成并可 CI diff 审查 |
| **asynq**（Redis） | 任务队列：重试/优先级队列/限速/定时/取消 + asynqmon 监控 UI | **替代自研 dispatcher 大部分代码**（重试、优先级、并发控制、死信）；Phase 1 不用上 Kafka |
| **gobreaker + golang.org/x/time/rate** | 供应商调用熔断与限流 | model-adapter 稳定性基建不自研 |
| **golang-jwt/jwt** | JWT 签发校验 | user-svc 认证 |
| **sonyflake / oklog-ulid** | 分布式 ID | 主键与 OSS 路径 ID |
| **minio-go（S3 协议）** | 对象存储 SDK | 线上对接 OSS 的 S3 兼容端点，本地开发用 MinIO，一套代码 |
| **ffmpeg + ffprobe（exec 封装）** | 时间线合成、转码、截帧、元数据 | media-svc 核心；不引入包装库，直接编排命令行（可控性最高） |
| **slog + OpenTelemetry SDK** | 结构化日志、trace/metrics | 可观测标准化 |
| **testify + gomock + testcontainers-go** | 单测断言、mock、真库集成测试 | repo 层测试不靠 H2 式假库 |

### 2.2 前端（tommax-web）

| 工具 | 用途 | 替代/收益 |
|---|---|---|
| **React Flow (xyflow)** | 无限画布：节点/连线/框选/小地图/吸附/缩放 | **画布底座省掉数月自研**，只写自定义节点与交互 |
| **TanStack Query** | 服务端状态、任务轮询（refetchInterval）、缓存失效 | 替代手写轮询与加载态管理 |
| **zustand** | 画布编辑器等本地交互态 | 轻量、无样板代码 |
| **react-hook-form + zod** | 生成参数表单与校验（模型参数 Schema 驱动） | 表单代码标准化 |
| **shadcn/ui + Radix + Tailwind** | 组件库与设计 token | UI 一致性；组件代码在自己仓里可改 |
| **openapi-typescript + openapi-fetch** | 从 proto→OpenAPI 生成类型与客户端 | 前后端类型零手写、永不漂移 |
| **@xzdarcy/react-timeline-editor**（评估）/ 自研轨道 | 时间线剪辑 UI 底座 | 先评估复用，不合适再自研轨道组件 |
| **video.js** | 视频预览播放 | 兼容性兜底 |
| **Vite + vitest + Playwright** | 构建、单测、E2E | 工具链标准化 |

### 2.3 工程化与 DevOps

| 工具 | 用途 | 收益 |
|---|---|---|
| **golangci-lint / ESLint+Prettier / husky+lint-staged / commitlint** | 规范工具化 | 04 文档的规范条款全部变成 CI 卡点 |
| **gitleaks** | 密钥泄漏扫描 | CI 卡点 |
| **Renovate** | 依赖自动升级 PR | 多仓依赖不腐化 |
| **Helm + ArgoCD** | 部署与 GitOps | 多仓统一发布模式 |
| **KEDA** | 按 Redis 队列深度扩缩 media/generation worker | 替代手写 HPA 逻辑 |
| **external-secrets + KMS** | 密钥注入 | 供应商 Key 不进镜像不进仓 |
| **Grafana + Prometheus + Loki + Tempo** | 可观测全家桶 | 与 OTel 直连 |
| **MinIO + docker compose** | 本地一键起 PG/Redis/MinIO | 本地开发环境标准化（tommax-infra 提供 compose 文件） |
| **asynqmon** | 任务队列监控面板 | 生成任务运维可视化，零开发 |

### 2.4 评估过但不引入的

| 候选 | 结论 |
|---|---|
| ComfyUI（节点式生成引擎） | 面向本地 GPU 工作流，与"调用商用 API"的形态不符；仅作画布交互参考 |
| one-api / LiteLLM（模型网关） | 只覆盖 LLM 文本协议，图片/视频供应商异构协议帮不上；model-adapter 仍自研（它就是我们的壁垒） |
| Kafka（Phase 1） | 5 个服务的事件量用不上，asynq/Redis 足够；终态引入时机：协作与社区回归、事件消费者变多之后 |
| Remotion（React 服务端合成视频） | 需要 Node+Chromium 渲染农场，重；FFmpeg 编排已覆盖时间线导出需求 |
| Yjs/CRDT | 协作已延后；且终态方案本就是节点级锁+op，不需要字符级 CRDT |

---

## 2.5 纵向切片落地修订（2026-08-18 实施记录）

切片实际落地采用**轻装组合**，与 §2.1 表格的差异已在各服务 README「例外登记」中记录，回收条件明确：

| §2.1 原选型 | 切片实际 | 原因与回收条件 |
|---|---|---|
| Kratos v2 | chi + grpc-go（分层结构不变：handler/service/domain/repo） | 切片阶段框架收益小于成本；服务数 ≥3 或需 gRPC 对外时统一接线 |
| ent + Atlas | pgx 手写 SQL + golang-migrate（嵌入式启动迁移） | 单表场景代码生成收益为负；canvas/media 落地时评估回改 |
| wire | 手动装配（main.go） | 依赖图浅；装配超 ~100 行时引入 |
| Casdoor | DevAuth 临时中间件（X-Dev-User，仅本地） | Casdoor 部署后删除 |

不变的部分：asynq 队列、buf/proto 契约、pgx、minio-go(objstore)、错误码/日志/配置基建（tommax-go-kit）、命名规范全套。

## 3. Phase 1 开工顺序（更新版）

1. 基建三仓：`tommax-proto`（user/generation/canvas/media 四域 v1 契约）、`tommax-go-kit`（中间件链/错误码/asynq 封装/otel）、`tommax-svc-template`；
2. 纵向切片：user 登录 → 提交 `image.text2img` → model-adapter 接**第一个供应商** → 产物入 MinIO → 前端轮询出图；
3. 横向铺开：canvas-svc（画布+节点工具逐个接 generation 任务类型）→ media-svc 时间线导出 → 分镜脚本链路；
4. 全程用 compose 本地环境开发，K8s dev 环境随纵向切片一起立起来。
