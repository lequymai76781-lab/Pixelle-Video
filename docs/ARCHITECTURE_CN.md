# Pixelle-Video 架构分析（中文）

> 目标：在**不改动核心代码**的前提下，帮助你理解当前仓库结构，便于后续做“千问广告短片 / NBA 球员分析短视频”等定制。

---

## 1. 仓库结构总览

- `web/`：Streamlit 前端入口与页面组件。
- `pixelle_video/`：核心业务层（配置、LLM、媒体、TTS、工作流执行、视频合成、持久化）。
- `workflows/`：ComfyUI/RunningHub 的 JSON 工作流。
- `templates/`：视频画面模板（静态/图片/视频模板）。
- `output/`：生成结果与任务历史（运行后自动创建）。
- `docs/`：中英文文档与说明。

---

## 2. 核心运行流程（端到端）

### 2.1 Web 入口与用户输入

1. 启动命令：`uv run streamlit run web/app.py`。
2. `web/app.py` 建立多页面导航（Home/History）。
3. Home 页面在 `web/pages/1_🎬_Home.py`：渲染系统配置、流水线 tab，并把用户输入委托给 pipeline UI。

### 2.2 Core 初始化

`PixelleVideoCore`（`pixelle_video/service.py`）是总调度器：

- 初始化 `LLMService`、`TTSService`、`MediaService`、`VideoService`、`FrameProcessor`、`PersistenceService`。
- 注册三类 pipeline：`standard`、`custom`、`asset_based`。
- `generate_video` 默认包装到 pipeline 调用。

### 2.3 主题 -> 文案（LLM）

以 `StandardPipeline` 为例（默认主流程）：

1. `generate_content()`：根据模式生成文案或拆分固定脚本。
2. 关键调用在 `pixelle_video/utils/content_generators.py`：
   - `generate_narrations_from_topic()`：主题生成分镜文案。
   - `split_narration_script()`：固定脚本分段。
3. 底层通过 `LLMService.__call__()` 发起 OpenAI 协议兼容请求。

### 2.4 文案 -> 图像/视频素材（ComfyUI / RunningHub）

1. `plan_visuals()`：先生成每段文案对应的 image/video prompt。
2. `produce_assets()`：逐帧执行：
   - TTS 生成音频；
   - Media 生成图像或视频；
   - HTML 模板渲染合成单帧画面；
   - 生成分段视频。
3. ComfyUI 与 RunningHub 都是通过 workflow JSON + ComfyKit 执行。

### 2.5 语音合成（TTS）

- `pixelle_video/services/tts_service.py` 统一入口：
  - 本地 `edge-tts`；
  - 或走 ComfyUI/RunningHub 的 TTS workflow。

### 2.6 背景音乐（BGM）

- pipeline 后处理阶段会判断并叠加 BGM（如果配置了音乐）。
- `VideoService.add_bgm()` 负责把 BGM 混入最终视频。

### 2.7 ffmpeg 合成最终视频

- `VideoService.concat_videos()` 负责拼接分镜视频段。
- `VideoService.check_ffmpeg()` 在执行前检查系统是否安装 ffmpeg。

### 2.8 持久化与输出

- `PersistenceService(output_dir="output")` 管理任务目录与索引。
- 每个任务写入 `output/{task_id}/`，包含 `metadata.json`、`storyboard.json`、`final.mp4` 等。

---

## 3. 各功能对应主要文件与函数

### 3.1 启动与页面

- `web/app.py`
  - `main()`：注册并运行页面导航。
- `web/pages/1_🎬_Home.py`
  - `main()`：初始化会话、配置面板、pipeline 渲染。

### 3.2 核心编排

- `pixelle_video/service.py`
  - `class PixelleVideoCore`
  - `initialize()`：初始化服务与 pipeline。
  - `_create_generate_video_wrapper()`：对外统一生成接口。

### 3.3 Pipeline 主流程

- `pixelle_video/pipelines/linear.py`
  - `LinearVideoPipeline.__call__()`：模板方法（setup/content/visuals/assets/post/finalize）。
- `pixelle_video/pipelines/standard.py`
  - `setup_environment()`
  - `generate_content()`
  - `plan_visuals()`
  - `produce_assets()`
  - `post_production()`
  - `finalize()`

### 3.4 LLM

- `pixelle_video/services/llm_service.py`
  - `_create_client()`
  - `__call__()`
- `pixelle_video/utils/content_generators.py`
  - `generate_narrations_from_topic()`
  - `generate_title()`
  - `generate_image_prompts()`

### 3.5 图像/视频生成

- `pixelle_video/services/media.py`
  - `__call__()`：统一媒体生成入口（图片/视频）。
- `pixelle_video/services/comfy_base_service.py`
  - `_get_default_workflow()`
  - `_execute_workflow()`（内部 workflow 执行核心）

### 3.6 语音

- `pixelle_video/services/tts_service.py`
  - `__call__()`
  - `_generate_with_local_edge_tts()`
  - `_call_comfyui_workflow()`

### 3.7 分镜处理与合成

- `pixelle_video/services/frame_processor.py`
  - `__call__()`：逐帧处理主入口。
- `pixelle_video/services/video.py`
  - `check_ffmpeg()`
  - `concat_videos()`
  - `add_bgm()`

### 3.8 配置与持久化

- `pixelle_video/config/manager.py`
  - `update()` / `save()` / `reload()`
  - `get_llm_config()` / `get_comfyui_config()`
- `pixelle_video/services/persistence.py`
  - `save_task_metadata()`
  - `save_storyboard()`
  - `list_tasks()`

---

## 4. DeepSeek / 通义千问 / OpenAI / Ollama 配置入口

### 文件入口

1. `config.example.yaml` -> 复制成 `config.yaml`。
2. `llm` 段填写：

```yaml
llm:
  api_key: "..."
  base_url: "..."
  model: "..."
```

### 代码入口

- `pixelle_video/services/llm_service.py`：读取 `config_manager.config.llm`。
- `pixelle_video/llm_presets.py`：内置 Qwen / OpenAI / DeepSeek / Ollama 预设。

---

## 5. ComfyUI / RunningHub 工作流配置位置

### 5.1 配置文件

`config.yaml` 中的 `comfyui` 段：

- `comfyui_url`（selfhost）
- `comfyui_api_key`（可选）
- `runninghub_api_key`（runninghub 必填）
- `tts.default_workflow` / `image.default_workflow` / `video.default_workflow`

### 5.2 工作流文件目录

- 本地：`workflows/selfhost/*.json`
- 云端：`workflows/runninghub/*.json`

---

## 6. 最终视频保存位置

默认都落在：

- `output/{task_id}/final.mp4`

并伴随：

- `output/{task_id}/metadata.json`
- `output/{task_id}/storyboard.json`
- `output/.index.json`

---

## 7. 做“千问广告短片 / NBA 球员分析短视频”优先改哪些文件

> 当前阶段建议先做“最小可控定制”，避免动底层执行器。

### 第一优先级（建议先改）

1. **提示词层**（决定文案风格）
   - `pixelle_video/prompts/topic_narration.py`
   - `pixelle_video/prompts/content_narration.py`
   - `pixelle_video/prompts/image_generation.py`
   - `pixelle_video/prompts/video_generation.py`
2. **内容生成工具层**
   - `pixelle_video/utils/content_generators.py`
3. **工作流选择层（不改引擎，只换 workflow）**
   - `config.yaml` 里的 `image/video/tts default_workflow`
   - 对应 `workflows/runninghub/*.json` 或 `workflows/selfhost/*.json`
4. **模板层（视觉风格）**
   - `templates/` 下 `image_*.html` / `video_*.html`

### 第二优先级（谨慎）

- `pixelle_video/pipelines/standard.py`
  - 如果你要加入“广告分镜结构模板”“NBA分析固定段落结构”，可以在 `generate_content/plan_visuals` 小范围改。

---

## 8. 不要轻易改动的核心文件（高风险）

1. `pixelle_video/service.py`
   - 全局服务装配与 pipeline 注册中心。
2. `pixelle_video/pipelines/linear.py`
   - 全 pipeline 的模板方法骨架。
3. `pixelle_video/services/video.py`
   - ffmpeg 拼接与 BGM 混音核心。
4. `pixelle_video/services/frame_processor.py`
   - 每帧素材处理与片段生成关键路径。
5. `pixelle_video/services/comfy_base_service.py`
   - ComfyUI/RunningHub 统一执行抽象层。
6. `pixelle_video/config/schema.py`
   - 配置校验模型，改坏会导致配置加载失败。

建议策略：**优先改 prompts / workflow / template，最后才碰 pipeline 和 core。**

---

## 9. 结论（当前阶段）

你现在已经完成“项目可启动”，下一步做定制最稳妥的路线是：

1. 固定一个可用 LLM（如 Qwen）
2. 固定一组稳定 workflow（先 runninghub，再考虑 selfhost）
3. 先改 prompts + template 做风格化
4. 最后按需微调 `standard` pipeline

这样可以最大限度降低“改崩项目”的风险。
