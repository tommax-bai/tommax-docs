# Tommax 服务内分层设计（03）：Go 服务标准结构与前端工程分层

所有 Go 服务共用同一套骨架（脚手架收敛在 `tommax-go-kit` 仓 + 服务模板仓 `tommax-svc-template`），保证任何人打开任何服务仓库，目录结构、依赖方向、代码位置的预期完全一致。

## 1. Go 服务标准目录（以 canvas-svc 为例）

```
tommax-canvas-svc/
├── cmd/
│   └── server/
│       └── main.go              # 唯一入口：读配置 → wire 装配 → 启动 gRPC/HTTP/consumer
├── configs/
│   └── config.yaml              # 本地开发配置模板（线上由 ConfigMap/env 覆盖）
├── internal/                    # 全部业务代码放 internal，禁止被其他仓 import
│   ├── conf/                    # 配置结构体（由 proto 或 struct 定义，禁止散落 os.Getenv）
│   ├── server/                  # ── 传输层 ──
│   │   ├── grpc.go              # gRPC server 装配、拦截器注册
│   │   ├── http.go              # HTTP server 装配（由 proto 注解生成路由）
│   │   └── middleware/          # 本服务特有中间件（鉴权提取、审计等）
│   ├── handler/                 # ── 接口层 ──
│   │   ├── canvas_handler.go    # 实现 proto Service 接口：DTO 校验、组装、调 service
│   │   └── assembler/           # DTO ↔ 领域对象 转换器（双向映射只写在这里）
│   ├── service/                 # ── 应用层（用例编排）──
│   │   ├── canvas_service.go    # 一个用例一个方法：事务边界、跨聚合编排、事件发布
│   │   └── share_service.go
│   ├── domain/                  # ── 领域层（纯 Go，不 import 任何基础设施包）──
│   │   ├── canvas.go            # 实体/值对象/领域行为（Canvas、Node、Edge、Group）
│   │   ├── canvas_repo.go       # Repository 接口（定义在消费方）
│   │   ├── errors.go            # 领域错误（sentinel errors）
│   │   └── events.go            # 领域事件定义
│   ├── repo/                    # ── 基础设施层 ──
│   │   ├── canvas_pg.go         # domain.CanvasRepo 的 PostgreSQL 实现
│   │   ├── canvas_cache.go      # Redis 缓存装饰器
│   │   ├── client/              # 对其他服务的 gRPC 客户端封装（防腐层）
│   │   │   └── asset_client.go  # 把 asset.v1 的响应转成本域值对象
│   │   └── eventbus/            # Kafka 生产者封装
│   ├── consumer/                # ── 异步入口层（与 handler 平级）──
│   │   └── generation_consumer.go  # 订阅 generation.task-succeeded → 调 service
│   └── job/                     # 定时任务（快照清理等），入口同样只调 service
├── migrations/                  # SQL 迁移（golang-migrate，编号递增，只增不改）
│   ├── 000001_init.up.sql
│   └── 000001_init.down.sql
├── deployments/
│   ├── Dockerfile
│   └── helm/                    # values.yaml（chart 模板统一放 tommax-infra）
├── scripts/                     # make 调用的辅助脚本
├── Makefile                     # init/generate/lint/test/build/run 五板斧，所有仓一致
├── .golangci.yaml               # 从 tommax-go-kit 同步的统一 lint 配置
├── go.mod
└── README.md                    # 服务职责一句话 + 本地启动方式 + 负责人
```

### 1.1 分层与依赖方向（唯一铁律）

```
        handler / consumer / job        （接口层：协议、校验、转换）
                  │  只准向下调用
                  ▼
              service                   （应用层：用例、事务、事件）
                  │
                  ▼
              domain          ◄─────┐   （领域层：模型与规则，零依赖）
                  ▲                 │ 实现接口
                  │ 定义接口         │
              repo / client / eventbus  （基础设施层：DB/缓存/MQ/外部服务）
```

- **domain 不 import 任何框架、驱动、proto 包**；repo 实现 domain 定义的接口（依赖倒置），由 wire 在 `cmd` 装配。
- **DTO 不下穿、实体不上穿**：proto 生成的 request/response 只活在 handler；domain 实体不直接序列化给前端，经 assembler 转换。
- **事务边界只在 service**：repo 不开事务；跨 repo 的一致性由 service 用 `TxManager`（go-kit 提供）包裹。
- **横向禁止**：handler 不得调 repo；service 不得感知 HTTP/gRPC 细节；repo 之间互不调用。
- **对外调用必须过防腐层**：其他服务的 gRPC client 封装在 `repo/client/`，返回本域的值对象，不允许把别人的 proto 类型漏进 service/domain。

### 1.2 一次请求的生命周期（约定即文档）

```
HTTP/gRPC 请求
 → 网关注入 X-User-Id / trace 头
 → 中间件链（tommax-go-kit 统一顺序）: recovery → tracing → metrics → logging → auth → ratelimit → validate
 → handler: DTO 校验(protovalidate) → assembler → service
 → service: 权限断言(资源 owner) → 领域操作 → repo 持久化 → 发领域事件(事务内 outbox)
 → handler: 领域结果 → assembler → 响应 DTO
错误一路 wrap 上抛，只在中间件层统一打日志、转错误码（见 04 规范）
```

### 1.3 worker 型服务的形态差异

generation-svc（dispatcher）、media-svc、moderation-svc（机审流水线）在同一骨架上增加：

```
internal/
├── consumer/           # Kafka 消费入口（受理即 ack + 业务表状态机去重）
├── worker/             # 执行器池：并发控制(semaphore)、心跳、优雅退出
└── pipeline/           # 多步执行的编排（如：下载→FFmpeg→上传→回执）
```

约定：**队列消息只带 task_id 等标识，不带大 payload**；worker 回源数据库取任务详情，凭状态机（`queued→running` 的 CAS）保证同一任务不被重复执行；所有 worker 必须支持 SIGTERM 后完成在手任务再退出（K8s 滚动更新安全）。

### 1.4 model-adapter-svc 的插件式结构（特例）

```
internal/
├── core/                  # 统一协议定义：InferenceRequest/Result、错误类别、能力描述
├── provider/              # 一个供应商一个包，实现 core.Provider 接口
│   ├── seedance/
│   ├── kling/
│   ├── midjourney/
│   ├── flux/
│   ├── mureka/
│   └── .../
├── router/                # 能力路由：model_key → provider 实例；备用路由/熔断(gobreaker)
├── secret/                # KMS 密钥装载，只在本包可见
└── callbackd/             # 供应商回调归一化入口（验签→core.Result）
```

新接一个供应商 = 新增一个 `provider/xxx` 包 + 注册表登记 + admin 配置，**不改动任何既有代码**（开闭原则是这个服务的立身之本）。

## 2. 前端工程分层（tommax-web）

React 18 + TypeScript + Vite。按 Feature-Sliced 思路分层，层间单向依赖（上层可用下层，禁止反向）：

```
tommax-web/
├── src/
│   ├── app/                  # 应用壳：路由、Providers(Query/Theme/Auth)、全局样式、错误边界
│   ├── pages/                # 路由页面：home / generator / canvas / assets / work-detail / wallet
│   │                         #   页面只做布局与 feature 组装，不写业务逻辑
│   ├── features/             # 业务功能单元（一个目录 = 一个可独立演进的功能）
│   │   ├── generator-panel/  #   快速生成面板（文生图/参考生图/视频/音乐 表单与参数）
│   │   ├── camera-control/   #   摄像机控制面板
│   │   ├── prompt-templates/ #   常用提示词管理
│   │   ├── canvas-editor/    #   无限画布（React Flow 定制渲染器、节点组件、连线交互）
│   │   ├── canvas-collab/    #   协作接入（WS 客户端、光标层、锁状态渲染）
│   │   ├── image-tools/      #   扩图/重绘/擦除/打光/多角度/裁剪/标记 等弹层工具
│   │   ├── storyboard/       #   分镜脚本表格、批量生成
│   │   ├── timeline-editor/  #   时间线剪辑器（轨道、属性面板、预览）
│   │   ├── assistant-chat/   #   AI 小助手（SSE 流式）
│   │   ├── community-feed/   #   发现页 Feed、模板/资源广场
│   │   └── wallet/           #   积分与充值
│   │   # └── director3d/     #   【预留】3D 导演台，暂不实现
│   ├── entities/             # 领域模型层：类型 + 数据访问 hook + 局部 store
│   │   ├── user/  task/  canvas/  asset/  work/  model-catalog/
│   │   #   每个 entity: model.ts(类型) api.ts(请求) store.ts(zustand) hooks.ts
│   ├── shared/               # 与业务无关的底座
│   │   ├── api/              #   openapi-typescript 生成的客户端 + fetch 封装(拦截器/错误归一)
│   │   ├── ui/               #   基础组件库(Button/Modal/Toast…，设计系统)
│   │   ├── ws/               #   WebSocket/SSE 连接管理(重连/心跳)
│   │   ├── lib/              #   工具函数、常量
│   │   └── config/           #   环境配置
│   └── main.tsx
├── public/
└── package.json
```

- 状态管理：服务端状态用 TanStack Query（任务轮询/失效重取），客户端交互态用 zustand（画布编辑器局部 store）。
- 画布：React Flow（xyflow）做节点/连线底座 + 自定义节点渲染；小地图/框选/吸附用其扩展；时间线剪辑器自研轨道组件，预览用 `<video>` 分段播放，最终效果以服务端导出为准。
- API 类型链路：`tommax-proto` → 生成 OpenAPI → `openapi-typescript` 生成 TS 类型，前后端字段永不手写重复定义。
- tommax-admin-web 同构此结构（Ant Design 组件库，功能密度优先）。

## 3. 共享仓库的边界

| 仓库 | 放什么 | 坚决不放什么 |
|---|---|---|
| tommax-proto | 全部 gRPC/REST 契约（buf 模块化：`tommax/{domain}/v1/`）、错误码注册表、事件 payload schema | 任何实现代码 |
| tommax-go-kit | 日志(slog 封装)、错误与错误码基建、中间件链、TxManager、Kafka outbox、config loader、taskkit(状态机基类)、测试工具 | 业务逻辑、任何领域类型 |
| tommax-infra | Helm chart 模板、ArgoCD app、Terraform、网关路由声明、告警规则 | 应用代码 |

共享库升级策略：语义化版本 + 各服务按需升级；**go-kit 禁止出现"改一次全体必须同步升级"的破坏性变更**（加新不改旧）。
