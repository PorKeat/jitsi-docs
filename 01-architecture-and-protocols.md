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

## 🏛️ Microservices Stack Breakdown

| Service | Container | Role (Simple Explanation) | Tech Stack |
| :--- | :--- | :--- | :--- |
| **Web Portal** | `jitsi-web-1` (Port 3000) | The main website with dashboard, camera preview, and video call controls. | Next.js 16 (Turbopack), React 19, Tailwind CSS |
| **FastAPI Backend** | `jitsi-api-1` (Port 8000) | Signs login tokens, encrypts invite links, checks room status, and manages bans. | Python 3.11, FastAPI, Pydantic v2, PyJWT |
| **Nginx Web Proxy** | `jitsi-jitsi-web-1` (Port 8443) | Secure HTTPS gateway handling SSL certificates and static WebRTC files. | Nginx, OpenSSL TLS 1.3 |
| **Prosody XMPP** | `jitsi-prosody-1` | Handles chat, validates JWT tokens, and manages participant presence. | Lua, Prosody XMPP Server |
| **Jicofo** | `jicofo` | Room focus manager that allocates video bridges for each conference. | Java 17, XMPP Focus Component |
| **JVB SFU** | `jitsi-jvb-1` (Port 10000 UDP) | Video Media Server that routes live audio and video streams between attendees. | Java, WebRTC, Colibri Protocol |
| **Whiteboard Relay** | `jitsi-whiteboard-1` (Port 3002) | Real-time WebSocket relay server for the shared collaborative drawing canvas. | Node.js, Excalidraw Backend |

---

## 🔄 End-to-End Call Flow (Step-by-Step)

```text
1. User clicks "Start Instant Meeting" on Next.js 16 Web UI (Port 3000)
   ▼
2. Web UI asks FastAPI Backend (Port 8000): "Create a new meeting room"
   ▼
3. FastAPI generates a Host Secret (sec_<hex>) and signs a secure HS256 JWT token
   ▼
4. User enters Green Room to preview camera and test microphone volume
   ▼
5. User enters meeting -> Nginx Gateway (Port 8443) verifies JWT with Prosody XMPP
   ▼
6. Prosody validates token and asks Jicofo: "Allocate a video bridge for this room"
   ▼
7. Jicofo connects to JVB (Videobridge SFU)
   ▼
8. JVB opens encrypted DTLS-SRTP media stream on UDP Port 10000
   ▼
9. Live HD Video & Audio streams flow smoothly between all attendees!
```

---

## 💡 Why SFU (Videobridge) is Better than Mesh P2P

* **Mesh (P2P):** If you are in a 5-person call, your laptop must send 4 separate video streams and receive 4 streams. Your computer overheats and slows down.
* **SFU (Unity Meet):** You send **only 1 video stream** to the JVB server. The server forwards it to everyone else. Your computer stays fast and cool, supporting 50+ participants easily!
