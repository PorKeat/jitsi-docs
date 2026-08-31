# 05. Network, Port Allocation & Firewall Rules

This document outlines the network ports and firewall requirements for Unity Meet.

---

## 🌐 Port Allocation Table

| Port | Protocol | Service | Container | Public Exposure | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`3000`** | `TCP` | Next.js Web App | `portal` | **Public** | Unity Meet landing page, room generator, and meeting UI |
| **`8443`** | `TCP` | Nginx HTTPS / Web | `web` | **Public** | Jitsi web static assets, BOSH (`/http-bind`), and WebSocket (`/xmpp-websocket`) |
| **`8080`** | `TCP` | Nginx HTTP | `web` | Optional | HTTP redirect to HTTPS |
| **`10000`**| `UDP` | JVB WebRTC Media | `jvb` | **Public (Critical)** | Encrypted DTLS-SRTP audio & video stream transport |
| **`3002`** | `TCP` | Excalidraw Backend | `whiteboard` | **Public / Local** | WebSocket collaborative drawing relay |
| **`4443`** | `TCP` | JVB TCP Fallback | `jvb` | Optional | TCP fallback for restrictive firewalls blocking UDP 10000 |
| **`5222`** | `TCP` | Prosody Client C2S | `prosody` | Internal Only | Internal client-to-server XMPP communication |
| **`5347`** | `TCP` | Prosody Component | `prosody` | Internal Only | Internal XMPP component connection for Jicofo |

---

## 🛡️ Firewall Configuration Rules (UFW / Cloud Security Groups)

```bash
# Allow Next.js Portal
sudo ufw allow 3000/tcp

# Allow Jitsi Web Interface
sudo ufw allow 8443/tcp
sudo ufw allow 8080/tcp

# Allow Excalidraw Whiteboard Backend
sudo ufw allow 3002/tcp

# Allow WebRTC Media UDP Traffic (MANDATORY)
sudo ufw allow 10000/udp

# Allow WebRTC TCP Fallback
sudo ufw allow 4443/tcp
```
