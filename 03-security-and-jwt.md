# 03. Security, Encryption & Access Control

This guide explains all security layers, encryption methods, and access controls in Unity Meet in **simple, plain English**.

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

## 🛡️ The 7 Security Layers (Who Can See What?)

| Security Layer | Technology | How It Works (Simple English) | Who Can Access It? |
| :--- | :--- | :--- | :--- |
| **1. Video & Audio Security** | **DTLS-SRTP (UDP 10000)** | Encrypts all live camera and microphone packets over the internet. | Attendees & the Jitsi server. Blocked for all outside sniffers. |
| **2. Entry Gatekeeper** | **HS256 JWT Tokens** | A digital ticket signed by the FastAPI backend specifying if you are a Host or a Guest. | Prosody XMPP verifies the digital signature before opening the room. |
| **3. Invite Link Protection** | **AES-256-GCM (AEAD)** | Hides real meeting names inside random-looking secure links with a 12-byte Nonce. | Only users who click the invite link and possess the decryption key. |
| **4. Host Authority Key** | **Ephemeral Secret (`sec_<hex>`)** | A private key stored in your browser tab (`sessionStorage`). Cleared on tab close. | Only the meeting creator's browser. |
| **5. Container Hardening** | **Non-Root Docker User** | Web container runs as an unprivileged user (`nextjs` UID: 1001) with zero root powers. | Protects the host operating system from container breakout attacks. |
| **6. Lobby Gatekeeper** | **Knocking Waiting Room** | Guests must knock outside until the host clicks "Approve". | Host controls admission; guests cannot sneak in. |
| **7. Ban Enforcement** | **Server-Side Blacklist** | Kicked disruptive users are added to a ban registry. | Banned users are blocked from re-entering. |

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
