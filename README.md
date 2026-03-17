# video-knowledge-ingest-skill

一个面向 OpenClaw / AgentSkills 工作流的视频知识入库技能仓库。

它解决的不是“下载一个视频”这么朴素的问题，而是更实用的那类脏活：**把跨平台视频或本地媒体，稳定地变成可沉淀到本地知识库的文本资产**。仓库当前已经实现的主流程是：

1. 接收远程 URL 或本地文件
2. 优先尝试获取字幕
3. 没有可用字幕时，自动回退到 `yt-dlp + ffmpeg + Whisper`
4. 对转录结果生成摘要
5. 把转录、摘要、元数据和索引写入本地知识库目录

一句话概括：**字幕优先，Whisper 兜底，最后沉淀为本地可检索的知识条目。**

---

## 项目简介

这个仓库包含一个名为 `video-knowledge-ingest` 的 Agent Skill，以及配套脚本、参考文档和打包产物。

它的目标很明确：

- 统一处理 **YouTube / Bilibili / 小红书 / 本地文件** 的视频知识采集
- 尽量优先使用平台已有字幕，减少转录成本与错误率
- 在字幕缺失、下载不完整、平台不给面子的情况下，用本地 Whisper 转录兜底
- 生成结构化落盘结果，便于后续检索、归档、再加工
- 适合作为主代理或子代理里的标准“视频入库管线”使用

这个仓库不是通用下载器，也不是在线视频站点的全功能爬虫。它聚焦的是：

- **视频 → 文本**
- **文本 → 摘要**
- **结果 → 本地知识库**

非常务实。没有多余花活，这反而是优点。

---

## 当前已实现的功能

基于仓库内现有 `SKILL.md`、`video_ingest.py`、Whisper 包装脚本以及 reference 文档，当前项目已经实现这些能力：

### 1. 统一接收多种输入源

支持两大类输入：

- 远程 URL
  - YouTube
  - Bilibili
  - 小红书（Xiaohongshu）
- 本地文件
  - 本地视频 / 音频媒体文件
  - 本地字幕文件（`.srt` / `.vtt`）
  - 本地文本文件（`.txt` / `.md`）

### 2. URL 规范化

脚本会在处理前先做 URL 规范化，减少平台兼容问题：

- **Bilibili**
  - 把 `bilibili.com` / `m.bilibili.com` 统一成 `www.bilibili.com`
  - 去掉常见 `spm_*` 跟踪参数，降低 403 或元数据获取失败概率
- **YouTube**
  - 保留有意义的时间/播放列表参数：`v`、`t`、`list`、`index`、`start`
  - 移除常见追踪参数

### 3. 字幕优先提取

对远程 URL，脚本首先尝试：

- 只下载字幕，不下载媒体
- 同时尝试普通字幕和自动字幕
- 优先语言范围：`zh.*`、`en.*`
- 支持 `.srt` / `.vtt`
- 自动选择“更合适”的字幕文件

字幕选择策略来自代码本身，主要考虑：

- 中文优先
- 英文次之
- 非 auto 字幕优于 auto 字幕
- `.srt` 优于 `.vtt`
- 文件更大通常意味着内容更完整

如果至少有一个可用字幕文件落地，即使某些字幕请求返回非零退出码，流程仍会继续。这一点对 429 或某一语言字幕失败的情况很重要，算是这个项目里很实用的一笔。

### 4. Whisper 转录回退

当没有可用字幕时，脚本会自动回退到：

- 使用 `yt-dlp` 下载媒体（优先 `bestaudio/best`）
- 用 `ffmpeg` 把输入统一转换为 16kHz 单声道 WAV
- 调用 bundled Whisper 包装器执行本地转录
- 优先尝试 GPU
- GPU 失败时自动退回 CPU

这部分不是纸面设计，仓库里已经有完整实现：

- `scripts/whisper-gpu`
- `scripts/whisper_gpu_transcribe.py`

并且输出不只是纯文本，还能生成：

- `.txt`
- `.srt`
- `.vtt`
- `.json`

在实际主流程中，`video_ingest.py` 会把 Whisper 产出的主文本复制到最终的 `transcript.txt`。

### 5. 本地文本/字幕文件直接入库

对于本地输入，项目并不会一股脑走“下载 + 转录”这条笨路：

- `.srt` / `.vtt`：直接清洗为纯文本 transcript
- `.txt` / `.md`：直接复制为 transcript
- 媒体文件：再走 Whisper

这意味着它不仅能处理在线视频，也能处理你已经下载好的资产，或者人工整理过的字幕/笔记文件。

### 6. 自动摘要

拿到 `transcript.txt` 后，脚本会调用：

- `summarize --cli codex --force-summary`

生成摘要文本，并写入最终的 `summary.md`。

### 7. 本地知识库落盘与索引

每次导入都会生成一个完整条目目录，并追加写入全局索引：

- 单条记录目录
- `record.json`
- `summary.md`
- `transcript.txt`
- `source.info.json`
- `index.jsonl`

这意味着它不是“一次性脚本输出”，而是可持续累积的本地知识库工作流。

---

## 仓库目录结构

当前仓库主要结构如下：

```text
video-knowledge-ingest-skill/
├── .gitignore
├── README.md
├── dist/
│   └── video-knowledge-ingest.skill
└── video-knowledge-ingest/
    ├── SKILL.md
    ├── references/
    │   ├── toolchain.md
    │   └── troubleshooting.md
    └── scripts/
        ├── video-ingest
        ├── video_ingest.py
        ├── whisper-gpu
        └── whisper_gpu_transcribe.py
```

### 各目录/文件说明

#### `video-knowledge-ingest/SKILL.md`
技能说明入口，定义：

- skill 名称与触发描述
- 推荐使用方式
- 标准工作流
- 参考文档使用说明

#### `video-knowledge-ingest/scripts/video-ingest`
Bash 入口脚本，负责把调用转交给 Python 主程序：

- 简单
- 稳定
- 适合作为统一入口

#### `video-knowledge-ingest/scripts/video_ingest.py`
本项目核心实现，负责：

- 解析输入
- 规范化 URL
- 获取远程元数据
- 先尝试字幕
- 失败后下载媒体
- 调用 Whisper
- 调用摘要工具
- 生成知识库条目和索引

#### `video-knowledge-ingest/scripts/whisper-gpu`
Whisper 的运行包装器，负责：

- 解析工作区路径
- 定位 GPU 专用虚拟环境
- 注入 CUDA / cuBLAS / cuDNN 所需库路径
- 调用 Python Whisper 转录程序

#### `video-knowledge-ingest/scripts/whisper_gpu_transcribe.py`
基于 `faster-whisper` 的本地转录实现，负责：

- 用 `ffmpeg` 规范化音频
- 调用 WhisperModel 做转录
- 输出 TXT / SRT / VTT / JSON
- 支持 GPU / CPU

#### `video-knowledge-ingest/references/toolchain.md`
对工具链和输出结构的补充说明。

#### `video-knowledge-ingest/references/troubleshooting.md`
常见故障排查参考，覆盖：

- YouTube anti-bot
- Bilibili 403
- 字幕部分失败
- 小红书无字幕
- Whisper 环境问题
- `ffmpeg` 缺失
- `summarize` / `codex` 认证失败

#### `dist/video-knowledge-ingest.skill`
当前已打包好的 skill 分发产物。

---

## 核心工作流

这是项目最关键的部分。没有必要把事情说得神神叨叨，真实流程就下面这条。

### 流程总览

```text
输入源（URL / 本地文件）
  ↓
规范化与识别
  ↓
远程：先抓元数据
  ↓
优先尝试字幕
  ├─ 成功：字幕清洗 → transcript.txt
  └─ 失败：下载媒体 → ffmpeg 规范化 → Whisper 转录 → transcript.txt
  ↓
summarize 生成摘要
  ↓
写入 item 目录 + record.json + index.jsonl
```

### 1. 输入识别

`video_ingest.py` 先判断输入是：

- URL
- 本地路径

URL 走远程流程；本地路径走本地流程。

### 2. URL 规范化

远程 URL 会先做平台适配：

- Bilibili 链接修正 host，并移除 `spm_*`
- YouTube 保留必要参数，去掉追踪参数

这样做的原因很现实：平台链接往往混着分享参数、移动端 host、追踪 query，不先清洗，后面很容易让 `yt-dlp` 吃瘪。

### 3. 远程元数据获取

通过：

- `yt-dlp --dump-single-json --skip-download`

先拉取基础元数据，用于：

- 提取标题
- 识别平台/提取器
- 生成稳定目录名
- 写入 `source.info.json`

### 4. 字幕优先路径

这是默认首选路径，因为它通常：

- 更快
- 成本更低
- 文本质量通常优于自动 ASR
- 不需要额外转码和长时间转录

具体做法：

- `yt-dlp --skip-download --write-subs --write-auto-subs`
- 语言优先 `zh.*`、`en.*`
- 转换字幕为 `srt`
- 如果已有可用字幕文件，则直接清洗为纯文本 transcript

字幕清洗逻辑会移除：

- 序号行
- 时间轴行
- `WEBVTT`
- `NOTE`
- 某些样式/元信息行
- HTML 标签
- 字幕控制标记

最终生成 `transcript.txt`。

### 5. Whisper fallback 路径

如果字幕不存在、不可用，或者平台天然不给字幕，小锤子就换成电钻：

1. 下载媒体
2. 用 `ffmpeg` 统一采样率和声道
3. 用 `faster-whisper` 转录
4. 优先 GPU
5. GPU 失败自动 CPU fallback
6. 取 Whisper 输出里的 `.txt` 作为最终 transcript

代码里明确记录了转录来源，最终 `record.json` / `summary.md` 里能看到类似：

- `subtitles:source.zh.vtt`
- `subtitles:source.en.srt`
- `whisper-gpu`
- `whisper-cpu`
- `text:filename.txt`

这对于后续质量判断和排障很有用。

### 6. 摘要生成

当 `transcript.txt` 成功生成后，脚本调用：

```bash
summarize <transcript> --plain --cli codex --force-summary --length <preset> --timeout 5m
```

然后将结果包成一个带头部元信息的 `summary.md`，其中包含：

- title
- source
- platform
- transcript_source
- transcript_file
- generated_at
- Summary 正文

### 7. 本地知识库写入

每条内容都会写入一个独立目录，并追加到 `index.jsonl`。

默认知识库根目录：

```text
/home/jason/.openclaw/workspace/knowledge/video-notes/
```

目录命名规则大致为：

```text
YYYY-MM-DD/<platform>-<id>-<slug>/
```

也就是按日期分层，再放具体条目。

---

## 外部工具说明

这个项目本质上是一个“工具编排器”。真正干活的，是下面这些外部工具；脚本负责把它们有序串起来。

### 1. `yt-dlp`

**用途：**

- 拉取远程视频元数据
- 获取平台字幕 / 自动字幕
- 在无字幕时下载媒体文件

**为什么要用它：**

- 是目前最实用、最成熟的跨平台视频提取工具之一
- 对 YouTube / B 站 / 小红书这类站点兼容性高
- 能直接输出结构化 metadata
- 能分别处理“字幕下载”和“媒体下载”两条路径

**在本项目中的角色：**

- 远程入口的第一站
- 字幕优先策略的关键执行器
- Whisper fallback 前的媒体获取器

### 2. `ffmpeg`

**用途：**

- 将输入媒体规范化为适合语音识别的 WAV 音频

**为什么要用它：**

- Whisper 对音频输入很宽容，但前置统一采样率/声道更稳定
- 不同平台下载下来的音视频编码五花八门，直接喂识别模型容易出问题
- `ffmpeg` 是事实标准，不用它基本是在给自己制造新宗教

**在本项目中的角色：**

- 转录前的媒体清洗与标准化层
- `whisper_gpu_transcribe.py` 内部直接调用

### 3. `ffprobe`

**用途：**

- 作为同套工具链的一部分，用于保证媒体探测/处理环境完整

**为什么要用它：**

- 虽然主脚本中未直接显式调用 `ffprobe`，但 reference 文档把它列为必要依赖
- 在很多音视频处理环境里，`ffmpeg` / `ffprobe` 往往成套存在
- 缺任何一个，排障体验通常都会变得很有教育意义——而且是那种你不会喜欢的教育

### 4. `faster-whisper`

**用途：**

- 本地语音识别转录

**为什么要用它：**

- 当平台无字幕时，必须有本地 ASR 兜底
- `faster-whisper` 相比原始实现更适合本地实用场景
- 支持 GPU / CPU，便于在不同机器上降级运行

**在本项目中的角色：**

- 字幕不可用时的主力兜底方案
- 由 `whisper_gpu_transcribe.py` 直接调用 `WhisperModel`

### 5. `summarize`

**用途：**

- 基于 transcript 生成摘要

**为什么要用它：**

- 项目目标不是只拿到转录文本，而是进一步形成可读摘要
- `summarize` 提供统一的摘要 CLI 接口，便于后续替换或扩展后端

**在本项目中的角色：**

- transcript 到 summary 的转换器
- 当前配置下通过 `--cli codex` 使用 Codex 作为摘要后端

### 6. `codex`

**用途：**

- 作为 `summarize` 的当前摘要后端

**为什么要用它：**

- 当前仓库实现里，摘要阶段明确依赖 `summarize --cli codex`
- 所以环境里需要有已可用的 `codex`

### 7. Python 运行环境 / GPU venv

**用途：**

- 承载 `video_ingest.py`
- 承载基于 `faster-whisper` 的 GPU 转录

**为什么要用它：**

- Whisper 相关依赖较重，独立 venv 更容易控环境
- 仓库里的 `whisper-gpu` 会定位到工作区下的：

```text
/home/jason/.openclaw/workspace/.venv-whisper-gpu
```

- 并设置 CUDA / cuBLAS / cuDNN 的库路径

---

## 平台支持

### YouTube

**支持情况：** 已支持

**典型路径：**

- 有字幕：优先字幕
- 无字幕：下载媒体后走 Whisper

**仓库内已有的兼容处理：**

- 清理 URL 追踪参数
- 保留时间戳和播放列表相关参数

**已知限制：**

- 可能遇到 anti-bot / 登录要求
- 某些 IP 或环境下需要 cookies

### Bilibili（B站）

**支持情况：** 已支持

**典型路径：**

- 先尝试字幕
- 实际上较常回退到 Whisper

**仓库内已有的兼容处理：**

- 自动把 `bilibili.com` / `m.bilibili.com` 归一化为 `www.bilibili.com`
- 自动移除 `spm_*` 参数

**已知限制：**

- 分享链接、带追踪参数链接更容易出问题
- 某些内容的字幕能力不稳定

### 小红书（Xiaohongshu）

**支持情况：** 已支持

**典型路径：**

- 可拉取 metadata / 媒体
- 多数情况没有字幕，通常走 Whisper fallback

**已知限制：**

- “无字幕”通常是平台常态，不是脚本故障

### 本地文件

**支持情况：** 已支持

支持类型包括：

- 字幕文件：`.srt` / `.vtt`
- 文本文件：`.txt` / `.md`
- 媒体文件：`.mp4`、`.mkv`、`.webm`、`.mov`、`.m4v`、`.avi`、`.flv`、`.wmv`、`.mp3`、`.m4a`、`.aac`、`.wav`、`.flac`、`.ogg`、`.opus`

这类输入不依赖远程抓取，适合：

- 已下载视频归档
- 本地录音/播客
- 人工保存的字幕文件
- 先前整理过的文本资料

---

## 输出产物与本地知识库结构

默认输出根目录：

```text
/home/jason/.openclaw/workspace/knowledge/video-notes/
```

典型结构如下：

```text
knowledge/video-notes/
  index.jsonl
  YYYY-MM-DD/
    <platform>-<id>-<slug>/
      source.url            # 远程输入时存在
      source.path           # 本地输入时存在
      source.info.json      # 元数据
      downloads/            # 远程下载得到的字幕/媒体
      whisper/              # Whisper 输出产物
      transcript.txt        # 最终转录文本
      summary.md            # 摘要结果
      record.json           # 当前条目的结构化记录
```

### 各文件说明

#### `source.url`
原始或规范化后的远程地址。

#### `source.path`
本地输入文件的绝对路径。

#### `source.info.json`
来源元数据：

- 远程：来自 `yt-dlp --dump-single-json`
- 本地：脚本生成的最小元信息对象

#### `downloads/`
保存：

- 下载的字幕文件
- 下载的媒体文件
- 某些附带的 info json

#### `whisper/`
保存 Whisper 的原始输出，如：

- `.txt`
- `.srt`
- `.vtt`
- `.json`

#### `transcript.txt`
最终供摘要阶段使用的标准 transcript 文件。

#### `summary.md`
最终给人看的摘要文档，带基础元信息头。

#### `record.json`
这次导入的结构化结果记录，典型包含：

- `created_at`
- `title`
- `platform`
- `source`
- `item_dir`
- `transcript_path`
- `transcript_source`
- `summary_path`

#### `index.jsonl`
全局追加式索引，每一行对应一个导入条目。适合后续：

- 搜索
- 汇总
- 再索引
- 构建更上层知识系统

---

## 使用方式

### 入口命令

推荐统一使用仓库内入口脚本：

```bash
video-knowledge-ingest/scripts/video-ingest "<source>"
```

如果你是在 OpenClaw workspace 技能目录上下文中调用，对应的 skill 使用形式通常是：

```bash
skills/video-knowledge-ingest/scripts/video-ingest "<source>"
```

### 远程 URL 示例

```bash
video-knowledge-ingest/scripts/video-ingest "https://www.youtube.com/watch?v=..."
video-knowledge-ingest/scripts/video-ingest "https://www.bilibili.com/video/BV..."
video-knowledge-ingest/scripts/video-ingest "https://www.xiaohongshu.com/explore/..."
```

### 本地文件示例

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/file.srt
video-knowledge-ingest/scripts/video-ingest /path/to/file.mp4
video-knowledge-ingest/scripts/video-ingest /path/to/notes.txt
```

### 常用参数

#### 指定转录语言

```bash
video-knowledge-ingest/scripts/video-ingest "<source>" --language zh
```

默认是：

```text
zh
```

#### 指定摘要长度

```bash
video-knowledge-ingest/scripts/video-ingest "<source>" --length long
```

#### 指定知识库输出目录

```bash
video-knowledge-ingest/scripts/video-ingest "<source>" --kb-root /some/other/root
```

### 成功输出

脚本成功后会向 stdout 打印 JSON，包含本次导入的核心路径信息，例如：

- `item_dir`
- `transcript_path`
- `summary_path`
- `transcript_source`

这对于代理流程串联很友好，机器和人都能读。

---

## 依赖与运行前提

根据仓库当前实现，运行本项目需要以下环境：

### 必需命令行工具

- `yt-dlp`
- `ffmpeg`
- `ffprobe`
- `summarize`
- `codex`
- `python3`

### Whisper 相关环境

- 工作区 GPU 虚拟环境：

```text
/home/jason/.openclaw/workspace/.venv-whisper-gpu
```

- 该环境内需要可用：
  - `faster-whisper`
  - 对应 CUDA 依赖
  - cuBLAS / cuDNN 动态库

### PATH 相关说明

`video_ingest.py` 在调用外部命令时，会主动把以下路径补进环境变量：

- `/home/jason/.local/bin`
- `/home/jason/.npm-global/bin`

所以某些本地安装的 CLI 不必恰好都在系统默认 PATH 里。

---

## 常见问题与限制

### 1. YouTube 可能要求登录或 cookies

如果出现：

- `LOGIN_REQUIRED`
- “Sign in to confirm you're not a bot”

这通常不是脚本问题，而是平台访问限制。需要：

- 提供有效 cookies
- 或改用本地媒体文件输入

### 2. 部分字幕失败不代表整次任务失败

项目代码已经专门处理这种情况：

- 即使某个字幕语言下载失败
- 只要已有可用 `.srt` / `.vtt` 文件落地
- 就会继续使用字幕路径

这对 HTTP 429 场景尤其重要。

### 3. 小红书经常没有字幕

这是平台现实，不是你今天水逆。

预期行为就是：

- 无字幕
- 下载媒体
- Whisper fallback

### 4. Whisper GPU 环境可能是最脆的环节

如果 GPU 转录失败，脚本会自动尝试 CPU fallback。

但如果以下内容本身缺失，仍会失败：

- `.venv-whisper-gpu`
- `faster-whisper`
- CUDA 相关库
- `ffmpeg`

### 5. 摘要阶段依赖 `summarize + codex`

即使 transcript 成功，如果摘要后端不可用，也不会得到完整最终产物。

要检查：

- `summarize` 是否安装
- `codex` 是否可用并已登录

### 6. 不是所有平台问题都能靠脚本完全规避

该项目已经做了一些平台兼容处理，但它仍然依赖第三方平台行为，例如：

- 页面结构变化
- 登录策略变化
- 反爬限制
- 区域限制

所以它是“尽可能稳”，不是“神迹保证”。

---

## 开发、打包与验证

这个仓库已经包含源码与打包产物，适合继续迭代。

### 1. 开发时优先改这些文件

核心逻辑：

- `video-knowledge-ingest/scripts/video_ingest.py`
- `video-knowledge-ingest/scripts/whisper_gpu_transcribe.py`
- `video-knowledge-ingest/scripts/whisper-gpu`

说明文档：

- `video-knowledge-ingest/SKILL.md`
- `video-knowledge-ingest/references/toolchain.md`
- `video-knowledge-ingest/references/troubleshooting.md`

### 2. 基础验证建议

至少做这些 smoke tests：

#### 本地字幕文件测试

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/test.srt
```

验证：

- 能生成 `transcript.txt`
- 能生成 `summary.md`
- 能写入 `record.json`
- 能追加 `index.jsonl`

#### 本地文本文件测试

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/test.txt
```

验证：

- 文本是否被直接复用为 transcript
- 摘要是否正常生成

#### 本地媒体文件测试

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/test.mp4
```

验证：

- Whisper GPU 是否可用
- GPU 失败时 CPU fallback 是否可用
- `whisper/` 是否产出中间文件

#### 远程 URL 测试

分别测：

- 一个有字幕的 YouTube 视频
- 一个 B 站视频
- 一个小红书视频

验证：

- URL 规范化是否生效
- 字幕优先路径是否生效
- 无字幕时是否正确 fallback

### 3. 打包 skill

如果要重新生成 skill 包，按 skill 规范通常使用打包脚本：

```bash
scripts/package_skill.py /path/to/video-knowledge-ingest
```

产出会是 `.skill` 文件。当前仓库里已经有一个现成产物：

```text
dist/video-knowledge-ingest.skill
```

### 4. 验证打包结果

建议检查：

- `SKILL.md` frontmatter 是否有效
- `scripts/`、`references/` 是否都被包含
- 没有引入 symlink
- 打包后入口路径是否仍正确

### 5. 调试顺序建议

如果某次导入失败，不要一上来就怀疑摘要模型或者 Whisper。按这个顺序查：

1. 是否成功创建 item 目录
2. `source.info.json` 是否存在
3. `downloads/` 是否有字幕或媒体
4. `whisper/` 是否有输出
5. `transcript.txt` 是否为空
6. `summary.md` 是否生成
7. `record.json` / `index.jsonl` 是否写入

这比“凭感觉 debug”有效得多。后者通常只会产生更多感觉。

---

## 适用场景

这个项目特别适合这些场景：

- 把视频内容沉淀到本地知识库
- 给代理链路提供统一的视频转文本入口
- 处理多平台内容而不想每个平台写一套脚本
- 先拿字幕，必要时再跑 ASR，降低成本
- 对已经下载好的本地媒体/字幕做二次整理

---

## 不在当前实现里的能力

为了避免 README 开始虚构文学，这里也明确一下：根据当前仓库内容，**没有看到**以下能力的实现证据，因此不应当把它们当成已支持功能：

- 视频内容向量化 / embedding 入库
- 全文检索服务端
- 自动标签提取
- 多语言自动翻译输出
- 批量任务队列 / 并发调度
- Web UI
- 数据库后端存储
- 自动发布到外部平台

仓库当前的核心是：**单条输入的稳健入库流水线**。

---

## 小结

`video-knowledge-ingest-skill` 的价值，不在于它“做了很多炫技功能”，而在于它把最常见、最烦人的视频知识入库问题收敛成了一条可复用的管线：

- 跨平台输入
- 字幕优先
- Whisper 兜底
- 自动摘要
- 本地结构化落盘

它已经是一个相当清晰、可运行、可继续扩展的基础设施型 skill。想继续做强，下一步通常会是：

- 增强批量处理能力
- 增强摘要/结构化提取能力
- 增强索引与检索层

但那是下一阶段的事。当前这套实现，已经足够把“视频内容变成本地知识”这件事干得像样，而不是像一团即兴脚本事故现场。
