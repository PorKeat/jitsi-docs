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
| `API_PORT` | FastAPI Backend port | `8000` |
| `ENABLE_AUTH` | Require token authentication | `1` |
| `ENABLE_GUESTS` | Allow unauthenticated guests | `0` (Disabled - all guests use signed JWTs) |
| `AUTH_TYPE` | Authentication mechanism | `jwt` |
| `JWT_APP_ID` | Application identifier for JWT | `my_jitsi_app` |
| `JWT_APP_SECRET` | Secret key for signing tokens | Configured in `.env` |
| `JVB_PORT` | Media UDP port for WebRTC audio/video | `10000` |

---

## 🗂️ Project Directory Structure

```text
Jitsi/
├── .env                  # Environment secrets and port configurations
├── docker-compose.yml    # Docker services (web, prosody, jicofo, jvb, api, web, whiteboard)
├── api/                  # High-performance FastAPI Backend Service (Port 8000)
│   ├── app/
│   │   ├── api/          # Endpoints: rooms, token, settings, bans, telemetry
│   │   ├── core/         # Config & Cryptography (AES-256-GCM, HS256 JWT)
│   │   ├── schemas/      # Strongly typed Pydantic models
│   │   └── services/     # Room state, Token service, Ban service
│   ├── tests/            # Automated test suite
│   └── Dockerfile        # Python 3.11 container
├── config/               # Volume mounts for Jitsi daemon configurations
│   ├── jicofo/           # Jicofo conference focus daemon config
│   ├── jvb/              # Jitsi Videobridge media relay config
│   ├── prosody/          # Prosody XMPP signaling & auth config
│   └── web/              # Jitsi web frontend images & config
└── web-app/              # Next.js 16 (Turbopack) Full-Stack Meeting Application
    ├── app/              # Next.js App Router (dashboard, prejoin, meeting room)
    ├── components/       # Meeting UI, Green Room, Drawers, Modals, Whiteboard
    ├── public/           # Logo assets & brand watermarks
    └── eslint.config.mjs # ESLint 9 Flat Config
```
