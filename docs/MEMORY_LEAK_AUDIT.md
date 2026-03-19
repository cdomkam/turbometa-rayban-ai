# Memory Leak Audit Report

**Date:** 2026-03-19
**Trigger:** macOS killed the application due to excessive memory usage

---

## P0 - Critical (Likely Cause of Crash)

### 1. OmniRealtimeView.swift - Timer Never Invalidated

**File:** `CameraAccess/Views/OmniRealtimeView.swift` (lines 75-86)

**Problem:** A `Timer.scheduledTimer` is created in `onAppear` but never stored in a variable and never invalidated in `onDisappear`. This timer fires 10 times per second forever, even after the view disappears. It strongly captures `streamViewModel` and `viewModel`, preventing them and their entire object graph (audio engines, WebSocket connections, UIImage video frames) from being deallocated. Each navigation to this view creates a new immortal timer.

**Code:**

```swift
.onAppear {
    viewModel.connect()
    Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) { timer in
        if let frame = streamViewModel.currentVideoFrame {
            viewModel.updateVideoFrame(frame)
        }
    }
}
.onDisappear {
    viewModel.disconnect()
}
```

**Fix:** Add `@State private var frameTimer: Timer?`, assign the timer, and invalidate in `onDisappear`:

```swift
@State private var frameTimer: Timer?

// in onAppear:
frameTimer = Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) { _ in
    if let frame = streamViewModel.currentVideoFrame {
        viewModel.updateVideoFrame(frame)
    }
}

// in onDisappear:
frameTimer?.invalidate()
frameTimer = nil
viewModel.disconnect()
```

---

### 2. URLSession Delegate Retain Cycle - OmniRealtimeService

**File:** `CameraAccess/Services/OmniRealtimeService.swift` (lines 162-163)

**Problem:** `URLSession` retains its delegate strongly. The service creates a `URLSession` with `self` as delegate but never calls `invalidateAndCancel()` in `disconnect()`. This creates a retain cycle: `self` -> `urlSession` (strong property) -> `self` (strong delegate). The service, along with two `AVAudioEngine` instances, `AVAudioPlayerNode`, audio buffers, and all callback closures are permanently leaked.

**Code:**

```swift
urlSession = URLSession(configuration: configuration, delegate: self, delegateQueue: OperationQueue())
```

```swift
func disconnect() {
    webSocket?.cancel(with: .goingAway, reason: nil)
    webSocket = nil
    stopRecording()
    stopPlaybackEngine()
    // MISSING: urlSession?.invalidateAndCancel(); urlSession = nil
}
```

**Fix:** Add to `disconnect()`:

```swift
func disconnect() {
    webSocket?.cancel(with: .goingAway, reason: nil)
    webSocket = nil
    urlSession?.invalidateAndCancel()
    urlSession = nil
    stopRecording()
    stopPlaybackEngine()
}
```

---

### 3. URLSession Delegate Retain Cycle - GeminiLiveService

**File:** `CameraAccess/Services/GeminiLiveService.swift` (lines 136-137)

**Problem:** Same retain cycle as issue #2. `URLSession` with `self` as delegate is never invalidated.

**Fix:** Add `urlSession?.invalidateAndCancel()` and `urlSession = nil` to `disconnect()`.

---

### 4. URLSession Delegate Retain Cycle - LiveTranslateService

**File:** `CameraAccess/Services/LiveTranslateService.swift` (lines 136-137)

**Problem:** Same retain cycle as issue #2. `URLSession` with `self` as delegate is never invalidated.

**Fix:** Add `urlSession?.invalidateAndCancel()` and `urlSession = nil` to `disconnect()`.

---

## P1 - High (Significant Memory Growth)

### 5. Strong `self` in DispatchQueue Closures - OmniRealtimeService

**File:** `CameraAccess/Services/OmniRealtimeService.swift` (lines 172-175, 393-398)

**Problem:** `handleServerEvent()` dispatches to the main queue capturing `self` strongly. Combined with the URLSession retain cycle, these closures keep the service alive indefinitely. The `sendEvent()` completion handler also captures `self` strongly (lines 315-319).

**Code:**

```swift
// handleServerEvent
DispatchQueue.main.async {
    self.onConnected?()
}

// sendEvent
webSocket?.send(message) { error in
    self.onError?("Send error: ...")
}

// connect
DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
    self.configureSession()
}
```

**Fix:** Use `[weak self]` in all closures:

```swift
DispatchQueue.main.async { [weak self] in
    self?.onConnected?()
}
```

---

### 6. Strong `self` in DispatchQueue Closures - GeminiLiveService

**File:** `CameraAccess/Services/GeminiLiveService.swift` (lines 325-330, 405-411, 584-586)

**Problem:** Same pattern as issue #5. Additionally, the `didOpenWithProtocol` delegate callback dispatches to main with strong `self`:

```swift
func urlSession(_ session: URLSession, ...) {
    DispatchQueue.main.async {
        self.configureSession()  // strong self
    }
}
```

**Fix:** Use `[weak self]` in all closures.

---

### 7. Strong `self` in DispatchQueue Closures - LiveTranslateService

**File:** `CameraAccess/Services/LiveTranslateService.swift` (lines 146-149, 405-409, 469-474)

**Problem:** Same pattern as issues #5 and #6.

**Fix:** Use `[weak self]` in all closures.

---

### 8. OmniRealtimeViewModel.deinit Ineffective

**File:** `CameraAccess/ViewModels/OmniRealtimeViewModel.swift` (lines 335-341)

**Problem:** The `deinit` tries to disconnect services via `[weak omniService, weak geminiService]`, but these are strong `private var` properties. Even if deinit fired, `disconnect()` doesn't break the URLSession retain cycle anyway. If the ViewModel is held by a leaked timer (issue #1), deinit never fires at all.

**Fix:** Ensure `disconnect()` is always called explicitly (from `onDisappear`), and fix the URLSession invalidation in the services.

---

## P2 - Medium

### 9. WearablesViewModel - Retain Cycle in Task Closures

**File:** `CameraAccess/ViewModels/WearablesViewModel.swift` (lines 44-55, 68-77)

**Problem:** `Task` closures in `init` capture `self` strongly via `self.registrationState = ...` etc. Since `self` holds the tasks via `registrationTask` and `deviceStreamTask`, a retain cycle exists: `self` -> `task` -> `self`. The `deinit` cancels the tasks, but deinit can never fire because of the cycle. In practice this ViewModel lives for the app lifetime as a `@StateObject`, limiting the impact.

**Code:**

```swift
registrationTask = Task {
    for await registrationState in wearables.registrationStateStream() {
        self.registrationState = registrationState  // strong self
    }
}
```

**Fix:** Use `[weak self]` inside the Task closure:

```swift
registrationTask = Task { [weak self] in
    guard let self else { return }
    for await registrationState in wearables.registrationStateStream() {
        self.registrationState = registrationState
    }
}
```

---

### 10. Video Frame UIImage Accumulation

**File:** `CameraAccess/ViewModels/StreamSessionViewModel.swift` (line 32)

**Problem:** `currentVideoFrame: UIImage?` is updated at up to 24fps. Each frame at camera resolution can be ~8MB. The frame is passed to multiple ViewModels (`OmniRealtimeViewModel.currentVideoFrame`, `LiveTranslateViewModel.currentVideoFrame`) and SwiftUI views. When combined with leaked timers and services (issues #1-4), old frames are retained by zombie objects, causing rapid memory growth.

**Impact:** Not a leak by itself, but multiplies the impact of all other leaks.

---

### 11. StreamSessionViewModel.cleanup() Never Called

**File:** `CameraAccess/ViewModels/StreamSessionViewModel.swift` (lines 286-298)

**Problem:** The `cleanup()` method exists to release all listener tokens and stop sessions, but is never called. The ViewModel is a `@StateObject` in `MainAppView`, so it lives for the app lifetime. If the architecture changes, listeners would leak.

**Fix:** Call `cleanup()` when appropriate, or implement cleanup in `deinit`.

---

### 12. NotificationCenter Observers Never Removed

**Files:**
- `CameraAccess/Intents/QuickVisionIntent.swift` (lines 261-266)
- `CameraAccess/Managers/LiveAIManager.swift` (lines 39-45)

**Problem:** NotificationCenter observers are added but never removed. Both are effectively singletons so this doesn't cause leaks currently, but it's a code smell that could cause crashes if the lifecycle changes.

**Fix:** Add `removeObserver` in `deinit` or use the block-based `addObserver` API that returns a token.

---

## How to Verify Fixes

1. **Xcode Instruments - Leaks:** Profile with the Leaks instrument to detect retain cycles.
2. **Xcode Instruments - Allocations:** Monitor live memory with the Allocations instrument. Look for monotonically increasing `UIImage`, `AVAudioEngine`, and `URLSession` instance counts.
3. **Debug Memory Graph:** Use Xcode's Debug Memory Graph (Runtime > Debug Memory Graph) to visualize retain cycles.
4. **Add `deinit` logging:** Add `print("deinit")` to all ViewModels and Services to verify they are actually deallocated when views disappear.

---

## Quick Checklist

- [ ] Fix `OmniRealtimeView` timer (P0)
- [ ] Add `urlSession?.invalidateAndCancel()` to `OmniRealtimeService.disconnect()` (P0)
- [ ] Add `urlSession?.invalidateAndCancel()` to `GeminiLiveService.disconnect()` (P0)
- [ ] Add `urlSession?.invalidateAndCancel()` to `LiveTranslateService.disconnect()` (P0)
- [ ] Add `[weak self]` to all `DispatchQueue` closures in `OmniRealtimeService` (P1)
- [ ] Add `[weak self]` to all `DispatchQueue` closures in `GeminiLiveService` (P1)
- [ ] Add `[weak self]` to all `DispatchQueue` closures in `LiveTranslateService` (P1)
- [ ] Fix `OmniRealtimeViewModel.deinit` (P1)
- [ ] Add `[weak self]` to `WearablesViewModel` Task closures (P2)
- [ ] Address `StreamSessionViewModel.cleanup()` (P2)
- [ ] Remove NotificationCenter observers properly (P2)
