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

## Settings Window Lifecycle

The settings window is destroyed on close, not hidden, because a hidden window keeps its WebKit process (~120 MB) resident even though the window is rarely open.

**Implementation:** `lib.rs` (`build_main_window`, `show_main_window`, the `CloseRequested` and `ExitRequested` handlers), `shortcut/mod.rs` (`change_overlay_style_setting`)

- With a tray icon, closing the `main` window destroys it and `show_main_window` rebuilds it via `build_main_window` on demand. Without a tray (disabled or `--no-tray`) the window is hidden but kept alive, as upstream does
- The app routinely runs with no windows, so `RunEvent::ExitRequested { code: None }` is prevented; only an explicit exit (`code: Some`) ends the process
- The `recording_overlay` webview would also keep a WebKit process alive, so it is not created at startup while the overlay style is `None` (`initialize_core_logic` skips it) and `change_overlay_style_setting` creates it on demand when the style is enabled. Switching to `None` leaves the window untouched (neither hidden nor destroyed); destroying a live overlay window crashed the app (GTK layer shell), so its webview stays until restart

## Builds and CI

- Installers are unsigned. Windows shows a SmartScreen warning on first launch. On macOS, allow the first launch under System Settings → Privacy & Security → Open Anyway, or run `xattr -dr com.apple.quarantine /Applications/Handy.app`
- `tauri.conf.json` has `createUpdaterArtifacts` off and no Windows `signCommand`, since the fork has no signing secrets. `build.yml` has a separate unsigned macOS build step
- `fork-build.yml` is the only automatic build: it runs on push to `main` for macOS Apple Silicon only. Run it manually with "all-platforms" to also build macOS Intel, Linux and Windows
- `main-build.yml` and `nix-check.yml` (they need upstream's signing and Cachix secrets), `build-test.yml`, `pr-test-build.yml` and `release.yml` are manual-only

## Upstream Sync

`upstream-sync.yml` runs daily (or manually, with an optional `tag` input), merges the newest upstream `vX.Y.Z` tag into an `upstream-sync/<tag>` branch and opens a PR, or an issue listing the conflicted files. It never merges on its own. An optional `SYNC_TOKEN` PAT (repo + workflow scope) lets it touch workflow files and start checks.

To sync by hand: `git fetch upstream --tags && git merge vX.Y.Z`, resolve conflicts (usually `transcription.rs`, `settings.rs`, `lib.rs`), and keep the remote transcription code.

Never use "Discard commits" in GitHub's "Sync fork" button: it resets `main` to upstream and drops the remote transcription feature.
