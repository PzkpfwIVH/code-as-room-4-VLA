# Code-as-Room 管线 Stage 功能详解

> 本文档基于 `run_pipeline.py`、`README.md` 及各 Stage 模块源码分析编写，描述每个 Stage 的功能、输入/输出和涉及的核心模块。

---

## 管线总览

```
Stage 0   Scene Classification        — 场景类型分类（实验室/住宅/办公等）
Stage 1   Spatial Semantic Analysis   — 空间语义分析（区域划分 + 物体清单）
Stage 2   Scene Graph Construction    — 场景图构建（物体关系图）
Stage 3   Blender Code Generation     — Blender 布局代码生成 + 迭代渲染比对
  └─ (可选) Rotation Correction      — 物体旋转角度矫正
Stage 4   Small-object Insertion      — 墙面装饰 + 小型物体占位插入
  └─ Arrow Cleanup                   — 方向箭头清除
Stage 5   Object Description          — 物体描述生成（外观/材质/颜色）
Stage 6   Detailed Geometry           — 每个物体的精细复合几何体生成
Stage 7   Surface Small-objects       — 基于支撑面的小型物体摆放
Stage 8   Small-object Descriptions   — (可选) 小物体描述
Stage 9   Small-object Geometry       — (可选) 小物体精细几何体
Stage 10  Material Generation         — 每部件 PBR 材质参数生成
Stage 11  Texture Generation          — 真实纹理图生成 + 注入代码
Stage 12  Render Script               — 光照 + 渲染设置
```

---

## Stage 0：Scene Classification（场景分类）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/scene_classifier.py` |
| **输入** | 参考图片 (top-down 视图)，可选文件名 |
| **输出** | `scene_type` (lab/residential/office/industrial/retail/other)，存入 Memory (`stage="meta"`) |
| **LLM 调用** | 仅 heuristic 未命中时调用一次 VLM |

**功能**：
- **混合策略**（heuristic 优先，LLM 兜底）：
  1. 先从文件名关键词判断（如含 `lab`/`bedroom`/`office` 等）
  2. 命中则直接返回，未命中则调用 VLM 分类
  3. 任何失败 fallback 为 `"other"`，不阻塞管线
- 下游 Stage 1/3 根据 `scene_type` 加载对应的 prompt addendum（如实验室场景加载 `Stage1_task_lab_addendum` / `Stage3_lab_addendum`）

---

## Stage 1：Spatial Semantic Analysis（空间语义分析）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage1/run_stage1.py` |
| **输入** | 参考图片 |
| **输出** | `stage1_output.json` — 解耦区域 (`decoupled_zones`) + 物体层级 (`object_hierarchy`) + 建筑特征 (`architectural_features`) |
| **LLM 调用** | VLM（含图片），最多 5 次迭代重试 |
| **验证器** | `agent_utils/validators.py` — JSON Schema 验证 |

**功能**：
- 分析 top-down 房间图片，识别：
  - **解耦区域** (zones)：如"休息区"、"主卧区"、"步入式衣帽间"
  - **物体层级**：每个区域内的 major/minor 物体，含父物体关系
  - **建筑特征**：墙体（边界墙/内隔墙）、窗户、门、门洞等
- 内置**场景尺度自动修正** (`_compute_scale_audit`)：检测家具占地密度，若 LLM 低估房间尺寸则自动放大
- 支持场景路由：读取 Stage 0 的 `scene_type`，拼接对应的 prompt addendum

---

## Stage 2：Scene Graph Construction（场景图构建）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage2/run_stage2.py` |
| **输入** | Stage 1 JSON + 参考图片 |
| **输出** | `stage2_output.json` — 节点 (nodes) + 关系边 (edges) + 区域图 (zone_graphs) + 全局图 (global_graph)；`stage2_minor_objects.json` (minor 物体附属) |
| **LLM 调用** | VLM，最多 5 次迭代 |

**功能**：
- **节点骨架自动派生**：从 Stage 1 的 object_hierarchy 代码生成节点结构（major + architecture）
- LLM 仅填充 `visual_attributes`（尺寸/颜色/材质描述）和 `geometry_hint`（2D 归一化坐标）
- **关系边**：LLM 生成前向边 (`left_of`/`right_of`/`against_wall`/`in_corner` 等)，代码自动生成逆向边
- v3.2 特性：**虚拟角落锚点** (`corner_NE/NW/SE/SW`) 帮助定位；自动推导 `against_wall` 和 `in_corner` 关系
- Minor 物体剥离到 sidecar 文件，不进入主图

---

## Stage 3：Blender Code Generation（Blender 代码生成）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage3/run_stage3.py`（主 Runner）、`agent_utils/stage3/pipeline.py`（备选 Pipeline） |
| **子模块** | `code_gen_agent.py` (代码生成)、`analyze_agent.py` (差异分析)、`fix_agent.py` (代码修复)、`code_patcher.py` (规则补丁)、`validator.py` (语法验证)、`core.py` (LLMClient/PromptManager) |
| **输入** | Stage 1 JSON + Stage 2 JSON + 参考图片 |
| **输出** | `stage3_output.py` — 可执行的 Blender Python 布局代码 |
| **LLM 调用** | VLM 生成初始代码；每轮迭代调用 VLM 分析差异 + 修复代码 |
| **外部依赖** | **Blender** (`blender --background --python` 渲染 top-down 视图) |

**功能**：
- 这是管线的**核心阶段**，将语义分析转化为可执行的 Blender 场景
- **迭代精炼循环** (默认最多 5 轮)：
  1. LLM 生成初始 Blender Python 布局代码
  2. Blender 后台渲染 top-down 视图
  3. 提取 live scene 中的家具物体 → `_layout.json`（比正则解析源码更可靠）
  4. VLM 对比渲染图与参考图，分析差异（缺失/位置错误/缩放问题）
  5. LLM 修复代码（Code Critic：自动尝试修复语法错误/渲染错误）
  6. 直至 target score ≥ 85% 或无新修正建议
- 支持场景路由（lab addendum 等）
- **旋转矫正** (Sub-stage，需 `--enable-rotation`)：迭代矫正物体朝向

---

## Stage 4：Small-object Insertion（小型物体插入）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage4/run_stage4.py` |
| **输入** | Stage 1/2/3 输出 + 参考图片 |
| **输出** | `stage4_output.py`（含墙面装饰 + minor 占位）|
| **LLM 调用** | 可选 VLM（无 wall-mounted 物体时跳过） |

**功能**：
- **Phase A**：墙面装饰 (wall-mounted objects) — 从 Stage 1 提取墙上物体，LLM 生成放置代码
- **Phase B**：Minor 占位符 — 从 Stage 2 sidecar 选择显著 minor 物体，插入简单几何占位
- **Arrow Cleanup** (`agent_utils/stage_clean_arrows.py`)：清除 LLM 生成的方向箭头标记

---

## Stage 5：Object Description（物体描述生成）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage5_describe.py` |
| **输入** | Stage 4 代码 + 参考图片 |
| **输出** | `describe_output.json`（每个物体的类型/外观/材质/颜色描述 + 房间风格）|
| **LLM 调用** | VLM |

**功能**：
- **Part A**：正则解析 Stage 4 代码中的 `create_box`/`create_cylinder` 调用，提取物体名称/位置/尺寸
- **Part B**：VLM 为每个物体生成详细描述（`object_type`、`appearance`、`material`、`color`）
- **Part C**：分析整体房间风格（如 Modern Minimalist）、色调和氛围
- 输出 orphan 报告（Stage 1 有但代码中未找到的物体）

---

## Stage 6：Detailed Geometry（精细几何体生成）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage6_geometry.py` |
| **输入** | Stage 5 描述 JSON |
| **输出** | `geometry_output.py` + `geometry_progress.json`（每个物体的多部件复合几何定义）|
| **LLM 调用** | 每个物体一次 LLM 调用（支持并行，`--parallel` 控制 worker 数） |

**功能**：
- 将 Stage 4 的简单 `create_box`/`create_cylinder` 替换为 `create_detailed_object()` — 最多 5 个几何部件的复合体
- **增量生成**：每生成一个物体立即保存，支持断点续传 (`--resume`)
- 保持物体中心对齐、前方面向 -Y 方向
- 支持按名称/索引生成指定物体

---

## Stage 7：Surface Small-object Placement（支撑面小物体摆放）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage7_small_objects.py` |
| **输入** | Stage 6 geometry JSON + 参考图片 |
| **输出** | `small_objects.json` + 更新后的 Blender 代码 |
| **LLM 调用** | 每个父物体一次 VLM 调用 |

**功能**：
1. **平面检测** (rule-based)：遍历 Stage 6 的 `DETAILED_GEOMETRY`，检测可摆放平面（桌面、柜顶、开放搁板、座椅面）
2. **LLM 摆放** (image-grounded)：VLM 决定每个平面上放哪些小物体（书籍、水杯、花瓶等），输出 plane-local UV 坐标和尺寸
3. **代码生成**：生成 `create_box`/`create_cylinder` 调用追加到 geometry 代码末尾

---

## Stage 8：Small-object Descriptions（小物体描述，可选）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage8_small_describe.py` |
| **启用条件** | `--detail-small-objects` |
| **输入** | Stage 7 `small_objects.json` + 参考图片 |
| **输出** | 每个小物体的描述记录 (object_type/appearance/material/color/part_hierarchy_hint) |
| **LLM 调用** | VLM 批量调用（`--small-describe-batch-size` 控制每批物体数） |

---

## Stage 9：Small-object Geometry（小物体精细几何，可选）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage9_small_geometry.py` |
| **启用条件** | `--detail-small-objects` |
| **输入** | Stage 8 describe JSON + Stage 7 Blender 代码 |
| **输出** | 将 Stage 7 的简单 `create_box`/`create_cylinder` 小物体替换为 `create_detailed_object_small()`（≤5 个几何部件） |
| **LLM 调用** | 每个小物体一次 LLM 调用（支持并行 + 重试） |

---

## Stage 10：Material Generation（材质生成）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage10_material.py` |
| **输入** | Stage 6/7 几何代码 + 参考图片 |
| **输出** | `material_output.py`（每部件 PBR 材质参数）+ `material_config.json` |
| **LLM 调用** | VLM 批量调用（scene-palette 预传递 + 并行 per-part 批次） |

**功能**：
- **Option-C 批处理架构**：
  - Pass 1：LLM 分析场景整体调色板
  - Pass 2：并行批处理每个物体的每个部件（`--material-batch-size` 控制批量大小）
- 生成标准 PBR 材质参数：base color、roughness、metallic、specular 等
- 同时生成增强的地板和墙体材质

---

## Stage 11：Texture Generation（真实纹理生成）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage11_texture.py` |
| **输入** | Stage 10 material 代码 + 材质配置 |
| **输出** | `texture_output.py`（含 `ShaderNodeTexImage` 纹理加载）+ `images/`（PNG 纹理）+ `texture_manifest.json` |
| **外部 API** | 图像生成端点（如 Gemini `generateContent` API，即 "nanobanana"） |

**功能**：
- 生成三类真实纹理 PNG：
  - `floor.png` — 地板纹理
  - `wall.png` — 墙面纹理
  - `art_<id>.png` — 墙面装饰画（每个 detected wall art 一张）
- 重写 Blender 代码：将 Stage 10 的程序化材质替换为 `ShaderNodeTexImage` 纹理加载
- 支持 `--wall-intensity` 参数 (subtle/bold/mural_like)

---

## Stage 12：Render Script（渲染脚本生成）

| 项目 | 内容 |
|------|------|
| **模块文件** | `agent_utils/stage12_render.py` |
| **输入** | Stage 10/11 代码 + 参考图片 |
| **输出** | `render_output.py` — 含 `setup_lighting_and_render()` 的最终 Blender 脚本 |
| **LLM 调用** | VLM 分析灯光（仅用于灯光分析部分） |

**功能**：
- **灯光分析**：VLM 分析参考图的光照方向、强度和色温
- **确定性灯光构建器**：默认使用 Sky+Portal+Sun+AgX 风格灯光（deterministic-by-default）
- 注入 `setup_lighting_and_render()` 函数，配置：相机、灯光、Cycles 渲染引擎、分辨率、采样数等
- Arrow Cleanup 再次执行
- 这是管线的**最终输出**，可直接用 Blender 渲染：

```bash
blender --python stage12_render/render_output.py
```

---

## 支撑模块

### Memory（记忆系统）

| 文件 | 功能 |
|------|------|
| `agent_utils/memory.py` | JSONL 格式的持久化记忆，支持按 stage/type 查询、add/clear 操作 |

每个 Stage 通过 `memory.add(stage="stageN", type="result", content=...)` 写入，下游 Stage 通过 `memory.get_latest(stage="stageN", type="result")` 读取。每个 run 目录有独立的 `agent_memory.jsonl`。

### LLMClient / PromptManager

| 文件 | 功能 |
|------|------|
| `agent_utils/stage3/core.py` | LLMClient（langchain_openai ChatOpenAI 封装）、PromptManager（agent_prompt 目录下 prompt 文件的加载/缓存） |

### Vision Fallback（视觉模型自动切换）

| 文件 | 功能 |
|------|------|
| `run_pipeline.py` (`_patch_chatopenai_invoke`) | Monkey-patch ChatOpenAI.invoke，当文本模型不支持视觉输入时自动切换到 Vision 模型 |

配置方式：`.env` 中设置 `SCENEGEN_VISION_MODEL` / `SCENEGEN_VISION_BASE_URL` / `SCENEGEN_VISION_API_KEY`。

### 辅助工具

| 文件 | 功能 |
|------|------|
| `agent_utils/image_utils.py` | 图片压缩、尺寸获取、管线预处理 |
| `agent_utils/stage_clean_arrows.py` | 清除 LLM 生成的 Blender 方向箭头代码 |
| `agent_utils/stage3_rotation.py` | Stage 3 子阶段：物体旋转角度迭代矫正 |
| `agent_utils/stage3/composite_helpers.py` | Stage 3 后处理：展开 `create_double_deck_bench` 等复合辅助函数为原始 `create_box` 调用 |
| `agent_utils/validators.py` | Stage 1/2 JSON Schema 验证器 |
| `agent_utils/scene_classifier.py` | Stage 0 场景分类器 |
| `agent_utils/prompt_manager.py` | Prompt 文件管理 |
| `agent_utils/blender_code_syntax_fix.py` | Blender 代码语法修复工具 |

### Prompt 模板目录

| 目录 | 内容 |
|------|------|
| `agent_prompt/` | 各 Stage 的 System Prompt + Fix Template + Lab/Residential Addendum |

### 批量运行

| 文件 | 功能 |
|------|------|
| `batch_run_pipeline.py` | 对文件夹内所有图片批量运行 `run_pipeline.py`，支持并行和自定义输出组织 |
| `IncrementalLayoutEngine.py` | 增量布局引擎 |
| `image_prompt_gen/` | 合成 top-down 房间图片的生成器（prompt 生成 + 图片生成） |

---

## LLM/VLM 使用总览

| Stage | 调用次数 | 需要视觉？ | 说明 |
|-------|---------|-----------|------|
| **Stage 0** | 0-1 次 | ✅ 是 | heuristic 命中时跳过 |
| **Stage 1** | 1-5 次 | ✅ 是 | 图片空间分析 |
| **Stage 2** | 1-5 次 | ✅ 是 | 场景图构建 |
| **Stage 3** | 1 + 2N 次 | ✅ 是 | 生成 1 次 + 每轮分析/修复各 1 次 |
| **Stage 4** | 0-1 次 | 可选 | 无 wall-mounted 物体时跳过 |
| **Stage 5** | 1 次 | ✅ 是 | 物体描述 + 房间风格 |
| **Stage 6** | N 次 | ❌ 否 (text) | 每个物体 1 次并行调用 |
| **Stage 7** | M 次 | ✅ 是 | 每个父物体 1 次 |
| **Stage 8** | batch 次 | ✅ 是 | 批量小物体描述 |
| **Stage 9** | K 次 | ❌ 否 (text) | 每个小物体 1 次 |
| **Stage 10** | 1 + batch 次 | ✅ 是 | 调色板 + 批量材质 |
| **Stage 11** | N/A | 图像生成 API | 不是 LLM，是图片生成端点 |
| **Stage 12** | 1 次 | ✅ 是 | 灯光分析 |
