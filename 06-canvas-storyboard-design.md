# Tommax 详细设计（06）：画布工作流 · 分镜与剪辑 · 业界方案与高质量产出

> 展开 01 文档中「画布工作流」「分镜与剪辑」两个功能域的实现设计（Phase 1 范围内），并分析业界 AI 视频工具的设计思路，给出"可高质量产出内容"的产品与工程方法论。

---

## Part A：画布工作流展开设计

### A.1 数据模型

画布文档 = 节点 + 边 + 组三张表（canvas-svc，见 02 文档），节点用 `payload jsonb` 承载各类型差异：

```
canvas_nodes
├── id / canvas_id / node_type / x / y / w / h / z_index
├── group_id（可空）/ label（用户自定义标签，缩放时字号恒定）
├── status: IDLE | GENERATING | SUCCEEDED | FAILED   ← 生成型节点的展示状态
├── payload jsonb（按 node_type 区分，schema 在 proto 中镜像定义）：
│   ├── text:       { content }
│   ├── image:      { asset_id, source: UPLOAD|GENERATED|SCREENSHOT, gen_task_id?, mask_layers[]? }
│   ├── video:      { asset_id, source, gen_task_id?, poster_asset_id }
│   ├── audio:      { asset_id, source, gen_task_id? }
│   ├── storyboard: { title, aspect_ratio, shots[]（见 Part B）}
│   └── timeline:   { composition（见 B.4 编排 schema）, export_history[] }
└── created_at / updated_at

canvas_edges: id / canvas_id / from_node_id / to_node_id
  边的语义 = 数据流依赖：`to` 节点生成时，`from` 节点作为上下文输入
canvas_groups: id / canvas_id / name / bg_color / layout: FREE|GRID|HORIZONTAL|VERTICAL
```

要点：
- **节点不存文件，只存 asset_id 引用**（文件归 media-svc 管），删节点不删资产；
- **生成型节点自带任务引用** `gen_task_id`，前端凭它轮询 generation-svc，任务完成后 canvas-svc 消费事件把 `asset_id` 写回 payload（终态经事件，Phase 1 由前端收到结果后直接 PATCH 节点，服务端二次校验任务归属）；
- 重绘/擦除的 mask 不进原图：作为 `mask_layers[]`（独立小 PNG 资产）挂在图片节点上，提交任务时与原图一起传给 adapter。

### A.2 连线驱动生成：输入收集与请求组装

这是画布区别于普通生成器的核心机制。规则要少而确定，用户才能建立稳定预期：

1. **触发点在目标节点**：用户从某节点拉线新建"AI 生成图片/视频"节点，或对已有空节点点"生成"。生成动作永远发生在目标节点上，入边决定上下文。
2. **输入收集算法**（前端执行，服务端校验）：沿目标节点的**直接入边**（只看一跳，不递归）收集上游节点，按类型装槽：

| 上游节点类型 | 目标=图片生成 | 目标=视频生成 |
|---|---|---|
| text | → prompt 槽（多个 text 拼接，按节点 y 坐标排序） | → prompt 槽 |
| image | → 参考图槽（多图，Shotlab 支持多参考图/组图框选引用） | → 参考图槽 / 首帧槽（用户在面板中指认首帧或尾帧） |
| video | ✗（不支持） | → 视频参考槽（video2video） |
| storyboard | ✗ | 由分镜节点内部批量触发（见 Part B），不走通用收集 |

3. **槽位可见即可改**：收集结果显示在目标节点的生成面板里（参考图列表、prompt 文本框），用户可增删改——连线只是"预填"，面板才是提交事实。这避免了隐式规则带来的不可控。
4. **请求组装**：`(task_type, model, prompt, ref_asset_ids[], first_frame, last_frame, params)` → POST /v1/generations，请求体带 `canvas_ctx: {canvas_id, node_id}` 用于回写与历史溯源。
5. **一目标多结果**：一次生成多张时，结果平铺为多个子图片节点并自动连线到生成节点（网格排布在其右侧），符合"从多个画面挑选"的工作方式。

### A.3 图片节点工具 → 任务类型映射与交互契约

| 工具 | task_type | 输入 | 关键参数/交互 |
|---|---|---|---|
| 扩图 | `image.outpaint` | 原图 | 11 种画幅比预设 + 自由拖拽扩展区域；输出 1K/2K/4K |
| 重绘 | `image.inpaint` | 原图 + mask + prompt | 前端画笔/框选生成 mask 图层（canvas 2D 导出 PNG） |
| 擦除 | `image.erase` | 原图 + mask | 同上，无 prompt |
| 打光 | `image.relight` | 原图 + 光照参数 | 光源角度（拖拽球面坐标）/颜色/亮度/轮廓光开关 + 6 预设；参数结构化传给支持的模型，不支持的模型由 adapter 转译为 prompt 描述 |
| 多角度 | `image.multi_view` | 原图 + 相机参数 | 立方体拖拽/数值滑块 → (yaw, pitch, fov)；广角一键 |
| 图片高清 | `image.upscale` | 原图 | 目标 2K/4K |
| 宫格拆分 | —（media-svc 同步接口） | 原图 | 4/9/16 宫格服务端切图，产物为 N 个新图片节点 |
| 裁剪 | —（前端 + media 同步） | 原图 | 比例预设，前端框选、服务端出图保真 |
| 标记 | —（纯前端图层） | 原图 | 涂鸦/形状/文字存为标注图层 jsonb，导出时可选烧录 |

交互契约：所有异步工具在节点右上角显示进度环，完成前原图仍可用；失败在节点内显示原因（错误码映射文案）+「重试」。

### A.4 前端画布架构（tommax-web / features/canvas-editor）

- **底座**：React Flow (xyflow)——节点/边渲染、拖拽、框选、缩放、小地图、吸附对齐开箱即用；自定义 nodeTypes 7 种 + 自定义边（贝塞尔 + 端点加号交互）。
- **性能预算**：500 节点 / 60fps 拖拽。手段：节点组件全部 `memo` + zustand selector 精准订阅；图片用缩略图渲染（media-svc 生成多档缩略图），原图仅在放大/编辑时加载；拖拽 rAF 节流；视口外节点 React Flow 自带虚拟化。
- **保存策略（Phase 1 单人）**：本地即时 → 500ms 防抖批量 PATCH（节点粒度 diff）；乐观更新 + 失败回滚提示；浏览器关闭前 sendBeacon 兜底。版本安全靠 canvas-svc 的 `updated_at` 乐观锁。
- **快照**：每 50 次写或 10 分钟触发服务端快照（canvas_snapshots），支持"恢复到某时间点"（Phase 1 仅保留最近 10 个）。
- **画布智能整理**：canvas-svc 内置算法——① 按连通分量聚类；② 每个分量内按边方向做分层布局（Sugiyama 简化版：生成链从左到右）；③ 分量间网格排布；返回新坐标数组，前端动画过渡。成组的 GRID/水平/垂直排列是同一算法的局部应用。

### A.5 画布 ←→ 生成任务的状态一致性

- 节点 `status=GENERATING` 的唯一事实源是 generation-svc 的任务状态，前端轮询驱动 UI；canvas-svc 存的 status 仅是最后已知值（打开画布时先渲染再校准）。
- 用户删除生成中节点：前端先调 `POST /v1/generations/{id}:cancel` 再删节点；取消失败不阻塞删除（任务成为孤儿任务，结果只进历史不回写）。
- 打开画布时批量校准：`GET /v1/generations?ids=...` 一次拉齐所有 GENERATING 节点的真实状态。

---

## Part B：分镜与剪辑展开设计

### B.1 分镜数据模型（storyboard 节点 payload）

```jsonc
{
  "title": "香水广告-30s",
  "aspectRatio": "16:9",          // 从分镜阶段对齐成片画幅（产品文档明确要求）
  "globalStyle": "胶片质感, 冷青色调, 24mm 广角...",   // 全局风格块，注入每个 shot
  "characters": [ { "name": "女主", "refAssetIds": ["..."] } ],  // 角色参考资产
  "shots": [
    {
      "no": 1,
      "duration": 3.5,             // 秒
      "shotSize": "CLOSE_UP",      // 景别: 远/全/中/近/特
      "cameraMove": "DOLLY_IN",    // 运镜: 固定/推/拉/摇/移/跟/升降/环绕
      "description": "清晨逆光，香水瓶特写，雾气缓慢流动",
      "dialogue": "",              // 台词/旁白
      "sfx": "环境音: 鸟鸣",
      "frameAssetId": null,        // 分镜图（首帧）
      "frameTaskId": null,         // 出图任务引用
      "videoAssetId": null,        // 分镜视频
      "videoTaskId": null,
      "status": "DRAFT"            // DRAFT | FRAME_READY | VIDEO_READY
    }
  ]
}
```

shot 是**最小创作单元**：出图、出视频、重试、重排、删除都以 shot 为粒度。表格视图直接编辑字段；画幅比例改动会提示已生成内容需重出。

### B.2 文本 → 分镜脚本（`script.text2storyboard`）

- LLM 走 model-adapter 的 LLM 通道，**强制结构化输出**（JSON Schema 约束 + 失败重试一次 + 服务端 schema 校验），产物即 B.1 的 shots 数组。
- Prompt 设计分三段：系统段（专业分镜师人设 + 字段定义 + 景别/运镜受控词表）、控制段（目标时长/镜头数上限/画幅/风格要求）、用户段（剧本原文）。受控词表保证 shotSize/cameraMove 是枚举值而非自由文本——下游出图 prompt 组装和时间线时长计算都依赖它。
- 生成后进入表格确认态，用户改完再触发出图（产品文档的"脚本内容确认"步骤）。

### B.3 视频 → 反推分镜（`script.video2storyboard`）

流水线（generation-svc 编排，媒体步骤委托 media-svc）：

```
上传视频 → ① 镜头切分（PySceneDetect 内容检测；供应商有 shot-detection API 则优先）
        → ② 每镜头抽 3 帧（首/中/尾）+ 记录时长
        → ③ VLM 逐镜头描述（画面/人物/动作/场景/景别/运镜，受控词表同 B.2）
        → ④ 汇总为 shots[]，首帧图登记为 frameAssetId
```

产物支持产品要求的两种导出：单镜头"首帧图+描述"导出、全选批量导出；首帧图可直接作为后续 img2video 的起点（"让每个镜头都有明确的画面起点"）。

### B.4 分镜图与分镜视频的批量生成

- **单条生成**：`image.ref2img`，prompt = `globalStyle + 角色参考(characters.refAssetIds) + shot.description + 景别/运镜词 + aspectRatio`。
- **批量生成**：前端勾选 N 条 → 一次 API 提交 N 个任务（服务端按 asynq group 编排，共享限速配额），表格内逐条显示进度；单条失败单条重试，不整批重跑。
- **分镜图 → 视频**：`video.img2video`，首帧 = frameAssetId，prompt = 运镜词 + description + duration。**连贯模式（可选）**：勾选后第 N 镜取第 N-1 镜视频尾帧作为参考图传入，提升镜头衔接的空间连续性。
- 生成完成的 shot 可一键"发送到时间线"：按 no 顺序生成 timeline 节点的初始 composition。

### B.5 时间线：编排 Schema 与双渲染器契约

**核心设计：一份 composition JSON，两个渲染器。** 前端渲染器做近似实时预览（分段 `<video>` + Web Audio 增益），media-svc 的 FFmpeg 渲染器做权威输出。两个渲染器实现同一份 schema 语义，前端不做 FFmpeg 不支持的效果，保证"预览≈成片"。

```jsonc
{
  "canvasSize": { "w": 1920, "h": 1080 },
  "duration": 30.0,
  "tracks": [
    { "kind": "video", "clips": [
        { "assetId": "...", "start": 0, "in": 0.5, "out": 4.0,   // 时间线位置 / 素材裁剪
          "transform": { "x":0,"y":0,"scale":1 },                 // 布局(位置/尺寸)
          "speed": 1.0, "muted": false } ] },
    { "kind": "audio", "clips": [
        { "assetId": "...", "start": 0, "in": 0, "out": 30,
          "volume": 0.8, "fadeIn": 1.0, "fadeOut": 2.0 } ] },
    { "kind": "text", "clips": [
        { "content": "标题", "start": 1, "duration": 3, "style": {...}, "position": {...} } ] },
    { "kind": "subtitle", "srtAssetId": "...", "style": {...}, "burnIn": true }
  ],
  "export": { "format": "MP4_H264", "resolution": "1080p" }   // MP4(H.264) / WEBM(VP8)
}
```

- **字幕自动生成**：时间线内点击 → `script.subtitle_asr` 任务（音轨抽出 → ASR → SRT 资产）→ 回填 subtitle 轨，可编辑后选择烧录或软字幕。
- **FFmpeg 编排器**（media-svc `pipeline/`）：composition → 规划器（素材归一化：统一 fps/分辨率/像素格式的中间转码）→ `filter_complex` 生成器（trim/setpts/scale/overlay/concat + amix/afade/atempo + subtitles/drawtext）→ 执行 → 进度解析（`-progress` 输出 → 任务 progress 字段）。复杂度控制：先归一化再 concat，避免一条巨型 filter graph 不可调试。
- **导出**：自定义尺寸与时长校验（clip 越界裁剪）；产物登记资产 + 出现在"历史"，支持重复导出不同规格。

---

## Part C：业界 AI 视频创作工具的设计思路分析

业界产品可归为四种形态，Shotlab（即我们的蓝本）是 ②+③ 的混合体：

| 形态 | 代表 | 核心思路 | 优劣 |
|---|---|---|---|
| ① 单次生成器 | Midjourney、Runway/可灵/即梦的基础面板、海螺 | 一次输入一次输出，参数面板即产品；靠模型本身的质量取胜 | 上手最快；但多镜头叙事、迭代复用全靠用户自己在外部组织 |
| ② 节点画布工作流 | **Krea Nodes**（50+ 模型一张画布）、**Flora**、**Freepik Spaces**（画布上实时协作）、**Weavy/Wireflow**（生成-编辑-音频链式编排）、ComfyUI（技术流极致） | 生成过程显式化为图：每一步可换模型、可分叉、可回溯；"每个节点用最擅长该步骤的模型" | 自由度最高、复用性强；心智门槛高，需要用"预填面板"（A.2）降低连线规则复杂度 |
| ③ 剧本→分镜→成片（storyboard-first） | **LTX Studio**（脚本自动拆场景/镜头、Persistent Character、逐镜头改机位/换模型）、即梦"故事创作"（Seedream 关键帧 → Seedance 成片）、Sora 的 storyboard 时间轴 | 用影视工业的结构（场-镜-帧）来约束生成，先结构后画面；角色/场景作为跨镜头资产 | 叙事内容质量上限最高；灵活探索不如画布 |
| ④ 时间线中心 | 剪映/CapCut + AI、Descript | 传统 NLE 加 AI 素材生成入口 | 剪辑强；生成只是配菜 |

**跨形态的共性设计（我们必须吸收的六条）：**

1. **模型无关的编排层**：所有头部工具都把"模型"降级为节点/镜头上的一个下拉选项——护城河在编排与资产管理，不在某个模型。这正是 model-adapter + 统一任务模型的架构依据。
2. **Keyframe-first（图定画面、视频定运动）**：即梦（Seedream 出关键帧 → Seedance 动起来）、LTX（先 board 后 shot）都是先用便宜可控的图片环节锁定构图/风格/角色，再进昂贵的视频环节。首尾帧生视频是该思想的产品化。
3. **角色/场景作为一等资产**：LTX 的 Persistent Character Profiles/Element（定义一次，Shot 3 和 Shot 27 长得一样）、可灵 3.0 Omni 的人像特征库（从视频提取特征）、Seedance 2.5 的多模态参考（最多 50 个参考素材：造型/服装/动作/场景/声音）、Shotlab 资源广场的人物资产包（面部特写/九宫格/头部与全身三视图/人物板）。**一致性问题的业界共识解法 = 参考资产工程，而不是祈祷模型稳定。**
4. **Shot 是最小创作单元**：拆到镜头粒度才能局部重试、局部换模型、控制成本；整段直出（"一句话出 30 秒成片"）的产品都在向 shot 化回退。
5. **草稿-精修双轨**：fast/mini/lite 模型快速试错构图 → 满意后切 Pro 模型出成片；批量出图 + 宫格挑选。价格梯度是产品能力，不只是商业化设计。
6. **受控词表 + 结构化提示词**：景别/运镜/相机参数做成受控枚举（LTX 的 camera framing 建议、Shotlab 摄像机控制的相机/镜头/焦距/光圈），由系统拼 prompt，而不是让用户写小作文。

---

## Part D：如何做到"可高质量产出内容"

质量 = 模型上限 × **流程对结构的约束** × **一致性工程** × **迭代成本**。平台能改变的是后三项：

### D.1 结构先行：五层漏斗，每层可审查可返工

```
剧本 → 分镜表(文本) → 分镜图(关键帧) → 分镜视频(镜头) → 时间线(成片)
```

每层产物独立可编辑、可单独重做，错误在便宜的层被拦截（改一行分镜描述的成本 ≪ 重出一段视频）。产品上的对应：B.2 的"脚本确认态"、B.4 的单条重试、时间线的 shot 级替换。**这是质量方法论的骨架，其余手段都挂在这个漏斗上。**

### D.2 一致性工程（角色/场景/风格三个维度）

- **角色**：三视图工作流（Shotlab 教程 4.3.2 的"人物三视图提示词模板"产品化为一键动作）→ 角色参考资产挂在 storyboard.characters → 每个 shot 出图自动携带；优先路由到支持多参考/人像锁定的模型（香蕉 2.0、可灵 Omni、Seedance 2.5）。
- **场景**：场景空镜作为参考资产复用于同场景的所有 shot。
- **风格**：globalStyle 风格块 + 摄像机参数全局默认值，注入每个 shot 的 prompt；换风格 = 改一处重出全部。
- **镜头衔接**：连贯模式（前镜尾帧 → 后镜参考）+ 首尾帧生视频做精确转场。

### D.3 迭代成本工程（让"多试几次"变得便宜）

- 草稿-精修双轨（fast 模型试错 → Pro 出片）作为分镜出图面板的显式开关；
- 批量 + 宫格 + 多张出图默认开启，挑选而非赌单张；
- 生成参数完整留痕（历史创作引用可"带参重开"），好结果可复制其配方；
- 失败归因到人话（错误码 → "提示词涉敏，请修改"/"供应商繁忙已重试"），减少无效重试。

### D.4 提示词与参数的系统化

- 常用提示词模板库（内置专业模板 + 用户自定义，Shotlab 6.1.2 的产品化）；
- 摄像机控制、打光、景别、运镜全部结构化参数 → 系统拼 prompt（受控词表），用户不写小作文；
- LLM 提示词增强（可选按钮）：短输入 → 专业长 prompt，走 LLM 通道；
- 参数 lint：提交前校验"模型 × 分辨率 × 时长 × 参考图数量"的合法组合（模型目录里声明能力矩阵），把"生成必失败"消灭在提交前。

### D.5 平台侧质量兜底（Phase 2+ 预留）

- 模型目录的场景化标签与选型指引（"亚洲人像→Seedream""电影运镜→Seedance"——把模型评测沉淀成产品信息）；
- 自动质量初筛：VLM judge 对批量产物打分排序（人挑之前先机器排序）；
- 社区优质作品的"配方可见"（生成词与参数公开 + 一键引用）——让高质量方法在用户间复制，这是 Shotlab 作品展示区的深层价值。

---

## 附：本篇引用的行业参考

- 节点画布类：[Krea Nodes / Freepik Spaces / Flora 等节点式工具综述](https://www.wireflow.ai/blog/best-node-based-image-generation-tools-in-2026)、[创作者转向节点式工具的原因](https://toolfolio.io/productive-value/why-creators-are-switching-to-node-based-ai-tools)
- 分镜/影视工作流：[LTX Studio 功能（脚本拆镜/Persistent Character）](https://ltx.io/blog/top-ltx-studio-features)、[分镜技巧（blocking/连贯性/风格参考）](https://ltx.io/blog/storyboard-techniques)、[AI 短剧：突破在生产工作流](https://mcplato.com/en/blog/ai-short-drama-generation-tools-2026-production-workflow/)
- 国内双雄与一致性：[可灵 vs 即梦 2026 实测](https://www.eshowai.com/index.php/2026/06/18/%E5%8F%AF%E7%81%B5-vs-%E5%8D%B3%E6%A2%A6%EF%BC%9A%E5%9B%BD%E4%BA%A7-ai-%E8%A7%86%E9%A2%91%E5%8F%8C%E9%9B%84%E7%BB%88%E6%9E%81%E5%AF%B9%E5%86%B3%EF%BC%882026-%E5%B9%B4%E5%AE%9E%E6%B5%8B%EF%BC%89/)、[即梦全流程教程](https://www.aitoollab.cn/articles/jimeng-ai-complete-guide-2026/)
