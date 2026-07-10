# /watch

paste url. claude watches.

Give Claude vision for any video. YouTube, TikTok, Loom, screen recordings, local files — anything `yt-dlp` touches. Captions first (free, instant). Frames extracted only when needed. Answers grounded in what's actually on screen, not the title.

[![Release](https://img.shields.io/github/v/release/DanielZYoffe/claude-video-lite?label=release&color=238636)](https://github.com/DanielZYoffe/claude-video-lite/releases/latest)
[![License](https://img.shields.io/github/license/DanielZYoffe/claude-video-lite?color=0969da)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.11%2B-3572A5)](https://python.org)

[See it](#before--after) · [Install](#install) · [Detail modes](#detail-modes) · [Benchmarks](#benchmarks) · [How it works](#how-it-works)

---

/watch is a skill for Claude Code, Codex, Cursor, Copilot, and 50+ other agents. Install once. Paste a URL or path, ask a question. The skill pulls captions when they exist, otherwise downloads the minimum needed audio for Whisper. It extracts frames using scene-aware adaptive sampling — spending tokens on distinct content, not held slides or static screens. By the time Claude answers, it has seen the video. Not a description of it. The video.

## Before / After

| 🗣️ Without /watch | /watch |
|---|---|
| *"I'm unable to access YouTube or watch videos. Based on the title 'Docker multi-stage builds', I can explain how multi-stage builds work in general terms — they allow you to use multiple FROM statements in a Dockerfile, which helps reduce the final image size..."* | She covers three patterns: builder stage at 4:12, distroless base at 11:30 (smallest output), BuildKit cache mounts at 18:45. Cache mounts are the one you'd use for a Node project. |
| *"I can't play video files. Could you describe what's happening on screen, share the relevant code, or paste any error messages you see?"* | Crash at 0:23. Stack trace: `NullPointerException` in `UserService.getById` — `user` is null when the token expires. You added a null check at 0:41 but it's on the wrong branch. |

Same question. Completely different answer.

```
┌────────────────────────────────────────────┐
│   frame tokens vs uniform    ████████   −64% │
│   captioned videos           █████████  free │
│   sources supported          █████████   any │
│   code / paths in transcript █████████  100% │
└────────────────────────────────────────────┘
```

/watch no add hallucination. /watch add vision.

---

## Install

One command. Finds every agent on your machine.

**Claude Code** (marketplace, auto-updates):
```
/plugin marketplace add bradautomates/claude-video
/plugin install watch@claude-video
```

**Codex, Cursor, Copilot, Gemini CLI, and 50+ others:**
```bash
npx skills add bradautomates/claude-video -g
```

`-g` installs globally (`~/.codex/skills`, `~/.cursor/skills`, etc.). Drop it for per-project only. Safe to re-run.

> **Tip** — First `/watch` call runs a preflight check. On macOS it auto-installs `ffmpeg` and `yt-dlp` via brew. On Linux and Windows it prints the exact commands. Takes ~30 seconds once, then silent forever.

**claude.ai (web):** Download [`watch.skill`](https://github.com/bradautomates/claude-video/releases/latest) → Settings → Capabilities → Skills → `+`. Enable **Code execution and file creation** first.

**Manual:** `git clone` then `ln -s "$(pwd)/claude-video-lite/skills/watch" ~/.claude/skills/watch`.

---

## Detail modes

Four modes. Switch per-call with `--detail`.

| Mode | Engine | Frame cap | Extraction | When to use |
|---|---|---|---|---|
| `transcript` | captions only | 0 | ~4.5 s | Full-length talks, podcasts, anything with good captions |
| `efficient` | keyframes | 50 | ~0.5 s | Quick visual scan, fast iteration |
| `balanced` *(default)* | adaptive (PySceneDetect) | 100 | ~21 s | General use — scene-aware, deduped |
| `token-burner` | adaptive, uncapped | none | ~21 s | Cut-heavy content, don't want to miss anything |

Numbers from a real **49:08** video (1280×720, auto-captions), pre-downloaded.

`efficient` is ~40× faster than the scene modes — it only reconstructs keyframes rather than decoding every frame to find cuts. On low-motion footage it can return *more* frames than `balanced`.

Set `WATCH_DETAIL` in `~/.config/watch/.env` to change your default.

---

## What you get

| Flag | What it does |
|---|---|
| `/watch <url> [question]` | Core command. URL or local path. Question optional. |
| `--detail transcript\|efficient\|balanced\|token-burner` | Fidelity dial. |
| `--start` / `--end` | Focus window (`SS`, `MM:SS`, `HH:MM:SS`). Denser budget, lower cost. |
| `--timestamps T1,T2,…` | Grab a frame at exact moments. Added on top of detail frames. With `--detail transcript`, these become the only frames. |
| `--max-frames N` | Hard cap for a tighter token budget. |
| `--resolution 1024` | Wider frames (default 512 px). Use when Claude needs to read on-screen text. |
| `--no-dedup` | Keep near-duplicate frames. Off by default — dedup drops held slides and static screens before they reach Claude. |
| `--no-whisper` | Frames only. No transcript. |
| `--whisper groq\|openai` | Force a specific Whisper backend (default: Groq if key present, else OpenAI). |

```
/watch https://youtu.be/dQw4w9WgXcQ what happens at the 30 second mark?
/watch ~/Downloads/bug-repro.mov when does it crash?
/watch "$URL" --start 2:15 --end 2:45
/watch "$URL" --detail transcript --timestamps 4:12,11:30,18:45
```

---

## Benchmarks

Real frame counts, structural selection only (dedup disabled). Formula: **1,683 tokens per 512 px frame** (512×288 → 1 tile → 1,598 + 85).

| Content type | Old pipeline | Old frames | Adaptive | vs. old | vs. 80-frame uniform |
|---|---|---|---|---|---|
| Talking head (10 min, 2 sections) | uniform fallback | 80 | **19** | **−76%** | −76% |
| Lecture + slides (8 min, 16 slides) | scene engine | 14 | **16** | +14% | −80% |
| Code tutorial (5 min, 8 sections) | uniform fallback | 80 | **19** | **−76%** | −76% |
| Quick demo (90 s, 18 cuts) | scene engine | 16 | **15** | −6% | −81% |
| Long course (20 min, 12 chapters) | scene engine | 11 | **12** | +9% | −85% |
| Screen recording (6 min, 4 switches) | uniform fallback | 80 | **19** | **−76%** | −76% |
| **Total** | | **281** | **100** | **−64%** | **−75%** |

**64% fewer frame tokens. 265k tokens saved across 6 scenarios.**

The old pipeline had a hard threshold: if fewer than 8 cuts were detected by ffmpeg's scene filter, it discarded the scene data and fell back to uniform sampling — up to 80 frames regardless of content. Three of the six benchmark types hit that fallback. PySceneDetect finds those section boundaries reliably; the adaptive sampler then allocates by duration. A parity cap ensures adaptive never exceeds the old scene engine's 1-frame-per-cut baseline on cut-heavy content — the +9%/+14% cases above are 1–2 extra frames from PySceneDetect detecting slightly more boundaries than ffmpeg's filter.

> **Honest numbers.** The −64% is frame *input* tokens only. Output tokens and reasoning tokens are untouched. On already-efficient content (cut-heavy videos the old scene engine handled well), adaptive is roughly neutral. On long videos with transcript mode, frame tokens are 0 — the dominant cost is the transcript itself. When adaptive wins hardest: 10+ minute talking-head content that would have burned 80 frames on uniform sampling.

---

## How it works

1. **Captions first.** `yt-dlp` checks for native captions (manual or auto-generated). At `transcript` detail, captioned URLs return without downloading any video.
2. **Download what's needed.** When frames are required, `yt-dlp` downloads the video. When Whisper is needed for audio, a mono 16 kHz 64 kbps mp3 is extracted (~480 kB/min).
3. **Adaptive frame extraction.** PySceneDetect finds scene boundaries. Frames are allocated per scene by duration (1–2 for short scenes, ~1 per 90 s for long chapters). A perceptual dedup pass drops visually identical consecutive frames before they reach Claude.
4. **Frames + transcript handed to Claude.** Frame paths print with `t=MM:SS` markers. Claude `Read`s each JPEG in parallel — they render directly as images in context.
5. **Cleanup.** Working directory printed at the end. Claude removes it after follow-ups finish.

---

## API keys

Captions cover most public videos for free. Whisper only fires for local files, TikToks, some Vimeos, and caption-less uploads.

| What you need | Why | Cost |
|---|---|---|
| `yt-dlp` + `ffmpeg` | Download + captions + frames | Free |
| [Groq API key](https://console.groq.com/keys) | Whisper fallback (`whisper-large-v3`) | Cheap, fast |
| [OpenAI API key](https://platform.openai.com/api-keys) | Whisper fallback alt (`whisper-1`) | Standard |

Keys live in `~/.config/watch/.env` (mode `0600`). `setup.py` scaffolds the file on first run.

---

## Privacy

/watch no phone home. No telemetry, no analytics, no accounts, no backend. After install, zero network calls except the ones you explicitly trigger — `yt-dlp` to fetch the video, Whisper API if you have a key and the video has no captions. Both are spelled out in the skill output. Everything else runs locally.

---

MIT — built on [`yt-dlp`](https://github.com/yt-dlp/yt-dlp), [`ffmpeg`](https://ffmpeg.org), [`PySceneDetect`](https://github.com/Breakthrough/PySceneDetect), and Claude's multimodal `Read` tool.
