# 音频驱动生成视频 —— 改造方案

> 目标：把现有「输入主题 → LLM 生成文案 → TTS 配音 → 匹配素材 → 合成」的流程，
> 改成「上传一段配音音频 → 按句子拆分 → 每句匹配一个画面 → 音频转字幕 → 合成输出视频」。
>
> 约束：
> - 音频只用用户上传的，不做 TTS、不做背景音乐、不做变速、不做音量调节。
> - 字幕用 **faster-whisper 本地识别**（带时间戳）。
> - 画面素材来源：**Pexels**（保留 Pixabay / Coverr 等可选搜索源的配置能力）。
> - 拆分粒度：**尽量一句话一个画面**。
> - 没用到的代码 **不删除，只注释**，目的是启动时少装一堆包。

---

## 1. 现有流程梳理（改造前）

入口：`webui/Main.py`（Streamlit）/ `app/router.py`（FastAPI）→ `app/services/task.py: start()`

`start()` 的 7 个步骤：

| 步骤 | 函数 | 说明 | 改造后 |
|------|------|------|--------|
| 1. 生成文案 | `generate_script` | 没有 `video_script` 时调用 `llm.generate_script` | **删/注释**（改为从音频识别得到文本） |
| 2. 生成关键词 | `generate_terms` | 调用 `llm.generate_terms` 生成搜索词 | **改造**（改成按字幕句子逐句生成关键词） |
| 3. 生成音频 | `generate_audio` | 有 `custom_audio_file` 用上传音频，否则 TTS | **简化**（只保留上传音频分支） |
| 4. 生成字幕 | `generate_subtitle` | `edge` 用 TTS 时间戳，否则 `whisper` | **简化**（强制走 whisper） |
| 5. 获取素材 | `get_video_materials` | 本地素材 or 在线下载 | **改造**（改成逐句匹配下载） |
| 6. 合成视频 | `generate_final_videos` → `video.combine_videos` | 按音频总时长堆素材 | **改造**（改成按字幕时间轴逐句放画面） |
| 7. 跨平台发布 | `upload_post` | 自动上传到 YouTube/TikTok 等 | **删/注释**（不需要） |

### 关键发现：当前画面和字幕是**不对齐**的

`combine_videos`（`app/services/video.py:535`）现在的逻辑是：把所有下载的素材切成小片段，
然后**按音频总时长**一段段拼接，直到盖住音频长度为止。它**完全不看字幕时间戳**，
所以"一句话对应一个画面"目前是做不到的 —— 这是本次改造的核心新增逻辑。

### 当前画面匹配（搜索词）怎么来的

`llm.generate_terms(video_subject, video_script, amount)` 用大模型从整篇文案里提炼出
一组英文搜索词，再丢给 Pexels 搜索。也就是说**当前完全依赖 LLM**，而且是"整篇一组词"，
不是"逐句一组词"。

---

## 2. 改造后目标流程

```
上传音频 (audio.mp3/wav)
   │
   ▼
[whisper 本地识别] → 句子级字幕 (text + start + end)   ← subtitle.create()
   │
   ▼
[逐句生成画面关键词]
   ├─ 方案 A：LLM 逐句提炼英文关键词（保留，需要 LLM 包 + key）
   └─ 方案 B：直接用字幕文本/分词做关键词（默认，可关掉所有 LLM 包）
   │
   ▼
[逐句搜索 + 下载素材] (Pexels / Pixabay / Coverr)   ← material.py
   │
   ▼
[按字幕时间轴逐句拼接画面] 第 i 句字幕时长 = 第 i 个画面时长   ← video.combine_videos 改造
   │
   ▼
[烧录字幕] generate_video()（保留）
   │
   ▼
输出 final-1.mp4
```

新的 `start()` 步骤简化为：

1. 解析上传音频，拿到时长。
2. whisper 识别音频 → 句子级字幕（带时间戳）。
3. 逐句生成画面关键词（LLM 或直接文本）。
4. 逐句搜索 + 下载素材。
5. 按字幕时间轴逐句拼接画面 → 合成。
6. 烧录字幕、输出视频。

---

## 3. 逐文件改造清单

### 3.1 `app/models/schema.py`

`VideoParams` 新增 / 调整字段（建议保留旧字段不删，只是用不到）：

```python
# 新增：明确这是“音频驱动”模式
audio_driven: bool = True
# custom_audio_file 已存在，继续用它接收上传音频路径

# 用不到但先保留（不删），避免别处引用报错：
# video_subject / video_script / video_terms
# voice_name / voice_rate / voice_volume
# bgm_type / bgm_file / bgm_volume
```

> 说明：`video_subject` 目前是必填（`video_subject: str`）。音频驱动模式下没有主题，
> 建议改成 `video_subject: Optional[str] = ""`，否则构造 `VideoParams` 会报错。

新增一个"逐句关键词来源"的开关（也可以放 config.toml）：

```python
# "llm" 用大模型逐句提炼英文关键词；"text" 直接用字幕文本/分词
term_source: Optional[str] = "text"
```

### 3.2 `app/services/task.py`

- `start()`：
  - **注释掉** 步骤 1 `generate_script` 调用（保留函数定义）。
  - **注释掉** 步骤 2 中"整篇生成关键词"的逻辑；改为在拿到字幕后逐句生成。
  - **注释掉** 步骤 7 `upload_post` 整段。
  - `generate_audio`：保留 `custom_audio_file` 分支，**注释掉** TTS 分支（`voice.tts(...)`）。
    没有上传音频时直接判失败。
  - `generate_subtitle`：**注释掉** `edge` 分支，强制走 `subtitle.create(audio_file, subtitle_file)`。
    `subtitle.correct(...)` 依赖原始文案做纠错，音频驱动下没有"标准文案"，
    **建议注释掉 correct**，直接用 whisper 原始识别结果。
- 新增函数（建议）：
  - `build_segment_terms(subtitles, params)`：输入字幕列表，输出 `[(start, end, [terms...]), ...]`。
    - `term_source == "llm"`：对每句调用 `llm.generate_terms`（或新写一个逐句版）。
    - `term_source == "text"`：用字幕文本本身/`jieba` 分词当关键词（见 §5 中文问题）。
  - `download_materials_per_segment(...)`：对每句的关键词搜索并下载 1 个素材，
    返回**按句顺序**的素材路径列表（顺序很重要）。

### 3.3 `app/services/subtitle.py`

- 基本不用改，`create()` 已经是 whisper 句子级识别（按标点断句，带时间戳）。
- `correct()` 在音频驱动模式下不调用即可（不用删）。
- 需要把每条字幕的 `start/end` 暴露给 task.py 用来对齐画面 —— 
  现在写进 `.srt` 文件，再用 `file_to_subtitles()` 读回来即可（已有该函数）。

### 3.4 `app/services/material.py`

- 搜索源逻辑**保留**（Pexels / Pixabay / Coverr 都留着，由 `video_source` 切换）。
- 新增 `download_one_video(search_term, ...)`：只下载该关键词的 **1 个**合适素材，
  供"逐句匹配"调用。现有 `download_videos` 是"按总时长批量下载"，保留即可。

### 3.5 `app/services/video.py` —— 核心改造

`combine_videos` 现在不看时间戳。新增一个对齐版本（建议新函数，旧的保留注释或留作 fallback）：

```python
def combine_videos_by_timeline(
    combined_video_path,
    segment_clips,   # [(start, end, video_path), ...] 按句顺序
    audio_file,
    video_aspect,
    ...
):
    # 对每一句：取该句对应素材，裁剪/循环到 (end - start) 这么长
    # 缩放 + letterbox 到目标分辨率（复用现有 resize 逻辑）
    # 按顺序拼接 → 总时长 == 音频时长
```

要点：
- 每个画面时长 = 该句字幕 `end - start`，**不再**按"凑满音频总时长"来堆。
- 素材比该句短就循环，比该句长就裁剪。
- `generate_video()`（烧字幕）保留不动。

### 3.6 入口 `webui/Main.py` / `app/router.py`

- Streamlit 界面：
  - **注释掉** 主题输入、文案生成、配音(voice)、背景音乐(bgm) 相关 UI。
  - 保留/强化 **音频上传** 控件，把路径塞进 `custom_audio_file`。
  - 保留 视频比例、字幕样式、`video_source`(Pexels)、`term_source` 等控件。
- FastAPI：保留 `/video` 任务接口，确保能接收 `custom_audio_file`。

---

## 4. 依赖瘦身（`requirements.txt` / `pyproject.toml`）

> 原则：用不到的**注释掉**，启动时不安装。下面是按本方案的判断，最终以你确认为准。

| 包 | 用途 | 处理 |
|----|------|------|
| `moviepy` | 视频合成 | **保留** |
| `faster-whisper` | 本地字幕识别 | **保留** |
| `fastapi` / `uvicorn` | API 服务 | 保留（用 API 才需要） |
| `streamlit` | Web 界面 | 保留（用界面才需要） |
| `requests` / `socksio` | 素材下载 | **保留** |
| `loguru` / `pyyaml` | 日志/配置 | **保留** |
| `python-multipart` | 上传音频 | **保留** |
| `edge_tts` | TTS 配音 | **注释**（不配音） |
| `azure-cognitiveservices-speech` | Azure TTS | **注释** |
| `pydub` | 音频处理(BGM/拼接) | **注释**（不处理音频，但需确认 §5） |
| `openai` | LLM | term_source=llm 才需要，否则**注释** |
| `google.generativeai` | Gemini | 同上，默认**注释** |
| `dashscope` | 通义 | 同上，默认**注释** |
| `litellm` | LLM 聚合 | 同上，默认**注释** |
| `redis` | 任务队列状态 | 看部署模式，单机内存模式可**注释** |

> 注意：直接注释 `requirements.txt` 不够，代码里 `import openai` 等会在导入时报错。
> 配套要做的是把 `app/services/llm.py`、`voice.py`、`upload_post.py` 里对这些包的
> **顶层 import 改成「延迟导入」或 try/except**，否则注释了包但代码一 import 就崩。
> 当前 `subtitle.py` 已经用了 `try: from faster_whisper import ... except ImportError:` 这种写法，
> 可以照抄这个模式。

---

## 5. 需要你拍板的几个点（重要）

### (1) 中文字幕 → Pexels 搜索的语言问题 ⚠️
Pexels 主要按**英文**关键词搜索。如果 `term_source="text"` 直接拿中文字幕去搜，
大概率搜不到合适素材。可选：
- **A：保留 LLM**，让它把中文句子翻译+提炼成英文画面关键词（效果最好，但要装 LLM 包 + 配 key）。
- **B：不用 LLM**，但接一个轻量翻译（如本地词典/小翻译接口），或者你的音频本身是英文。
- **C：先用 text 跑通流程，画面质量后面再优化。**

> 你倾向哪种？这直接决定 LLM 那几个包到底注释不注释。

### (2) whisper 模型大小
`config.toml` 里 `whisper.model_size`（默认 `large-v3`，约 3GB）。
本地识别精度高但下载慢、占内存。要不要默认改成 `medium` 或 `small` 平衡速度？

### (3) "一句话一个画面" 太碎怎么办
whisper 按标点断句，有些句子可能只有 1~2 秒，画面切太快会晃。
是否需要一个"最短画面时长"（比如 < 2 秒的相邻句子合并成一个画面）？

### (4) 素材搜不到时的兜底
某一句关键词在 Pexels 搜不到素材时，怎么办？
- 用上一个画面延续 / 用一个默认空镜 / 用纯色背景？

### (5) `pydub` 是否真的能去掉
合成时如果完全不碰音频（直接把上传音频塞进视频轨），`pydub` 可以注释。
但要确认 `video.py` 合成那段没有用 pydub 做音频处理。我会在实现时核对。

---

## 6. 建议的改动顺序（你自己改的话）

1. **先改 `schema.py`**：`video_subject` 改可选，加 `term_source`。
2. **改 `task.py: start()`**：注释 script/terms/tts/upload，串起新流程骨架。
3. **改 `generate_audio` / `generate_subtitle`**：只走上传音频 + whisper。
4. **新增逐句关键词 + 逐句下载**（material.py + task.py）。
5. **新增 `combine_videos_by_timeline`**（video.py），按时间轴对齐画面。
6. **改入口 UI**（webui/Main.py），只留音频上传 + 必要参数。
7. **最后再动 `requirements.txt`**，配合把 llm/voice/upload 的顶层 import 改成延迟/try-except。

> 先保证流程跑通（text 关键词 + 不删包），再做依赖瘦身，风险最低。
```
