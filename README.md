# Unity Meet — Production Documentation

> Modern, ultra-secure, enterprise-grade WebRTC video conferencing built with **Next.js 16 (Turbopack)**, **Go (Golang) Microservice Backend**, **Valkey 8 In-Memory Datastore**, **PostgreSQL 15**, and **Jitsi WebRTC SFU** orchestrated via **Kubernetes Helm (GitOps)** on **Traefik v3**.

---

## 🌟 Core Features

* 🎛️ **Modern Workspace Dashboard:**
  * Clean, minimalist UI with Light and Dark mode theme switching.
  * Real-time client RTT ping latency, server uptime, and active meetings status.
  * Quick Access & Recent Rooms with **In Progress** (1-click Rejoin) and **Ended** badges.
  * Live Mini Calendar of the current month with Keycloak sync preview.
* 🎥 **Pre-Join "Green Room" Lobby:** Live camera test, Web Audio volume equalizer visualizer, device toggles, robot avatar selector (DiceBear bottts), and custom background studio.
* 🖥️ **Smooth Screen Sharing & Stage Controls:** GPU-accelerated smooth zoom (`transform: scale()`), 1-click Fit / Fill aspect ratio toggle, floating stage pill controls, and zero glitching.
* 🔒 **Meeting Room Lock & Knocking Lobby:** Host admission control, meeting PINs, and anti-hijack token gates.
* ✏️ **Built-in Collaborative Whiteboard:** Interactive Next.js drawing canvas with 60 FPS neon laser pointer, shapes, text, and 1-click Color Studio.
* 🛡️ **Cryptographic Token Auth & AES-256-GCM:** HMAC-SHA256 tokens and AEAD ciphertext invite links issued by the Go backend microservice.
* ⚡ **Ultra-Low Latency SFU:** 100% in-browser WebRTC via JVB over DTLS-SRTP (Port 10000 UDP) scaled across active cluster nodes.
* 🚀 **Enterprise Kubernetes Architecture:** GitOps Helm deployment with Traefik v3 IngressRoute, MetalLB VIP (`10.1.18.200`), Longhorn distributed storage, and automated lifecycle bug patches.

---

## 📚 Documentation Index

| File | Topic | Description |
| :--- | :--- | :--- |
| [**01. Architecture & Protocols**](./01-architecture-and-protocols.md) | System Design & Protocols | Kubernetes cluster topology, Go API microservice, Valkey datastore, WebRTC, XMPP, DTLS-SRTP, Colibri, and SFU flows. |
| [**02. Setup & Deployment**](./02-setup-and-deployment.md) | Installation & Startup | Production Kubernetes Helm deployment (GitOps), MetalLB VIP, Docker Compose local dev, and TLS SAN SSL certs. |
| [**03. Security, Passwords & Lobby**](./03-security-and-jwt.md) | Security & Access Control | AES-256-GCM links, Host Secret validation, Knocking Lobby mode, meeting passwords, and JWT tokens. |
| [**04. Customization & Features**](./04-customization-and-branding.md) | UI, Green Room & Stage Controls | Native Next.js 16 UI, Green Room camera lobby, smooth screen sharing fit/fill zoom, Whiteboard, and toolbar. |
| [**05. Network & Port Allocation**](./05-network-and-ports.md) | Networking & Firewalls | Kubernetes cluster networking, MetalLB VIP (10.1.18.200), Traefik v3 IngressRoute, JVB hostPort 10000/UDP, and port mapping. |
| [**06. Operations & Troubleshooting**](./06-operations-and-troubleshooting.md) | DevOps & Maintenance | Kubernetes cluster management, Helm upgrade runbooks, JVB node sizing, stuck pod cleanup, Longhorn recovery, and Docker cheat sheets. |
| [**07. Configuration & Reference**](./07-config-and-scripts-reference.md) | Config & Architecture Guide | Detailed breakdown of Helm values (`values-prod.yaml`), environment variables, Go JWT signing, and directory layout. |
| [**08. Next.js Integration & Customization**](./08-nextjs-integration-and-customization.md) | Frontend Dev & SDK Guide | Next.js 16 App Router, Turbopack, dynamic SDK imports, IFrame API commands/events, custom toolbar, Green Room, and Whiteboard. |
| [**09. Valkey Cache & Distributed State**](./09-valkey-cache-and-distributed-state.md) | High-Performance In-Memory Cache | Distributed room state, knocking lobby sync, rate limiting, and Unity Calendar holiday & event query acceleration. |
| [**10. Real-Time Sync & SSE Architecture**](./10-realtime-sync-and-sse-architecture.md) | Push Synchronization & Zero-Cache | Valkey Pub/Sub channel, Server-Sent Events (SSE) streaming, zero-cache invalidation, and real-time meeting sync. |

---

## 🚀 Quick Deployment Reference

### Production Kubernetes (Helm GitOps)
```bash
# SSH into cluster master node (jitsi-meet2 / 10.1.18.10)
ssh -i ~/.ssh/unity-workspace-key root@10.1.18.10

# Upgrade Helm release with production values
cd /root/unity-meet-helm
helm upgrade unity-meet . -n jitsi -f values-prod.yaml

# Check cluster pod health
kubectl get pods -n jitsi -o wide
```

### Local Development (Docker Compose)
```bash
# Start all services locally
docker compose up -d

# Check status of local containers
docker compose ps

# View live real-time logs
docker compose logs -f
```
