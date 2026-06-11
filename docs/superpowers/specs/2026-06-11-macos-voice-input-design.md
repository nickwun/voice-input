# macOS Voice Input App Design

Date: 2026-06-11

## Goal

Build a native macOS menu bar app that replaces the user's current Typeless workflow for frequent Chinese voice input.

The first version focuses on two pain points:

- Customizable text refinement instead of fixed vendor behavior.
- More reliable conversion by using the user's own speech and LLM API credentials.

## First Version Scope

The app runs as a menu bar utility. A global hotkey starts recording, the same hotkey stops recording, and the app sends the audio through a provider pipeline:

1. Record microphone audio.
2. Send audio to the first speech recognition provider, initially Doubao/Volcengine.
3. Send the transcript to the first text refinement provider, initially DeepSeek.
4. Write the refined text to the clipboard.
5. Simulate Command+V to paste into the currently focused input field.

The first version supports one editable default prompt. Multiple prompt profiles, provider fallback, and advanced history management are deferred.

## User Experience

The app appears in the macOS menu bar and shows its current state:

- Idle
- Recording
- Transcribing
- Refining
- Pasting
- Failed

The menu includes:

- Start or stop recording
- Open settings
- Show the most recent result
- Quit

The default global hotkey is `Option+Space`. The first version may keep the hotkey fixed while keeping the code structured so editable hotkeys can be added later.

## Permissions

The app needs these macOS permissions:

- Microphone permission for recording.
- Accessibility permission for simulated paste.

If Accessibility permission is missing, the app still writes the result to the clipboard and shows a clear message that automatic paste needs Accessibility access.

## Architecture

Use a native Swift macOS app with small, focused components:

- `AppStateController`: owns the current app state and coordinates the workflow.
- `HotkeyController`: registers the global hotkey and sends start/stop events.
- `AudioRecorder`: records microphone input to a temporary audio file.
- `SpeechRecognitionProvider`: protocol for speech-to-text providers.
- `VolcengineSpeechProvider`: first speech recognition implementation.
- `TextRefinementProvider`: protocol for transcript refinement providers.
- `DeepSeekRefinementProvider`: first text refinement implementation.
- `ClipboardPaster`: writes text to the clipboard and triggers Command+V.
- `SettingsStore`: loads and saves non-secret preferences.
- `SecretsStore`: stores API keys in Keychain.
- `MenuBarController`: renders menu bar status and user commands.

Provider protocols keep vendor-specific code isolated so future providers can be added without changing the recording or paste workflow.

## Data Flow

1. User presses `Option+Space`.
2. `HotkeyController` asks `AppStateController` to start recording.
3. `AudioRecorder` records microphone input to a temporary file.
4. User presses `Option+Space` again.
5. `AudioRecorder` stops and returns the audio file URL.
6. `VolcengineSpeechProvider` uploads the audio and returns the raw transcript.
7. `DeepSeekRefinementProvider` sends the raw transcript plus the configured prompt and returns refined text.
8. `ClipboardPaster` writes the refined text to the clipboard.
9. `ClipboardPaster` simulates `Command+V`.
10. Temporary audio is deleted.
11. App state returns to idle.

## Settings

Store non-secret settings locally:

- Default refinement prompt.
- Automatic paste enabled or disabled.
- Last raw transcript.
- Last refined text.

Store secrets in Keychain:

- Volcengine/Doubao speech API credentials.
- DeepSeek API key.

## Default Prompt

The built-in default prompt should lightly refine Chinese speech input:

- Remove filler words and repeated fragments.
- Add natural punctuation.
- Preserve the user's meaning and tone.
- Do not add new facts.
- Return only the final text.

The user can edit this prompt in settings.

## Error Handling

The app handles these failures:

- Missing microphone permission: show a permission message and do not start recording.
- Missing Accessibility permission: copy result to clipboard, skip automatic paste, and show a message.
- Missing API credentials: ask the user to open settings.
- Speech recognition failure: show the provider error and stop the workflow.
- Text refinement failure: offer to use the raw transcript as the clipboard/paste result.
- Network or service failure: show a readable error and preserve any generated text.
- Empty transcript: show a short message and do not call the refinement provider.

The app should not crash or lose already generated text when an API call fails.

## Deferred Features

These are intentionally outside the first version:

- Multiple prompt profiles.
- Editable global hotkey UI.
- Provider fallback, such as Xunfei, Baidu, or local Whisper.
- Local offline transcription.
- Rich history search.
- Packaging, auto-update, and notarization.
- Team or cloud sync.

## Acceptance Criteria

The first version is successful when:

- The app can run as a macOS menu bar utility.
- The user can press the global hotkey from another app to start recording.
- The user can press the hotkey again to stop recording.
- A Chinese recording can be transcribed through the configured speech provider.
- The transcript can be refined through the configured DeepSeek prompt.
- The refined text is copied to the clipboard.
- The refined text is automatically pasted into common apps when Accessibility permission is enabled.
- When automatic paste is not possible, the text remains available in the clipboard.
- Missing permissions and missing API keys produce readable guidance instead of crashes.

## Open Questions For Implementation

- Confirm the exact Volcengine/Doubao speech API product and audio format requirements.
- Decide whether first-version recording format should be WAV, M4A, or provider-specific.
- Decide whether the default hotkey should be `Option+Space` or another combination if it conflicts with the user's current setup.
- Decide whether last transcript storage is enabled by default or opt-in for privacy.
