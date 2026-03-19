# TurboMeta — Ray-Ban Meta Smart Glasses AI Assistant

<div align="center">

<img src="./rayban.png" width="120" alt="TurboMeta Logo"/>

**Multimodal AI assistant for Ray-Ban Meta smart glasses**

[![iOS](https://img.shields.io/badge/iOS-17.0%2B-blue.svg)](https://www.apple.com/ios/)
[![Swift](https://img.shields.io/badge/Swift-5.0-orange.svg)](https://swift.org)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

> **This is a fork** of [Turbo1123/turbometa-rayban-ai](https://github.com/Turbo1123/turbometa-rayban-ai). The original project is a full-featured Chinese-first AI assistant for Ray-Ban Meta glasses. This fork restructures the project for maintainability, adds secret management, and provides English-first documentation.

---

## Features

- **Live AI** — Real-time multimodal conversation through the glasses camera and microphone (Alibaba Qwen Omni or Google Gemini Live)
- **Quick Vision** — Siri/Shortcuts-triggered image recognition with TTS voice announcement, no phone unlock needed
- **LeanEat** — Photograph food to get nutritional analysis and a health score
- **RTMP Streaming** — Push live video to YouTube, Twitch, TikTok, or any RTMP endpoint
- **Multi-Provider** — Alibaba Cloud Dashscope, OpenRouter (500+ models), and Google Gemini
- **Bilingual UI** — Full English and Chinese interface with runtime switching

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Hardware | Ray-Ban Meta smart glasses (Gen 1 or Gen 2) |
| iPhone | iOS 17.0+ |
| Xcode | 15.0+ |
| Meta View App | Latest version, with **DAT SDK Preview Mode enabled** (see below) |
| API Key | Alibaba Cloud Dashscope, OpenRouter, or Google Gemini |

### Enable DAT SDK Preview Mode

The Meta Wearables DAT SDK is in Preview. You must enable it in the Meta View app before TurboMeta can communicate with the glasses:

1. Update your Ray-Ban Meta firmware to **v20+**
2. Update the **Meta View** (or Meta AI) app to the latest version
3. Open Meta View → **Settings** → **App Info**
4. **Tap the version number 5 times** rapidly
5. A confirmation message appears — Preview Mode is now active

## Getting Started

### 1. Clone and configure secrets

```bash
git clone https://github.com/<your-username>/turbometa-rayban-ai.git
cd turbometa-rayban-ai

# Create your local secrets file (gitignored, never committed)
cp Secrets.xcconfig.example Secrets.xcconfig
```

Open `Secrets.xcconfig` and fill in your Meta Developer credentials:

```
META_CLIENT_TOKEN = AR|your_app_id|your_client_token_hash
META_APP_ID = your_app_id
```

You can find these values in your [Meta Developer Dashboard](https://developers.facebook.com/apps/) under your app's settings.

### 2. Open in Xcode

```bash
open CameraAccess.xcodeproj
```

- Select your **Development Team** in Signing & Capabilities
- Change the **Bundle Identifier** if needed to avoid conflicts
- Connect your iPhone and press **Cmd+R** to build and run

### 3. Configure API keys in the app

TurboMeta stores API keys securely in the iOS Keychain — they are never in source code.

1. Open TurboMeta on your phone
2. Go to **Settings** → **API Key Management**
3. Enter your API key for your chosen provider:

| Provider | Get a key | Region notes |
|----------|-----------|--------------|
| Alibaba Cloud Dashscope | [Console](https://bailian.console.aliyun.com/) → API-KEY Management → Create | Beijing (mainland China) or Singapore (international) |
| OpenRouter | [openrouter.ai/keys](https://openrouter.ai/keys) | Global |
| Google Gemini | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) | Requires non-China network |

### 4. Pair your glasses

1. Pair your Ray-Ban Meta glasses in the **Meta View** app
2. Ensure Bluetooth is on
3. Open TurboMeta — it will detect the glasses automatically

## Project Structure

```
CameraAccess/
├── TurboMetaApp.swift          # App entry point
├── Models/                     # Data models (conversations, modes, nutrition)
├── Views/                      # SwiftUI views
│   └── Components/             # Reusable UI components
├── ViewModels/                 # MVVM view models
├── Services/                   # API clients (Gemini, Omni, QuickVision, TTS)
├── Managers/                   # App-wide state (API providers, language, modes)
├── Intents/                    # Siri / App Intent definitions
├── Utils/                      # API key storage, permissions, helpers
└── Utilities/                  # Design system (colors, typography, spacing)
```

## Quick Vision Setup (Siri / Shortcuts)

Quick Vision lets you identify objects through the glasses without unlocking your phone.

1. Open the **Shortcuts** app on iPhone
2. Tap **+** → **Add Action** → search **TurboMeta**
3. Select **Quick Vision** (or **Live Conversation** for Live AI)
4. Rename the shortcut to something easy to say (e.g., "What's this")
5. Say **"Hey Siri, What's this"** — the glasses will capture a frame, run AI recognition, and speak the result

**iPhone 15 Pro+ users:** You can also bind the shortcut to the Action Button via Settings → Action Button → Shortcut.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| UI | SwiftUI |
| Architecture | MVVM |
| Glasses SDK | Meta Wearables DAT SDK v0.4.0 |
| Streaming | HaishinKit (RTMP) |
| AI Providers | Alibaba Qwen Omni-Realtime, Qwen VL-Plus, Google Gemini Live, OpenRouter |
| Audio | AVAudioEngine + AVAudioPlayerNode |
| TTS | Alibaba qwen3-tts-flash / system AVSpeechSynthesizer fallback |
| Secrets | iOS Keychain (API keys), xcconfig (build-time tokens) |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Glasses won't connect | Ensure DAT SDK Preview Mode is enabled in Meta View. Restart the glasses (place in charging case). |
| "Glasses not connected" on Quick Vision | Open TurboMeta at least once after install so the app registers with the system. |
| AI not responding | Check your API key in Settings. Verify network connectivity. Check provider quota. |
| No audio from TTS | Check phone isn't on silent. Check Bluetooth audio routing. TTS requires network. |
| Xcode build fails | Ensure `Secrets.xcconfig` exists (see Getting Started). Check your signing team and bundle ID. |

## Privacy

- Audio and video are sent to your configured AI provider for processing and are not stored by the app
- API keys are stored in the iOS Keychain
- No analytics or telemetry is collected
- All API traffic uses HTTPS

## Contributing

1. Fork this repo
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes
4. Push and open a Pull Request

## License

This project is based on sample code from Meta Platforms, Inc. and follows the original project's license. See [LICENSE](LICENSE) for details.

## Acknowledgments

- [Turbo1123/turbometa-rayban-ai](https://github.com/Turbo1123/turbometa-rayban-ai) — original project
- **Meta Platforms, Inc.** — DAT SDK and sample code
- **Alibaba Cloud Qwen Team** — multimodal AI models
