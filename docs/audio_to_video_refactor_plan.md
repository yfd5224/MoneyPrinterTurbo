# 音频 → 视频 改造方案（基于 MoneyPrinterTurbo）

> 目标：把入口从「主题文案自动生成」改成「上传一段中文配音音频」，
> 流程变为：**上传音频 → 按句拆分 → 每句匹配画面 → 音频转字幕 → 合成视频**。
> 不要 TTS、不要背景音乐、不要变速、不要跨平台发布。
> 用不到的代码**不删，只注释**；目标之一是启动时少装一堆包。

## 最终确认结论（你已拍板）

- **LLM 提供商：全部保留**（用来把中文句子提炼成英文画面关键词，保证画面质量）。
- **字幕识别：whisper 本地 + DashScope 在线 两种都保留**，界面/配置可切换。
  - whisper 默认模型 `large-v3`（中文最准，CPU 慢可改 medium）。
  - DashScope 在线用阿里 Paraformer 语音识别（更快，依赖网络 + key）。
- **入口：Streamlit 界面 + FastAPI 接口 两个都改**。

---

## 一、新流程总览

```
上传音频(.mp3/.wav)
   │
   ▼
[1] 语音识别（whisper 本地 / DashScope 在线，二选一）→ 句子级字幕(带时间戳 start/end)
   │
   ▼
[2] 遍历每句字幕文本 → LLM 把「中文句子」提炼成「英文画面关键词」
   │
   ▼
[3] 每句按关键词去 Pexels(可选 Pixabay/Coverr) 搜索并下载 1 个画面素材
   │
   ▼
[4] 按字幕时间轴拼接：第 i 句的画面时长 = 该句 end - start（一句一画面）
   │
   ▼
[5] 叠加字幕 + 挂上原始音频 → 输出最终视频
```

与原流程对比（原：生成文案 → 生成关键词 → TTS 配音 → 字幕 → 下载素材 → 合成 → 发布），
本次**砍掉** generate_script / TTS / 背景音乐 / 跨平台发布；**新增** “按句匹配画面 + 按时间轴拼接 + 在线 ASR”。

---

## 二、关键现状分析（决定改造难度）

逐文件读过代码后，有几个必须知道的点：

1. **`task.py` 已支持 `custom_audio_file`**：`generate_audio()` 的 else 分支直接用上传音频，
   并返回 `sub_maker=None`。所以“用上传音频”不用从零做，主要是**改编排顺序**。

2. **`subtitle.py` 的 whisper 已是“句子级 + 时间戳”**（按标点断句，输出 `start_time/end_time`），
   非常契合“一句一画面”。**当前只有本地 whisper，没有在线识别**——在线识别要新增。

3. **最大的改造点在 `video.py` 的 `combine_videos()`**：
   它现在**按音频总时长堆素材**（`required_video_duration = 音频时长`，素材循环填满），
   **完全不看字幕时间戳**。所以“画面跟着句子切换”**目前实现不了**，
   必须**新增按时间轴拼接的函数**（`combine_videos_by_timeline`）。

4. **`material.py`** 已有 Pexels/Pixabay/Coverr 三个搜索源 + 顺序匹配逻辑，
   但是“一批关键词整体下载”，需新增“**按句**，每句下 1 个”的函数。

5. **中文直接搜 Pexels 几乎搜不到**（Pexels 主要吃英文），所以**保留 LLM 做中文→英文关键词**
   是画面质量关键（已按你的要求保留全部 LLM 提供商）。

---

## 三、逐文件改造清单

### 1. `app/models/schema.py`（参数模型）

`VideoParams` 新增/调整字段（旧字段保留不删，新增的给默认值即可向后兼容）：

```python
class VideoParams(BaseModel):
    # ===== 新流程核心参数 =====
    custom_audio_file: str = ""        # 上传的音频路径（已存在字段，继续用）
    asr_provider: str = "whisper"      # "whisper"(本地) | "dashscope"(在线)
    keyword_by_llm: bool = True        # True=用LLM把中文句子转英文关键词
    min_scene_duration: float = 2.0    # 最短画面时长，短句合并用（见风险点）

    # video_subject 目前是必填，音频驱动下没有主题，改成可选避免构造报错：
    # video_subject: Optional[str] = ""

    # ===== 旧字段保留默认值，不要删（webui/api 可能仍引用）=====
    # video_script / video_terms / paragraph_number
    # voice_name / voice_rate / voice_volume / bgm_type / bgm_file / bgm_volume ...
```

---

### 2. `app/services/task.py`（流程编排，**改动最大**）

`start()` 重排为新流程；旧逻辑用注释包起来保留。

- **`generate_script()`**：上传音频模式**跳过**（注释原调用）。`video_script` 置空或等识别后回填。
- **`generate_terms()`**：不再整体生成，改为“按句生成关键词”（见下，注释原整体调用）。
- **`generate_audio()`**：只保留 `else`(custom_audio_file) 分支；
  `if not custom_audio_file:` 的 TTS 分支整段注释。没有上传音频直接判失败。
- **`generate_subtitle()`**：强制走 ASR。把 `if subtitle_provider == "edge"` 整段注释，
  按 `params.asr_provider` 调 `subtitle.create()`(whisper) 或新增的
  `subtitle.create_by_dashscope()`(在线)。**不要再调用 `subtitle.correct()`**
  （原 correct 拿 LLM 文案校正字幕，新流程没有原始文案）。
- **新增 `build_segments_and_materials(subtitles, params)`**：
  - 读字幕得到 `[(start, end, 中文文本), ...]`（用现有 `subtitle.file_to_subtitles()`）。
  - 可选“短句合并”（< `min_scene_duration` 的相邻句合并成一个画面段）。
  - 每段：`关键词 = llm.generate_terms(该句)` → `material.download_one_video(关键词)` → `clip_path`。
  - 产出 `segments = [{start, end, text, clip_path}, ...]`。
- **`generate_final_videos()`**：调用新增的 `video.combine_videos_by_timeline(segments, ...)`，
  原 `video.combine_videos()` 调用注释保留。
- **第 7 步跨平台发布**：整段注释（`upload_post` 相关）。

`start()` 新顺序：
```
update(progress=5)
audio_file, audio_duration = use_uploaded_audio()          # 取代 generate_audio 的 TTS
subtitle_path, subs        = generate_subtitle_by_asr()    # whisper / dashscope
segments                   = build_segments_and_materials(subs)  # 按句关键词+按句下素材
combined                   = video.combine_videos_by_timeline(segments)  # 按时间轴拼接
final                      = video.generate_video(combined, audio, subtitle)  # 叠字幕+挂音频
update(progress=100, ...)
```

---

### 3. `app/services/subtitle.py`（字幕识别，两种都保留）

- **本地 whisper**：基本不动（已是句子级+时间戳）。
  `from faster_whisper import WhisperModel` 已是 try/except 延迟保护，注释包后也不崩。
- **新增在线 `create_by_dashscope(audio_file, subtitle_file)`**：
  调用阿里 DashScope 语音识别（Paraformer，如 `paraformer-v2`），
  把返回的句子+时间戳写成同样 `.srt`（复用 `utils.text_to_srt`）。
  → key 复用现有 `qwen_api_key`（DashScope 通用）或新增 `[dashscope] api_key`。
- `correct()`：新流程不调用，保留函数体。

---

### 4. `app/services/material.py`（素材下载）

- 三个搜索源 `search_videos_pexels/pixabay/coverr` **全部保留**（Pexels 为主，其余备用）。
- **新增 `download_one_video(search_term, video_aspect, min_duration)`**：
  搜该关键词 → 取第 1 个满足时长的素材 → `save_video()` → 返回本地路径。
  搜不到时降级：换备用源 / 用通用兜底词（city/nature）/ 复用上一句画面。
- 原 `download_videos()` / `_download_videos_by_script_order()` 保留不动。

---

### 5. `app/services/video.py`（**新增按时间轴拼接**，核心）

新增函数（原 `combine_videos` 完整保留不删）：

```python
def combine_videos_by_timeline(
    combined_video_path: str,
    segments: list,          # [{start, end, clip_path}, ...] 来自字幕时间轴
    audio_file: str,
    video_aspect=VideoAspect.portrait,
    threads=2,
) -> str:
    """
    一句一画面：第 i 个片段时长 = segments[i].end - segments[i].start。
    每个 clip：截取/循环到目标时长 → resize+letterbox 到目标分辨率
    （复用现有逻辑）→ 顺序 concat；总时长对齐音频时长。
    """
```

实现要点：
- 复用现有 resize / 居中 letterbox / `concat_video_clips_with_ffmpeg` / 编码回退逻辑。
- 素材短于该句 → 循环或定格尾帧（不变速更安全）；长于该句 → `subclipped(0, 句时长)` 截断。
- `generate_video()`（叠字幕+挂音频）**基本不动**，继续复用。

---

### 6. LLM（`app/services/llm.py`，全部保留）

- 所有提供商保留。复用/新增 `generate_terms()` 的“单句”用法：
  输入一句中文，输出 3~5 个英文画面关键词。
  prompt 改为：“给定一句中文旁白，输出最适合作为空镜/B-roll 画面搜索的英文关键词”。
- 原 `generate_script` / `generate_social_metadata` 保留不动（新流程不调用）。

---

### 7. 入口（两个都改）

- **Streamlit `webui/`**：
  - 新增「上传音频」控件（`st.file_uploader`，存到 task 目录，写入 `custom_audio_file`）。
  - 注释隐藏：主题输入、文案编辑、TTS 音色/语速/音量、背景音乐相关 UI。
  - 新增：ASR 方式(whisper/dashscope)、画面比例、素材源、是否用 LLM 关键词、最短画面时长。
- **FastAPI `app/`（router/controller）**：
  - 新增上传音频接口（`multipart/form-data`）或复用现有 `/tasks` + `custom_audio_file`。
  - 任务请求体对齐新的 `VideoParams` 字段。

---

## 四、依赖瘦身（注释 + 延迟导入，避免启动即崩）

> 原则：**先注释使用处 → 再把顶层 import 改延迟/try 导入 → 最后注释包**。
> 否则注释了包、代码顶层还 `import` 就直接崩。`subtitle.py` 的 try/except 写法可照抄。

可注释掉的包：

| 包 | 用途 | 处理 |
|---|---|---|
| `edge_tts` | TTS 配音 | 注释 voice.py 用法 + import |
| `azure-cognitiveservices-speech` | Azure TTS | 同上 |
| `elevenlabs` / siliconflow 相关 | TTS | 同上 |
| `redis` | 任务状态（默认 enable_redis=false） | 注释 import |
| `pydub` | 音频处理(变速/BGM) | 注释 BGM/变速用法 + import |
| `upload_post` 相关 | 跨平台发布 | 注释整模块用法 + import |

**必须保留**：`faster-whisper`(本地ASR)、`dashscope`(在线ASR + 可做关键词)、
全部 LLM 包(openai/gemini/qwen/litellm... 你要求保留)、
`moviepy`/`imageio-ffmpeg`/`Pillow`/`numpy`(合成)、`requests`(下载)、
`fastapi`/`uvicorn`/`streamlit`(入口)、`loguru`/`pyyaml`/`python-multipart`。

---

## 五、配置变更（`config.example.toml`）

- `subtitle_provider`：语义改为 ASR 选择，默认 `"whisper"`，可选 `"dashscope"`
  （或在 schema 用 `asr_provider`，二者取其一，建议统一）。
- `[whisper] model_size = "large-v3"`（中文最准；CPU 慢可改 medium）。
- 新增（可选）`[dashscope] api_key`，或直接复用 `qwen_api_key`。
- TTS / BGM / upload_post 配置项**保留但注释“新流程未使用”**。

---

## 六、建议实施顺序

1. `schema.py` 加字段（最小、向后兼容）。
2. `subtitle.py` 加 `create_by_dashscope()`（本地 whisper 已可用，先打通在线）。
3. `material.py` 加 `download_one_video()`。
4. `video.py` 加 `combine_videos_by_timeline()`（核心，单独写脚本测一段音频）。
5. `llm.py` 让 `generate_terms()` 支持“单句→英文关键词”。
6. `task.py` 重排 `start()`，串起 2→5（旧逻辑注释保留）。
7. 改 `webui` + `api` 入口。
8. 最后做依赖瘦身（先延迟 import，再注释包）。

> 先保证流程跑通（不删包），再做依赖瘦身，风险最低。

---

## 七、风险与注意点

- **画面切换太碎**：中文一句可能只有 1~2 秒，画面频繁闪。用 `min_scene_duration`
  把过短的相邻句合并成一个画面（在 `build_segments_and_materials` 里做）。
- **Pexels 配额**：按句下载请求量大，用多 key 轮询（已支持）+ 失败降级兜底。
- **whisper large-v3 在 CPU 上很慢**：无 GPU 时长音频识别耗时明显，可优先选 dashscope 在线。
- **素材时长不足该句**：不变速前提下用“循环/定格尾帧”，避免黑屏或拉伸变形。
- **字幕与音频对齐**：直接用 ASR 时间戳，天然对齐，无需 `subtitle.correct()`。

---

## 八、改之前可再拍板的细节（非阻塞）

1. **DashScope key**：复用现有 `qwen_api_key`，还是新建 `[dashscope] api_key`？（建议复用）
2. **素材搜不到的兜底**：通用关键词兜底 / 复用上一句画面 / 纯色背景？（建议：先备用源，再通用词，最后复用上一句）
3. **画面比例默认**：竖屏 9:16 还是横屏 16:9？（Pexels 竖屏少、Coverr 偏横屏）
4. **最短画面时长默认值**：建议 2 秒，可在界面调。
