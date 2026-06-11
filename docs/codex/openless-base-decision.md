# OpenLess Base Decision

Date: 2026-06-11

## Decision

Use [Open-Less/openless](https://github.com/Open-Less/openless) as the implementation base for this project instead of continuing with the from-scratch Swift Package plan.

## Why

OpenLess already implements most of the requested Typeless replacement workflow:

- macOS support.
- Global hotkey dictation.
- Speech-to-text pipeline.
- AI polishing with configurable prompts/style packs.
- Cursor insertion with clipboard fallback.
- Volcengine ASR support.
- DeepSeek/OpenAI-compatible chat-completions style polishing.
- OS credential vault storage, including macOS Keychain.
- MIT license.

This reduces duplicated work compared with building the full macOS recording, hotkey, paste, settings, credentials, and provider pipeline from scratch.

## Import Details

- Imported upstream repository: `Open-Less/openless`.
- Imported upstream branch snapshot: `beta`.
- Imported qwen-asr submodule: `Open-Less/qwen-asr` at `b00b789`.
- Local feature branch: `codex/openless-base`.
- Local worktree: `.worktrees/openless-base`.

## Baseline Verification

Commands run from `openless-all/app` or repository root:

```bash
npm ci
npm run build
cargo check --manifest-path openless-all/app/src-tauri/Cargo.toml
```

Results:

- `npm ci`: passed.
- `npm run build`: passed, with Vite chunk-size/dynamic-import warnings.
- `cargo check`: initially failed because `Coordinator.retranscribe_pcm` did not handle `ActiveAsr::AppleSpeech` on macOS.
- Added the missing Apple Speech match arm using the same dynamic timeout pattern as the existing local Qwen/QA paths.
- `cargo check`: passed after the fix, with existing unused-code warnings.

## Next Adaptation Steps

1. Rename/rebrand app identity for this project.
2. Simplify the UI to focus on voice input, credentials, prompts, hotkey, and insertion.
3. Set preferred defaults for the user's workflow:
   - Volcengine/Doubao ASR.
   - DeepSeek-compatible polishing.
   - Chinese light-cleanup prompt.
   - Clipboard fallback enabled.
4. Decide whether to keep OpenLess streaming Volcengine ASR or add the flash recording-file API from the original design spec.
5. Remove or hide nonessential surfaces such as marketplace, QA panel, and complex history if they slow down the first usable version.

