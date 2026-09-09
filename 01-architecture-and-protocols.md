# 01. Architecture, Protocols & Plain English Guide

This guide explains the complete architecture, networking protocols, and technical terms used in Unity Meet in **simple, easy-to-understand English**.

---

## 📖 Plain English Dictionary: What It Is, How It Works & Who Can See

| Hard Term | What Is It? | How Does It Work? | Who Can See It? |
| :--- | :--- | :--- | :--- |
| **DTLS-SRTP** | The encryption shield for live video & audio streams. | Encrypts UDP packets as they travel over the internet to the video server. | **You, other attendees, and the Jitsi server.** (Hackers on Wi-Fi and your ISP CANNOT see anything). |
| **True E2EE** | True End-to-End Encryption where nobody in the middle can see anything. | Your browser locks raw video frames with a private password before sending. | **ONLY You and other attendees.** (Even the server administrator CANNOT see your video). |
| **SFU (Videobridge / JVB)** | A smart video router (Selective Forwarding Unit). | Receives 1 video stream from you and forwards it to others without decoding or mixing. | Routes packets to all connected attendees. |
| **XMPP (Prosody)** | The live messaging and presence protocol. | Sends instant updates whenever someone joins, leaves, mutes, or raises their hand. | The Prosody server broadcasts presence events to attendees in the room. |
| **Jicofo** | The meeting room traffic controller / allocator. | Automatically assigns rooms to the best available video server (JVB). | Internal system coordinator. |
| **JWT (HS256 Token)** | A digital entry ticket signed by the backend. | Contains your username, room name, and whether you are a Host or a Guest. | Checked by the Prosody server before letting you into the room. |
| **AES-256-GCM** | Military-grade authenticated encryption for links. | Scrambles the real room name with a 12-byte random number and 16-byte tag. | Only users with the invite link and decryption key can see the real room name. |
| **ICE / STUN / TURN** | Connection pathfinders through routers & firewalls. | STUN finds your public IP address; ICE tests the fastest path; TURN relays if blocked. | Used to establish peer-to-server WebRTC network channels. |
| **WebSockets (WSS)** | An open, two-way communication pipe in your browser. | Keeps a permanent connection open so chat, signaling, and whiteboard draw in real-time. | Encrypted under TLS 1.3 HTTPS/WSS. |

---

## 🏛️ Microservices Stack Breakdown (Production Kubernetes)

| Service | Pod / Component | Role (Simple Explanation) | Tech Stack |
| :--- | :--- | :--- | :--- |
| **Ingress Gateway** | Traefik v3 IngressRoute | Routes traffic from MetalLB VIP (`10.1.18.200`) to microservices with SSL & WebSockets. | Traefik v3, MetalLB, Let's Encrypt |
| **Web Portal UI** | `unity-meet-web` (Port 3000) | Full Next.js 16 UI with dashboard, green room, smooth stage zoom, and meeting frame. | Next.js 16 (Turbopack), React 19, Tailwind CSS |
| **API Microservice**| `unity-meet-api` (Port 8000) | High-performance Go service signing JWTs, encrypting links, room locks, and bans. | Go (Golang) 1.22, Gin/Chi, HMAC-SHA256, AES-256-GCM |
| **In-Memory Cache** | `unity-meet-valkey` (Port 6379) | Sub-millisecond distributed cache for active rooms, knocking lobby, and rate limits. | Valkey 8.0, Longhorn CSI Persistence |
| **Database** | `unity-meet-postgres` (Port 5432)| Relational database for persistent users, audit logs, and meeting archives. | PostgreSQL 15, Longhorn CSI Persistence |
| **Jitsi Web Gateway**| `unity-meet-jitsi-meet-web` | Serves Jitsi WebRTC core assets, BOSH (`/http-bind`), and WebSocket endpoints. | Nginx, lib-jitsi-meet (patched via postStart) |
| **Prosody XMPP** | `unity-meet-jitsi-meet-prosody-0`| Real-time XMPP signaling, JWT authentication, and participant presence. | Lua 5.4, Prosody XMPP Server |
| **Jicofo** | `unity-meet-jitsi-meet-jicofo` | Conference focus allocator assigning media bridges for rooms. | Java 17, XMPP Focus Component |
| **JVB Videobridge** | `unity-meet-jitsi-meet-jvb-0` (2 pods)| High-throughput SFU routing WebRTC audio/video on UDP 10000 per node. | Java 17, WebRTC, Colibri Protocol |
| **Jibri Recorder** | `unity-meet-jitsi-meet-jibri` | Headless Chromium recorder recording meetings to Longhorn storage. | Java, ALSA Virtual Audio, FFmpeg |

---

## 🔄 End-to-End Call Flow (Step-by-Step)

```text
1. User clicks "Start Instant Meeting" on Next.js 16 Web UI (meet.unity-workspace.com)
   ▼
2. Web UI calls Go API Microservice (/api/generate-room-and-token):
   - Signs secure HS256 JWT token with room name and user identity
   - Generates AES-256-GCM encrypted room invite link
   - Stores active room state in Valkey 8 (6379) & PostgreSQL 15
   ▼
3. User enters Green Room to preview camera and test microphone volume
   ▼
4. User enters meeting -> Traefik IngressRoute proxies WebSockets to Prosody XMPP
   ▼
5. Prosody validates JWT token and asks Jicofo: "Allocate a video bridge for this room"
   ▼
6. Jicofo connects to JVB Videobridge (Node 2: 10.1.18.10 or Node 3: 10.1.18.11)
   ▼
7. JVB opens encrypted DTLS-SRTP media stream on UDP Port 10000 (host network)
   ▼
8. Live HD Video & Audio streams flow with GPU-accelerated smooth zoom & stage fit/fill!
```

---

## 💡 Why SFU (Videobridge) is Better than Mesh P2P

* **Mesh (P2P):** If you are in a 5-person call, your laptop must send 4 separate video streams and receive 4 streams. Your computer overheats and slows down.
* **SFU (Unity Meet):** You send **only 1 video stream** to the JVB server. The server forwards it to everyone else. Your computer stays fast and cool, supporting 50+ participants easily!
