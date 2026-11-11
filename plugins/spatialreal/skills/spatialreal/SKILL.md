---
name: spatialreal
description: Integrate SpatialReal realtime 3D avatars and voice agents into a web, iOS or Android app. Use when the user wants to put a SpatialReal avatar or agent on a web page or in an iOS or Android app, add the server endpoint that issues session tokens, send their own audio to an avatar, connect an avatar to LiveKit, Agora or Pipecat, or fix a SpatialReal SDK error.
---

<!-- spatialreal skill -->

# SpatialReal

SpatialReal renders a realtime 3D avatar on the user's device. Audio goes in, the avatar speaks
it with matching face and body motion. Two things can be built, and both use the same pieces: a
token endpoint on the customer's server, and an SDK in the client.

| The person wants | The SDK session gets | Where the audio comes from |
| --- | --- | --- |
| A voice agent with a face: it listens, thinks and answers by itself | `agentId` | SpatialReal runs the agent |
| An avatar that speaks audio their own code produces (their LLM, their TTS) | `avatarId` | The customer's code sends it |

Ask which one they want if it is not clear. Every agent has an avatar; an agent integration
without one does not exist.

## What you need from the account

| Value | What it is | Secret |
| --- | --- | --- |
| App ID | Identifies the customer's app; goes into client code | no |
| API key | Lets the customer's server request session tokens | **yes, server only** |
| `avatarId` or `agentId` | The avatar to show, or the agent to talk to | no |

They come from the web console, https://app.spatialreal.ai. When the `spatialreal` command is
installed (`spatialreal --version`), use it instead: `spatialreal init` writes the App ID and a
new API key to `.env`, and it has its own skill for everything in the account (agents, avatars,
keys, embed links, billing). If it is not installed, you may offer it:
`npm install -g @spatialreal/cli` (Node.js 20 or newer), only when the person agrees.

Never put the API key in browser or mobile code, a bundler variable exposed to the browser, a
log line or a URL. Do not print it.

## The steps

The same four steps on the Web, iOS and Android; the three SDKs have the same calls.

### 1. Add a token endpoint to the customer's server

The client never sees the API key: it asks the customer's server for a session token.
`expire_at` (Unix seconds, at most 24 hours ahead) is required. Page: `overview/session-tokens`.

```js
// server (Express). Decide here who may get a token, e.g. require a signed-in user.
app.post('/api/session-token', async (req, res) => {
  const r = await fetch('https://api.spatialreal.com/v1/auth/session-token', {
    method: 'POST',
    headers: { 'X-API-KEY': process.env.SPATIALREAL_API_KEY, 'Content-Type': 'application/json' },
    body: JSON.stringify({ expire_at: Math.floor(Date.now() / 1000) + 3600 }),
  })
  if (!r.ok) return res.sendStatus(502)
  res.json({ token: (await r.json()).session_token })
})
```

### 2. Install the SDK

Take the current version from the platform's page; do not guess one.

| Platform | Install | Page |
| --- | --- | --- |
| Web | `npm install @spatialreal/web-sdk`, then add `spatialRealVitePlugin()` from `@spatialreal/web-sdk/vite` to the Vite plugins, or wrap the Next.js config with `withSpatialReal()` from `@spatialreal/web-sdk/next` (else `wasm-load-failed`) | `avatar-integration/sdk-mode/web` |
| iOS (16+, Xcode 16+) | Swift package `https://github.com/SpatialReal-ai/ios-sdk-release.git`, product `SpatialRealSDK` (a CocoaPods podspec is on the page) | `avatar-integration/sdk-mode/ios` |
| Android (API 24+, 64-bit ARM, Vulkan) | Gradle dependency `ai.spatialreal:android:<version>`, and the `INTERNET` permission in `AndroidManifest.xml` | `avatar-integration/sdk-mode/android` |

### 3. Create the session, start it, send audio

The container (an element, a `UIView`, a `ViewGroup`) must be on screen with a real size before
`createSession`, or it fails with `resource-load-failed`.

```ts
// Web
import { SpatialReal } from '@spatialreal/web-sdk'

const sr = new SpatialReal({ appId: 'YOUR_APP_ID' }) // the App ID is not a secret
const avatar = await sr.createSession({
  avatarId: 'YOUR_AVATAR_ID',
  credential: token, // from the endpoint of step 1
  container: document.getElementById('avatar')!,
  audioFormat: { codec: 'pcm16', sampleRate: 16000 }, // the rate of the audio you send
})
avatar.on('error', ({ error }) => console.error(error.code, error.message))

startButton.onclick = async () => {
  await avatar.start() // must run inside a click or tap; send() only after it resolved
  avatar.send(pcmChunk, false) // mono 16-bit PCM at the declared rate, in chunks as it arrives
  avatar.send(new ArrayBuffer(0), true) // `true` ends the utterance
}
stopButton.onclick = () => avatar.interrupt()
addEventListener('pagehide', () => avatar.dispose())
```

```swift
// iOS
import SpatialRealSDK

let sr = try SpatialReal(appId: "YOUR_APP_ID")
let avatar = try await sr.createSession(AvatarSessionOptions(
    avatarId: "YOUR_AVATAR_ID",
    credential: token,
    container: avatarView,              // a UIView; in SwiftUI wrap one in UIViewRepresentable
    audioFormat: try .pcm16(16000)
))
avatar.onError { e in print("[\(e.code.rawValue)] \(e.message)") }

try await avatar.start()
try avatar.send(chunk)                  // Data, as it arrives
try avatar.send(Data(), end: true)      // ends the utterance
try avatar.interrupt()                  // the user talks over the avatar
await avatar.dispose()                  // the screen goes away
```

```kotlin
// Android: these are suspend functions, call them in a coroutine (lifecycleScope.launch { … })
import ai.spatialreal.android.facade.*

val sr = SpatialReal(context, appId = "YOUR_APP_ID")
val avatar = sr.createSession(AvatarSessionOptions(
    avatarId = "YOUR_AVATAR_ID",
    credential = token,
    container = avatarView,             // a ViewGroup such as FrameLayout; in Compose use AndroidView
    audioFormat = SessionAudioFormat.pcm16(16000),
))
avatar.onError { e -> Log.w("avatar", "[${e.code}] ${e.message}") }

avatar.start()
avatar.send(chunk)                      // ByteArray, as it arrives
avatar.send(ByteArray(0), end = true)   // ends the utterance
avatar.interrupt()
avatar.dispose()
```

**An agent session** is the same with `agentId` in place of `avatarId` and no `audioFormat`
(on iOS and Android the options type is `ChatSessionOptions`): the agent listens on the
microphone and speaks by itself. The app needs microphone access: `NSMicrophoneUsageDescription`
in `Info.plist` on iOS, the `RECORD_AUDIO` permission (declared and requested) on Android.
Page, with a complete screen for each platform: `agent/reach/your-own-app`.

### 4. Check these before saying it works

- Only the server reads the API key; the client gets the App ID and a token per session.
- The declared sample rate is the real rate of the audio sent, or the voice is too fast, slow
  or distorted.
- Billing starts at `start()`, not `createSession()`; call `end()` or `dispose()` when done.
- The SDK does not reconnect by itself. On a `session-ended` error show that the session ended
  (`error.reason` says why); do not reconnect automatically. On `credential-expired` fetch a
  new token and start again with it.
- `send()` failing with `backpressure`: more than 60 seconds of audio is queued; send the chunk
  again once playback has caught up.
- `401 invalid api key` from the token endpoint: the key is wrong, revoked, or belongs to
  another environment than the host called.

An agent can also be shown with no code at all: an embed link is a public page or an iframe
for the customer's site. Page: `agent/reach/page-and-embed`.

Not using the SDK to send audio (a LiveKit or Agora room, a Pipecat pipeline, your own
transport)? Start at `avatar-integration/introduction`; it says which mode fits.

## Documentation

Read the page before writing code for anything not shown above; do not guess an API. Every page
of https://docs.spatialreal.ai is plain Markdown at `https://docs.spatialreal.ai/<page>.md`, and
`https://docs.spatialreal.ai/llms.txt` lists them all. Fetch single pages: `llms-full.txt` holds
the whole site in one file of about 500 KB, which fetch tools cut off after the first part.
With the CLI: `spatialreal docs show <page>`, `spatialreal docs search "<words>"`.

| Section | Start with | Also there |
| --- | --- | --- |
| Overview | `overview/how-it-works` | `overview/session-tokens`, `studio/api-keys`, `resources/error-codes` |
| SpatialReal Agent | `agent/quickstart` | `agent/reach/…` (embed, own app, webhooks), `agent/design/…` (persona, voice, tools, knowledge) |
| Avatar Integration | `avatar-integration/introduction` | `…/sdk-mode/` web, ios, android; `…/livekit/`; `…/host-mode/`; `…/pipecat/` |
| SDK Reference | `sdk-reference/web-sdk/api-reference` | `…/ios-sdk/…`, `…/android-sdk/…`, `…/python-sdk/python-sdk` |
| API Reference | `api-reference/overview` | one page per server endpoint |
