# 03. Security, Token Auth, Passwords & Knocking Lobby

Unity Meet implements an enterprise multi-layer security architecture designed to prevent unauthorized room creation, meeting hijacking, and unvetted entry.

---

## 🛡️ Security Layers

| Layer | Feature | How It Protects You |
| :--- | :--- | :--- |
| **Layer 1: Token Gate** | **Cryptographic JWT Tokens** | Rooms cannot be initialized or joined without an HMAC-SHA256 token signed by the Next.js server. |
| **Layer 2: Direct URL Lockdown** | **Disabled Guests & Welcome Page** | `ENABLE_GUESTS=0`, `ENABLE_WELCOME_PAGE=0`. Typing `https://localhost:8443/room` directly is rejected with `403 Forbidden`. |
| **Layer 3: Knocking Lobby Mode** | **Host Admission Gate** | When enabled by the host, participants must "knock" and wait in a waiting room until approved by the host. |
| **Layer 4: Meeting Passcode (PIN)** | **Room Passwords** | Hosts can set a custom meeting password via the **Security** toolbar button for VIP meetings. |
| **Layer 5: Media Encryption** | **DTLS-SRTP Stream Encryption** | WebRTC audio and video packets are encrypted with AES-256 over UDP port `10000`. |

---

## 🚪 How Knocking Lobby & Room Lock Work

1. **Enabling Lobby / Room Lock:**
   * Inside any active meeting, the host clicks the **Security** button in the bottom toolbar.
   * Toggle **"Lobby Mode"** or click **"Add Password"** to lock the room.
2. **Guest Entry with Lobby Enabled:**
   * When a guest joins, they enter a waiting room screen with the message *"Waiting for the host to let you in"*.
   * The host receives a real-time prompt with **"Admit"** or **"Reject"** buttons.
3. **Guest Entry with Room Password:**
   * Guests are prompted to enter the meeting PIN before audio/video streams are unlocked.

---

## 🔑 JWT Token Structure & Signing

The Next.js backend generates JSON Web Tokens on the server using HMAC-SHA256 (`HS256`).

```typescript
import jwt from "jsonwebtoken";

const JWT_APP_ID = process.env.JWT_APP_ID || "my_jitsi_app";
const JWT_APP_SECRET = process.env.JWT_APP_SECRET || "your_secret";

export function generateJitsiToken(room: string, userName: string, isModerator = false) {
  const payload = {
    aud: JWT_APP_ID,
    iss: JWT_APP_ID,
    sub: "*",
    room: room,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 86400, // 24-hour expiration
    context: {
      user: {
        name: userName,
        email: `${userName.toLowerCase()}@unitymeet.local`,
        avatar: `https://api.dicebear.com/7.x/identicon/svg?seed=${userName}`,
        moderator: isModerator,
      },
      features: {
        recording: true,
        livestreaming: true,
        "screen-sharing": true,
      },
    },
  };

  return jwt.sign(payload, JWT_APP_SECRET, { algorithm: "HS256" });
}
```
