# video-knowledge-ingest-skill

A video knowledge ingestion skill repository for OpenClaw / AgentSkills workflows.

This project is not just a "download a video" utility. It is designed for the messier and more useful real-world job: **turning cross-platform video content or local media into durable text assets that can be stored in a local knowledge base**.

The current pipeline already implements the following flow:

1. Accept a remote URL or a local file
2. Try subtitles first
3. Fall back to `yt-dlp + ffmpeg + Whisper` when subtitles are unavailable
4. Generate a summary from the transcript
5. Save transcript, summary, metadata, and index entries into a local knowledge base

In one sentence: **subtitle-first, Whisper as fallback, and finally a locally stored knowledge entry you can inspect and reuse later.**

---

## Project Overview

This repository contains an Agent Skill named `video-knowledge-ingest`, along with its supporting scripts, reference documents, and packaged distribution artifact.

Its purpose is straightforward:

- Handle video knowledge ingestion from **YouTube, Bilibili, Xiaohongshu, and local files** using a single workflow
- Prefer platform subtitles whenever possible to reduce transcription cost and error rate
- Fall back to local Whisper transcription when subtitles are missing, incomplete, or unavailable
- Produce structured on-disk outputs that are easy to inspect, index, archive, and process further
- Serve as a standard "video ingestion pipeline" for a main agent or any sub-agent

This repository is **not** a general-purpose downloader and **not** a full crawler for every video site feature. It is focused on three practical steps:

- **video -> text**
- **text -> summary**
- **result -> local knowledge base**

Pragmatic beats flashy here.

---

## What the Project Currently Does

Based on the current repository contents (`SKILL.md`, `video_ingest.py`, Whisper wrappers, and reference documents), the project already supports the following capabilities.

### 1. Accept Multiple Input Types

The pipeline supports two major input categories:

- Remote URLs
  - YouTube
  - Bilibili
  - Xiaohongshu
- Local files
  - Local video / audio media files
  - Local subtitle files (`.srt` / `.vtt`)
  - Local text files (`.txt` / `.md`)

### 2. Normalize Problematic URLs

Before processing, the script normalizes URLs to reduce platform-specific failures.

- **Bilibili**
  - Converts `bilibili.com` / `m.bilibili.com` into `www.bilibili.com`
  - Removes common `spm_*` tracking parameters to reduce 403s and metadata failures
- **YouTube**
  - Preserves meaningful playback parameters such as `v`, `t`, `list`, `index`, and `start`
  - Removes common tracking parameters

This is not cosmetic. Shared links often contain noisy parameters, mobile hosts, or tracking junk that makes `yt-dlp` unnecessarily fragile.

### 3. Prefer Subtitles First

For remote URLs, the script first attempts to:

- download subtitles without downloading the media itself
- request both normal subtitles and auto-generated subtitles
- prefer `zh.*` and `en.*`
- work with `.srt` / `.vtt`
- choose the best available subtitle file automatically

The subtitle ranking logic is based on the current code and roughly prefers:

- Chinese over English
- English over other languages
- non-auto subtitles over auto subtitles
- `.srt` over `.vtt`
- larger files when that likely means more complete content

If at least one usable subtitle file lands successfully, the pipeline continues even when another subtitle variant exits non-zero. That detail matters a lot for real-world 429 or partial subtitle failure scenarios.

### 4. Fall Back to Whisper Transcription

When no usable subtitles are available, the script automatically falls back to:

- downloading media with `yt-dlp`
- converting the input into 16kHz mono WAV with `ffmpeg`
- calling the bundled Whisper wrapper for local transcription
- trying GPU first
- falling back to CPU automatically if GPU transcription fails

This is not theoretical. The repository already contains the working implementation:

- `scripts/whisper-gpu`
- `scripts/whisper_gpu_transcribe.py`

The transcription layer can output:

- `.txt`
- `.srt`
- `.vtt`
- `.json`

In the main pipeline, the generated text is copied into the final `transcript.txt` used by the summary stage.

### 5. Ingest Local Subtitle and Text Files Directly

For local inputs, the repository does not force everything through the expensive "download + transcription" path:

- `.srt` / `.vtt`: cleaned directly into plain transcript text
- `.txt` / `.md`: copied directly into the transcript path
- media files: sent to Whisper

This makes the skill useful not only for remote content but also for:

- already-downloaded archives
- local recordings and podcasts
- manually saved subtitles
- previously prepared text notes

### 6. Generate Summaries Automatically

Once `transcript.txt` exists, the script calls:

- `summarize --cli codex --force-summary`

The final summary is written to `summary.md`.

### 7. Persist Everything into a Local Knowledge Base

Each ingest run creates a full item directory and appends an entry to the global index:

- per-item directory
- `record.json`
- `summary.md`
- `transcript.txt`
- `source.info.json`
- `index.jsonl`

This means the project is not just a one-shot script. It is a reusable, accumulating local knowledge pipeline.

---

## Repository Structure

Current repository layout:

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

### What each file or directory does

#### `video-knowledge-ingest/SKILL.md`
The skill entry point. It defines:

- the skill name and trigger description
- recommended usage
- the standard workflow
- when to read the bundled reference documents

#### `video-knowledge-ingest/scripts/video-ingest`
A thin Bash entrypoint that forwards calls to the Python implementation.

It is intentionally simple and stable, making it a good canonical entrypoint.

#### `video-knowledge-ingest/scripts/video_ingest.py`
The core implementation. It is responsible for:

- parsing input
- normalizing URLs
- loading remote metadata
- attempting subtitles first
- downloading media on fallback
- calling Whisper
- calling the summary tool
- generating the knowledge-base item and index entry

#### `video-knowledge-ingest/scripts/whisper-gpu`
The Whisper runtime wrapper. It is responsible for:

- locating the workspace root
- locating the GPU-specific virtual environment
- exporting CUDA / cuBLAS / cuDNN library paths
- calling the Python Whisper transcription script

#### `video-knowledge-ingest/scripts/whisper_gpu_transcribe.py`
The local transcription implementation based on `faster-whisper`. It is responsible for:

- normalizing audio with `ffmpeg`
- calling `WhisperModel`
- generating TXT / SRT / VTT / JSON outputs
- supporting both GPU and CPU execution

#### `video-knowledge-ingest/references/toolchain.md`
Additional documentation for the toolchain and output structure.

#### `video-knowledge-ingest/references/troubleshooting.md`
Troubleshooting notes for common failures, including:

- YouTube anti-bot issues
- Bilibili 403s
- partial subtitle failures
- Xiaohongshu missing subtitles
- Whisper environment issues
- missing `ffmpeg`
- `summarize` / `codex` authentication failures

#### `dist/video-knowledge-ingest.skill`
The packaged skill artifact ready for distribution.

---

## Core Workflow

This is the most important part of the project. No mysticism required; the real workflow is just this.

### High-level flow

```text
Input source (URL / local file)
  ↓
Normalize and classify
  ↓
Remote input: fetch metadata first
  ↓
Try subtitles first
  ├─ success: clean subtitles -> transcript.txt
  └─ failure: download media -> ffmpeg normalize -> Whisper -> transcript.txt
  ↓
Run summarize
  ↓
Write item directory + record.json + index.jsonl
```

### 1. Identify the input

`video_ingest.py` first decides whether the input is:

- a URL
- a local path

URLs use the remote flow. Local paths use the local flow.

### 2. Normalize the URL

Remote URLs are normalized for platform compatibility:

- Bilibili links are converted to `www.bilibili.com` and stripped of `spm_*`
- YouTube links keep meaningful playback parameters and drop tracking junk

This matters because real-world shared links are noisy, and noisy links make `yt-dlp` fail in boring but expensive ways.

### 3. Fetch remote metadata

The script uses:

- `yt-dlp --dump-single-json --skip-download`

This is used to:

- extract the title
- identify the platform / extractor
- generate a stable output directory name
- write `source.info.json`

### 4. Subtitle-first path

This is the preferred path because it is usually:

- faster
- cheaper
- textually cleaner than ASR
- less dependent on long transcription runs

The script:

- runs `yt-dlp --skip-download --write-subs --write-auto-subs`
- prioritizes `zh.*` and `en.*`
- converts subtitles to `srt`
- continues as soon as at least one usable subtitle file exists

The subtitle cleaning logic removes:

- numeric sequence lines
- timeline lines
- `WEBVTT`
- `NOTE`
- style / metadata lines
- HTML tags
- subtitle control markers

The result becomes `transcript.txt`.

### 5. Whisper fallback path

If subtitles are missing or unusable, the project switches to the heavy path:

1. download media
2. normalize audio with `ffmpeg`
3. transcribe with `faster-whisper`
4. try GPU first
5. fall back to CPU if GPU fails
6. use the generated `.txt` as the final transcript

The source of the final transcript is recorded clearly, so you can later see values like:

- `subtitles:source.zh.vtt`
- `subtitles:source.en.srt`
- `whisper-gpu`
- `whisper-cpu`
- `text:filename.txt`

This is useful for both debugging and downstream quality decisions.

### 6. Generate the summary

Once `transcript.txt` exists, the script calls:

```bash
summarize <transcript> --plain --cli codex --force-summary --length <preset> --timeout 5m
```

The result is wrapped into `summary.md`, which includes:

- title
- source
- platform
- transcript source
- transcript file
- generation timestamp
- summary body

### 7. Write the local knowledge entry

Each ingestion writes a dedicated item directory and appends to `index.jsonl`.

Default root:

```text
/home/jason/.openclaw/workspace/knowledge/video-notes/
```

Directory naming is roughly:

```text
YYYY-MM-DD/<platform>-<id>-<slug>/
```

This gives you a date-based structure with one directory per ingested item.

---

## External Tools Used in This Project

This project is fundamentally a tool orchestrator. The scripts provide structure, but the actual heavy lifting is done by the tools below.

### 1. `yt-dlp`

**What it is used for**

- loading remote video metadata
- downloading platform subtitles or auto-subtitles
- downloading media when subtitles are not available

**Why it is used**

- it is one of the most mature and practical cross-platform media extraction tools available
- it works across YouTube, Bilibili, Xiaohongshu, and many other sites
- it can return structured metadata
- it supports separate subtitle-first and media-download paths

**Role in this project**

- first stage of the remote ingestion flow
- key enabler of the subtitle-first strategy
- media fetcher before Whisper fallback

### 2. `ffmpeg`

**What it is used for**

- converting arbitrary input media into a normalized WAV format suitable for transcription

**Why it is used**

- Whisper is tolerant, but standardizing sample rate and channel layout improves reliability
- media from different platforms comes in wildly different formats and codecs
- `ffmpeg` is the industry standard; not using it would mostly be a creative way to invent avoidable problems

**Role in this project**

- media preprocessing layer before transcription
- called directly by `whisper_gpu_transcribe.py`

### 3. `ffprobe`

**What it is used for**

- part of the expected media toolchain used to ensure the environment is complete

**Why it is used**

- although the main script does not explicitly call `ffprobe` in every path, it belongs to the same operational toolchain as `ffmpeg`
- in real media-processing environments, `ffmpeg` and `ffprobe` are usually treated as a pair
- when one is missing, debugging becomes educational in all the wrong ways

### 4. `faster-whisper`

**What it is used for**

- local ASR transcription

**Why it is used**

- when subtitles do not exist, the project needs a local speech-to-text fallback
- `faster-whisper` is practical for local use
- it supports both GPU and CPU execution

**Role in this project**

- main fallback path when subtitles are unavailable
- called directly via `WhisperModel` inside `whisper_gpu_transcribe.py`

### 5. `summarize`

**What it is used for**

- generating summaries from transcript text

**Why it is used**

- the project goal is not just transcription; it also wants concise, reusable summaries
- `summarize` provides a stable CLI abstraction for the summary stage

**Role in this project**

- converts transcript text into a human-readable summary
- currently used with `--cli codex`

### 6. `codex`

**What it is used for**

- the current summary backend behind `summarize`

**Why it is used**

- the current implementation explicitly depends on `summarize --cli codex`
- therefore the environment must have a working, authenticated `codex`

### 7. Python runtime and GPU virtual environment

**What it is used for**

- running `video_ingest.py`
- hosting the Whisper GPU transcription dependencies

**Why it is used**

- Whisper-related dependencies are heavy and easier to manage in an isolated virtual environment
- the `whisper-gpu` wrapper resolves the workspace GPU venv at:

```text
/home/jason/.openclaw/workspace/.venv-whisper-gpu
```

- it also injects CUDA / cuBLAS / cuDNN library paths before running transcription

---

## Platform Support

### YouTube

**Support status:** supported

**Typical path:**

- if subtitles exist: subtitle-first
- if subtitles do not exist: download media and use Whisper

**Compatibility work already included:**

- tracking parameter cleanup
- preservation of meaningful timing and playlist parameters

**Known limitations:**

- may hit anti-bot or login-required behavior
- some environments require cookies

### Bilibili

**Support status:** supported

**Typical path:**

- tries subtitles first
- often falls back to Whisper in practice

**Compatibility work already included:**

- automatic normalization to `www.bilibili.com`
- automatic removal of `spm_*` parameters

**Known limitations:**

- shared links and tracking-heavy links are more fragile
- subtitle availability can be inconsistent

### Xiaohongshu

**Support status:** supported

**Typical path:**

- metadata and media can be resolved
- subtitles are often unavailable, so Whisper fallback is common

**Known limitations:**

- missing subtitles are often normal platform behavior, not a script bug

### Local files

**Support status:** supported

Supported types include:

- subtitle files: `.srt`, `.vtt`
- text files: `.txt`, `.md`
- media files: `.mp4`, `.mkv`, `.webm`, `.mov`, `.m4v`, `.avi`, `.flv`, `.wmv`, `.mp3`, `.m4a`, `.aac`, `.wav`, `.flac`, `.ogg`, `.opus`

This is useful for:

- previously downloaded video archives
- local recordings or podcasts
- manually saved subtitle files
- already prepared text material

---

## Outputs and Local Knowledge Base Structure

Default output root:

```text
/home/jason/.openclaw/workspace/knowledge/video-notes/
```

Typical structure:

```text
knowledge/video-notes/
  index.jsonl
  YYYY-MM-DD/
    <platform>-<id>-<slug>/
      source.url            # exists for remote inputs
      source.path           # exists for local inputs
      source.info.json      # metadata
      downloads/            # remote subtitles or media
      whisper/              # Whisper intermediate outputs
      transcript.txt        # final transcript
      summary.md            # final summary
      record.json           # structured record for this item
```

### What each output file means

#### `source.url`
The original or normalized remote URL.

#### `source.path`
The absolute local path when the input is local.

#### `source.info.json`
Source metadata:

- remote inputs: produced from `yt-dlp --dump-single-json`
- local inputs: generated as a minimal local metadata object by the script

#### `downloads/`
Holds:

- downloaded subtitle files
- downloaded media files
- related metadata files

#### `whisper/`
Holds raw Whisper outputs such as:

- `.txt`
- `.srt`
- `.vtt`
- `.json`

#### `transcript.txt`
The standardized transcript file used by the summary stage.

#### `summary.md`
The final human-readable summary document with basic metadata in the header.

#### `record.json`
The structured result record for the ingest run, typically containing:

- `created_at`
- `title`
- `platform`
- `source`
- `item_dir`
- `transcript_path`
- `transcript_source`
- `summary_path`

#### `index.jsonl`
The append-only global index, one JSON line per ingested item. This is intended for later:

- search
- aggregation
- re-indexing
- integration into higher-level knowledge workflows

---

## Usage

### Main entrypoint

Use the repository entrypoint:

```bash
video-knowledge-ingest/scripts/video-ingest "<source>"
```

Within an OpenClaw workspace skill context, the typical invocation is:

```bash
skills/video-knowledge-ingest/scripts/video-ingest "<source>"
```

### Remote URL examples

```bash
video-knowledge-ingest/scripts/video-ingest "https://www.youtube.com/watch?v=..."
video-knowledge-ingest/scripts/video-ingest "https://www.bilibili.com/video/BV..."
video-knowledge-ingest/scripts/video-ingest "https://www.xiaohongshu.com/explore/..."
```

### Local file examples

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/file.srt
video-knowledge-ingest/scripts/video-ingest /path/to/file.mp4
video-knowledge-ingest/scripts/video-ingest /path/to/notes.txt
```

### Common options

#### Specify transcription language

```bash
video-knowledge-ingest/scripts/video-ingest "<source>" --language zh
```

Default:

```text
zh
```

#### Specify summary length

```bash
video-knowledge-ingest/scripts/video-ingest "<source>" --length long
```

#### Specify a custom knowledge-base root

```bash
video-knowledge-ingest/scripts/video-ingest "<source>" --kb-root /some/other/root
```

### Successful output

On success, the script prints JSON to stdout containing key paths such as:

- `item_dir`
- `transcript_path`
- `summary_path`
- `transcript_source`

This makes the workflow easy to chain in agent pipelines while still being human-readable.

---

## Dependencies and Runtime Requirements

Based on the current repository implementation, the project expects the following environment.

### Required CLI tools

- `yt-dlp`
- `ffmpeg`
- `ffprobe`
- `summarize`
- `codex`
- `python3`

### Whisper-related environment

- workspace GPU virtual environment:

```text
/home/jason/.openclaw/workspace/.venv-whisper-gpu
```

The environment should contain working:

- `faster-whisper`
- the necessary CUDA dependencies
- cuBLAS / cuDNN shared libraries

### PATH behavior

`video_ingest.py` prepends these paths when calling external commands:

- `/home/jason/.local/bin`
- `/home/jason/.npm-global/bin`

This makes locally installed CLIs easier to resolve even when the system PATH is incomplete.

---

## Common Problems and Limitations

### 1. YouTube may require login or cookies

If you see errors such as:

- `LOGIN_REQUIRED`
- "Sign in to confirm you're not a bot"

That is usually a platform access problem, not a transcription problem. In that case you need:

- valid YouTube cookies
- or a local media file / mirror input instead

### 2. Partial subtitle failure does not necessarily mean total failure

The code already handles this case deliberately:

- even if one subtitle variant fails
- as long as at least one usable `.srt` / `.vtt` file exists
- the subtitle path continues

This matters especially for HTTP 429 situations.

### 3. Xiaohongshu often has no subtitles

That is normal platform behavior, not a bad day in the pipeline.

Expected behavior:

- no subtitles
- media download
- Whisper fallback

### 4. The Whisper GPU environment is usually the most fragile part

If GPU transcription fails, the script automatically tries CPU fallback.

But if the following are missing entirely, the run can still fail:

- `.venv-whisper-gpu`
- `faster-whisper`
- CUDA-related libraries
- `ffmpeg`

### 5. The summary stage depends on `summarize + codex`

Even if transcription succeeds, the final pipeline is not complete if the summary backend fails.

Check:

- whether `summarize` is installed and usable
- whether `codex` is installed and authenticated

### 6. Not every platform problem can be solved purely in code

The repository includes real compatibility work, but it still depends on third-party platform behavior such as:

- site structure changes
- login policy changes
- anti-bot systems
- region restrictions

So the project aims to be **robust**, not miraculous.

---

## Development, Packaging, and Validation

This repository already includes source code and a packaged artifact, and it is intended to be iterated further.

### 1. Main files to edit during development

Core logic:

- `video-knowledge-ingest/scripts/video_ingest.py`
- `video-knowledge-ingest/scripts/whisper_gpu_transcribe.py`
- `video-knowledge-ingest/scripts/whisper-gpu`

Documentation:

- `video-knowledge-ingest/SKILL.md`
- `video-knowledge-ingest/references/toolchain.md`
- `video-knowledge-ingest/references/troubleshooting.md`

### 2. Recommended basic smoke tests

At minimum, run these:

#### Local subtitle file

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/test.srt
```

Verify:

- `transcript.txt` is generated
- `summary.md` is generated
- `record.json` is written
- `index.jsonl` is appended

#### Local text file

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/test.txt
```

Verify:

- the text is reused directly as transcript
- the summary is generated correctly

#### Local media file

```bash
video-knowledge-ingest/scripts/video-ingest /path/to/test.mp4
```

Verify:

- Whisper GPU works
- CPU fallback works when GPU fails
- `whisper/` contains intermediate outputs

#### Remote URL tests

Test at least:

- one YouTube video with subtitles
- one Bilibili video
- one Xiaohongshu video

Verify:

- URL normalization works
- subtitle-first behavior works where expected
- no-subtitle fallback works where expected

### 3. Package the skill

To regenerate the packaged artifact, use the packaging script:

```bash
scripts/package_skill.py /path/to/video-knowledge-ingest
```

The current repository already includes one packaged artifact:

```text
dist/video-knowledge-ingest.skill
```

### 4. Validate the package result

Recommended checks:

- `SKILL.md` frontmatter is valid
- `scripts/` and `references/` are included
- no symlinks were introduced
- entrypoint paths still work after packaging

### 5. Recommended debugging order

When an ingest run fails, do not immediately blame the summary model or Whisper. Check in this order:

1. Was the item directory created?
2. Does `source.info.json` exist?
3. Does `downloads/` contain subtitles or media?
4. Does `whisper/` contain outputs?
5. Is `transcript.txt` empty?
6. Was `summary.md` generated?
7. Were `record.json` and `index.jsonl` updated?

This is far better than debugging by vibes. The latter mainly produces more vibes.

---

## Good Use Cases

This project is especially useful when you want to:

- store video content in a local knowledge base
- give agents a standard video-to-text entrypoint
- handle multiple platforms without building a separate script for each one
- prefer subtitles first and only pay the ASR cost when necessary
- process already-downloaded media or subtitles in a clean, structured way

---

## What Is Not Implemented Yet

To avoid README fiction, here are capabilities that are **not** represented by the current repository implementation and should not be claimed as already supported:

- vector embeddings / embedding-based retrieval
- full-text search backend service
- automatic tag extraction
- multilingual translation output
- task queues or batch schedulers
- web UI
- database-backed storage
- automatic publishing to external platforms

The current core is: **a robust single-item ingestion pipeline into a local file-based knowledge base**.

---

## Summary

The value of `video-knowledge-ingest-skill` is not that it tries to do everything. Its value is that it turns one of the most common and annoying media knowledge tasks into a reusable pipeline:

- cross-platform input
- subtitle-first processing
- Whisper fallback
- automatic summary generation
- structured local persistence

It is already a clear, runnable, and extensible infrastructure-style skill.

The next stage, if needed, would usually involve:

- stronger batch processing
- richer structured extraction
- indexing and retrieval layers

That is for later. Right now, the current implementation already does the main job well: **turn video content into local knowledge instead of leaving it as a pile of one-off scripts and forgotten links.**
