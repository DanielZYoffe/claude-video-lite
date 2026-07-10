# /watch

**Give Claude the ability to watch any video.**

Paste a URL or a local file path, ask a question, and Claude fetches captions, extracts frames using scene-aware adaptive sampling, and answers based on what it actually saw — not the title, not a transcript excerpt, the video.

[![Release](https://img.shields.io/github/v/release/DanielZYoffe/claude-video-lite?label=release&color=238636)](https://github.com/DanielZYoffe/claude-video-lite/releases/latest)
[![License](https://img.shields.io/github/license/DanielZYoffe/claude-video-lite?color=0969da)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.11%2B-3572A5)](https://python.org)

---

**Claude Code** (auto-updates via marketplace):
```
/plugin marketplace add bradautomates/claude-video
/plugin install watch@claude-video
```

**Codex, Cursor, Copilot, Gemini CLI, or any of 50+ [Agent Skills](https://agentskills.io) hosts:**
```bash
npx skills add bradautomates/claude-video -g
```

More install options in [Install](#install).

---

```
/watch https://youtu.be/dQw4w9WgXcQ what happens at the 30 second mark?
/watch ~/Downloads/bug-repro.mov what's going wrong?
/watch https://vimeo.com/123456789 --start 2:15 --end 2:45
```

Zero config to start. `yt-dlp` and `ffmpeg` install on first run via `brew` on macOS. Linux and Windows print the exact commands. Captions cover most public videos for free — Whisper is only needed when a video genuinely has no caption track.

---

## Adaptive frame selection

Frames dominate token cost. A naïve uniform sampler sends up to 80 frames for a 10-minute video. The `balanced` and `token-burner` modes use a four-step adaptive pipeline instead:

1. **Scene detection** — [PySceneDetect](https://github.com/Breakthrough/PySceneDetect) finds scene boundaries more reliably than ffmpeg's scene filter.
2. **Per-scene sampling** — frame count per scene scales with duration: short scenes get 1–2 frames, long chapters get 1 per 90 s.
3. **Perceptual dedup** — 16×16 grayscale thumbnail comparison drops frames that are visually identical to the one before them (held slides, static screen recordings).
4. **Parity cap** — if the old pipeline's scene engine would have run (≥ 8 detected cuts), output is capped at `n_scenes` so adaptive never exceeds the old 1-frame-per-cut baseline.

The selection stats print to **stderr** on every run so they don't add to Claude's context:

```
────────────────────────────────────────────────
Adaptive Frame Selection
  Video duration:              12:34
  Detected scenes:             8
  Frames before deduplication: 8
  Frames after deduplication:  7
  Final frames sent to Claude: 7
  Reduction vs uniform:        91%
────────────────────────────────────────────────
```

### Token savings benchmark

Measured structurally (dedup disabled) across 6 representative video types.
Formula: **1,683 tokens per 512 px frame** (512×288 → 1 tile → 1,598 + 85).

| Content type | Old engine | Old frames | Adaptive | vs. old | vs. uniform |
|---|---|---|---|---|---|
| Talking head (10 min, 2 sections) | uniform fallback | 80 | **19** | **−76%** | −76% |
| Lecture + slides (8 min, 16 slides) | scene engine | 14 | **16** | +14% | −80% |
| Code tutorial (5 min, 8 sections) | uniform fallback | 80 | **19** | **−76%** | −76% |
| Quick demo (90 s, 18 cuts) | scene engine | 16 | **15** | −6% | −81% |
| Long course (20 min, 12 chapters) | scene engine | 11 | **12** | +9% | −85% |
| Screen recording (6 min, 4 switches) | uniform fallback | 80 | **19** | **−76%** | −76% |
| **Total (6 scenarios)** | | **281 frames** | **100 frames** | **−64%** | **−75%** |

**64% fewer tokens overall. 265k tokens saved across 6 scenarios.**

The old pipeline had a hard threshold (`SCENE_MIN_FRAMES = 8`): if fewer than 8 cuts were detected by ffmpeg's filter, it discarded the scene data and fell back to uniform sampling — up to 80 frames regardless of content. Three of the six benchmark types hit that fallback (talking head, 5-min tutorial, screen recording) because they have clear visual sections but fewer than 8 hard cuts. PySceneDetect finds those sections reliably; the adaptive sampler then allocates by duration instead.

For cut-heavy content already handled efficiently by the old scene engine, the parity cap ensures adaptive never exceeds the old baseline. The +9%/+14% cases above are 1–2 extra frames from PySceneDetect detecting slightly more scene boundaries than ffmpeg's filter — negligible in practice.

---

## How it works

1. **You paste a URL or path.** Anything `yt-dlp` supports — YouTube, Loom, TikTok, X, Instagram, Vimeo, plus hundreds more — or a local `.mp4`, `.mov`, `.mkv`, `.webm`.
2. **Captions first.** At `transcript` detail, captioned URLs return without downloading video. Otherwise `yt-dlp` pulls native captions (manual or auto-generated) from the source.
3. **Frames extracted at the chosen detail.** `efficient` decodes keyframes only (near-instant). `balanced`/`token-burner` run the adaptive pipeline above. JPEGs are 512 px wide by default, clamped to 1998 px tall for Claude `Read` compatibility.
4. **Whisper fallback for audio.** When no caption track exists, the script extracts a mono 16 kHz 64 kbps mp3 (~480 kB/min) and ships it to Whisper — Groq's `whisper-large-v3` (preferred) or OpenAI's `whisper-1`.
5. **Frames + transcript handed to Claude.** Frame paths print with `t=MM:SS` markers. Claude `Read`s each frame in parallel — JPEGs render directly as images in context.
6. **Cleanup.** Working directory printed at the end; Claude removes it after follow-ups finish.

---

## Detail modes

Numbers from a real run against a **49:08** YouTube video (1280×720, auto-captions). Pre-downloaded; extraction time only.

| Mode | Engine | Frames | Cap | Extraction | Est. image tokens |
|------|--------|--------|-----|------------|-------------------|
| `transcript` | none (captions only) | 0 | — | **~4.5 s** | 0 (~26.6k text) |
| `efficient` | keyframe (`-skip_frame nokey`) | 50 | 50 | **~0.5 s** | ~9.8k |
| `balanced` | adaptive (PySceneDetect) | 100 | 100 | **~20.9 s** | ~19.7k |
| `token-burner` | adaptive (uncapped) | 116 | none | **~21.0 s** | ~22.8k |

- `efficient` is ~40× faster than the scene modes — it only reconstructs keyframes rather than decoding every frame to find cuts. It can return *more* frames than `balanced` on low-motion footage.
- `token-burner` only diverges from `balanced` past the cap. On this clip (116 scene cuts) `balanced` sampled 100 and `token-burner` kept all 116.
- Image token estimate uses Anthropic's tile formula at 512 px width. `--resolution 1024` roughly 4×s that.

---

## Frame deduplication

Even after scene-aware sampling, consecutive frames can still be near-identical — a slide held for 90 seconds, a screen recording with no movement. The dedup pass drops them before they reach Claude. Runs by default; `--no-dedup` turns it off.

1. Each extracted JPEG is scaled to a 16×16 grayscale thumbnail (one `ffmpeg` call per frame). Pure-stdlib Python from here.
2. Mean absolute pixel difference against the *last kept frame* — not the previous one. This catches slow fades that never trip a frame-to-frame threshold.
3. Frames at or below threshold (`2.0`) are dropped. Above threshold: kept, becomes new reference.

The `Frames:` line in Claude's report shows what collapsed: `7 selected from 14 candidates (7 near-duplicates dropped)`. On always-moving footage nothing drops and you pay what you'd have anyway.

---

## Install

| Surface | Command |
|---------|---------|
| **Claude Code** | `/plugin marketplace add bradautomates/claude-video` then `/plugin install watch@claude-video` |
| **Codex, Cursor, Copilot, Gemini CLI, +50 more** | `npx skills add bradautomates/claude-video -g` |
| **claude.ai (web)** | [Download `watch.skill`](https://github.com/bradautomates/claude-video/releases/latest) → Settings → Capabilities → Skills → `+` |
| **Manual / dev** | `git clone` then symlink `skills/watch/` into your host's skills dir |

### Claude Code

```
/plugin marketplace add bradautomates/claude-video
/plugin install watch@claude-video
```

Update: `/plugin update watch@claude-video`.

### Codex, Cursor, Copilot, Gemini CLI, and 50+ others

```bash
npx skills add bradautomates/claude-video -g
```

`-g` installs globally (`~/.codex/skills`, `~/.cursor/skills`, etc.). Drop it for per-project install. Useful flags: `-a codex -a cursor` to target specific hosts, `--copy` for filesystems without symlink support.

Update: `npx skills update watch -g`.

### claude.ai (web)

1. [Download `watch.skill`](https://github.com/bradautomates/claude-video/releases/latest) from the latest release.
2. Settings → Capabilities → Skills → `+`.
3. Enable **Code execution and file creation** under Capabilities — the skill shells out to `ffmpeg` and `yt-dlp`.

### Manual (dev)

```bash
git clone https://github.com/DanielZYoffe/claude-video-lite.git
ln -s "$(pwd)/claude-video-lite/skills/watch" ~/.claude/skills/watch
```

Build the `.skill` bundle from source: `bash skills/watch/scripts/build-skill.sh` → `dist/watch.skill`.

---

## First run

On the first `/watch` call, `scripts/setup.py --check` runs automatically. If `ffmpeg` / `yt-dlp` aren't on PATH or no API key is set, it walks through fixing it:

- **macOS** — auto-runs `brew install ffmpeg yt-dlp`
- **Linux** — prints the exact `apt` / `dnf` / `pipx` commands
- **Windows** — prints the `winget` / `pip` commands
- **API key** — scaffolds `~/.config/watch/.env` (mode `0600`) with commented placeholders

After setup, preflight is silent — sub-100ms on subsequent runs.

---

## Usage

```
/watch <url-or-path> [question]
/watch https://youtu.be/dQw4w9WgXcQ what happens at the 30 second mark?
/watch https://www.tiktok.com/@user/video/123 summarize this
/watch ~/Movies/screen-recording.mp4 when does the UI break?
```

Focus on a section for denser per-second budgets (capped at 2 fps):
```
/watch https://youtu.be/abc --start 2:15 --end 2:45
/watch video.mp4 --start 50 --end 60
/watch "$URL" --start 1:12:00            # from 1h12m to end
```

### Flags (passed to `scripts/watch.py`)

| Flag | Default | Description |
|------|---------|-------------|
| `--detail` | `balanced` | `transcript` (captions only), `efficient` (keyframes, cap 50), `balanced` (adaptive, cap 100), `token-burner` (adaptive, uncapped) |
| `--start` / `--end` | — | Focus window. Accepts `SS`, `MM:SS`, `HH:MM:SS`. |
| `--timestamps T1,T2,…` | — | Grab a frame at each timestamp. Added on top of detail frames. With `--detail transcript`, these become the only frames. |
| `--max-frames N` | — | Override the frame cap for a tighter token budget. |
| `--resolution W` | `512` | Frame width in pixels. Use `1024` when Claude needs to read on-screen text (slides, terminals, code). |
| `--fps F` | auto | Override auto-fps (still capped at 2 fps). |
| `--whisper groq\|openai` | auto | Force a specific Whisper backend. |
| `--no-whisper` | — | Disable transcription entirely; frames only. |
| `--no-dedup` | — | Keep near-duplicate frames. |
| `--out-dir DIR` | auto tmp | Keep working files in a specific location. |

Set `WATCH_DETAIL` in `~/.config/watch/.env` to change the default detail level.

---

## API keys

Captions cover most public videos for free. Whisper is only needed for local files, TikToks, some Vimeos, and the occasional caption-less YouTube upload.

| Capability | What you need | Cost |
|---|---|---|
| Download + native captions | `yt-dlp` + `ffmpeg` | Free |
| Whisper fallback (preferred) | [Groq API key](https://console.groq.com/keys) — `whisper-large-v3` | Cheap, fast |
| Whisper fallback (alt) | [OpenAI API key](https://platform.openai.com/api-keys) — `whisper-1` | Standard |
| Frames only, no transcript | `--no-whisper` | Free |

---

## Limits

- **Long-video accuracy depends on detail mode.** On capped modes, coverage thins past ~10 minutes — the frame budget spreads across the full clip. The script prints a "sparse scan" warning. Re-run with `--start`/`--end` focused on the section that matters, or use `--detail token-burner` to lift the cap.
- **PySceneDetect optional.** If the `scenedetect` package isn't installed, `balanced` and `token-burner` fall back to the existing ffmpeg-based scene engine. Install with `pip install scenedetect` to enable the adaptive pipeline.

---

## Structure

```
.
├── skills/watch/                 # self-contained skill — copied as a unit by every installer
│   ├── SKILL.md                  # skill contract — source of truth across all surfaces
│   └── scripts/
│       ├── watch.py              # entry point — download → frames → transcript
│       ├── frames.py             # adaptive frame extraction, scene detection, dedup
│       ├── download.py           # yt-dlp wrapper
│       ├── transcribe.py         # VTT parsing + dedup + Whisper orchestration
│       ├── whisper.py            # Groq / OpenAI clients (pure stdlib)
│       ├── config.py             # shared config (~/.config/watch/.env)
│       ├── setup.py              # preflight + auto-installer
│       └── build-skill.sh        # build dist/watch.skill for claude.ai upload
├── hooks/                        # SessionStart status hook (Claude Code only)
├── .claude-plugin/               # plugin.json + marketplace.json (Claude Code)
├── .codex-plugin/                # Codex/agents manifest
├── tests/                        # pytest suite (ffmpeg-synthesized clips, no network)
└── .github/workflows/            # release.yml — builds watch.skill on tag push
```

---

## Develop

```bash
# Run the test suite (ffmpeg required for frame tests):
python3 -m pytest -q

# Build the claude.ai upload bundle:
bash skills/watch/scripts/build-skill.sh      # → dist/watch.skill
```

Releasing: tag `vX.Y.Z`, push the tag. The workflow builds `dist/watch.skill` and attaches it to the GitHub release. Keep the version in sync across `skills/watch/SKILL.md`, `.claude-plugin/plugin.json`, and `.codex-plugin/plugin.json`.

---

## Star History

<a href="https://www.star-history.com/?repos=bradautomates%2Fclaude-video&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=bradautomates/claude-video&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=bradautomates/claude-video&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=bradautomates/claude-video&type=date&legend=top-left" />
 </picture>
</a>

---

MIT license. Built on [`yt-dlp`](https://github.com/yt-dlp/yt-dlp), [`ffmpeg`](https://ffmpeg.org), [`PySceneDetect`](https://github.com/Breakthrough/PySceneDetect), and Claude's multimodal `Read` tool.
