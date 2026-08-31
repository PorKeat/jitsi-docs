# 08 — Next.js Integration & Customization Guide

> Comprehensive developer guide for integrating Jitsi WebRTC with Next.js 14 App Router and building custom conferencing user interfaces.

---

## 1. Architecture Overview

Unity Meet decouples the **WebRTC Media Engine** (Jitsi Videobridge, Prosody XMPP, Jicofo) from the **User Interface** (Next.js 14).

```mermaid
graph TD
    subgraph Frontend ["Next.js 14 App Router (Port 3000)"]
        UI[Custom Native UI<br/>Floating Toolbar, Drawers, Modals]
        WB[Collaborative Whiteboard<br/>Laser Pointer & Color Studio]
        SDK["@jitsi/react-sdk (JitsiMeeting)"]
        TokenAPI["/api/token & /api/create-room"]
    end

    subgraph Backend ["Jitsi WebRTC Stack (Port 8443)"]
        Web[Jitsi Web / Nginx]
        Prosody[Prosody XMPP & JWT Auth]
        Jicofo[Jicofo Focus Allocator]
        JVB[JVB Media Bridge (UDP 10000)]
        Excalidraw[Excalidraw Relay (Port 3002)]
    end

    SDK <-->|IFrame API Commands & Events| Web
    TokenAPI <-->|Signed HS256 JWT| Prosody
    Web <--> Prosody
    Prosody <--> Jicofo
    Jicofo <--> JVB
    WB <-->|Live Sync| Excalidraw
```

---

## 2. Server-Side Token Generation (`web-app/lib/jwt.ts`)

To prevent room hijacking and control moderator permissions, JWT tokens are generated **server-side only** using HS256:

```typescript
import jwt from "jsonwebtoken";

export function generateJitsiToken({
  room,
  userName = "Guest",
  isModerator = false,
}: {
  room: string;
  userName?: string;
  isModerator?: boolean;
}) {
  const secret = process.env.JWT_APP_SECRET || "your_secret";
  const appId = process.env.JWT_APP_ID || "my_jitsi_app";

  const payload = {
    aud: appId,
    iss: appId,
    sub: "*",
    room: room,
    context: {
      user: {
        name: userName,
        email: `${userName.toLowerCase().replace(/\s+/g, ".")}@local.meet`,
        avatar: `https://api.dicebear.com/7.x/identicon/svg?seed=${encodeURIComponent(userName)}`,
        moderator: isModerator,
      },
      features: {
        recording: true,
        livestreaming: true,
        "screen-sharing": true,
      },
    },
  };

  return jwt.sign(payload, secret, {
    algorithm: "HS256",
    expiresIn: "24h",
  });
}
```

---

## 3. Dynamic SDK Import (Disabling SSR)

The `@jitsi/react-sdk` package requires client-side `window` and `document` APIs. Always import dynamically with `ssr: false`:

```tsx
import dynamic from "next/dynamic";

const JitsiMeeting = dynamic(
  () => import("@jitsi/react-sdk").then((mod) => mod.JitsiMeeting),
  {
    ssr: false,
    loading: () => (
      <div className="flex-1 flex items-center justify-center bg-[#0a0814] text-white">
        <div className="w-12 h-12 border-4 border-purple-500 border-t-transparent rounded-full animate-spin"></div>
      </div>
    ),
  }
);
```

---

## 4. Jitsi IFrame API Control & Event Listeners

Store the API instance in a `useRef` to trigger actions and listen to state changes:

```tsx
const apiRef = useRef<any>(null);

<JitsiMeeting
  domain="localhost:8443"
  roomName={room}
  jwt={jwt}
  userInfo={{
    displayName: userName,
    email: `${userName.toLowerCase()}@local.meet`,
  }}
  configOverwrite={{
    startWithAudioMuted: false,
    startWithVideoMuted: false,
    toolbarButtons: [], // Disables default Jitsi toolbar so Next.js owns the UI
    whiteboard: {
      enabled: true,
      collabServerBaseUrl: "http://localhost:3002",
    },
  }}
  interfaceConfigOverwrite={{
    SHOW_JITSI_WATERMARK: false,
    SHOW_BRAND_WATERMARK: false,
    TOOLBAR_BUTTONS: [],
  }}
  onApiReady={(externalApi) => {
    apiRef.current = externalApi;

    // Listen to conference events
    externalApi.addListener("videoConferenceLeft", () => setMeetingEnded(true));
    externalApi.addListener("audioMuteStatusChanged", ({ muted }) => setIsMuted(muted));
    externalApi.addListener("participantJoined", (p) => handleParticipantJoin(p));
  }}
/>
```

### Essential API Commands:
* `apiRef.current.executeCommand("toggleAudio")`: Toggle microphone.
* `apiRef.current.executeCommand("toggleVideo")`: Toggle webcam.
* `apiRef.current.executeCommand("toggleShareScreen")`: Toggle screen share.
* `apiRef.current.executeCommand("toggleChat")`: Open/close text chat.
* `apiRef.current.executeCommand("toggleRaiseHand")`: Raise or lower hand.
* `apiRef.current.executeCommand("muteEveryone")`: Host action to mute all guests.
* `apiRef.current.executeCommand("endConference")`: Host action to terminate meeting for all attendees.
* `apiRef.current.executeCommand("password", pinCode)`: Locks room with passcode.
* `apiRef.current.executeCommand("toggleWhiteboard")`: Synchronizes Excalidraw board across all attendees.

---

## 5. UI Customization Examples

### A. Pre-Join Green Room (Live VU Meter)
Using Web Audio API and `navigator.mediaDevices.getUserMedia`:

```tsx
const audioCtx = new AudioContext();
const analyser = audioCtx.createAnalyser();
const source = audioCtx.createMediaStreamSource(stream);
source.connect(analyser);

const data = new Uint8Array(analyser.frequencyBinCount);
analyser.getByteFrequencyData(data);
const volume = data.reduce((a, b) => a + b, 0) / data.length;
setAudioLevel(volume);
```

### B. Custom Floating Capsule Toolbar
Positioned at bottom center with frosted glass styling:

```tsx
<div className="absolute bottom-6 left-1/2 -translate-x-1/2 z-40 flex items-center gap-2.5 bg-[#120e26]/90 backdrop-blur-2xl border border-purple-500/30 px-5 py-2.5 rounded-full shadow-2xl">
  <button onClick={() => apiRef.current?.executeCommand("toggleAudio")}>
    {isMuted ? <MicOff /> : <Mic />}
  </button>
  <button onClick={() => apiRef.current?.executeCommand("toggleVideo")}>
    {isVideoMuted ? <CameraOff /> : <Camera />}
  </button>
  <button onClick={() => setShowWhiteboard(true)}>
    <Edit3 />
  </button>
</div>
```

### C. 60 FPS Real-Time Laser Pointer
Rendered on a separate overlay canvas:

```tsx
const renderLaser = () => {
  const ctx = laserCanvas.getContext("2d");
  ctx.clearRect(0, 0, width, height);
  const now = Date.now();
  const points = laserPoints.filter((p) => now - p.time < 650);

  points.forEach((p, i) => {
    const alpha = 1 - (now - p.time) / 650;
    ctx.strokeStyle = `rgba(239, 68, 68, ${alpha})`;
    ctx.stroke();
  });

  requestAnimationFrame(renderLaser);
};
```

---

## 6. Theme & Branding Customization

1. **Color Palette:** Modify `web-app/tailwind.config.js` to change primary accent colors (e.g. from purple `#a855f7` to blue `#3b82f6` or brand colors).
2. **Logo Asset:** Replace `web-app/public/logo.png` and `config/web/images/watermark.png` with your company logo.
3. **App Title:** Set `APP_NAME` in `web-app/app/meeting/[room]/page.tsx` and `app/layout.tsx`.
