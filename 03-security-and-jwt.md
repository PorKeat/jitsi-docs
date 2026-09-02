# 03. Security, Encryption & Access Control

This guide lists **all 16 implemented security mechanisms**, highlighting the **Top 5 Most Critical Security Pillars** and the **WebRTC Real-Time DRM Content Protection Layer** in plain English.

---

## 🌟 Top 5 Most Critical Core Security Pillars

```text
┌────────────────────────────────────────────────────────────────────────┐
│               🌟 TOP 5 MOST CRITICAL CORE SECURITY PILLARS             │
├────┬─────────────────────────────┬─────────────────────────────────────┤
│ 1  │ ⭐ 🔒 DTLS-SRTP Media       │ Encrypts live video/audio in transit│
│ 2  │ ⭐ 🛡️ True E2EE             │ Server CANNOT see or decode video   │
│ 3  │ ⭐ 🔑 HS256 JWT Token Gate  │ Only authorized users can join      │
│ 4  │ ⭐ 🔗 AES-256-GCM Links     │ Hides real room names in URL links  │
│ 5  │ ⭐ ⚡ Host Authority & Lock │ Kick users & permanently end rooms  │
└────┴─────────────────────────────┴─────────────────────────────────────┘
```

| # | Security Feature | What It Does (Simple English) | Who Can See It? | Priority |
|---|---|---|---|---|
| **1** | **⭐ 🔒 DTLS-SRTP Media Encryption** | Encrypts all live camera and microphone streams across the internet (UDP `10000`). | Attendees & Jitsi server (Safe from Wi-Fi hackers & ISPs). | **CRITICAL** |
| **2** | **⭐ 🛡️ True E2EE (End-to-End)** | Extra browser-level lock on raw video frames before sending. | **ONLY You and other attendees.** (Even server admin CANNOT see it). | **CRITICAL** |
| **3** | **⭐ 🔑 HS256 JWT Token Gatekeeper** | Creates a digitally signed ticket for every user joining the call. | Verified by Prosody before letting anyone in; assigns Host vs Guest role. | **CRITICAL** |
| **4** | **⭐ 🔗 AES-256-GCM Encrypted Links** | Turns real meeting room names into secret, unguessable invite URLs. | Only people who have the link and decryption key. | **CRITICAL** |
| **5** | **⭐ ⚡ Host Authority & "End for All"** | Host can kick disruptive people, ban them, and permanently destroy the room. | Host has complete meeting control. | **CRITICAL** |

---

## 🛡️ Full List of Additional Security Layers

| # | Additional Security Feature | What It Does | Threat Prevented |
|---|---|---|---|
| **6** | **🌐 TLS 1.3 / HTTPS & WSS** | Encrypts all web requests, XMPP signaling, and whiteboard collaboration traffic | Eavesdropping, MITM on web data |
| **7** | **🗝️ Ephemeral Host Secrets (`sec_<hex>`)** | Stores private host key strictly in tab memory (`sessionStorage`), cleared on tab close | Host impersonation, Replay attacks |
| **8** | **🚫 Server-Side Ban Blacklist** | When a host kicks a user, the backend (`/api/kick`) bans them from rejoining | Disruptive attendee return, Ban evasion |
| **9** | **🐳 Non-Root Docker Hardening** | Next.js container runs as unprivileged user (`USER nextjs` UID: 1001) | Container escape, Root privilege escalation |
| **10**| **🏰 Private Docker Network (DMZ)** | Backend microservices (`api`, `prosody`, `jicofo`) talk privately inside `meet.jitsi` | Direct port exposure, External API attacks |
| **11**| **🚪 Knocking Lobby Mode** | Guests must knock and wait in a waiting room until the host approves them | "Zoombombing", Uninvited intrusions |
| **12**| **🧹 Input Sanitization (Regex & Pydantic)** | Restricts room names and inputs to safe alphanumeric sets | XSS, SQL/NoSQL Injection, Path Traversal |
| **13**| **🚦 In-Memory Rate Limiting** | Limits `/api/create-room` (20/min) and `/api/token` (60/min) per IP | API Flooding, DDoS, Room creation spam |
| **14**| **🌐 Dynamic CORS Whitelist** | Restricts cross-origin API requests to verified domains via `CORS_ORIGINS` | Cross-site unauthorized API abuse |
| **15**| **💧 WebRTC Dynamic Watermark (DRM)** | Forensic overlay of attendee Name, Email, and live UTC Timestamp | Screen recording leaks, External phone cameras |
| **16**| **🔒 Host-Only Recording Governance** | Guests are cryptographically blocked from recording via JWT claims | Unauthorized meeting capture & data theft |

---

## 💧 WebRTC Real-Time DRM & Forensic Watermarking

Unity Meet uses the same **Gold Standard** content protection used by **Zoom Enterprise** and **Microsoft Teams Premium**:

1. **Forensic Video Watermark:** A subtle, semi-transparent (`opacity-18`) tiled overlay displays the viewer's **Name, Email/ID, and live ticking UTC Timestamp**.
2. **Anti-Leak Traceability:** If someone uses an external phone camera or screen recording tool, their identity is irreversibly stamped onto the captured video.
3. **Host-Only Recording Lock:** The FastAPI backend signs JWT tokens where `features.recording` is `false` for all guests, ensuring only the meeting host can record.

---

## 🔒 DTLS-SRTP vs True E2EE: What Is the Difference?

| Feature | **DTLS-SRTP** (In-Transit Encryption) | **True E2EE** (End-to-End Encryption) |
| :--- | :--- | :--- |
| **What It Does** | Encrypts the video traveling over the internet between your browser and the video server. | Locks the raw video with a secret key *before* it leaves your browser. |
| **Who Can See The Video?** | **You, other attendees, and the Jitsi server.** | **ONLY You and other attendees.** |
| **Can ISP / Hackers on Wi-Fi see it?** | ❌ **No (100% Protected)** | ❌ **No (100% Protected)** |
| **Can the Server Administrator see it?** | 🖥️ Yes (Server decrypts packet headers to route video) | ❌ **No (Mathematically impossible)** |
| **When to Use It?** | Standard video meetings (supports 50+ people with high efficiency). | Ultra-confidential private conversations. |

---

## 🚪 How the Knocking Lobby & Room Passcodes Work

1. **Lobby Waiting Room Mode:**
   * When enabled, any new guest who joins sees: *"Waiting for the host to let you in"*.
   * The host sees a popup with **"Admit"** or **"Reject"** buttons.
2. **Room Passcode Protection:**
   * When set, guests must type the secret PIN before camera/microphone streams connect.
3. **End Meeting for All:**
   * When the host clicks **"End Meeting for All"**, the server sends an immediate disconnect signal, kicks all participants, and permanently closes the room.

---

## 🔑 JWT Token Structure & Claims Explained

```json
{
  "aud": "my_jitsi_app",
  "iss": "my_jitsi_app",
  "sub": "test-room",
  "room": "test-room",
  "exp": 1788280000,
  "context": {
    "user": {
      "name": "Alex",
      "moderator": true,
      "role": "moderator"
    },
    "features": {
      "recording": true,
      "livestreaming": true,
      "screen-sharing": true
    }
  }
}
```
