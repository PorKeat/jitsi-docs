# 05. Network, Port Allocation & Multi-Zone Security Guide

This document details the complete networking topology, port allocations, and communication paths for Unity Meet across three specific security zones:
1. **🌐 Category 1: External / Public Facing** (Internet ➔ Edge Router ➔ Cluster)
2. **🔄 Category 2: Inter-VM & Inter-Service** (Private Subnet / Cross-VM Communication)
3. **🔒 Category 3: Strictly Internal** (Pod-to-Pod / ClusterIP Network Only)

---

## 🏛️ Comprehensive Architecture & Traffic Flow

```
                      ┌────────────────────────────────────────────────────────┐
                      │                 ZONE 1: PUBLIC INTERNET                │
                      └───────────────────────────┬────────────────────────────┘
                                                  │
                  ┌───────────────────────────────┴───────────────────────────────┐
                  │ 443/TCP (HTTPS/WSS) & 80/TCP (HTTP)                           │ 10000/UDP (Direct WebRTC Media)
                  ▼                                                               ▼
┌──────────────────────────────────────────────┐              ┌──────────────────────────────────────────────┐
│ MetalLB Virtual IP (VIP): 10.1.18.200        │              │ Physical Node HostPorts (JVB SFU Media)      │
│ └── Traefik v3 Ingress Controller            │              │  ├── Node 2 (jitsi-meet2): 10.1.18.10:10000  │
└──────────────────────┬───────────────────────┘              │  └── Node 3 (jitsi-meet3): 10.1.18.11:10000  │
                       │                                      └──────────────────────┬───────────────────────┘
                       │                                                             │
═══════════════════════╪═════════════════════════════════════════════════════════════╪═════════════════════════
                       │ ZONE 2: INTER-VM & CROSS-SERVICE LAN (10.1.18.0/24)         │
                       │  • Kubernetes API: 6443/TCP (Control Plane)                 │
                       │  • etcd Quorum: 2379-2380/TCP (Node 2 ◄► Node 3)            │
                       │  • Longhorn CSI Storage Engine: 9500-9504/TCP               │
                       │  • Keycloak SSO Server: 8080/443 (10.1.18.8)                │
                       │  • Calendar Service API: 5434/TCP (Database / Microservice) │
                       │  • SSH Management: 22/TCP (Admin VPN Only)                  │
═══════════════════════╪═════════════════════════════════════════════════════════════╪═════════════════════════
                       │
                       ▼ ZONE 3: STRICTLY INTERNAL POD OVERLAY (ClusterIP)
       ┌───────────────────────────────┬───────────────────────────────┐
       │                               │                               │
       ▼                               ▼                               ▼
┌──────────────┐               ┌──────────────┐               ┌───────────────────┐
│unity-meet-web│               │unity-meet-api│               │jitsi-meet-web     │
│  Port 3000   │               │  Port 8000   │               │  Port 80          │
└──────┬───────┘               └──────┬───────┘               └─────────┬─────────┘
       │                              │                                 │
       │     ┌────────────────────────┼────────────────────────┐        │
       │     ▼                        ▼                        ▼        ▼
┌──────┴───────────────┐       ┌──────────────┐       ┌───────────────────────────┐
│  unity-meet-valkey   │       │unity-meet-db │       │ jitsi-meet-prosody (XMPP) │
│  Port 6379 (Cache)   │       │Port 5432(PG) │       │ Port 5222(C2S) & 5347(MUC)│
└──────────────────────┘       └──────────────┘       └───────────────────────────┘
```

---

## 🌐 Category 1: External / Public Facing (Edge Router ➔ Cluster)

These ports **MUST be allowed through your perimeter firewall, edge router, or NAT port-forwarding** so users from the internet can join meetings:

| Inbound Port | Protocol | Target Destination | Scope | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`443`** | `TCP` | MetalLB VIP (`10.1.18.200:443`) | **Mandatory** | Public HTTPS entrypoint for Next.js Web UI, Go REST API, and WebSocket tunneling (`/xmpp-websocket`). |
| **`80`** | `TCP` | MetalLB VIP (`10.1.18.200:80`) | **Mandatory** | Automatic HTTP-to-HTTPS redirect (301) and Let's Encrypt ACME HTTP-01 SSL challenge validation. |
| **`10000`** | `UDP` | Cluster Worker Nodes (`10.1.18.10`, `10.1.18.11`) | **Critical** | Direct WebRTC audio/video media streams (DTLS-SRTP). Binds directly to the physical host network (`useHostNetwork: true`). |
| **`4443`** | `TCP` | Cluster Worker Nodes (`10.1.18.10`, `10.1.18.11`) | Optional | TCP media fallback for corporate environments that block outbound UDP 10000. |

---

## 🔄 Category 2: Inter-VM & Cross-Service Communication (Private LAN)

These ports operate within the private LAN/VLAN (`10.1.18.0/24`). They **must NOT be exposed to the public internet**, but **MUST be allowed between cluster nodes and related workspace VMs**:

| Port / Range | Protocol | Source | Destination | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`6443`** | `TCP` | Worker Nodes / Admin | `jitsi-meet2` (`10.1.18.10`) | Kubernetes API Server for kubelet scheduling and cluster management. |
| **`2379` – `2380`**| `TCP` | Control Plane Nodes | Control Plane Nodes (`.10`, `.11`) | etcd Raft consensus database synchronization between cluster masters. |
| **`8472`** | `UDP` | Cluster Nodes | Cluster Nodes (`.10`, `.11`) | CNI Pod Network Overlay (Cilium / VXLAN cross-node encapsulation). |
| **`9500` – `9504`**| `TCP` | Cluster Nodes | Cluster Nodes (`.10`, `.11`) | Longhorn Distributed CSI Engine (volume replication & instance managers for persistent disks). |
| **`8080` / `443`** | `TCP` | Meet Web / API Pods | Keycloak VM (`auth.unity-workspace.com` / `10.1.18.8`) | OpenID Connect / OAuth2 token validation, user authentication, and profile sync. |
| **`8200`** | `TCP` | Vault Agent Injector / Sidecars | Vault Host (`10.1.18.8:8200`) | HTTPS / mTLS API for token login and dynamic secret retrieval. |
| **`8443`** | `TCP` | Vault Host (`10.1.18.8`) | `jitsi-meet2` (`10.1.18.10:8443`) | K8s API Server TokenReview proxy for Vault to validate ServiceAccount JWT tokens. |
| **`5434`** | `TCP` | Meet Web BFF | Calendar Service | Calendar event query acceleration and meeting synchronization. |
| **`22`** | `TCP` | Admin VPN / Bastion | All Nodes (`10.1.18.10`, `10.1.18.11`)| SSH administrative access, GitOps deployments, and node maintenance. |

---

## 🔒 Category 3: Strictly Internal (Pod-to-Pod / ClusterIP Only)

These ports operate strictly inside the Kubernetes virtual ClusterIP network. They **must NEVER be exposed to the outside internet or published on VM host ports**:

| Port | Protocol | Service / Component | Service DNS Name | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`8000`** | `TCP` | Go API Microservice | `unity-meet-api.jitsi.svc.cluster.local` | Core backend: HS256 JWT tokens, AES-256-GCM link encryption, room locks, and bans. |
| **`3000`** | `TCP` | Next.js 16 Web UI | `unity-meet-web.jitsi.svc.cluster.local` | Next.js App Router, SSR pages, and BFF API route handlers. |
| **`6379`** | `TCP` | Valkey In-Memory Cache| `unity-meet-valkey.jitsi.svc.cluster.local` | Sub-millisecond distributed cache for active rooms, knocking lobby, and rate limiting. |
| **`5432`** | `TCP` | PostgreSQL Database | `unity-meet-postgres.jitsi.svc.cluster.local` | Relational storage for user accounts, persistent room records, and audit history. |
| **`5222`** | `TCP` | Prosody XMPP Client | `unity-meet-jitsi-meet-prosody.jitsi.svc.cluster.local` | Internal client-to-server XMPP signaling and presence broadcast. |
| **`5347`** | `TCP` | Prosody Component | `unity-meet-jitsi-meet-prosody.jitsi.svc.cluster.local` | Internal XMPP component protocol connection between Jicofo and Prosody. |
| **`80`** | `TCP` | Jitsi Web Gateway | `unity-meet-jitsi-meet-web.jitsi.svc.cluster.local` | Internal Nginx serving WebRTC frontend assets and BOSH proxy. |
| **`3002`** | `TCP` | Excalidraw Whiteboard | `unity-meet-whiteboard.jitsi.svc.cluster.local` | Internal WebSocket relay for collaborative drawing. |

---

## 📋 Firewall & Routing Rule Summary

1. **At your Edge Gateway / Public Router:**
   * Forward **`80/TCP`** and **`443/TCP`** to MetalLB VIP **`10.1.18.200`**.
   * Forward **`10000/UDP`** directly to the physical cluster nodes (**`10.1.18.10`** and **`10.1.18.11`**).
2. **Between Cluster VMs / Subnet (`10.1.18.0/24`):**
   * Allow full inter-node traffic for Kubernetes CNI (`8472/UDP`), etcd (`2379-2380/TCP`), and Longhorn storage (`9500-9504/TCP`).
   * Allow outbound HTTP/HTTPS to Keycloak (`10.1.18.8`) and Calendar service.
3. **Inside Kubernetes (Pods):**
   * Handled automatically by Kubernetes CNI and ClusterIP services; no manual firewall rules required.


