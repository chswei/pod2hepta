# pod2hepta

Codex skill for turning podcast episode links into Heptabase note cards.

The skill lives at:

```text
.codex/skills/podcast-to-heptabase
```

## What It Does

- Resolves podcast episode audio from supported podcast platforms and RSS feeds.
- Downloads and verifies audio in a disposable per-run workspace.
- Transcribes episodes locally with Whisper-compatible tooling.
- Synthesizes concise Markdown study notes.
- Imports notes into Heptabase with the Heptabase CLI.
- Tags podcast notes and places them on the `Podcasts` whiteboard when available.

## Usage

Install or copy this repository's skill directory into a Codex skills path, then ask Codex to process one or more podcast episode links.

Example request:

```text
把這集 podcast 做成 Heptabase 筆記：https://example.com/episode
```

The skill defaults to Traditional Chinese notes when the request is written in Chinese.

## Requirements

- macOS
- Codex with local skills enabled
- `uv` for Python package and script execution
- `ffmpeg` / `ffprobe` for audio handling
- Heptabase CLI with CLI access enabled in Heptabase

## Repository Notes

Runtime artifacts such as downloaded audio, transcripts, and generated import Markdown are intentionally ignored. The durable output should be the Heptabase note card, not local files.
