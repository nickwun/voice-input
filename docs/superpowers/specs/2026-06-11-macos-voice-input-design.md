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
5. If automatic paste is enabled, simulate Command+V to paste into the currently focused input field.

The first version supports one editable default prompt. Multiple prompt profiles, provider fallback, and advanced history management are deferred.

The first speech provider is Volcengine/Doubao Big Model Recording File Flash Recognition. It uses:

- Endpoint: `POST https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash`
- Resource ID: `volc.bigasr.auc_turbo`
- Authentication: first-version settings use the newer `X-Api-Key` flow. Legacy app key/access key support is deferred.
- Required headers: `X-Api-Key`, `X-Api-Resource-Id: volc.bigasr.auc_turbo`, `X-Api-Request-Id: <UUID>`, and `X-Api-Sequence: -1`.
- Request body: upload local audio as base64 in `audio.data`, set `user.uid` to a stable locally generated UUID, and set `request.model_name` to `bigmodel`.
- Success parsing: require response header `X-Api-Status-Code` to equal `20000000`; capture `X-Tt-Logid` when present for debugging; read transcript from `result.text`.
- First-version audio format: mono WAV, 16 kHz, 16-bit PCM.

The first text refinement provider is DeepSeek Chat Completions. It uses:

- Endpoint: `POST https://api.deepseek.com/chat/completions`
- Authentication: `Authorization: Bearer <DeepSeek API key>`
- Model: `deepseek-v4-flash`
- Thinking mode: disabled, because the task is short text cleanup and should prioritize latency.
- Streaming: disabled for the first version.
- Request body: send the editable prompt as the system message and the transcript as the user message, with `thinking: {"type": "disabled"}` and `stream: false`.
- Success parsing: read refined text from `choices[0].message.content`.

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

Hotkey behavior is state-specific:

- `Idle`: start recording.
- `Recording`: stop recording and begin the provider workflow.
- `Transcribing`, `Refining`, or `Pasting`: ignore the hotkey and keep the current workflow running.
- `Failed`: clear the error and return to `Idle`.

The first version uses a Settings window, not only prompts or menu dialogs. Settings are needed for API keys, prompt editing, and the automatic paste toggle.

Automatic paste is enabled by default because the first version is meant to match the Typeless replacement workflow. If Accessibility permission is missing, the app copies to clipboard and explains how to enable automatic paste.

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

Core protocol boundaries:

```swift
enum VoiceInputError: Error, Equatable {
    case missingMicrophonePermission
    case missingAccessibilityPermission
    case missingSpeechCredentials
    case missingRefinementCredentials
    case recordingFailed(String)
    case transcriptionFailed(String)
    case refinementFailed(String)
    case pasteFailed(String)
    case emptyTranscript
    case cancelled
}

struct SpeechRecognitionRequest {
    let audioFileURL: URL
    let requestID: UUID
}

struct SpeechRecognitionResult: Equatable {
    let text: String
    let providerLogID: String?
}

protocol SpeechRecognitionProvider {
    func transcribe(_ request: SpeechRecognitionRequest) async throws -> SpeechRecognitionResult
}

struct TextRefinementRequest {
    let transcript: String
    let prompt: String
}

struct TextRefinementResult: Equatable {
    let text: String
}

protocol TextRefinementProvider {
    func refine(_ request: TextRefinementRequest) async throws -> TextRefinementResult
}

protocol AudioRecorder {
    func startRecording() async throws
    func stopRecording() async throws -> URL
    func cancelRecording() async
}

protocol ClipboardPaster {
    func copy(_ text: String)
    func pasteCopiedText() throws
}
```

`AppStateController` owns workflow ordering and converts thrown provider errors into user-visible states. Provider implementations do not update UI directly.

## Data Flow

1. User presses `Option+Space`.
2. `HotkeyController` asks `AppStateController` to start recording.
3. `AudioRecorder` records microphone input to a temporary file.
4. User presses `Option+Space` again.
5. `AudioRecorder` stops and returns the audio file URL.
6. `VolcengineSpeechProvider` uploads the audio and returns the raw transcript.
7. `DeepSeekRefinementProvider` sends the raw transcript plus the configured prompt and returns refined text.
8. `ClipboardPaster` writes the refined text to the clipboard.
9. If automatic paste is enabled, `ClipboardPaster` simulates `Command+V`; otherwise the workflow stops after copying.
10. Temporary audio is deleted.
11. App state returns to idle.

Temporary audio is deleted in a `defer`-style cleanup path after stop-recording succeeds, including transcription, refinement, paste, and cancellation failures.

## Settings

Store non-secret settings locally:

- Default refinement prompt.
- Automatic paste enabled or disabled.
- Stable anonymous local user ID for the Volcengine `user.uid` request field.

Store secrets in Keychain:

- Volcengine/Doubao speech API credentials.
- DeepSeek API key.

The most recent raw transcript and refined text are kept in memory only for the current app session so the menu can show the latest result. They are not persisted in the first version.

## Default Prompt

The built-in default prompt should lightly refine Chinese speech input:

- Remove filler words and repeated fragments.
- Add natural punctuation.
- Preserve the user's meaning and tone.
- Do not add new facts.
- Return only the final text.

The user can edit this prompt in settings.

The settings UI must make it clear that the prompt is sent to the configured refinement provider together with each transcript.

## Error Handling

The app handles these failures:

- Missing microphone permission: show a permission message and do not start recording.
- Missing Accessibility permission: copy result to clipboard, skip automatic paste, and show a message.
- Missing API credentials: ask the user to open settings.
- Hotkey registration failure or conflict: show a menu-bar error, keep the app running, and allow recording from the menu item.
- Speech recognition failure: show the provider error and stop the workflow.
- Text refinement failure: automatically copy the raw transcript instead, do not auto-paste it, and show a menu-bar notification that refinement failed but the raw transcript is available in the clipboard.
- Network or service failure: show a readable error and preserve any generated text.
- Empty transcript: show a short message and do not call the refinement provider.
- Recording timeout: if one recording exceeds 180 seconds, stop recording and continue with the captured audio.
- Provider timeout: speech recognition and text refinement each time out after 30 seconds.
- Cancellation: if the app quits or recording is cancelled, stop recording and delete temporary audio.

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
- The refined text is automatically pasted into common apps when Accessibility permission is enabled. The first acceptance targets are WeChat, Apple Notes, Safari text fields, and Chrome text fields.
- When automatic paste is not possible, the text remains available in the clipboard.
- Missing permissions and missing API keys produce readable guidance instead of crashes.

## Decisions For Implementation

- Use Volcengine/Doubao Big Model Recording File Flash Recognition for speech recognition.
- Use mono WAV, 16 kHz, 16-bit PCM for the first recording format.
- Use `Option+Space` as the first default hotkey.
- Enable automatic paste by default.
- Do not persist transcript history in the first version. The app may keep only the most recent result in memory until quit.

## References

- Volcengine/Doubao Big Model Recording File Flash Recognition API: https://www.volcengine.com/docs/6561/1631584
- DeepSeek API Quick Start: https://api-docs.deepseek.com/
- DeepSeek Chat Completions API: https://api-docs.deepseek.com/api/create-chat-completion
