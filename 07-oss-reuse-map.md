# Tommax 开源复用地图（07）：功能模块 → 开源项目

> 与 05 文档的区别：05 列的是工程底座（框架/工具链），本篇按**功能模块**逐一回答"这块能不能不自己写"。分四类：A 直接替代整块功能、B 底座级框架（在其上写业务）、C 延后功能的未来复用预案、D 必须自研清单。

---

## A. 直接替代整块功能（部署即用 / 接入即用）

| 功能模块 | 开源项目 | License | 替代掉的工作 | 集成方式与风险 |
|---|---|---|---|---|
| **登录 / 用户体系** | **Casdoor**（Go） | Apache-2.0 | user-svc 的绝大部分：手机号+短信验证码登录（内置阿里云/腾讯云短信 provider）、JWT/OIDC 签发、用户管理界面、第三方登录扩展 | 独立部署，APISIX 校验其 JWT；user-svc 缩为 profile 薄接口甚至砍掉。风险：登录页 UI 定制需用其 SDK 自建页面。备选：Logto（体验更现代，TS 技术栈） |
| **图片静态处理**（缩略图/裁剪/格式转换/水印） | **imgproxy**（Go） | MIT | media-svc 的整条图片处理管线：多档缩略图、裁剪、WebP/AVIF 转换全部 URL 参数 on-the-fly + CDN 缓存；**宫格拆分 = 前端算好 N 个 crop 区域生成 N 个 imgproxy URL**，服务端零开发 | 独立部署在 OSS 前面，签名 URL 防盗刷。成熟度高（生产广泛使用） |
| **任务队列 + 运维面板** | **asynq + asynqmon**（Go） | MIT | generation/media 的 dispatcher 自研：重试、优先级队列、限速、定时、取消、死信、Web 监控面板 | 库形式引入 + asynqmon 独立部署。Phase 1 免上 Kafka 的前提 |
| **时间线服务端合成** | **editly**（Node）✅ 已定 | MIT | media-svc 渲染器的大部分：声明式 JSON（clips/转场/音轨/字幕/文字）→ FFmpeg filtergraph 生成与执行 | **决策（2026-08-18）：接受 Node 例外**。集成形态：worker 容器内仍由 Go 消费 asynq（asynq 任务编码是 Go 侧协议，Node 不直接消费队列）→ 把我们的 composition schema **翻译为 editly spec** → `exec editly spec.json` → 解析输出进度 → 产物上传登记。镜像内打包 Node runtime + editly + ffmpeg。**composition schema 仍是唯一契约**（06 文档双渲染器约定不变），editly spec 只是内部目标格式；editly 覆盖不了的能力（如软字幕封装）用一道 ffmpeg 后处理补齐。风险控制：锁定 editly 版本；翻译层做能力探测，超出 editly 能力的 composition 提交时报参数错误。备选 FFCreator（转场更多，依赖 node-canvas/GL）暂不用 |
| **字幕 ASR** | **FunASR**（达摩院）或 faster-whisper | MIT | 自动字幕的语音识别引擎，FunASR 中文效果与时间戳质量好 | 独立 CPU 推理容器，media-svc 走内部接口。省事备选：先买云 ASR，接口抽象不变 |
| **镜头切分**（视频反推分镜第一步） | **PySceneDetect**；零依赖起步可用 FFmpeg `scdet` 滤镜 | BSD-3 | shot boundary detection 算法自研 | media-svc exec 调用 CLI；精度要求高再上 TransNetV2 |
| **前端上传** | **Uppy**（含 AWS S3 插件） | MIT | 上传组件全套：分片、断点续传、进度、S3 multipart 直传（OSS S3 兼容端点） | media-svc 只做 STS 签发。UI 可完全自定义 |
| **播放器** | **xgplayer**（字节）或 video.js | MIT | 画布/详情页的视频播放兼容性处理 | 国内场景 xgplayer 更顺手 |
| **API 网关** | **APISIX** | Apache-2.0 | 网关自研（已定选型，此处重申归类） | — |
| **本地开发对象存储** | **MinIO** | AGPL-3.0 | 本地/CI 的 S3 兼容存储 | ⚠️ 仅开发环境使用（AGPL）；线上用云 OSS 的 S3 端点，同一套 minio-go 客户端代码 |

## B. 底座级框架（替代不了业务，但决定业务代码量）

| 功能模块 | 项目 | 说明 |
|---|---|---|
| 无限画布 | **React Flow (xyflow)**，MIT | 节点/连线/框选/小地图/吸附/缩放底座。⚠️ 不选 tldraw（自定义 license，需水印或付费）；excalidraw 是白板不是节点图 |
| 标记 / mask 画笔 | **Konva**（或 fabric.js），MIT | 重绘/擦除的 mask 涂抹、标记工具的涂鸦/形状/文字图层，替代手写 canvas 2D 交互 |
| 模型参数表单 | **JSON Schema 驱动表单**：react-jsonschema-form 或 zod + 自研轻渲染器 | 模型目录声明参数 Schema → 表单自动渲染。新接一个模型不写前端表单，这一条决定"接模型"的边际成本 |
| 时间线剪辑 UI | 参考 **OpenCut**（MIT，开源版剪映）与 designcombo 源码；`@xzdarcy/react-timeline-editor` 可评估 | 轨道/刻度/拖拽裁剪交互自研为主、抄成熟实现，不硬套停维护的组件 |
| 服务框架等 | Kratos / wire / buf+protovalidate / ent+Atlas / gobreaker / slog+OTel（见 05） | 已定 |

## C. 延后功能的复用预案（现在不做，但别将来自研）

| 延后功能 | 届时优先评估 | 备注 |
|---|---|---|
| 运营后台 admin-web | **百度 amis**（Apache-2.0，JSON 配置生成后台页面）或 Appsmith | 模型目录/定价/审核工作台这类 CRUD 页面用 amis 拼，省一个前端工程；NocoBase 功能强但 AGPL/商业版需评估 |
| 实时协作 | 终态方案是自研节点级 op+锁（见 02/06）；若未来要**文本级**共编再上 **Yjs + Hocuspocus**（MIT） | 不要为节点锁场景引入 CRDT 复杂度 |
| 积分计费 | **自研更省**：积分模型简单（冻结/结算/解冻），Lago/Kill Bill 面向订阅计量，模型不匹配 | 结论是"评估过，不引" |
| AI 小助手 LLM 网关 | one-api / new-api / LiteLLM | 仅覆盖 LLM；回归时评估，或直接走 model-adapter 的 LLM 通道 |
| 社区搜索 | **Meilisearch**（MIT）优先于 ES | 中文分词好、运维轻，量级足够 |
| 内容审核 | 云供应商内容安全 API（阿里绿网/数美） | 机审无自研价值 |

## D. 必须自研（开源帮不上，也是壁垒所在）

1. **model-adapter 的供应商插件**：Seedance/可灵/MJ/Flux/香蕉/万相/海螺等图片视频 API 没有任何开源聚合器覆盖（one-api 类只做 LLM 文本协议）；
2. **画布连线语义**：槽位收集、prompt 组装、受控词表（景别/运镜/摄像机参数）——这是产品体验的核心；
3. **generation 业务状态机**：asynq 之上的薄业务层（配额、计价挂点、历史、canvas 回写）；
4. **时间线 composition schema 与前端预览渲染器**：双渲染器契约（06 文档 B.5）必须自持，否则"预览≈成片"无法保证；
5. **分镜结构化生成的 prompt 工程**（B.2/B.3 的三段式与受控词表）。

## 集成后的工作量再分配（Phase 1）

引入 A 类项目后，各服务剩余的自研核心：

| 服务 | 引入后剩什么 |
|---|---|
| user-svc | 若用 Casdoor：仅 profile 薄接口（甚至并入网关配置），**约省 80%** |
| generation-svc | 业务状态机 + 配额 + 模型目录 + 提示词模板（队列/重试/面板归 asynq） |
| model-adapter-svc | 全部保留（核心自研） |
| canvas-svc | 全部保留（文档模型 + 整理算法；本就无现成替代） |
| media-svc | 图片线归 imgproxy、上传归 Uppy+STS、合成归 editly/FFCreator（或抄其设计）、ASR 归 FunASR——**剩任务编排与产物登记，约省 60%** |
| tommax-web | 画布归 React Flow、表单归 Schema 渲染、上传归 Uppy、播放归 xgplayer——重头戏剩画布交互定制与时间线 UI |
