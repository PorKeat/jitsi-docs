# 02. Setup & Deployment Guide

This guide details how to deploy Unity Meet to **Production Kubernetes (Helm GitOps)** or run it locally using **Docker Compose**.

---

## 🏛️ Part 1: Production Kubernetes Deployment (Helm & GitOps)

The production deployment at **`meet.unity-workspace.com`** runs on a bare-metal Kubernetes cluster orchestrated via Helm with Traefik v3, MetalLB VIP, Longhorn CSI, and automated lifecycle patches.

### 1. Cluster Topology & Access
* **Control Plane Node:** `jitsi-meet2` (`10.1.18.10`)
* **Worker / Control Plane Node:** `jitsi-meet3` (`10.1.18.11`)
* **MetalLB Virtual IP (VIP):** `10.1.18.200` (Traefik v3 Ingress Entrypoint)
* **SSH Access:**
  ```bash
  ssh -i ~/.ssh/unity-workspace-key root@10.1.18.10
  ```

### 2. Building & Pushing Container Images (`linux/amd64`)
Since cluster nodes run Linux x86_64, images must be compiled for `linux/amd64` using `docker buildx`:

```bash
cd /Users/alexkgm/Desktop/Jitsi

# Build Go API Microservice
docker buildx build --platform linux/amd64 -t alexkgm/unity-meet-api:latest --push ./api

# Build Next.js 16 Web Portal
docker buildx build --platform linux/amd64 -t alexkgm/unity-meet-web:latest --push ./web-app
```

### 3. Synchronizing & Deploying Helm Chart
The Helm chart is tracked in `unity-meet-helm.git`. Deployments are executed from the control plane node:

```bash
# 1. Commit and push Helm configuration locally
cd /Users/alexkgm/Desktop/jitsi-helm
git add .
git commit -m "chore: update helm configuration"
git push origin main

# 2. Rsync chart to master node
rsync -avz --exclude '.git' -e "ssh -i ~/.ssh/unity-workspace-key" /Users/alexkgm/Desktop/jitsi-helm/ root@10.1.18.10:/root/unity-meet-helm/

# 3. Apply Helm upgrade on master node (GitOps Safe)
ssh -i ~/.ssh/unity-workspace-key root@10.1.18.10 "cd /root/unity-meet-helm && helm upgrade unity-meet . -n jitsi -f values-prod.yaml"

# 4. Trigger rolling restart of application pods
ssh -i ~/.ssh/unity-workspace-key root@10.1.18.10 "kubectl rollout restart deployment/unity-meet-api deployment/unity-meet-web -n jitsi"
```

### 4. GitOps Guarantees Built into the Chart
* **Traefik v3 IngressRoute:** Explicitly configured via `templates/traefik-ingressroute.yaml` with host rules, path prefixes, and TLS certificates.
* **JVB Replica Sizing:** Fixed at `replicaCount: 2` to match the two active physical nodes, guaranteeing zero `Pending` pods due to host port 10000 binding.
* **ConnectionQuality Fix Hook:** Injected via `jitsi-meet.web.lifecycle.postStart` to automatically sanitize `lib-jitsi-meet.min.js` on every pod startup:
  ```bash
  perl -pi -e "s/\"stats\"===t\.type/t&&\"stats\"===t\.type/g" /usr/share/jitsi-meet/libs/lib-jitsi-meet.min.js
  ```
* **Vault Agent Sidecar Injection:** Automatically injects `vault-agent-init` and `vault-agent` sidecars into API pods when `vault.enabled: true`. Secrets are rendered exclusively in-memory (`tmpfs`) at `/vault/secrets/credentials.env` without touching etcd or disk.
* **Custom Assets Mount:** Next-gen watermark, `interface_config.js`, and `head.html` mounted cleanly from ConfigMaps.

---

## 💻 Part 2: Local Development Setup (Docker Compose)

For rapid offline UI development and API testing:

### 1. Prerequisites
* **Docker Desktop** (v24.0+) & **Docker Compose** (v2.0+)
* **Node.js** (v18+) & **pnpm** or **npm**
* **OpenSSL** (for TLS 1.3 self-signed SAN certs)

### 2. Configure Local `.env`
```bash
cd /Users/alexkgm/Desktop/Jitsi
cp env.example .env
```

### 3. Generate Local TLS 1.3 SAN Certificate
```bash
mkdir -p config/web/keys
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout config/web/keys/key.pem \
  -out config/web/keys/cert.pem \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment" \
  -addext "extendedKeyUsage=serverAuth"
```

### 4. Run Local Services
```bash
# Start backend stack (WebRTC, Prosody, Jicofo, JVB, API, Valkey)
docker compose up -d

# Start Next.js local dev server with Turbopack
cd web-app
pnpm dev
```
Open **`http://localhost:3000`** in Chrome/Brave to test. Accept the certificate on `https://localhost:8443`.
