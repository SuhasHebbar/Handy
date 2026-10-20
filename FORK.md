# FORK.md

This repository is a fork of [cjpais/Handy](https://github.com/cjpais/Handy). `AGENTS.md` tracks upstream and is left unchanged; everything specific to this fork is documented here.

## Overview of fork changes

- `managers/transcription.rs` processes speech either with the local engine or with a remote server (see [Remote Transcription](#remote-transcription))
- Pipeline: Audio → VAD → local model (Whisper/Parakeet) or remote server → Text output → Clipboard/Paste
- The self-updater is always off, so the `HANDY_DISABLE_UPDATER` variable mentioned for Nix/NixOS in `AGENTS.md` has no extra effect here

## Remote Transcription

This fork can send recordings to an OpenAI-compatible server instead of loading a local model, for machines with little RAM. Settings → Models → "Transcription Server"; onboarding also offers it so no model download is required.

**Settings:** remote transcription server (URL, API key, local fallback, busy timeout)

**Implementation:** `managers/transcription.rs` (dispatch + HTTP), `settings.rs` (`remote_transcription_*` fields, `SecretString`), `shortcut/mod.rs` (setting commands, `mark_onboarding_completed`), `components/settings/RemoteTranscription.tsx` (UI)

- Requests: `POST {remote_transcription_url}/audio/transcriptions` (or `/translations` when translating), multipart WAV, optional `Authorization: Bearer` key; the URL includes `/v1`
- The remote dispatch lives at the top of `TranscriptionManager::complete()`, which both `transcribe()` and `transcribe_recording()` funnel into. On failure with fallback on, it calls `initiate_local_model_load()` and falls through to the existing local path — don't reimplement local decoding
- `initiate_model_load()` is a dispatcher: in remote mode it runs a background probe at recording start (a silent clip against the busy timeout); unreachable/busy preloads the local model, and a later successful remote request unloads it
- Remote mode needs no local model to start recording (`actions.rs` skips the "can anything transcribe" gate). Fallback does: `change_remote_transcription_fallback_setting` refuses to enable it without a downloaded selected model, enabling remote with none leaves it off, and deleting the selected model turns it off
- Remote mode disables live streaming (`actions.rs`) and eager local model loads on model selection (`commands/models.rs`)
- The API key is a `SecretString`, so `Debug`-logging settings never prints it
- The self-updater is forced off (`update_checks_forced_disabled()` always returns `true`), so upstream releases can't replace this build and drop the feature
