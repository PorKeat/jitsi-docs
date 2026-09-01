# 01. Architecture & Protocol Overview

Unity Meet is built as an enterprise-grade WebRTC conferencing platform consisting of containerized microservices and standardized networking protocols.

---

## 🏛️ Microservices Stack

| Service | Container | Role | Key Technologies |
| :--- | :--- | :--- | :--- |
| **Web Portal** | `jitsi-web-1` (Port 3000) | Modern UI frontend, Workspace Dashboard, Green Room, native toolbar, Whiteboard | Next.js 16 (Turbopack), React 19, `@jitsi/react-sdk`, Tailwind CSS, TypeScript |
| **FastAPI Backend** | `jitsi-api-1` (Port 8000) | JWT token signing, room lifecycle, AES-256-GCM encryption, ban enforcement, telemetry | Python 3.11, FastAPI, Pydantic, Cryptography (AESGCM), PyJWT |
| **Nginx Web Proxy** | `jitsi-jitsi-web-1` (Port 8443) | Reverse proxy, static Jitsi assets, WebSockets/BOSH endpoint | Nginx, OpenSSL TLS 1.3 |
| **Prosody XMPP** | `jitsi-prosody-1` | XMPP signaling server, JWT validation module, MUC (Multi-User Chat) rooms | Lua, Prosody XMPP Server, `mod_auth_token` |
| **Jicofo** | `jicofo` | Conference focus agent, bridges participants, allocates JVB media channels | Java 17, XMPP Focus Component |
| **JVB SFU** | `jitsi-jvb-1` (Port 10000 UDP) | Selective Forwarding Unit (SFU), WebRTC audio/video relay, Colibri REST API | Java, Colibri Protocol, libnice, WebRTC |
| **Whiteboard Relay** | `jitsi-whiteboard-1` (Port 3002) | Real-time collaborative Excalidraw drawing canvas WebSocket relay | Node.js, WebSocket Server |

---

## 🌐 Network Protocols Used

### 1. WebRTC (Real-Time Communications)
- **Audio/Video Media:** Transmitted over **SRTP** (Secure Real-time Transport Protocol) with 128-bit / 256-bit AES encryption.
- **Key Exchange & Handshake:** Handled via **DTLS** (Datagram Transport Layer Security) over UDP port `10000`.
- **NAT Traversal & Candidate Discovery:** Uses **ICE** (Interactive Connectivity Establishment) and **STUN** (Session Traversal Utilities for NAT).
- **Data Channels:** Transmitted over **SCTP** (Stream Control Transmission Protocol) encapsulated in DTLS for chat, reactions, and dominant speaker events.

### 2. XMPP & Signaling Protocols
- **Client Signaling:** Browser clients establish secure WebSocket / BOSH connections to `wss://localhost:8443/xmpp-websocket` for presence and room state.
- **MUC (Multi-User Chat):** Prosody manages room occupancy (`muc.meet.jitsi`), presence, and speaker status.
- **Colibri Protocol (XMPP / REST):** Jicofo communicates with JVB over internal port `8088` to dynamically allocate and destroy video bridges for active conferences.

### 3. Authentication & Cryptography Protocols
- **JWT (JSON Web Token):** Cryptographically signed HMAC-SHA256 tokens issued by the FastAPI backend (`api/`) and verified by Prosody before any user can create or join a room.
- **AES-256-GCM AEAD:** URL-safe ciphertext invite slugs containing room credentials with random 12-byte IVs and 16-byte authentication tags.

---

## 🔄 End-to-End Call Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as User Browser
    participant Web as Next.js 16 (:3000)
    participant API as FastAPI Backend (:8000)
    participant Nginx as Web Proxy (:8443)
    participant Prosody as Prosody XMPP
    participant Jicofo as Jicofo Agent
    participant JVB as JVB SFU (:10000 UDP)

    User->>Web: 1. Click "Start Instant Meeting"
    Web->>API: 2. POST /api/create-room
    API->>API: 3. Generate Host Secret & Signed HS256 JWT
    API-->>Web: 4. Return Room Code + Host Token
    Web-->>User: 5. Transition to Green Room Lobby
    User->>Nginx: 6. Connect WebRTC Client with JWT
    Nginx->>Prosody: 7. Forward XMPP Auth & Validate Token
    Prosody-->>User: 8. JWT Validated, Join Room MUC
    Prosody->>Jicofo: 9. Trigger Conference Allocation
    Jicofo->>JVB: 10. Allocate Media Bridge (Colibri)
    JVB-->>User: 11. Establish DTLS-SRTP Media Streams (Port 10000 UDP)
    Note over User,JVB: Live Encrypted HD Video & Audio Active
```

---

## 💡 Selective Forwarding Unit (SFU) vs Mesh P2P

Unity Meet is optimized to use **SFU mode** (`p2p.enabled = false`):
* **No P2P Connection Timeouts:** Bypasses direct peer-to-peer ICE negotiation delays.
* **Scalability:** Each participant sends 1 video stream to JVB and receives only active speaker streams, reducing CPU and bandwidth consumption.
