# Colab-first podcast transcription

Load the independent `colab-gpu` skill for current CLI commands, authentication, compute authorization, and session management. This reference adds only the podcast-specific lifecycle. If that skill or CLI is unavailable, use the local fallback in `SKILL.md`; do not block podcast notes on installing cloud tools.

## Prepare before allocating

- Resolve the official audio enclosure, download it, verify duration, and retain the original in the run's temporary directory.
- Prepare a short representative sample and a remote worker. Whisper `turbo` with `task="transcribe"` is a suitable starting point; use a known source language or language detection. Keep the output as timestamped original-language segments, not a translation or invented speaker labels.
- For long episodes, use manageable chunks (for example, about ten minutes), with stable names, source offsets, and individual JSON checkpoints. Save the chunk manifest locally so a different backend can finish missing chunks without shifting the timeline.
- Prepare required packages and worker code locally. On the VM use uv to create a Python environment, manage dependencies, and run the worker; check CUDA from that actual environment. A T4 label alone does not prove the installed model uses the GPU.
- Use current `colab usage` and `colab sessions`, not remembered balances. A request to process this episode through the default workflow permits its necessary audio transfer and the compute within the session's authorized scope; follow `colab-gpu` if a spending limit or permission is genuinely unresolved. Do not repeatedly ask for permissions already given. Installing or editing the skill alone does not authorize a billable test.

## Run, retrieve, and always release

Use a unique session name for this episode, explicit `-s` on every session-specific command, and a cleanup path established before allocation. The lifecycle is:

```text
Record existing sessions and this run's unique session name.
try:
    Attempt to allocate a T4 session.
    Confirm hardware, CUDA availability, and current usage rate.
    Install dependencies and validate a representative sample.
    Transcribe only missing/unvalidated chunks.
    Download checkpoint JSONs as they finish.
    Download and validate all required transcription results.
finally:
    On error/cancellation, try to retrieve completed checkpoints first.
    Stop this run's session if it was created.
    If allocation failed ambiguously, reconcile sessions before assuming none exists.
    Confirm this run's session is absent from colab sessions.
Continue with note synthesis, or local fallback for missing audio.
```

Implement cleanup with a real `try/finally` in an orchestrator when writing one, or perform the same sequence explicitly across tool calls. Do not rely on `&& colab stop`, which skips cleanup after an earlier failure. Do not stop unrelated pre-existing sessions. If shutdown cannot be confirmed, report the exact owned session name and that resources may remain active; retaining a local checkpoint does not stop a VM.

For a workflow needing dependencies, audio upload, and downloaded JSON, use explicit `new → upload/install → exec → download → stop`. A bare `colab run` tears down before a separate download can retrieve files; use it only when its output handling truly satisfies the task. Run `colab <command> --help` for installed-version syntax. A CLI output-wait timeout is not a total job deadline; on timeout check whether work is still running before resubmitting.

Stream child process stdout/stderr back through the notebook kernel (for example with `Popen` and explicit printing) if the worker launches a uv subprocess. Otherwise the CLI may show no progress until the child exits. Keep monitoring and cleanup possible while the model downloads or processes audio.

## Validate before synthesis

- Require readable nonempty JSON, the expected language/task, and coverage through the episode's final spoken material. Inspect sample text and suspicious segments; do not claim word-for-word human verification unless performed.
- Restore source offsets when merging chunks. Do not blindly assume every chunk is exactly 600 seconds. Some segmented FLAC files report duration `N/A` and expose packet timestamps in original-episode time: chunk duration is the final packet end **minus the first packet start**, not the absolute final timestamp. Decoded sample counts provide another valid duration source.
- Check segment bounds, overlaps, missing coverage, repeated text and hallucinations near silence or chunk boundaries. Discard segments entirely outside the real audio bounds, cap overlong end timestamps, and recheck uncertain boundary speech as needed; do not let hallucinated tails enter the notes as facts.
- Preserve validated checkpoints if the remote worker fails. After releasing the GPU, transcribe only missing/invalid chunks locally, retaining the same manifest offsets and common segment schema.
- Record the actual backend, model, hardware and before/after unit balance for the final report. Treat observed balance changes as approximate at the CLI's display precision, and never promise a fixed free quota or rate.

## Fallback conditions

Use local transcription without seeking an additional fallback confirmation when T4 allocation is unavailable/denied, no usable GPU exists, the cloud path is not installed/authenticated, or authorized cloud processing cannot proceed. Do not bypass quota controls, switch accounts, buy units, or escalate hardware. Explain the reason briefly and proceed with the existing local Whisper route.

GPU release and local scratch cleanup are separate: release cloud resources immediately after transcription; remove local audio/transcripts/import drafts only after the Heptabase note is durable and placement/tagging is verified. Preserve the run directory on unrecovered failure.
