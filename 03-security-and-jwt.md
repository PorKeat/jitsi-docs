# 03. Security, Token Auth, Passwords & Knocking Lobby

Unity Meet implements an enterprise multi-layer security architecture designed to prevent unauthorized room creation, meeting hijacking, and unvetted entry.

---

## 🛡️ Security Layers

| Layer | Feature | How It Protects You |
| :--- | :--- | :--- |
| **Layer 1: Token Gate** | **Cryptographic JWT Tokens** | Rooms cannot be initialized or joined without an HMAC-SHA256 token signed by the FastAPI backend service (`api/`). |
| **Layer 2: AES-256-GCM Slugs** | **Authenticated Encryption (AEAD)** | Invite links carry 12-byte IV + 16-byte tag ciphertexts to prevent tampering and URL spoofing. |
| **Layer 3: Host Secret Validation** | **Creator Verification (`sec_<hex>`)** | Only verified room creators holding host secrets can modify room settings or trigger conference termination. |
| **Layer 4: Direct URL Lockdown** | **Disabled Guests & Welcome Page** | `ENABLE_GUESTS=0`, `ENABLE_WELCOME_PAGE=0`. Typing `https://localhost:8443/room` directly is rejected with `403 Forbidden`. |
| **Layer 5: Knocking Lobby Mode** | **Host Admission Gate** | When enabled by the host, participants must "knock" and wait in a waiting room until approved by the host. |
| **Layer 6: Participant Ban Enforcement** | **Token Revocation** | Removed/kicked participants are tracked in the backend ban service and blocked from requesting new tokens. |
| **Layer 7: Media Encryption** | **DTLS-SRTP Stream Encryption** | WebRTC audio and video packets are encrypted with AES-256 over UDP port `10000`. |

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

## 🔑 FastAPI JWT Token Structure & Signing (`api/app/core/security.py`)

The FastAPI backend generates JSON Web Tokens using HMAC-SHA256 (`HS256`):

```python
import time
import jwt
import secrets
from app.core.config import settings

def generate_jitsi_token(
    room: str,
    user_name: str = "Guest",
    is_moderator: bool = False,
    user_id: str | None = None,
    expiration_hours: int = 24,
) -> str:
    app_id = settings.JWT_APP_ID or "my_jitsi_app"
    secret = settings.JWT_APP_SECRET
    unique_id = user_id or (
        f"host_{secrets.token_hex(4)}" if is_moderator else f"guest_{secrets.token_hex(4)}"
    )

    now = int(time.time())
    exp = now + (expiration_hours * 3600)

    payload = {
        "aud": app_id,
        "iss": app_id,
        "sub": "*",
        "room": "*",
        "iat": now,
        "nbf": now - 10,
        "exp": exp,
        "context": {
            "user": {
                "id": unique_id,
                "name": user_name,
                "email": f"{user_name.lower()}.{unique_id}@local.meet",
                "avatar": f"https://api.dicebear.com/7.x/bottts/svg?seed={unique_id}",
                "moderator": bool(is_moderator),
                "affiliation": "owner" if is_moderator else "member",
                "role": "moderator" if is_moderator else "participant",
            },
            "group": "admin" if is_moderator else "member",
            "features": {
                "recording": bool(is_moderator),
                "livestreaming": bool(is_moderator),
                "screen-sharing": True,
            },
        },
    }

    return jwt.encode(payload, secret, algorithm="HS256")
```
