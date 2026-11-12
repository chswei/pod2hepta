---
name: podcast-to-heptabase
description: Use when the user provides podcast episode links and wants Heptabase notes. Resolve audio, prefer Colab GPU transcription through colab-gpu with automatic resource cleanup, fall back to local Whisper when a GPU is unavailable, and organize the resulting notes in Heptabase.
---

# Podcast To Heptabase

## Overview

Turn podcast episode links into durable Heptabase note cards: resolve and download audio, prefer Colab GPU transcription through `colab-gpu`, retrieve and validate the transcript, release the GPU, synthesize a Markdown study note, import it with `heptabase note create`, place it on the `Podcasts` whiteboard when available, and clean up local intermediate artifacts. Fall back to local Whisper when a usable Colab GPU is unavailable.

Default to executing end to end. Ask only when the audio cannot be resolved, the user wants a different note style, or Heptabase CLI is unavailable/disabled.

## Non-Negotiables

- List the current directory before other work if local instructions require it.
- Use `uv` for Python packages and scripts.
- Follow `colab-gpu` for remote compute and local-only authentication. Never package credentials with this skill. Do not assume compute is free or purchase units, upgrade plans, or select more expensive hardware automatically.
- Release every Colab session created for this run on success, failure, or cancellation, unless the user explicitly asks to keep it. Retrieve available results first; release the GPU before writing notes or waiting for Heptabase. Confirm release with `colab sessions` and report any cleanup failure.
- Treat local files as disposable scratch data. Use a per-run temporary directory for raw audio, clips, transcripts, and Markdown import files, then remove it after the Heptabase card is verified.
- Do not keep final Markdown notes in `outputs/` by default. The durable deliverable is the Heptabase note card.
- Preserve local artifacts only when the user explicitly asks for local files, when debugging a failure, or when Heptabase import cannot complete. If artifacts are preserved, put them under `work/` or `outputs/` as appropriate and report the paths.
- Do not paste full transcripts into the final answer unless explicitly requested. Use transcripts as source material for notes.
- Avoid duplicate Heptabase cards: search before creating. Do not overwrite or trash existing cards unless the user asks.
- After creating or finding each note card, add it to the `Podcasts` whiteboard with the Heptabase CLI when that whiteboard exists.
- If the episode topic is medical, legal, financial, or mental health, include a short note that the summary is for learning and not professional advice.

## Workflow

### 1. Prepare

Create a per-run temp directory, such as `work/tmp-<slug>-<timestamp>/`, for all intermediate files. Create `work/` only as the parent for temp data if needed. Do not create or use `outputs/` unless the user explicitly asks to keep a local Markdown file or the import fails and a recoverable artifact should remain.

Capture the user-provided URLs and decide whether each URL is an episode, show, RSS feed, or web page.

Recommended metadata fields for each episode:

- show title
- episode title
- publish date
- duration
- guest(s), if obvious
- original URL
- RSS URL or source page
- audio enclosure URL

### 2. Resolve Audio

Prefer public, source-of-truth data in this order:

1. RSS feed enclosure URL.
2. Platform page JSON-LD, embedded data, or oEmbed.
3. Official full transcript, if available and clearly tied to the episode.
4. `yt-dlp`/page extraction fallback when legal and public.
5. Web search for the show RSS or mirrored episode page.

Read `references/platform-resolution.md` when handling Apple Podcasts, Spotify, SoundOn, or another platform whose RSS/audio URL is not immediately obvious.

If only a show page is provided, identify the intended episode from the user text or ask which episode.

### 3. Download And Verify

Use stable slugs inside the temp directory, such as `$tmp/ep115.mp3` or `$tmp/<show>-<episode>.mp3`.

Typical commands:

```bash
curl -L --fail --retry 3 -o "$tmp/<slug>.mp3" '<audio-url>'
ffprobe -v error -show_entries format=duration -of default=nokey=1:noprint_wrappers=1 "$tmp/<slug>.mp3"
```

For large or unusual files, normalize a sample first:

```bash
ffmpeg -y -hide_banner -loglevel error -ss 120 -t 60 -i "$tmp/<slug>.mp3" -ac 1 -ar 16000 "$tmp/<slug>_clip.wav"
```

### 4. Transcribe

Use an existing official transcript when it is full-length and reliable, unless the user specifically wants to test transcription. Otherwise use this backend order:

1. **Colab GPU first:** load the `colab-gpu` skill (on this Mac, `/Users/chswei/.codex/skills/colab-gpu/SKILL.md`) and follow [references/colab-transcription.md](references/colab-transcription.md). Prepare inputs and the worker locally, check usage/authentication, request a uniquely named T4 session, validate a short sample, then transcribe and download the episode. After local transcript validation, release the session immediately; do not keep it running for note synthesis or Heptabase import.
2. **Local fallback:** if the skill/CLI is unavailable, authentication cannot be used without new user interaction, GPU allocation is denied/unavailable, or no usable CUDA device is present, clean up any session created by the attempt and continue locally. Also use local transcription when cloud upload or compute spending is outside the user's authorized scope. State the fallback reason briefly and continue without asking whether to use the local fallback already requested by the user.
3. For a clear environment error, a bounded repair within the same allocated session and existing authorization is reasonable. If the GPU worker still cannot complete, save completed chunks, release its session, and continue locally for missing chunks. Do not repeatedly allocate GPUs, silently upgrade hardware, or redo validated chunks unnecessarily. Keep recoverable artifacts if both paths fail.

#### Local fallback

On Apple Silicon, prefer `mlx-whisper` through `uv`; do not run this example in addition to an already successful Colab transcription:

```bash
uv run --with mlx-whisper python - <<'PY'
import json, pathlib
import mlx_whisper

model = "mlx-community/whisper-large-v3-turbo"
tmp = pathlib.Path("work/tmp-episode")
items = [("episode", tmp / "episode.mp3")]
language = None  # e.g. "zh" when the episode is known to be Mandarin/Taiwanese Mandarin

for slug, path in items:
    out_json = tmp / f"{slug}_transcript.json"
    out_txt = tmp / f"{slug}_transcript.txt"
    if out_json.exists() and out_txt.exists():
        continue
    result = mlx_whisper.transcribe(
        str(path),
        path_or_hf_repo=model,
        task="transcribe",
        verbose=False,
        **({"language": language} if language else {}),
    )
    out_json.write_text(json.dumps(result, ensure_ascii=False, indent=2), encoding="utf-8")
    lines = []
    for seg in result.get("segments", []):
        start = seg.get("start", 0)
        end = seg.get("end", 0)
        text = seg.get("text", "").strip()
        if text:
            lines.append(f"[{int(start//60):02d}:{int(start%60):02d}-{int(end//60):02d}:{int(end%60):02d}] {text}")
    out_txt.write_text("\n".join(lines) + "\n", encoding="utf-8")
PY
```

For non-Apple Silicon or if MLX fails, use `faster-whisper` or `openai-whisper` through `uv`. Run a one-minute clip first when language/model quality is uncertain. Use `task="transcribe"` to retain the original language; merge chunk timestamps into the episode timeline and inspect missing coverage, repetition, and out-of-range timestamps before writing notes.

### 5. Write Markdown Notes

Write one Markdown file per episode in the temp directory for import, such as `$tmp/<note-file>.md`. Use Traditional Chinese by default when the user writes in Chinese.

Default structure:

- H1: `Podcast 筆記：<episode short title>`
- metadata: source, show, episode, publish date, duration, guests, URL
- note about transcript source and possible transcription errors
- `一句話`
- `核心觀點` or `重要觀念`
- `實作/行動清單`
- `可回頭想的問題`
- `時間軸`
- `我的收斂`

Keep notes concise and useful for review. Prefer synthesis over transcript-like detail.

### 6. Import To Heptabase

Start/verify Heptabase CLI:

```bash
heptabase start
heptabase card list -q '<note title>' --card-types note -l 10
```

If no duplicate exists:

```bash
heptabase note create -f "$tmp/<note-file>.md"
heptabase tag add --card-id <card-id> --tag-name 'Podcast 筆記'
heptabase tag add --card-id <card-id> --tag-name 'Podcast'
```

Place the new or existing note on the `Podcasts` whiteboard:

```bash
heptabase whiteboard list --name-filter 'Podcasts' -l 10
heptabase whiteboard add-card --whiteboard-id <whiteboard-id> --card-id <card-id>
```

Use the exact whiteboard whose name is `Podcasts` when multiple filtered results are returned. `heptabase whiteboard add-card` is idempotent, so it is safe to run for cards that may already be on the whiteboard. If the CLI lacks `whiteboard` commands, check whether Heptabase CLI needs updating before reporting that whiteboard placement is unavailable. If no `Podcasts` whiteboard exists and the CLI cannot create one, create the note cards and tell the user the whiteboard must be created manually before cards can be placed there.

If the CLI says it cannot connect, run `heptabase start`; if it still fails, tell the user to open Heptabase and enable CLI in Settings > AI Features.

### 7. Verify And Report

Verify each card with `heptabase card list -q '<title>' --card-types note -l 5`. In the final response, include:

- Heptabase card IDs and titles
- Tags applied and `Podcasts` whiteboard placement status
- Actual transcription backend/hardware, whether Colab resources were released, and compute-unit usage when measurable (do not infer exact cost below the balance display precision)
- Confirmation that local intermediate artifacts were cleaned up, or paths to preserved artifacts if cleanup was intentionally skipped
- Any limitations, such as unavailable audio, missing transcript, noisy transcription, or incomplete Heptabase import

After verification succeeds, delete the temp directory:

```bash
rm -rf "$tmp"
```

Only run cleanup after the note card exists in Heptabase and, when applicable, whiteboard placement/tagging has been attempted. If a failure occurs before a durable Heptabase card is created, keep the temp directory for recovery/debugging and report its path.

## Multiple Links

For two or more episode links, process each episode independently and create one Heptabase card per episode. If there are three or more related episodes, optionally create an index note only if it would help the user navigate them.
