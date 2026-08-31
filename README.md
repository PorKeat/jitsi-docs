# Unity Meet — Production Documentation

> Modern, ultra-secure, custom-branded WebRTC video conferencing built on top of Jitsi Meet and Next.js 14.

---

## 🌟 Core Features

* 🎥 **Pre-Join "Green Room" Lobby:** Live camera test, audio level equalizer visualizer, device toggles, and custom display name input.
* 🔒 **Meeting Room Lock & Knocking Lobby:** Host admission control, meeting PINs, and anti-hijack token gates.
* ✏️ **Built-in Collaborative Whiteboard:** Interactive Next.js drawing canvas for real-time team diagramming and sketching.
* 🛡️ **Cryptographic Token Auth:** HMAC-SHA256 tokens blocking unauthorized room creation.
* ⚡ **Ultra-Low Latency SFU:** 100% in-browser WebRTC via JVB over DTLS-SRTP (Port 10000 UDP).
* 🎨 **3D Purple Glassmorphism UI:** Custom-branded dark interface, floating pill toolbar, and seamless post-call screens.

---

## 📚 Documentation Index

| File | Topic | Description |
| :--- | :--- | :--- |
| [**01. Architecture & Protocols**](./01-architecture-and-protocols.md) | System Design & Protocols | Microservices architecture, WebRTC, XMPP, DTLS-SRTP, Colibri, and SFU data flows. |
| [**02. Setup & Deployment**](./02-setup-and-deployment.md) | Installation & Startup | Step-by-step setup, TLS 1.3 SAN SSL certs, environment configuration, and startup commands. |
| [**03. Security, Passwords & Lobby**](./03-security-and-jwt.md) | Security & Access Control | Room hijacking prevention, Knocking Lobby mode, meeting passwords, and JWT tokens. |
| [**04. Customization & Features**](./04-customization-and-branding.md) | UI, Green Room & Whiteboard | Native Next.js UI, Green Room camera lobby, Whiteboard canvas, and toolbar controls. |
| [**05. Network & Port Allocation**](./05-network-and-ports.md) | Networking & Firewalls | Port mapping table (3000, 8443, 10000/udp, 4443), NAT traversal, and SFU traffic. |
| [**06. Operations & Troubleshooting**](./06-operations-and-troubleshooting.md) | DevOps & Maintenance | Docker Compose management cheat sheet, log inspection, and troubleshooting fixes. |
| [**07. Configuration & Reference**](./07-config-and-scripts-reference.md) | Config & Architecture Guide | Detailed breakdown of environment variables, Next.js JWT signing, and directory layout. |

---

## 🚀 Standard Docker Commands

```bash
# Start all services
docker compose up -d

# Check status of all containers
docker compose ps

# View live real-time logs
docker compose logs -f

# Restart all services
docker compose restart

# Stop all services
docker compose down
```
