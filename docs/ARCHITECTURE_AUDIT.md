# iOS Architecture Audit: TurboMeta

## Executive Summary

The app follows an MVVM pattern with SwiftUI and has a reasonable folder structure (`Models/`, `Views/`, `ViewModels/`, `Services/`, `Managers/`). It includes a design system, localization support, and Keychain-based secret storage. However, there are several **structural issues** that will increasingly slow down feature development, make bugs harder to isolate, and make unit testing nearly impossible.

**Severity rating: Medium-High risk.** The app works, but the architecture will fight you as it grows.

---

## 1. Singleton Overuse — No Dependency Injection

**Severity: Critical for testability and extensibility**

Nearly every manager and service is a `static let shared` singleton:

- `LiveAIManager.shared`, `TTSService.shared`, `ConversationStorage.shared`
- `APIProviderManager.shared`, `APIKeyManager.shared`, `LanguageManager.shared`
- `LiveAIModeManager.shared`, `QuickVisionModeManager.shared`

**Problems:**
- Dependencies are hidden inside method bodies, not declared in `init()` signatures. You can't look at a class's initializer and understand what it depends on.
- Mocking is impossible — you cannot substitute a fake `ConversationStorage` or `TTSService` in tests.
- Singletons create implicit coupling graphs that are invisible until something breaks.

**Fix:** Define protocols for each service/manager and inject them via `init()`. Keep the `.shared` convenience accessor if needed, but make the dependency injectable:

```swift
protocol ConversationStorageProtocol {
    func saveConversation(_ record: ConversationRecord)
    func loadAllConversations() -> [ConversationRecord]
}

class ConversationStorage: ConversationStorageProtocol { ... }
```

---

## 2. Massive Code Duplication Between Services

**Severity: High**

`OmniRealtimeService` and `GeminiLiveService` share ~70% identical code:

- Identical `setupPlaybackEngine()`, `startPlaybackEngine()`, `stopPlaybackEngine()` methods
- Identical `createPCMBuffer(from:format:)` implementations
- Identical callback signature patterns (`onConnected`, `onTranscriptDelta`, `onError`, etc.)
- Identical audio buffer management strategy (chunk collection, min-chunks-before-play)
- `TTSService` also duplicates the entire playback engine setup and `createPCMBuffer`

This means **a bug fix in audio playback must be applied in 3 places**, and adding a new AI provider means copying 500+ lines of boilerplate.

**Fix:** Extract a shared protocol and base audio infrastructure:

```swift
protocol RealtimeAIService: AnyObject {
    var onConnected: (() -> Void)? { get set }
    var onTranscriptDone: ((String) -> Void)? { get set }
    var onError: ((String) -> Void)? { get set }
    // ... shared callbacks
    func connect()
    func disconnect()
    func startRecording()
    func stopRecording()
    func sendImage(_ image: UIImage)
}
```

Extract `AudioPlaybackEngine` as a shared utility class that all three services use.

---

## 3. LiveAIManager is a God Object

**Severity: High**

`LiveAIManager` (~430 lines) handles:
- Session lifecycle orchestration
- Audio session configuration
- AI service initialization (with provider dispatch)
- Callback wiring for both Omni and Gemini (duplicate blocks)
- Video frame timer management
- Recording start/stop delegation
- Conversation history accumulation
- Conversation persistence
- NotificationCenter observation for Intents

The dual-provider dispatch is repeated in 6 separate methods via `switch provider`.

**Fix:** If `OmniRealtimeService` and `GeminiLiveService` conformed to a shared `RealtimeAIService` protocol, `LiveAIManager` would drop from ~430 lines to ~150 lines. The callback wiring would be written once. Adding a third provider would require zero changes to `LiveAIManager`.

---

## 4. Duplicate Manager Classes

**Severity: Medium**

`LiveAIModeManager` and `QuickVisionModeManager` are structurally identical:
- Same `supportedLanguages` array (duplicated verbatim)
- Same `translateTargetLanguage` logic
- Same `getTranslatePrompt()` pattern
- Same UserDefaults persistence pattern
- Same static access helpers

**Fix:** Extract a generic `ModeManager<Mode: ModeProtocol>` or at minimum share the language list and translate prompt logic.

---

## 5. Static UserDefaults Access Anti-Pattern

**Severity: Medium**

To work around `@MainActor` isolation, several managers have `nonisolated static` computed properties that re-read from `UserDefaults` independently. The UserDefaults keys are string literals duplicated between the instance and static accessors. If a key changes in one place but not the other, you get a silent bug with no compiler warning.

**Affected files:**
- `APIProviderManager.swift` — 6+ static accessors duplicating keys
- `LanguageManager.swift` — 3 static accessors
- `LiveAIModeManager.swift` — 3 static accessors

**Fix:** Use a single source of truth for UserDefaults keys (e.g., a `Keys` enum) and consider using `@AppStorage` or a dedicated settings repository.

---

## 6. Test Coverage is Nearly Zero

**Severity: Critical**

The test target contains **2 integration tests**, both for `StreamSessionViewModel`. There are:

- **0 tests** for any AI service (`GeminiLiveService`, `OmniRealtimeService`, `QuickVisionService`)
- **0 tests** for any manager (`LiveAIManager`, `APIProviderManager`, `LiveAIModeManager`)
- **0 tests** for models or storage (`ConversationStorage`, `ConversationRecord`)
- **0 tests** for utilities (`APIKeyManager`, `LanguageManager`, `TTSService`)

The singleton architecture makes writing unit tests very difficult, which likely explains the absence.

---

## 7. UserDefaults for Conversation Storage

**Severity: Medium**

`ConversationStorage` saves up to 100 conversation records (each with multiple messages) as a single JSON blob in `UserDefaults`. Every save re-encodes the entire history. Every load re-decodes the entire history. This degrades as conversation count grows, and `UserDefaults` is not designed for large data blobs.

**Fix:** Migrate to SwiftData or file-based JSON storage with indexed access.

---

## 8. Thread Safety Concerns

**Severity: Medium**

`LanguageManager.currentBundle` is declared as `nonisolated(unsafe)`. The comment claims "This is safe because we only read after initialization," but `updateBundle()` is called from the `currentLanguage.didSet` observer — meaning it's written to whenever the user changes language. The `String.localized` extension reads it from any thread. This is a data race.

---

## 9. Navigation Architecture

**Severity: Medium**

Navigation is managed through conditional `if/else` statements in `MainAppView`. `@State` properties like `hasCheckedPermissions` can be lost on view reconstruction. There's no coordinator or router — adding a new flow (e.g., onboarding, deep linking) requires modifying `MainAppView` directly.

---

## 10. Fragile Intent Communication

**Severity: Low-Medium**

The Siri intent triggers Live AI via `NotificationCenter`. This is untyped, has no delivery guarantee, and creates an invisible coupling. If `LiveAIManager` hasn't initialized yet (or has been deinitialized), the notification is silently lost.

---

## 11. Inconsistent Logging

**Severity: Low**

The codebase has two logging approaches:
- `StreamSessionViewModel` uses the proper `os.log` (`Logger`) system
- Everything else uses `print()` with emoji prefixes

`print()` statements don't appear in Console.app for release builds, don't support log levels, and can't be filtered.

---

## 12. Hardcoded / Non-Localized Error Messages

**Severity: Low**

Error enums (`LiveAIError`, `QuickVisionError`, `TTSError`) use hardcoded Chinese strings instead of the localization system used everywhere else. `ConversationRecord.formattedDate` also has hardcoded "今天" / "昨天".

---

## Priority Action Items

| Priority | Issue | Impact | Effort |
|----------|-------|--------|--------|
| **P0** | Extract `RealtimeAIService` protocol | Eliminates duplication, enables new providers | Medium |
| **P0** | Extract shared `AudioPlaybackEngine` | 3 files share identical audio code | Medium |
| **P0** | Introduce dependency injection | Enables testing, decouples everything | High |
| **P1** | Add unit tests for services/managers | Catches regressions | Ongoing |
| **P1** | Migrate `ConversationStorage` off UserDefaults | Performance, data integrity | Medium |
| **P1** | Fix `nonisolated(unsafe)` thread safety | Data race in `LanguageManager` | Low |
| **P2** | Unify `LiveAIModeManager`/`QuickVisionModeManager` | Reduces duplicate manager code | Low |
| **P2** | Consolidate UserDefaults keys | Prevents key-mismatch bugs | Low |
| **P2** | Adopt `os.log` everywhere | Proper diagnostics in production | Low |
| **P2** | Localize error messages | Consistency with existing i18n | Low |
| **P3** | Add a navigation coordinator | Cleaner flow management | Medium |
| **P3** | Replace NotificationCenter with direct calls | Typed, reliable intent handling | Low |

---

## What's Already Good

- **Clean folder structure** — MVVM separation is clear and well-organized
- **Design system** (`DesignSystem.swift`) — centralized colors, typography, spacing, corner radii
- **Keychain for API keys** — secure storage with proper migration logic
- **Localization infrastructure** — dual-language support with a `.localized` extension
- **`@MainActor` annotations** — concurrency is mostly handled correctly on ViewModels/Managers
- **`StreamSessionViewModel`** — well-architected with proper listener tokens, cleanup, and `os.log`
- **Codable models** — clean data models with proper `CodingKeys`

The bones are good. The main investment needed is in **protocol abstractions** and **dependency injection** to unlock testability and make adding a third AI provider a one-file change instead of a multi-file surgery.
