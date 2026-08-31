# 07. Configuration & Architecture Reference

This guide details the Jitsi backend configurations, environment variables, and directory layout.

---

## ⚙️ Environment Variables (`.env`)

| Variable | Description | Value in Unity Meet |
| :--- | :--- | :--- |
| `PUBLIC_URL` | Base public URL for the Jitsi web service | `https://localhost:8443` |
| `HTTP_PORT` | Nginx HTTP port | `8080` |
| `HTTPS_PORT` | Nginx HTTPS port | `8443` |
| `PORTAL_PORT` | Next.js Frontend port | `3000` |
| `ENABLE_AUTH` | Require token authentication | `1` |
| `ENABLE_GUESTS` | Allow unauthenticated guests | `0` (Disabled - all guests use signed JWTs) |
| `AUTH_TYPE` | Authentication mechanism | `jwt` |
| `JWT_APP_ID` | Application identifier for JWT | `my_jitsi_app` |
| `JWT_APP_SECRET` | Secret key for signing tokens | Configured in `.env` |
| `JVB_PORT` | Media UDP port for WebRTC audio/video | `10000` |

---

## 🔐 Built-In Next.js JWT Generation (`web-app/lib/jwt.ts`)

Tokens are signed on-demand by the Next.js server:

```typescript
import jwt from "jsonwebtoken";

const JWT_APP_ID = process.env.JWT_APP_ID || "my_jitsi_app";
const JWT_APP_SECRET = process.env.JWT_APP_SECRET || "secret";

export function generateJitsiToken(room: string, userName: string, isModerator = false) {
  const payload = {
    aud: JWT_APP_ID,
    iss: JWT_APP_ID,
    sub: "*",
    room: room,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 86400,
    context: {
      user: {
        name: userName,
        email: `${userName.toLowerCase()}@unitymeet.local`,
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

---

## 🗂️ Project Directory Structure

```text
Jitsi/
├── .env                  # Environment secrets and port configurations
├── docker-compose.yml    # Docker services (web, prosody, jicofo, jvb, portal)
├── docs/                 # Documentation (Symlinked to ~/Desktop/jitsi-docs)
├── config/               # Volume mounts for Jitsi daemon configurations
│   ├── jicofo/           # Jicofo conference focus daemon config
│   ├── jvb/              # Jitsi Videobridge media relay config
│   ├── prosody/          # Prosody XMPP signaling & auth config
│   └── web/              # Jitsi web frontend images & config
└── web-app/              # Next.js 14 Full-Stack Meeting Application
    ├── app/              # Next.js App Router (homepage, meeting room, API routes)
    ├── lib/              # JWT token generator and utility functions
    └── public/           # Logo assets & brand watermarks
```
