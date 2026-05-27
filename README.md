# 短视频小说创作智能体 — 模块A

> 端到端短视频创作流水线：小说文本 → 高潮前置配音 → 角色/场景提取 → 分镜剧本 → 视频生成 → 剪辑合成

**模块定位：** 短剧/小说视频化生成工具，面向有内容创作需求的创作者

**测试状态：** 179 passed, 3 skipped ✅

---

## 核心能力

```
小说文本输入
    │
    ▼
① 小说解析 ──────────── LLM ──── 章节结构化
    │
    ▼
② LLM改写 ──────────── LLM ──── 高潮前置配音文案
    │
    ▼
③ TTS配音 ──────────── TTS ──── 配音MP3
    │
    ▼
④ 场景提取 ─────────── LLM ──── 场景外观描述词
    │
    ▼
⑤ 角色提取 ─────────── LLM ──── 角色外观描述词
    │
    ▼
⑥ 分镜剧本 ─────────── LLM ──── 运镜/音效/台词脚本
    │
    ▼
⑦ 视频生成 ─────────── VideoGen ── 视频片段
    │
    ▼
⑧ 视频剪辑 ─────────── Editor ──── 最终MP4
```

---

## 项目结构

```
novel-video-pipeline/
├── llm/                      # 大语言模型调用层
│   ├── base.py               # BaseLLM 抽象基类
│   ├── openai_llm.py         # OpenAI Provider
│   ├── factory.py            # LLMFactory 工厂
│   └── test_llm.py           # 16 passed
│
├── generator/                 # 图像/视频生成层
│   ├── image_gen/            # 图像生成（角色/场景图）
│   │   ├── base.py           # BaseImageGenerator
│   │   ├── jimeng.py        # 即梦
│   │   ├── wan.py           # 腾讯混元
│   │   ├── sd20.py         # Stable Diffusion 2.0
│   │   └── factory.py       # create_image_gen_provider()
│   │
│   └── video_gen/            # 视频生成
│       ├── base.py           # BaseVideoGenerator (+ api_secret)
│       ├── jimeng.py        # 即梦
│       ├── kling.py         # 可灵
│       ├── pixverse.py      # PixVerse
│       └── factory.py        # create_video_gen_provider()
│
├── tts/                      # 文字转配音
│   ├── base.py              # TTSProvider + TTSResult
│   ├── cosyvoice.py         # 阿里云CosyVoice
│   ├── azure.py             # Azure TTS
│   ├── gpt_sovits.py        # GPT-SoVITS
│   ├── factory.py           # create_tts_provider()
│   └── test_tts.py          # 18 passed, 2 skipped
│
├── subtitle/                 # 字幕管理
│   ├── base.py              # SubtitleCue / SubtitleTrack + ASRProvider
│   ├── whisper_asr.py       # OpenAI Whisper ASR
│   ├── formatters.py        # SRTFormatter / ASSFormatter / TimelineAligner
│   ├── factory.py           # create_asr_provider()
│   └── test_subtitle.py     # 21 passed
│
├── editor/                   # 视频剪辑合成
│   ├── base.py              # VideoClip / AudioTrack / SubtitleOverlay + EditorProvider
│   ├── ffmpeg_editor.py     # FFmpeg 适配器
│   ├── composer.py          # VideoComposer 编排接口
│   ├── factory.py           # create_editor_provider()
│   └── test_editor.py       # 18 passed
│
├── audio/                    # BGM/音效
│   ├── base.py              # MusicGenProvider / SFXProvider
│   ├── music_gen.py         # SunoMusicGen / FreeMusic
│   ├── sfx.py               # LocalSFXProvider
│   ├── factory.py           # create_music_provider() / create_sfx_provider()
│   └── test_audio.py        # 27 passed
│
├── pipeline/                 # 主流程编排
│   ├── config.py            # PipelineConfig + 各阶段配置类
│   ├── dsl.py               # Pipeline DSL + 8个阶段实现
│   ├── runner.py            # PipelineRunner 执行器
│   └── test_pipeline.py     # 26 passed
│
├── cli/                      # 命令行入口
│   ├── parser.py            # argparse 参数解析
│   ├── commands.py          # 命令实现（run / config / status）
│   ├── main.py              # CLI 主入口
│   └── test_cli.py          # 17 passed
│
└── README.md                 # 本文件
```

---

## 快速开始

### Python API

```python
from pathlib import Path
from pipeline import PipelineConfig, PipelineRunner, LLMConfig, TTSConfig, VideoGenConfig

config = PipelineConfig(
    output_dir=Path("./output"),
    llm=LLMConfig(provider="openai", api_key="sk-..."),
    tts=TTSConfig(provider="cosyvoice", api_key="...", secret_key="...", app_id="..."),
    video_gen=VideoGenConfig(provider="jimeng", api_key="...", api_secret="..."),
)

runner = PipelineRunner(config)
result = runner.run(
    novel_text="小说文本...",
    progress_callback=lambda stage, p: print(f"{stage}: {p:.0%}"),
)

print(f"成功: {result['success']}")
print(f"输出文件: {result['final_output']}")
```

### CLI

```bash
# 安装
pip install -e .

# 运行
novel-video run "小说文本..." --llm-key sk-xxx --tts-key yyy --video-key zzz -v

# 交互式
novel-video run "" -i -o ./output

# 配置
novel-video config init
novel-video config show
```

---

## API Key 配置

| 阶段 | Provider | 凭证 | 配置 Key |
|------|----------|------|----------|
| LLM | `openai` | api_key | `config.llm.api_key` |
| LLM | `claude` | api_key | `config.llm.api_key` |
| LLM | `deepseek` | api_key | `config.llm.api_key` |
| TTS | `cosyvoice` | api_key + secret_key + app_id | `config.tts.*` |
| TTS | `azure` | api_key + region | `config.tts.*` |
| TTS | `gpt_sovits` | api_key (URL) | `config.tts.api_key` |
| VideoGen | `jimeng` | api_key + api_secret | `config.video_gen.*` |
| VideoGen | `kling` | api_key + api_secret | `config.video_gen.*` |
| VideoGen | `pixverse` | api_key | `config.video_gen.api_key` |
| Music | `suno` | api_key | `config.music.api_key` |

---

## 依赖

| 依赖 | 用途 | 必须 |
|------|------|------|
| `ffmpeg` + `ffprobe` | 视频剪辑 | ✅ 必须 |
| `openai` | OpenAI API | 可选 |
| `anthropic` | Claude API | 可选 |
| `requests` | HTTP 调用 | 可选 |
| `azure-cognitiveservices-speech` | Azure TTS | 可选 |
| `whisper` | 本地 ASR | 可选 |

---

## 测试

```bash
# 全部测试
python -m pytest

# 单模块测试
python -m pytest llm/test_llm.py -v
python -m pytest tts/test_tts.py -v
python -m pytest pipeline/test_pipeline.py -v
```

---

## 文档

| 文件 | 内容 |
|------|------|
| `README.md` | 项目总览（本文） |
| `ARCHITECTURE.md` | 详细架构设计 |
| `MODULE_INTERFACE.md` | 所有模块接口文档 |
| `CHANGELOG.md` | 版本变更记录 |
