# Voice Input Adaptation Notes

This fork uses OpenLess as the implementation base and adapts the first-run experience for the `nickwun/voice-input` project.

## Current Defaults

- Product name: `Voice Input`
- Bundle identifier: `com.nickwun.voice-input`
- Default ASR provider: `volcengine`
- Default LLM provider: `deepseek`
- Default style pack: `builtin.structured`
- Auto-update background check: disabled until this fork publishes signed update manifests
- Release/update links: `https://github.com/nickwun/voice-input`

## Intentional Compatibility Leftovers

Many internal identifiers still contain `OpenLess`, including storage directories, legacy credential migration paths, Windows IME integration, logs, tests, and upstream comments. They are left in place for this baseline import to reduce merge conflicts and avoid breaking mature platform-specific code.

Rename those internal identifiers only in a dedicated migration pass with explicit data-migration and platform tests.

## Next Product Tasks

1. Run the app on macOS and complete microphone/accessibility permission checks.
2. Validate a real Volcengine ASR credential and DeepSeek credential path.
3. Decide whether to keep the upstream style marketplace and remote-input features in this fork.
4. Replace or regenerate the icon before producing public builds.
