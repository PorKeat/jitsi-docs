# 05. Network, Port Allocation & Firewall Rules

This document outlines the network topology, port allocations, and firewall requirements for Unity Meet in **Production Kubernetes** and local environments.

---

## 🏛️ Production Kubernetes Network Topology

In the production cluster, traffic is routed through **MetalLB** to **Traefik v3**, with high-throughput WebRTC media routed directly to physical nodes via `hostPort`.

```
Clients (Web Browsers)
   │
   ├─► HTTPS (443/TCP) ──► MetalLB VIP: 10.1.18.200 ──► Traefik v3 IngressRoute
   │                        │
   │                        ├── /api/calendar, /api/room-schedule ──► unity-meet-web (Port 3000)
   │                        ├── /api/*, /health ──► unity-meet-api (Port 8000)
   │                        ├── /_next, /meeting, /join, / ──► unity-meet-web (Port 3000)
   │                        └── / (Jitsi WebRTC Assets & WSS) ──► jitsi-meet-web (Port 80)
   │
   └─► WebRTC Media (10000/UDP) ──► Direct Physical Node Binding (hostPort)
                            ├── Node 2 (jitsi-meet2): 10.1.18.10:10000 UDP
                            └── Node 3 (jitsi-meet3): 10.1.18.11:10000 UDP
```

### Key Kubernetes Network Details
1. **MetalLB Virtual IP (`10.1.18.200`):** Fronted by Traefik v3. Manages TLS termination, WebSocket upgrades (`/xmpp-websocket`), and HTTP-to-HTTPS redirection.
2. **JVB HostPort UDP 10000:** JVB uses `useHostNetwork: true` to eliminate container NAT overhead. Each physical node can host at most **one** JVB instance on UDP port 10000. With 2 active worker nodes (`jitsi-meet2` & `jitsi-meet3`), `replicaCount: 2` is the cluster capacity limit.
3. **Internal ClusterIP Network:**
   * `unity-meet-api`: Port `8000` (Go microservice)
   * `unity-meet-valkey`: Port `6379` (In-memory datastore)
   * `unity-meet-postgres`: Port `5432` (PostgreSQL)
   * `unity-meet-jitsi-meet-prosody`: Port `5222` (C2S XMPP) and `5347` (Component)
   * `unity-meet-jitsi-meet-web`: Port `80` (Static assets & Nginx internal proxy)

---

## 🌐 Port Allocation Table

| Port | Protocol | Scope | Service / Pod | Public Exposure | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`443`** | `TCP` | Production VIP | Traefik v3 Ingress | **Public (`10.1.18.200:443`)** | HTTPS entrypoint for Web UI, API, and WebSockets |
| **`80`** | `TCP` | Production VIP | Traefik v3 Ingress | **Public (`10.1.18.200:80`)** | HTTP entrypoint (redirects to HTTPS) |
| **`10000`**| `UDP` | Physical Node | JVB Media SFU | **Public (Critical)** | Direct DTLS-SRTP audio & video stream transport |
| **`3000`** | `TCP` | Internal / Local| Next.js Web App | Internal (ClusterIP) | React 19 / Next.js 16 Web Dashboard & Meeting Frame |
| **`8000`** | `TCP` | Internal / Local| Go API Microservice| Internal (ClusterIP) | JWT token generation, room locks, and AES decryption |
| **`6379`** | `TCP` | Internal | Valkey In-Memory | Internal Only | Sub-millisecond room state, rate limits, and bans |
| **`5432`** | `TCP` | Internal | PostgreSQL DB | Internal Only | Persistent user profiles and meeting history |
| **`5222`** | `TCP` | Internal | Prosody XMPP | Internal Only | Internal client-to-server XMPP signaling |
| **`5347`** | `TCP` | Internal | Prosody Component | Internal Only | Internal XMPP component connection for Jicofo |

## 🛡️ External Inbound Firewall & Network Exposure Requirements

To allow users outside the local network or across the internet to join meetings, the following ports must be allowed through your edge router, NAT port forwarding, cloud security groups, or perimeter firewall:

| Inbound Port | Protocol | Target Destination | Requirement | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`443`** | `TCP` | Ingress VIP (`10.1.18.200:443`) | **MANDATORY** | Public HTTPS web access, REST API microservice, and secure WebSockets (`/xmpp-websocket`) |
| **`80`** | `TCP` | Ingress VIP (`10.1.18.200:80`) | **MANDATORY** | Automatic HTTP-to-HTTPS redirect and Let's Encrypt ACME SSL certificate renewals |
| **`10000`** | `UDP` | Cluster Worker Nodes (`10.1.18.10`, `10.1.18.11`) | **CRITICAL** | WebRTC audio & video media streams (DTLS-SRTP). Must route directly to physical host ports |
| **`4443`** | `TCP` | Cluster Worker Nodes (`10.1.18.10`, `10.1.18.11`) | Optional | TCP media fallback for attendees behind strict firewalls that block UDP |

> [!IMPORTANT]
> **Internal Service Security (Do NOT Expose Outside):**
> All internal ports (`3000` Next.js Web UI, `8000` Go API, `6379` Valkey, `5432` PostgreSQL, `5222` Prosody XMPP) operate strictly on the internal Kubernetes ClusterIP network and **must NOT** be opened or forwarded to the public internet. External clients only reach them through Traefik Ingress on port 443/TCP.

