# 03. Security, Encryption & Access Control

This guide explains all security layers, encryption methods, and access controls in Unity Meet in **simple, plain English**.

---

## 🌟 The 5 Core Security Pillars

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   🌟 5 CORE SECURITY PILLARS                           │
├────┬─────────────────────────────┬─────────────────────────────────────┤
│ 1  │ 🔒 DTLS-SRTP Media Transit  │ Encrypts video/audio over internet  │
│ 2  │ 🛡️ True E2EE (Client-Level) │ Server CANNOT see or decode video   │
│ 3  │ 🔑 HS256 JWT Token Gate     │ Only authorized users can join      │
│ 4  │ 🔗 AES-256-GCM Secure Links │ Hides real room names in URL links  │
│ 5  │ ⚡ Host Authority & Ban Lock │ Kick users & permanently end rooms  │
└────┴─────────────────────────────┴─────────────────────────────────────┘
```

| # | Core Security | What It Does (Simple English) | Who Can See It? |
|---|---|---|---|
| **1** | **🔒 DTLS-SRTP Media Encryption** | Encrypts all live camera and microphone streams across the internet (UDP `10000`). | Attendees and the Jitsi server (Safe from Wi-Fi hackers & ISPs). |
| **2** | **🛡️ True E2EE (End-to-End Encryption)** | Extra browser-level lock on raw video frames before sending. | **ONLY You and other attendees.** (Even the server administrator CANNOT see it). |
| **3** | **🔑 HS256 JWT Token Gatekeeper** | Creates a digitally signed ticket for every user joining the call. | Verified by Prosody before letting anyone in; assigns Host vs Guest role. |
| **4** | **🔗 AES-256-GCM Encrypted Links** | Turns real meeting room names into secret, unguessable invite URLs. | Only people who have the link and decryption key. |
| **5** | **⚡ Host Authority & "End for All"** | Host can kick disruptive people, ban them, and permanently destroy the room. | Host has complete meeting control. |

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
  "sub": "*",
  "room": "room_name",
  "exp": 1788280000,
  "context": {
    "user": {
      "name": "Alex",
      "moderator": true,
      "role": "moderator"
    }
  }
}
```

* **`room`**: The exact meeting room name (prevents token reuse in other rooms).
* **`moderator: true`**: Confirms host powers (mute all, kick, end meeting).
* **`exp`**: Expiration timestamp so old tokens cannot be reused.
