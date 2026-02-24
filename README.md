# 20XX Vision

**by ALL SEEING EYES**

![20XX Vision](assets/teaserimage.png)

A real-time AI assistant for Meta Ray-Ban smart glasses. See what you see, hear what you say, and take actions on your behalf -- all through voice.

![Cover](assets/cover.png)

Forked from [VisionClaw](https://github.com/sseanliu/VisionClaw). Built on [Meta Wearables DAT SDK](https://github.com/facebook/meta-wearables-dat-ios) (iOS) + [Gemini Live API](https://ai.google.dev/gemini-api/docs/live) + [OpenClaw](https://github.com/nichochar/openclaw) (optional).

**Hardware:** Meta Ray-Ban Wayfarer Gen 2
**Device:** iOS -- iPhone 17 Pro
**Platform:** iOS (iPhone)

---

## What It Does

Put on your glasses, tap the AI button, and talk:

- **"What am I looking at?"** -- Gemini sees through your glasses camera and describes the scene
- **"Add milk to my shopping list"** -- delegates to OpenClaw, which adds it via your connected apps
- **"Send a message to John saying I'll be late"** -- routes through OpenClaw to WhatsApp/Telegram/iMessage
- **"Search for the best coffee shops nearby"** -- web search via OpenClaw, results spoken back

20XX Vision specializes in: visual analysis, creative direction, VFX research, content spotting, and sports analytics.

The glasses camera streams at ~1fps to Gemini for visual context, while audio flows bidirectionally in real-time.

## How It Works

![How It Works](assets/how.png)

```
Meta Ray-Ban Glasses (or iPhone camera)
       |
       | video frames + mic audio
       v
iOS App (20XX Vision)
       |
       | JPEG frames (~1fps) + PCM audio (16kHz)
       v
Gemini Live API (WebSocket)
       |
       |-- Audio response (PCM 24kHz) --> App --> Speaker
       |-- Tool calls (execute) -------> App --> OpenClaw Gateway
       |                                              |
       |                                              v
       |                                      56+ skills: web search,
       |                                      messaging, smart home,
       |                                      notes, reminders, etc.
       |                                              |
       |<---- Tool response (text) <----- App <-------+
       |
       v
  Gemini speaks the result
```

**Key pieces:**
- **Gemini Live** -- real-time voice + vision AI over WebSocket (native audio, not STT-first)
- **OpenClaw** (optional) -- local gateway that gives Gemini access to 56+ tools and all your connected apps
- **Phone mode** -- test the full pipeline using your iPhone camera instead of glasses
- **WebRTC streaming** -- share your glasses POV live to a browser viewer

---

## Quick Start

### 1. Clone and open

```bash
git clone https://github.com/allsxxing/XXVision.git
cd XXVision/samples/CameraAccess
open CameraAccess.xcodeproj
```

### 2. Add your secrets

Copy the example file and fill in your values:

```bash
cp CameraAccess/Secrets.swift.example CameraAccess/Secrets.swift
```

Edit `Secrets.swift` with your [Gemini API key](https://aistudio.google.com/apikey) (required) and optional OpenClaw/WebRTC config.

### 3. Configure signing

1. In Xcode, select the `CameraAccess` target
2. Under **Signing & Capabilities**, set your Apple Developer Team
3. Bundle Identifier: `com.allsxxing.xxvision`

### 4. Build and run

Connect your iPhone 17 Pro via USB, select it as the build target, and hit Run (Cmd+R).

### 5. Try it out

**Without glasses (iPhone mode):**
1. Tap **"Start on iPhone"** -- uses your iPhone's back camera
2. Tap the **AI button** to start a Gemini Live session
3. Talk to the AI -- it can see through your iPhone camera

**With Meta Ray-Ban glasses:**

First, enable Developer Mode in the Meta AI app:

1. Open the **Meta AI** app on your iPhone
2. Go to **Settings** (gear icon, bottom left)
3. Tap **App Info**
4. Tap the **App version** number **5 times** -- this unlocks Developer Mode
5. Go back to Settings -- you'll now see a **Developer Mode** toggle. Turn it on.

![How to enable Developer Mode](assets/dev_mode.png)

Then in 20XX Vision:
1. Tap **"Start Streaming"** in the app
2. Tap the **AI button** for voice + vision conversation

---

## Setup: OpenClaw (Optional)

OpenClaw gives Gemini the ability to take real-world actions: send messages, search the web, manage lists, control smart home devices, and more. Without it, Gemini is voice + vision only.

### 1. Install and configure OpenClaw

```bash
# Requires Node.js 22+
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

In `~/.openclaw/openclaw.json`:

```json
{
  "gateway": {
    "port": 18789,
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "your-gateway-token-here"
    },
    "http": {
      "endpoints": {
        "chatCompletions": { "enabled": true }
      }
    }
  }
}
```

Key settings:
- `bind: "lan"` -- exposes the gateway on your local network so your iPhone can reach it
- `chatCompletions.enabled: true` -- enables the `/v1/chat/completions` endpoint (off by default)
- `auth.token` -- the token your app will use to authenticate

### 2. Configure the app

In `Secrets.swift`:
```swift
static let openClawHost = "http://Your-Mac-Mini.local"
static let openClawPort = 18789
static let openClawGatewayToken = "your-gateway-token-here"
```

To find your Mac's Bonjour hostname: **System Settings > General > Sharing** -- it's shown at the top.

> The app also has an in-app Settings screen where you can change these values at runtime without editing source code.

### 3. Start the gateway

```bash
openclaw gateway restart
curl http://localhost:18789/health
```

Now when you talk to the AI, it can execute tasks through OpenClaw.

---

## Architecture

### Key Files

All source code is in `samples/CameraAccess/CameraAccess/`:

| File | Purpose |
|------|---------|
| `Gemini/GeminiConfig.swift` | API keys, model config, system prompt |
| `Gemini/GeminiLiveService.swift` | WebSocket client for Gemini Live API |
| `Gemini/AudioManager.swift` | Mic capture (PCM 16kHz) + audio playback (PCM 24kHz) |
| `Gemini/GeminiSessionViewModel.swift` | Session lifecycle, tool call wiring, transcript state |
| `OpenClaw/ToolCallModels.swift` | Tool declarations, data types |
| `OpenClaw/OpenClawBridge.swift` | HTTP client for OpenClaw gateway |
| `OpenClaw/ToolCallRouter.swift` | Routes Gemini tool calls to OpenClaw |
| `iPhone/IPhoneCameraManager.swift` | AVCaptureSession wrapper for iPhone camera mode |
| `WebRTC/WebRTCClient.swift` | WebRTC peer connection + SDP negotiation |
| `WebRTC/SignalingClient.swift` | WebSocket signaling for WebRTC rooms |

### Audio Pipeline

- **Input**: iPhone mic -> AudioManager (PCM Int16, 16kHz mono, 100ms chunks) -> Gemini WebSocket
- **Output**: Gemini WebSocket -> AudioManager playback queue -> iPhone speaker
- **iPhone mode**: Uses `.voiceChat` audio session for echo cancellation + mic gating during AI speech
- **Glasses mode**: Uses `.videoChat` audio session (mic is on glasses, speaker is on phone -- no echo)

### Video Pipeline

- **Glasses**: DAT SDK video stream (24fps) -> throttle to ~1fps -> JPEG (50% quality) -> Gemini
- **iPhone**: Camera capture (30fps) -> throttle to ~1fps -> JPEG -> Gemini

### Tool Calling

Gemini Live supports function calling. The app declares a single `execute` tool that routes everything through OpenClaw:

1. User says "Add eggs to my shopping list"
2. Gemini speaks "Sure, adding that now" (verbal acknowledgment before tool call)
3. Gemini sends `toolCall` with `execute(task: "Add eggs to the shopping list")`
4. `ToolCallRouter` sends HTTP POST to OpenClaw gateway
5. OpenClaw executes the task using its 56+ connected skills
6. Result returns to Gemini via `toolResponse`
7. Gemini speaks the confirmation

---

## Requirements

- iOS 17.0+
- Xcode 15.0+
- Gemini API key ([get one free](https://aistudio.google.com/apikey))
- Meta Ray-Ban glasses (optional -- use iPhone mode for testing)
- OpenClaw on your Mac (optional -- for agentic actions)
- Apple Developer account (for device deployment)

---

## Troubleshooting

**"Gemini API key not configured"** -- Add your API key in Secrets.swift or in the in-app Settings.

**Gemini doesn't hear me** -- Check that microphone permission is granted. Speak clearly and at normal volume.

**OpenClaw connection timeout** -- Make sure your iPhone and Mac are on the same Wi-Fi network, the gateway is running (`openclaw gateway restart`), and the hostname matches your Mac's Bonjour name.

**Echo/feedback in iPhone mode** -- The app mutes the mic while the AI is speaking. If you still hear echo, try turning down the volume.

**Build fails on SPM** -- Clean build folder (Cmd+Shift+K) and re-resolve packages.

**Glasses not detected** -- Ensure Developer Mode is ON, firmware is v20.0+, and the Meta AI app is open.

For DAT SDK issues, see the [developer documentation](https://wearables.developer.meta.com/docs/develop/).

---

## Upstream

This project is forked from [sseanliu/VisionClaw](https://github.com/sseanliu/VisionClaw). To sync with upstream:

```bash
git remote add upstream https://github.com/sseanliu/VisionClaw.git
git fetch upstream
git merge upstream/main
```

## License

This source code is licensed under the license found in the [LICENSE](LICENSE) file in the root directory of this source tree.

---

*20XX Vision -- ALL SEEING EYES*
