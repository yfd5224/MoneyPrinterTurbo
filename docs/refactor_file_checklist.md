# 音频转视频改造 —— 文件改动清单

> 场景：以 **WebUI（Streamlit）方式启动**。容器 / Docker 相关文件本轮先不动（仅末尾列出，标注「暂不改」）。
> 原则：**没用到的代码不删，只注释**；删包前必须先把对应的顶层 `import` 改成延迟导入或 try/except，否则一启动就崩。
> 配套设计文档见 `docs/audio_to_video_refactor_plan.md`，本文件只列「要改哪些文件 + 改什么」。

---

## 一、必须改（核心链路）

| # | 文件 | 改动要点 | 优先级 |
|---|------|----------|--------|
| 1 | `app/models/schema.py` | `VideoParams` 新增字段：`segment_mode`（句子/固定时长）、`min_clip_seconds` 等；TTS/BGM 相关字段保留但标注废弃。新增 ASR provider 相关字段（whisper/dashscope）。 | 高 |
| 2 | `app/services/task.py` | 重排 `start()` 流程：跳过 `generate_script`/`generate_terms` 的强制 LLM 生成；`generate_audio` 只走「自定义音频」分支；字幕强制走 whisper/dashscope；**新增按字幕时间线匹配画面**的逻辑；注释第 7 步 `upload_post` 跨平台发布。 | 高 |
| 3 | `app/services/subtitle.py` | whisper 本地识别保留（已是句子级带时间戳，基本不动）；**新增 DashScope（paraformer）在线识别**函数，按 `subtitle_provider` 切换。 | 高 |
| 4 | `app/services/material.py` | 保留 Pexels/Pixabay/Coverr 在线下载；**新增「按单句关键词下载/匹配一个素材」**的函数，供时间线拼接调用。 | 高 |
| 5 | `app/services/video.py` | **新增 `combine_videos_by_timeline()`**：按每句字幕的 `start/end` 决定每段画面时长（现有 `combine_videos` 只按音频总时长堆素材，不看时间戳，必须新增）。`generate_video`（烧字幕）保留。 | 高 |
| 6 | `app/services/llm.py` | 保留多 provider（你要求全保留）；**新增 / 复用「单句中文 → 英文画面关键词」**的函数（中文直接搜 Pexels 命中率低，需 LLM 提炼英文词）。`dashscope`、`google.generativeai` 已是函数内延迟导入，无需动；`openai` 顶层导入保留。 | 高 |

---

## 二、WebUI 入口改动

| # | 文件 | 改动要点 |
|---|------|----------|
| 7 | `webui/Main.py` | 1) **文案面板**（左栏 `Video Script Settings`，约 711-816 行）：注释「生成文案/关键词」按钮区，改为「上传音频」为主入口；`video_subject/script/terms` 保留但不再必填。<br>2) **音频面板**（`Audio Settings`，约 955-1316 行）：注释整个 TTS 服务器选择、试听、Azure/SiliconFlow/Gemini/MiMo/ElevenLabs key 区、语速/音量、BGM 区；只保留「Custom Audio File」上传控件（约 1267-1281 行）。<br>3) **字幕面板**（右栏，约 1318-1429 行）：保留；新增「字幕来源 whisper/dashscope」「拆分方式」选择。<br>4) **视频面板**（约 818-954 行）：保留 Pexels/Pixabay/Coverr/Local 源与 API key；`Clip Duration` 等改为时间线模式下的兜底值。<br>5) **生成按钮校验**（约 1534-1572 行）：把「文案和主题不能同时为空」改为「必须上传音频」；上传音频落盘逻辑保留。 |
| 8 | `webui/i18n/zh.json`（及 `en.json` 等其余 8 个语言文件） | 新增/调整文案 key：上传音频、字幕来源、拆分方式等。其余语言可只补 `zh.json`、`en.json`，剩下的回退到 key 即可。 |

---

## 三、API 入口改动（你要求 API 也保留）

| # | 文件 | 改动要点 |
|---|------|----------|
| 9 | `app/controllers/v1/video.py` | `/videos`、`/audio`、`/subtitle` 接口保留；确认 `TaskVideoRequest` 走新流程（音频为必填、文案可空）。BGM 上传接口 `/musics` 可标注废弃（暂留）。 |
| 10 | `app/models/schema.py`（请求模型部分） | `TaskVideoRequest` / `AudioRequest` 调整必填项：`custom_audio_file` 变为核心输入。（与 #1 同文件，一起改） |

---

## 四、依赖瘦身（requirements.txt / pyproject.toml）

> 你计划用 WebUI 启动；以下包**注释前必须先处理顶层 import**，否则启动崩溃。

| 包 | 能否注释 | 前置处理（关键！） |
|----|----------|--------------------|
| `edge_tts` | 可注释（不再用 TTS） | ⚠️ `app/services/voice.py` 顶层有 `import edge_tts` / `from edge_tts import SubMaker`，必须先改成延迟导入或 try/except，否则 task.py/Main.py 一 import voice 就崩。 |
| `azure-cognitiveservices-speech` | 可注释 | 确认 voice.py 中 azure 相关已是延迟导入再注释。 |
| `pydub` | 可注释（不做变速/混音） | 检查 voice.py / video.py 是否顶层引用。 |
| `redis` | 可注释（WebUI 不用任务队列） | `app/controllers/manager/redis_manager.py` 顶层 import redis；API 入口 `video.py` 会 import 它。若保留 API，需把 redis_manager 改成延迟导入，或 `enable_redis=False` 时不 import。 |
| `litellm` | 可注释（你保留其他 LLM 即可） | llm.py 中 `import litellm` 已是函数内延迟导入，直接注释包即可。 |
| `google.generativeai` / `dashscope` | **保留**（你要求全保留 LLM；dashscope 还要做在线 ASR） | 已是延迟导入，无需动。 |
| `openai` / `faster-whisper` / `moviepy` / `streamlit` / `fastapi` / `uvicorn` | **保留** | 核心依赖。 |

> 同步修改 `pyproject.toml`（项目主依赖在这里，`requirements.txt` 是 legacy 兼容）。两处保持一致。

---

## 五、需要确认是否处理的「次要文件」

| 文件 | 说明 | 建议 |
|------|------|------|
| `app/services/voice.py` | 现在同时负责 TTS + 音频时长 + 字幕生成。`get_audio_duration()` 仍要用（读自定义音频时长），所以**文件不能删**，只把 TTS 部分相关顶层 import 改延迟。 | 改（最小改动） |
| `app/services/upload_post.py` | 跨平台发布，新流程不用。 | 不动文件，只在 task.py 注释调用 |
| `app/controllers/manager/redis_manager.py` | 仅 API + redis 时用。 | 若注释 redis 包，需配套改延迟导入 |
| `app/services/utils/video_effects.py` | 转场特效，时间线拼接可复用。 | 保留 |
| `cli.py` | 命令行入口，走的也是 `tm.start`。 | 跟随 task.py 自动适配，可暂不单独改 |

---

## 六、文档改动

| 文件 | 改动要点 |
|------|----------|
| `docs/audio_to_video_refactor_plan.md` | 已存在（总体方案），无需重写。 |
| `docs/refactor_file_checklist.md` | 本文件。 |
| `README.md` / `README-en.md` / `README-ar.md` | 可选：把「输入主题自动生成」描述改为「上传音频生成」。优先级低，可最后改。 |
| `config.example.toml` | 把 `subtitle_provider` 默认值改为 `whisper`；新增 dashscope ASR 相关配置项注释；TTS/BGM 配置保留但标注废弃。 |

---

## 七、本轮先不改（容器 / CI）

- `Dockerfile`
- `docker-compose.yml` / `docker-compose.gpu.yml` / `docker-compose.release.yml`
- `.github/workflows/ci.yml` / `docker-ghcr.yml`
- `webui.sh` / `webui.bat`（启动脚本，逻辑不变，可不动）
- `test/` 下相关测试（改完核心逻辑后可能需要同步，但非启动必需）

---

## 八、改动顺序建议

1. `schema.py`（加字段）→ 2. `subtitle.py`（加 dashscope ASR）→ 3. `material.py`（按句下载）→ 4. `llm.py`（单句关键词）→ 5. `video.py`（时间线拼接）→ 6. `task.py`（串起新流程 + 注释 TTS/upload）→ 7. `webui/Main.py`（入口改造）→ 8. i18n 文案 → 9. 依赖瘦身（先改 voice.py 延迟导入，再注释包）→ 10. 文档/配置。
