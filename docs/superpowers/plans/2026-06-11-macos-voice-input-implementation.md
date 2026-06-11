# macOS Voice Input Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first usable native macOS voice input app: global hotkey recording, Volcengine/Doubao transcription, DeepSeek refinement, clipboard copy, and optional automatic paste.

**Architecture:** Use a Swift Package with a testable `VoiceInputCore` library and a small `VoiceInput` executable that starts an AppKit menu bar app. Keep system adapters, provider clients, state orchestration, and UI controllers in separate files so provider and workflow behavior can be tested without microphone, network, or Accessibility permissions.

**Tech Stack:** Swift 6.3, Swift Package Manager, AppKit, AVFoundation, Carbon hotkeys, Security/Keychain, Foundation `URLSession`, XCTest.

---

## Chunk 1: Testable Foundation

### File Structure

- Create: `Package.swift` - package manifest with `VoiceInputCore`, `VoiceInput`, and `VoiceInputCoreTests`.
- Create: `Sources/VoiceInput/main.swift` - executable entry point that starts the AppKit app.
- Create: `Sources/VoiceInputCore/App/VoiceInputState.swift` - app states and shared error type.
- Create: `Sources/VoiceInputCore/App/AppStateController.swift` - workflow orchestration.
- Create: `Sources/VoiceInputCore/Audio/AudioRecorder.swift` - recording protocol and request types.
- Create: `Sources/VoiceInputCore/Providers/SpeechRecognitionProvider.swift` - speech provider protocol and result types.
- Create: `Sources/VoiceInputCore/Providers/TextRefinementProvider.swift` - refinement provider protocol and result types.
- Create: `Sources/VoiceInputCore/System/ClipboardPaster.swift` - clipboard protocol.
- Create: `Sources/VoiceInputCore/System/TemporaryFileCleaner.swift` - injectable temp-file cleanup protocol.
- Create: `Sources/VoiceInputCore/Config/AppSettings.swift` - settings model and defaults.
- Test: `Tests/VoiceInputCoreTests/AppStateControllerTests.swift`
- Test: `Tests/VoiceInputCoreTests/AppSettingsTests.swift`

### Task 1: Scaffold The Swift Package

**Files:**
- Create: `Package.swift`
- Create: `Sources/VoiceInput/main.swift`
- Create: `Sources/VoiceInputCore/App/VoiceInputState.swift`

- [ ] **Step 1: Write the package manifest**

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "VoiceInput",
    platforms: [.macOS(.v14)],
    products: [
        .library(name: "VoiceInputCore", targets: ["VoiceInputCore"]),
        .executable(name: "VoiceInput", targets: ["VoiceInput"])
    ],
    targets: [
        .target(name: "VoiceInputCore"),
        .executableTarget(name: "VoiceInput", dependencies: ["VoiceInputCore"]),
        .testTarget(name: "VoiceInputCoreTests", dependencies: ["VoiceInputCore"])
    ]
)
```

- [ ] **Step 2: Add a minimal executable**

```swift
import AppKit
import VoiceInputCore

let app = NSApplication.shared
app.setActivationPolicy(.accessory)
NSApp.run()
```

- [ ] **Step 3: Add state and error types**

```swift
public enum VoiceInputState: Equatable {
    case idle
    case recording
    case transcribing
    case refining
    case pasting
    case failed(String)
}

public enum VoiceInputError: Error, Equatable {
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
```

- [ ] **Step 4: Verify the package builds**

Run: `swift build`

Expected: build succeeds and creates `.build/debug/VoiceInput`.

- [ ] **Step 5: Commit**

```bash
git add Package.swift Sources
git commit -m "Scaffold Swift voice input package"
```

### Task 2: Add Core Protocols And Settings Defaults

**Files:**
- Create: `Sources/VoiceInputCore/Audio/AudioRecorder.swift`
- Create: `Sources/VoiceInputCore/Providers/SpeechRecognitionProvider.swift`
- Create: `Sources/VoiceInputCore/Providers/TextRefinementProvider.swift`
- Create: `Sources/VoiceInputCore/System/ClipboardPaster.swift`
- Create: `Sources/VoiceInputCore/System/TemporaryFileCleaner.swift`
- Create: `Sources/VoiceInputCore/Config/AppSettings.swift`
- Test: `Tests/VoiceInputCoreTests/AppSettingsTests.swift`

- [ ] **Step 1: Write failing settings tests**

```swift
import XCTest
@testable import VoiceInputCore

final class AppSettingsTests: XCTestCase {
    func testDefaultsMatchSpec() {
        let settings = AppSettings.default

        XCTAssertEqual(settings.globalHotkeyDescription, "Option+Space")
        XCTAssertTrue(settings.automaticPasteEnabled)
        XCTAssertFalse(settings.defaultPrompt.isEmpty)
    }
}
```

- [ ] **Step 2: Run the failing test**

Run: `swift test --filter AppSettingsTests`

Expected: FAIL because `AppSettings` does not exist yet.

- [ ] **Step 3: Implement protocols and settings**

`Sources/VoiceInputCore/Providers/SpeechRecognitionProvider.swift`:

```swift
import Foundation

public struct SpeechRecognitionRequest {
    public let audioFileURL: URL
    public let requestID: UUID
}

public struct SpeechRecognitionResult: Equatable {
    public let text: String
    public let providerLogID: String?
}

public protocol SpeechRecognitionProvider {
    func transcribe(_ request: SpeechRecognitionRequest) async throws -> SpeechRecognitionResult
}
```

`Sources/VoiceInputCore/Providers/TextRefinementProvider.swift`:

```swift
import Foundation

public struct TextRefinementRequest {
    public let transcript: String
    public let prompt: String
}

public struct TextRefinementResult: Equatable {
    public let text: String
}

public protocol TextRefinementProvider {
    func refine(_ request: TextRefinementRequest) async throws -> TextRefinementResult
}
```

`Sources/VoiceInputCore/Audio/AudioRecorder.swift`:

```swift
import Foundation

public protocol AudioRecorder {
    func startRecording() async throws
    func stopRecording() async throws -> URL
    func cancelRecording() async
}
```

`Sources/VoiceInputCore/System/ClipboardPaster.swift`:

```swift
public protocol ClipboardPaster {
    func copy(_ text: String)
    func pasteCopiedText() throws
}
```

`Sources/VoiceInputCore/System/TemporaryFileCleaner.swift`:

```swift
import Foundation

public protocol TemporaryFileCleaner {
    func removeTemporaryFile(at url: URL)
}
```

`Sources/VoiceInputCore/Config/AppSettings.swift`:

```swift
import Foundation

public struct AppSettings: Equatable {
    public var defaultPrompt: String
    public var automaticPasteEnabled: Bool
    public var localUserID: String
    public var globalHotkeyDescription: String

    public static let defaultPromptText = """
    你是一个中文语音输入整理助手。请去掉口头语和重复片段，补充自然标点，保持原意和语气，不添加新事实，只输出最终文本。
    """

    public static var `default`: AppSettings {
        AppSettings(
            defaultPrompt: defaultPromptText,
            automaticPasteEnabled: true,
            localUserID: UUID().uuidString,
            globalHotkeyDescription: "Option+Space"
        )
    }
}
```

- [ ] **Step 4: Run tests**

Run: `swift test --filter AppSettingsTests`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources/VoiceInputCore Tests/VoiceInputCoreTests
git commit -m "Add core voice input protocols"
```

### Task 3: Implement Workflow Orchestration With Fakes

**Files:**
- Create: `Sources/VoiceInputCore/App/AppStateController.swift`
- Modify: `Sources/VoiceInputCore/System/TemporaryFileCleaner.swift`
- Test: `Tests/VoiceInputCoreTests/AppStateControllerTests.swift`

- [ ] **Step 1: Write failing workflow tests**

```swift
import XCTest
@testable import VoiceInputCore

final class AppStateControllerTests: XCTestCase {
    func testSuccessfulWorkflowCopiesAndPastesRefinedText() async throws {
        let recorder = FakeRecorder(audioURL: URL(fileURLWithPath: "/tmp/test.wav"))
        let speech = FakeSpeechProvider(result: .init(text: "今天这个产品挺好用", providerLogID: "log-1"))
        let refinement = FakeRefinementProvider(result: .init(text: "今天这个产品挺好用。"))
        let paster = FakeClipboardPaster()
        let controller = AppStateController(
            recorder: recorder,
            speechProvider: speech,
            refinementProvider: refinement,
            clipboardPaster: paster,
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: .default
        )

        try await controller.handleHotkey()
        try await controller.handleHotkey()

        XCTAssertEqual(paster.copiedText, "今天这个产品挺好用。")
        XCTAssertTrue(paster.didPaste)
        XCTAssertEqual(controller.lastRawTranscript, "今天这个产品挺好用")
        XCTAssertEqual(controller.lastRefinedText, "今天这个产品挺好用。")
        XCTAssertEqual(controller.state, .idle)
    }

    func testRefinementFailureCopiesRawTranscriptWithoutPasting() async throws {
        var settings = AppSettings.default
        settings.automaticPasteEnabled = true
        let paster = FakeClipboardPaster()
        let controller = AppStateController(
            recorder: FakeRecorder(audioURL: URL(fileURLWithPath: "/tmp/test.wav")),
            speechProvider: FakeSpeechProvider(result: .init(text: "原始文本", providerLogID: nil)),
            refinementProvider: FakeRefinementProvider(error: .refinementFailed("boom")),
            clipboardPaster: paster,
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: settings
        )

        try await controller.handleHotkey()
        try await controller.handleHotkey()

        XCTAssertEqual(paster.copiedText, "原始文本")
        XCTAssertFalse(paster.didPaste)
    }

    func testAutomaticPasteDisabledOnlyCopies() async throws {
        var settings = AppSettings.default
        settings.automaticPasteEnabled = false
        let paster = FakeClipboardPaster()
        let controller = AppStateController(
            recorder: FakeRecorder(audioURL: URL(fileURLWithPath: "/tmp/test.wav")),
            speechProvider: FakeSpeechProvider(result: .init(text: "原始文本", providerLogID: nil)),
            refinementProvider: FakeRefinementProvider(result: .init(text: "整理文本")),
            clipboardPaster: paster,
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: settings
        )

        try await controller.handleHotkey()
        try await controller.handleHotkey()

        XCTAssertEqual(paster.copiedText, "整理文本")
        XCTAssertFalse(paster.didPaste)
    }

    func testPasteFailureKeepsCopiedTextAndEndsFailed() async throws {
        let paster = FakeClipboardPaster(pasteError: .missingAccessibilityPermission)
        let controller = AppStateController(
            recorder: FakeRecorder(audioURL: URL(fileURLWithPath: "/tmp/test.wav")),
            speechProvider: FakeSpeechProvider(result: .init(text: "原始文本", providerLogID: nil)),
            refinementProvider: FakeRefinementProvider(result: .init(text: "整理文本")),
            clipboardPaster: paster,
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: .default
        )

        try await controller.handleHotkey()
        try await controller.handleHotkey()

        XCTAssertEqual(paster.copiedText, "整理文本")
        XCTAssertEqual(controller.state, .failed("missingAccessibilityPermission"))
    }

    func testTemporaryAudioIsRemovedOnSuccessAndFailurePaths() async throws {
        let audioURL = URL(fileURLWithPath: "/tmp/test.wav")
        let scenarios: [(String, FakeSpeechProvider, FakeRefinementProvider, FakeClipboardPaster)] = [
            ("success", FakeSpeechProvider(result: .init(text: "原始文本", providerLogID: nil)), FakeRefinementProvider(result: .init(text: "整理文本")), FakeClipboardPaster()),
            ("transcription failure", FakeSpeechProvider(error: .transcriptionFailed("network")), FakeRefinementProvider(result: .init(text: "unused")), FakeClipboardPaster()),
            ("empty transcript", FakeSpeechProvider(result: .init(text: " ", providerLogID: nil)), FakeRefinementProvider(result: .init(text: "unused")), FakeClipboardPaster()),
            ("refinement failure", FakeSpeechProvider(result: .init(text: "原始文本", providerLogID: nil)), FakeRefinementProvider(error: .refinementFailed("boom")), FakeClipboardPaster()),
            ("paste failure", FakeSpeechProvider(result: .init(text: "原始文本", providerLogID: nil)), FakeRefinementProvider(result: .init(text: "整理文本")), FakeClipboardPaster(pasteError: .missingAccessibilityPermission))
        ]

        for (_, speech, refinement, paster) in scenarios {
            let cleaner = FakeTemporaryFileCleaner()
            let controller = AppStateController(
                recorder: FakeRecorder(audioURL: audioURL),
                speechProvider: speech,
                refinementProvider: refinement,
                clipboardPaster: paster,
                temporaryFileCleaner: cleaner,
                settings: .default
            )

            try await controller.handleHotkey()
            try await controller.handleHotkey()

            XCTAssertEqual(cleaner.removedURLs, [audioURL])
        }
    }

    func testEmptyTranscriptFailsBeforeRefinement() async throws {
        let refinement = FakeRefinementProvider(result: .init(text: "should not run"))
        let controller = AppStateController(
            recorder: FakeRecorder(audioURL: URL(fileURLWithPath: "/tmp/test.wav")),
            speechProvider: FakeSpeechProvider(result: .init(text: "   ", providerLogID: nil)),
            refinementProvider: refinement,
            clipboardPaster: FakeClipboardPaster(),
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: .default
        )

        try await controller.handleHotkey()
        try await controller.handleHotkey()

        XCTAssertEqual(controller.state, .failed("emptyTranscript"))
        XCTAssertFalse(refinement.didRefine)
    }

    func testFailedStateHotkeyReturnsToIdle() async throws {
        let controller = AppStateController(
            recorder: FakeRecorder(startError: .recordingFailed("no mic")),
            speechProvider: FakeSpeechProvider(result: .init(text: "unused", providerLogID: nil)),
            refinementProvider: FakeRefinementProvider(result: .init(text: "unused")),
            clipboardPaster: FakeClipboardPaster(),
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: .default
        )

        try await controller.handleHotkey()
        XCTAssertNotEqual(controller.state, .idle)
        try await controller.handleHotkey()
        XCTAssertEqual(controller.state, .idle)
    }

    func testHotkeyIsIgnoredWhileProcessing() async throws {
        let speech = SlowSpeechProvider(result: .init(text: "原始文本", providerLogID: nil))
        let recorder = FakeRecorder(audioURL: URL(fileURLWithPath: "/tmp/test.wav"))
        let controller = AppStateController(
            recorder: recorder,
            speechProvider: speech,
            refinementProvider: FakeRefinementProvider(result: .init(text: "整理文本")),
            clipboardPaster: FakeClipboardPaster(),
            temporaryFileCleaner: FakeTemporaryFileCleaner(),
            settings: .default
        )

        try await controller.handleHotkey()
        let processing = Task { try await controller.handleHotkey() }
        try await Task.sleep(nanoseconds: 10_000_000)
        try await controller.handleHotkey()
        speech.release()
        try await processing.value

        XCTAssertEqual(recorder.stopCallCount, 1)
    }
}
```

- [ ] **Step 2: Run failing workflow tests**

Run: `swift test --filter AppStateControllerTests`

Expected: FAIL because `AppStateController` and fakes do not exist.

- [ ] **Step 3: Implement `AppStateController`**

Implement:

- `public private(set) var state: VoiceInputState`
- `public private(set) var lastRawTranscript: String?`
- `public private(set) var lastRefinedText: String?`
- `public func handleHotkey() async throws`
- `public func startRecording() async throws`
- `public func stopRecordingAndProcess() async throws`

Required behavior:

- `idle` hotkey starts recording.
- `recording` hotkey stops recording and runs the provider pipeline.
- `transcribing`, `refining`, and `pasting` hotkeys are ignored.
- `failed` hotkey clears to `idle`.
- Empty speech result throws/records `.emptyTranscript`.
- Refinement failure copies raw transcript and skips automatic paste.
- Automatic paste toggle is respected.
- Paste failure after a successful copy leaves copied text in the clipboard and moves to `failed`.
- Temporary audio is removed through `TemporaryFileCleaner` in a `defer` path after `stopRecording()` returns a URL, regardless of transcription, refinement, paste, or empty-transcript failures.
- Expected user-facing workflow failures are recorded in `state = .failed(...)`; `handleHotkey()` should not crash the app or leak handled provider/permission errors to the AppKit event loop.

- [ ] **Step 4: Add test fakes inside the test file**

Keep fakes local to tests with these minimum members:

- `FakeRecorder`: initializer `init(audioURL: URL? = nil, startError: VoiceInputError? = nil, stopError: VoiceInputError? = nil)`, records `didStart`, throws `startError` from `startRecording`, returns `audioURL` from `stopRecording`, and throws `stopError` from `stopRecording` when provided.
- `FakeRecorder` also tracks `stopCallCount`.
- `FakeSpeechProvider`: initializer `init(result: SpeechRecognitionResult? = nil, error: VoiceInputError? = nil)`, records `didTranscribe`, returns `result` or throws `error`.
- `SlowSpeechProvider`: suspends `transcribe` until `release()` is called so processing-state hotkey behavior can be tested.
- `FakeRefinementProvider`: initializer `init(result: TextRefinementResult? = nil, error: VoiceInputError? = nil)`, records `didRefine`, returns `result` or throws `error`.
- `FakeClipboardPaster`: initializer `init(pasteError: VoiceInputError? = nil)`, stores `copiedText`, sets `didPaste`, throws `pasteError` from `pasteCopiedText`.
- `FakeTemporaryFileCleaner`: stores `removedURLs` in call order.

- [ ] **Step 5: Run tests**

Run: `swift test --filter AppStateControllerTests`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add Sources/VoiceInputCore/App Tests/VoiceInputCoreTests/AppStateControllerTests.swift
git commit -m "Add voice input workflow controller"
```

---

## Chunk 2: Providers, macOS Adapters, UI, And Manual Verification

### File Structure

- Create: `Sources/VoiceInputCore/Providers/VolcengineSpeechProvider.swift`
- Create: `Sources/VoiceInputCore/Providers/DeepSeekRefinementProvider.swift`
- Create: `Sources/VoiceInputCore/Config/SettingsStore.swift`
- Create: `Sources/VoiceInputCore/Config/SecretsStore.swift`
- Create: `Sources/VoiceInputCore/System/MacClipboardPaster.swift`
- Create: `Sources/VoiceInputCore/System/MacHotkeyController.swift`
- Create: `Sources/VoiceInputCore/System/FileManagerTemporaryFileCleaner.swift`
- Create: `Sources/VoiceInputCore/Audio/AVFoundationAudioRecorder.swift`
- Create: `Sources/VoiceInputCore/UI/MenuBarController.swift`
- Create: `Sources/VoiceInputCore/UI/SettingsWindowController.swift`
- Modify: `Sources/VoiceInput/main.swift`
- Create: `Tests/VoiceInputCoreTests/VolcengineSpeechProviderTests.swift`
- Create: `Tests/VoiceInputCoreTests/DeepSeekRefinementProviderTests.swift`
- Create: `Tests/VoiceInputCoreTests/SettingsStoreTests.swift`
- Create: `Tests/VoiceInputCoreTests/SecretsStoreTests.swift`
- Create: `docs/manual-verification.md`
- Modify: `README.md`

### Task 4: Implement Provider HTTP Clients

**Files:**
- Create: `Sources/VoiceInputCore/Providers/VolcengineSpeechProvider.swift`
- Create: `Sources/VoiceInputCore/Providers/DeepSeekRefinementProvider.swift`
- Test: `Tests/VoiceInputCoreTests/VolcengineSpeechProviderTests.swift`
- Test: `Tests/VoiceInputCoreTests/DeepSeekRefinementProviderTests.swift`

- [ ] **Step 1: Write failing Volcengine request tests**

Verify:

- URL is `https://openspeech.bytedance.com/api/v3/auc/bigmodel/recognize/flash`.
- Headers include `X-Api-Key`, `X-Api-Resource-Id`, `X-Api-Request-Id`, and `X-Api-Sequence: -1`.
- `X-Api-Resource-Id` is exactly `volc.bigasr.auc_turbo`.
- JSON body includes stable configured `user.uid`, `audio.data`, and `request.model_name: bigmodel`.
- `user.uid` equals the `AppSettings.localUserID` value passed into the provider, not a new UUID per request.
- Success requires `X-Api-Status-Code: 20000000`.
- Transcript is parsed from `result.text`.
- A request that exceeds 30 seconds maps to `.transcriptionFailed`.

- [ ] **Step 2: Write failing DeepSeek request tests**

Verify:

- URL is `https://api.deepseek.com/chat/completions`.
- Header includes `Authorization: Bearer <key>`.
- JSON body includes `model: deepseek-v4-flash`, `thinking.type: disabled`, and `stream: false`.
- System message is the editable prompt.
- User message is the transcript.
- Refined text is parsed from `choices[0].message.content`.
- A request that exceeds 30 seconds maps to `.refinementFailed`.

- [ ] **Step 3: Run failing provider tests**

Run: `swift test --filter VolcengineSpeechProviderTests && swift test --filter DeepSeekRefinementProviderTests`

Expected: FAIL because provider implementations do not exist.

- [ ] **Step 4: Implement provider clients with injectable `URLSession`**

Use `URLSessionProtocol` or a small `HTTPClient` protocol so tests can stub responses without real network calls.

Implementation rules:

- Do not log API keys.
- Base64 encode the WAV file bytes.
- Trim returned text.
- Map provider errors to `.transcriptionFailed` or `.refinementFailed`.
- Include Volcengine `X-Tt-Logid` in `SpeechRecognitionResult.providerLogID`.
- Enforce a 30-second timeout per provider call by configuring the injected HTTP client/session timeout or wrapping the request in a timeout helper.

- [ ] **Step 5: Run provider tests**

Run: `swift test --filter VolcengineSpeechProviderTests && swift test --filter DeepSeekRefinementProviderTests`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add Sources/VoiceInputCore/Providers Tests/VoiceInputCoreTests/*ProviderTests.swift
git commit -m "Add speech and refinement providers"
```

### Task 5: Implement Settings And Secrets Storage

**Files:**
- Create: `Sources/VoiceInputCore/Config/SettingsStore.swift`
- Create: `Sources/VoiceInputCore/Config/SecretsStore.swift`
- Test: `Tests/VoiceInputCoreTests/SettingsStoreTests.swift`
- Test: `Tests/VoiceInputCoreTests/SecretsStoreTests.swift`

- [ ] **Step 1: Write failing settings store tests**

Test that:

- Missing settings load `AppSettings.default`.
- Saved prompt and automatic paste setting can be reloaded.
- Local user ID is created once and then reused.
- Transcript history is not persisted.

- [ ] **Step 2: Write failing secrets store tests**

Test that:

- Missing secrets return `.missingSpeechCredentials` or `.missingRefinementCredentials` through a validation helper.
- Saved Volcengine and DeepSeek keys can be reloaded.
- Empty strings are treated as missing credentials.
- Tests use an isolated Keychain service/account prefix so they do not touch real user credentials.

- [ ] **Step 3: Implement `SettingsStore` using `UserDefaults`**

Expose:

```swift
public protocol SettingsStore {
    func load() -> AppSettings
    func save(_ settings: AppSettings)
}
```

- [ ] **Step 4: Implement `SecretsStore` using Keychain**

Expose:

```swift
public struct VoiceInputSecrets: Equatable {
    public var volcengineAPIKey: String
    public var deepSeekAPIKey: String
}

public protocol SecretsStore {
    func load() throws -> VoiceInputSecrets
    func save(_ secrets: VoiceInputSecrets) throws
}
```

- [ ] **Step 5: Run tests**

Run: `swift test --filter SettingsStoreTests && swift test --filter SecretsStoreTests`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add Sources/VoiceInputCore/Config Tests/VoiceInputCoreTests/SettingsStoreTests.swift Tests/VoiceInputCoreTests/SecretsStoreTests.swift
git commit -m "Add settings and secrets storage"
```

### Task 6: Implement macOS System Adapters

**Files:**
- Create: `Sources/VoiceInputCore/Audio/AVFoundationAudioRecorder.swift`
- Create: `Sources/VoiceInputCore/System/MacClipboardPaster.swift`
- Create: `Sources/VoiceInputCore/System/MacHotkeyController.swift`
- Create: `Sources/VoiceInputCore/System/FileManagerTemporaryFileCleaner.swift`

- [ ] **Step 1: Implement `AVFoundationAudioRecorder`**

Requirements:

- Request/check microphone permission.
- Record mono WAV, 16 kHz, 16-bit PCM.
- Write to a temporary file.
- Stop automatically at 180 seconds.
- Return the temporary file URL to `AppStateController`; cleanup remains owned by the controller through `TemporaryFileCleaner`.

- [ ] **Step 2: Implement `MacClipboardPaster`**

Requirements:

- Use `NSPasteboard.general` to copy text.
- Check Accessibility trust before simulated paste.
- Use `CGEvent` to send Command+V.
- Throw `.missingAccessibilityPermission` or `.pasteFailed`.

- [ ] **Step 3: Implement `MacHotkeyController`**

Requirements:

- Use Carbon `RegisterEventHotKey` for `Option+Space`.
- Call an injected async handler when the hotkey fires.
- Surface registration failure without terminating the app.
- Leave menu-based recording usable if hotkey registration fails.

- [ ] **Step 4: Implement `FileManagerTemporaryFileCleaner`**

Requirements:

- Implement `TemporaryFileCleaner`.
- Remove the provided file URL with `FileManager.default.removeItem(at:)`.
- Ignore file-not-found cleanup errors.
- Never delete directories.

- [ ] **Step 5: Build**

Run: `swift build`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add Sources/VoiceInputCore/Audio Sources/VoiceInputCore/System
git commit -m "Add macOS recording hotkey and paste adapters"
```

### Task 7: Implement Menu Bar App And Settings Window

**Files:**
- Create: `Sources/VoiceInputCore/UI/MenuBarController.swift`
- Create: `Sources/VoiceInputCore/UI/SettingsWindowController.swift`
- Modify: `Sources/VoiceInput/main.swift`

- [ ] **Step 1: Build menu bar UI**

Menu items:

- Current state label.
- Start/Stop Recording.
- Show Last Result.
- Settings.
- Quit.

- [ ] **Step 2: Build settings window**

Fields:

- Volcengine API key secure text field.
- DeepSeek API key secure text field.
- Default prompt multiline text field.
- Automatic paste checkbox, enabled by default.

Settings copy must state that transcripts and prompts are sent to configured providers.

- [ ] **Step 3: Persist settings window changes**

When the user clicks Save:

- Load existing `AppSettings`.
- Save prompt and automatic paste through `SettingsStore`.
- Save Volcengine and DeepSeek API keys through `SecretsStore`.
- Keep API key fields blank when no key is stored; never print keys to logs.
- Rebuild or update provider dependencies so new credentials are used for subsequent recordings.

- [ ] **Step 4: Wire runtime dependencies in `main.swift`**

Create:

- `UserDefaultsSettingsStore`
- `KeychainSecretsStore`
- `AVFoundationAudioRecorder`
- `VolcengineSpeechProvider`
- `DeepSeekRefinementProvider`
- `MacClipboardPaster`
- `FileManagerTemporaryFileCleaner`
- `AppStateController`
- `MacHotkeyController`
- `MenuBarController`

Credential handling:

- On startup, load `AppSettings` immediately.
- Load secrets before creating providers.
- If one or both secrets are missing, create lightweight placeholder providers that throw `.missingSpeechCredentials` or `.missingRefinementCredentials`; keep the app running and show a settings-needed state/menu message.
- After settings are saved, rebuild the real providers with the saved keys and the stable `AppSettings.localUserID`.
- Menu-based recording should surface missing credentials as readable errors rather than crashing.

- [ ] **Step 5: Build and smoke-run**

Run: `swift build && swift run VoiceInput`

Expected: menu bar app launches. Stop manually after confirming the menu appears.

- [ ] **Step 6: Commit**

```bash
git add Sources/VoiceInput Sources/VoiceInputCore/UI
git commit -m "Add menu bar app UI"
```

### Task 8: Document Manual Verification And First-Run Usage

**Files:**
- Create: `docs/manual-verification.md`
- Modify: `README.md`

- [ ] **Step 1: Add manual verification doc**

Cover:

- Configure API keys.
- Grant microphone permission.
- Grant Accessibility permission.
- Test in WeChat, Apple Notes, Safari, and Chrome.
- Confirm automatic paste fallback copies to clipboard if Accessibility is missing.
- Confirm refinement failure copies raw transcript without auto-paste.
- Confirm hotkey conflict/failure still allows menu-based recording.

- [ ] **Step 2: Update README first-run instructions**

Include:

- Current first version runs with `swift run VoiceInput`.
- `.app` packaging, notarization, and auto-update are deferred.
- Required API keys and permissions.
- Link to `docs/manual-verification.md`.

- [ ] **Step 3: Run all automated checks**

Run: `swift test && swift build`

Expected: all commands succeed.

- [ ] **Step 4: Commit**

```bash
git add docs/manual-verification.md README.md
git commit -m "Document manual verification"
```

### Task 9: Final Validation And Push

**Files:**
- No new files expected.

- [ ] **Step 1: Check repository status**

Run: `git status -sb`

Expected: clean working tree.

- [ ] **Step 2: Push commits**

Run: `git push`

Expected: local branch is pushed to `origin/main`.

- [ ] **Step 3: Record validation results**

Update the final response with:

- Last commit hash.
- Automated checks run.
- Whether manual microphone/API/paste testing was completed or still needs the user's credentials/permissions.
