# Unity Meet — Production Documentation

> Modern, ultra-secure, custom-branded WebRTC video conferencing built with **Next.js 16 (Turbopack)**, **FastAPI Backend Service**, and **Jitsi WebRTC SFU**.

---

## 🌟 Core Features

* 🎛️ **Modern Workspace Dashboard:**
  * Clean, minimalist UI with Light and Dark mode theme switching.
  * Real-time client RTT ping latency, server uptime, and active meetings status.
  * Quick Access & Recent Rooms with **In Progress** (1-click Rejoin) and **Ended** badges.
  * Live Mini Calendar of the current month with Keycloak sync preview.
* 🎥 **Pre-Join "Green Room" Lobby:** Live camera test, Web Audio volume equalizer visualizer, device toggles, robot avatar selector (DiceBear bottts), and custom background studio.
* 🔒 **Meeting Room Lock & Knocking Lobby:** Host admission control, meeting PINs, and anti-hijack token gates.
* ✏️ **Built-in Collaborative Whiteboard:** Interactive Next.js drawing canvas with 60 FPS neon laser pointer, shapes, text, and 1-click Color Studio.
* 🛡️ **Cryptographic Token Auth & AES-256-GCM:** HMAC-SHA256 tokens and AEAD ciphertext invite links issued by the FastAPI backend service.
* ⚡ **Ultra-Low Latency SFU:** 100% in-browser WebRTC via JVB over DTLS-SRTP (Port 10000 UDP).
* 📱 **Native Next.js 16 UI:** Custom-branded dark/light interface, floating capsule toolbar, and seamless direct exit to dashboard.

---

## 📚 Documentation Index

| File | Topic | Description |
| :--- | :--- | :--- |
| [**01. Architecture & Protocols**](./01-architecture-and-protocols.md) | System Design & Protocols | Microservices architecture, FastAPI backend, WebRTC, XMPP, DTLS-SRTP, Colibri, and SFU data flows. |
| [**02. Setup & Deployment**](./02-setup-and-deployment.md) | Installation & Startup | Step-by-step setup, TLS 1.3 SAN SSL certs, environment configuration, and startup commands. |
| [**03. Security, Passwords & Lobby**](./03-security-and-jwt.md) | Security & Access Control | AES-256-GCM links, Host Secret validation, Knocking Lobby mode, meeting passwords, and JWT tokens. |
| [**04. Customization & Features**](./04-customization-and-branding.md) | UI, Green Room & Whiteboard | Native Next.js 16 UI, Green Room camera lobby, Whiteboard canvas, and toolbar controls. |
| [**05. Network & Port Allocation**](./05-network-and-ports.md) | Networking & Firewalls | Port mapping table (3000, 8000, 8443, 3002, 10000/udp, 4443), NAT traversal, and SFU traffic. |
| [**06. Operations & Troubleshooting**](./06-operations-and-troubleshooting.md) | DevOps & Maintenance | Docker Compose management cheat sheet, log inspection, and troubleshooting fixes. |
| [**07. Configuration & Reference**](./07-config-and-scripts-reference.md) | Config & Architecture Guide | Detailed breakdown of environment variables, FastAPI JWT signing, and directory layout. |
| [**08. Next.js Integration & Customization**](./08-nextjs-integration-and-customization.md) | Frontend Dev & SDK Guide | Next.js 16 App Router, Turbopack, dynamic SDK imports, IFrame API commands/events, custom toolbar, Green Room, and Whiteboard. |

---

## 🚀 Standard Docker Commands

```bash
# Start all services
docker compose up -d

# Check status of all containers
docker compose ps

# View live real-time logs
docker compose logs -f

# Run FastAPI automated tests
python -m unittest api/tests/test_api.py

# Lint & build Next.js 16 web application
cd web-app && pnpm build && pnpm lint

# Restart all services
docker compose restart

# Stop all services
docker compose down
```
