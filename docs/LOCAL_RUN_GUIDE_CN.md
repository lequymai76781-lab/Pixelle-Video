# Pixelle-Video 本地运行指南（中文）

## 1. 项目简介

Pixelle-Video 是一个基于 Streamlit + ComfyUI 工作流的 AI 短视频生成项目。输入主题后，系统可自动完成文案、配图/视频、配音和合成输出。

当前阶段目标是：**先跑通原项目，不做重构、不接入外部业务系统**。

---

## 2. Windows 环境准备

建议使用 **Windows 10/11（64位）**。

### 必备组件

1. **Python 3.11+**（源码运行需要）
2. **uv**（Python 包管理与运行工具）
3. **ffmpeg**（视频合成依赖）
4. 可选：
   - **ComfyUI**（如果你走 selfhost 本地工作流）
   - **RunningHub 账号与 API Key**（如果你走 runninghub 云工作流）

> 如果你用项目发布页提供的 Windows 一键整合包，可以不手动安装 Python/uv/ffmpeg。

---

## 3. uv 安装方式

官方安装文档：<https://docs.astral.sh/uv/getting-started/installation/>

在 Windows PowerShell 可使用（示例）：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

安装后验证：

```powershell
uv --version
```

---

## 4. ffmpeg 安装方式

官方下载：<https://ffmpeg.org/download.html>

Windows 常规步骤：

1. 下载 ffmpeg 压缩包并解压。
2. 将解压后的 `bin` 目录加入系统 `PATH`。
3. 新开终端验证：

```powershell
ffmpeg -version
```

---

## 5. 启动命令（源码方式）

在仓库根目录执行：

```bash
uv run streamlit run web/app.py
```

本次仓库实测该命令可正常启动，默认访问：

- `http://localhost:8501`

---

## 6. API Key 配置位置

### 配置文件

项目提供了示例配置：

- `config.example.yaml`

实际运行时请复制为：

- `config.yaml`

示例（Windows PowerShell）：

```powershell
Copy-Item config.example.yaml config.yaml
```

然后在 `config.yaml` 中填写你的密钥与地址。**不要将真实 API Key 提交到 Git 仓库。**

---

## 7. DeepSeek / 通义千问 / OpenAI 配置方式

在 `config.yaml` 的 `llm` 段配置：

```yaml
llm:
  api_key: "你的密钥"
  base_url: "服务地址"
  model: "模型名"
```

常见可用组合（来自项目注释与预设）：

- **通义千问**
  - `base_url: https://dashscope.aliyuncs.com/compatible-mode/v1`
  - `model: qwen-max`（或其他 qwen 系列）
- **OpenAI**
  - `base_url: https://api.openai.com/v1`
  - `model: gpt-4o`（或你有权限的模型）
- **DeepSeek**
  - `base_url: https://api.deepseek.com`
  - `model: deepseek-chat`

填写后在 Web 配置页保存即可。

---

## 8. ComfyUI 配置方式

若使用 **selfhost** 工作流（本地推理）：

在 `config.yaml` 中配置 `comfyui`：

```yaml
comfyui:
  comfyui_url: http://127.0.0.1:8188
  comfyui_api_key: ""   # 可选
```

要求：

1. 本地 ComfyUI 必须先启动。
2. 对应工作流 JSON 需位于 `workflows/selfhost/`。
3. 图像/视频/TTS 工作流选择 selfhost 版本（例如 `selfhost/image_flux.json`）。

---

## 9. RunningHub 配置方式

若使用 **runninghub** 云端工作流（推荐无本地 GPU 场景）：

```yaml
comfyui:
  runninghub_api_key: "你的 RunningHub Key"
  runninghub_concurrent_limit: 1
```

可选高级配置：

- `runninghub_instance_type`（如 24G/48G，具体以平台可用值为准）

并在工作流上选择 `workflows/runninghub/` 下对应 JSON（例如 `runninghub/image_flux.json`、`runninghub/video_wan2.1_fusionx.json`）。

---

## 10. 输出视频保存位置

默认输出目录：

- `output/`

历史任务与媒体文件都在该目录下管理。

---

## 11. 常见报错与解决办法

### 报错 1：`uv: command not found`

- 原因：未安装 uv 或 PATH 未生效。
- 解决：按上文安装 uv，重开终端后执行 `uv --version`。

### 报错 2：`No module named streamlit`

- 原因：未通过 uv 环境运行，或依赖未安装完成。
- 解决：使用 `uv run streamlit run web/app.py`，让 uv 自动解析并安装依赖。

### 报错 3：`ffmpeg not found`

- 原因：系统未安装 ffmpeg 或 PATH 未配置。
- 解决：安装 ffmpeg 并确保 `ffmpeg -version` 可执行。

### 报错 4：LLM 请求失败（401/403）

- 原因：`llm.api_key` 错误、过期，或 `base_url/model` 不匹配。
- 解决：检查 `config.yaml` 的 `llm` 三元组（`api_key/base_url/model`）是否对应同一厂商。

### 报错 5：ComfyUI 连接失败

- 原因：`comfyui_url` 错误、ComfyUI 未启动、端口不通。
- 解决：先本地打开 `http://127.0.0.1:8188` 验证，再回到项目测试连接。

### 报错 6：RunningHub 工作流调用失败

- 原因：`runninghub_api_key` 未填、无权限、并发限制超限、工作流不匹配。
- 解决：检查 API Key、账号额度、并发设置，并确认所选 workflow 为 `workflows/runninghub/` 下有效文件。

### 报错 7：启动成功但生成失败

- 原因：通常是配置缺失（LLM / 图像 / 视频 / TTS 任一环节）。
- 解决：在 Web 侧边栏「系统配置」逐项补齐后再重试。

---

## 12. 最小可运行建议（先跑通）

如果你只是想先验证链路：

1. 启动命令先跑起来：`uv run streamlit run web/app.py`
2. LLM 先配置一个可用模型（通义千问 / DeepSeek / OpenAI 三选一）
3. 图像与视频优先选择 `runninghub` 工作流（避免本地 ComfyUI 环境复杂度）
4. 生成一个短主题视频做冒烟测试

这样可以在**不改动核心代码结构**的前提下，最快完成“项目可运行”目标。
