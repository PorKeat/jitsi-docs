# 07. Configuration & Architecture Reference

This guide details the production Helm configuration parameters (`values-prod.yaml`), environment variables, and directory layout for Unity Meet.

---

## 🏛️ Production Helm Configuration (`values-prod.yaml`)

| Section | Parameter | Default Value | Description |
| :--- | :--- | :--- | :--- |
| **Global** | `global.domain` | `meet.unity-workspace.com` | Primary production domain |
| | `global.jwtAppId` | `unity_meet_enterprise` | Jitsi Prosody JWT application ID |
| | `global.jwtSecret` | *(Secret Token)* | HMAC-SHA256 signing secret |
| **API** | `api.replicaCount` | `3` (Autoscales 3-15) | Go API microservice pods |
| | `api.image.repository`| `alexkgm/unity-meet-api` | Prebuilt `linux/amd64` Go binary |
| **Web UI** | `web.replicaCount` | `3` (Autoscales 3-15) | Next.js 16 Web Portal UI pods |
| | `web.image.repository`| `alexkgm/unity-meet-web` | Next.js standalone container |
| **Valkey** | `valkey.persistence.storageClass` | `longhorn` | High-availability replicated storage |
| | `valkey.service.port` | `6379` | Distributed room & rate limit store |
| **Ingress** | `traefikIngressRoute.enabled` | `true` | Traefik v3 CRD with MetalLB VIP (`10.1.18.200`) |
| | `traefikIngressRoute.tlsSecretName` | `unity-meet-prod-tls` | Let's Encrypt TLS secret |
| **Jitsi** | `jitsi-meet.jvb.replicaCount` | `2` | 1 JVB per physical node (`hostPort: 10000/udp`) |
| | `jitsi-meet.jvb.useHostNetwork` | `true` | Direct host network binding for WebRTC media |
| | `jitsi-meet.web.lifecycle.postStart` | Script | Patches ConnectionQuality stats TypeError in `lib-jitsi-meet` |
| **Vault** | `vault.enabled` | `true` | Injects HashiCorp Vault Agent sidecar (`/vault/secrets`) |
| | `vault.role` | `unity-meet-api` | Vault Kubernetes auth role name |
| | `vault.tlsCaSecret` | `vault-ca-cert` | Kubernetes Secret with Vault internal root CA |
| | `vault.secretPath` | `secret/data/unity-workspace/meet/api` | KV-v2 path for dynamic in-memory credential delivery |

---

## ⚙️ Local Development Environment Variables (`.env`)

| Variable | Description | Value in Unity Meet |
| :--- | :--- | :--- |
| `PUBLIC_URL` | Base public URL for the Jitsi web service | `https://localhost:8443` |
| `HTTP_PORT` | Nginx HTTP port | `8080` |
| `HTTPS_PORT` | Nginx HTTPS port | `8443` |
| `PORTAL_PORT` | Next.js Frontend port | `3000` |
| `API_PORT` | Go API Microservice port | `8000` |
| `ENABLE_AUTH` | Require token authentication | `1` |
| `ENABLE_GUESTS` | Allow unauthenticated guests | `0` (Disabled - guests use signed JWTs) |
| `AUTH_TYPE` | Authentication mechanism | `jwt` |
| `JWT_APP_ID` | Application identifier for JWT | `unity_meet_enterprise` |
| `JVB_PORT` | Media UDP port for WebRTC audio/video | `10000` |

---

## 🗂️ Project Repositories & Directory Structure

```text
Desktop/
├── jitsi-helm/               # Production Kubernetes Helm Chart (GitOps)
│   ├── Chart.yaml            # Umbrella chart & Jitsi subchart
│   ├── values-prod.yaml      # Production configuration (Traefik, JVB=2, postStart hook)
│   ├── templates/            # Go API, Web, Valkey, Postgres, IngressRoute
│   └── scripts/              # deploy-prod.sh, validate.sh
│
├── Jitsi/                    # Source Code & Container Builds
│   ├── api/                  # Go Microservice (JWT, Valkey cache, Postgres, Room locks)
│   │   ├── handlers/         # Token, room check, and security endpoints
│   │   ├── services/         # Valkey cache & DB connectors
│   │   └── Dockerfile        # Multi-stage Go build
│   ├── web-app/              # Next.js 16 (Turbopack) Full-Stack Meeting Application
│   │   ├── app/              # App Router (dashboard, green room, meeting room)
│   │   └── components/       # Meeting UI, Stage Controls, Whiteboard, Modals
│   └── docker-compose.yml    # Local multi-container development environment
│
└── jitsi-docs/               # Technical Documentation & Runbooks
    ├── 01-10-*.md            # Architecture, Security, Networking, Valkey, Real-Time
    └── README.md             # Documentation index and quickstart
```
