<p align="center">
  <img src="openless-all/app/src-tauri/icons/128x128@2x.png" alt="OpenLess" width="128" height="128" />
</p>

<h1 align="center">Voice Input</h1>

<p align="center">
  <strong>Personal voice input for macOS, forked from OpenLess</strong>
</p>

<p align="center">
  Hold a hotkey, speak, and watch AI-polished text stream straight to your cursor —<br/>
  in the writing style <em>you</em> choose.
</p>

<p align="center">
  <a href="https://github.com/nickwun/voice-input"><strong>Repository</strong></a>
  &nbsp;·&nbsp;
  <a href="https://github.com/Open-Less/openless"><strong>Upstream OpenLess</strong></a>
</p>

<p align="center">
  <a href="https://github.com/nickwun/voice-input/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/badge/license-MIT-2f855a?style=flat-square" /></a>
</p>

<p align="center">
  <img alt="macOS" src="https://img.shields.io/badge/macOS-12%2B-1f425f?style=flat-square&logo=apple&logoColor=white" />
  <img alt="Windows" src="https://img.shields.io/badge/Windows-10%2B-0078d4?style=flat-square&logo=windows&logoColor=white" />
  <img alt="Tauri" src="https://img.shields.io/badge/Tauri-2-24c8db?style=flat-square&logo=tauri&logoColor=white" />
  <img alt="Rust" src="https://img.shields.io/badge/Rust-2021-ce422b?style=flat-square&logo=rust&logoColor=white" />
</p>

> This repository is the `nickwun/voice-input` adaptation of [Open-Less/openless](https://github.com/Open-Less/openless). The goal is a Typeless-like macOS voice input tool that supports personal prompts, bring-your-own Volcengine ASR, and DeepSeek/OpenAI-compatible polishing without relying on a shared SaaS quota.

## Current Adaptation

- Default ASR provider is Volcengine streaming ASR.
- Default LLM provider is DeepSeek.
- Default style is structured Chinese prompt cleanup.
- Automatic update checks are disabled until this fork has its own signed releases.
- User-facing app name and repository links point to Voice Input / `nickwun/voice-input`.

The imported upstream implementation still contains internal `OpenLess` identifiers, storage paths, and Windows/Linux subsystems. Those are intentionally left mostly intact for now so this fork can keep taking upstream fixes with low friction.

## Development

```bash
cd openless-all/app
npm ci
npm run build
cargo check --manifest-path src-tauri/Cargo.toml
```

See [docs/codex/openless-base-decision.md](docs/codex/openless-base-decision.md) for the base-project selection notes.
