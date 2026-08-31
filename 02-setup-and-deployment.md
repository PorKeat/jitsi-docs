# 02. Setup & Deployment Guide

This guide details how to install, configure, and launch Unity Meet locally or on a production cloud server.

---

## 📋 Prerequisites

* **Docker Engine** (v24.0+) & **Docker Compose** (v2.0+)
* **Node.js** (v18+) & **npm** (for building the Next.js portal)
* **OpenSSL** (for SSL certificate generation)

---

## 🛠️ Step 1: Clone & Configure `.env`

Copy the environment template and verify your secrets:

```bash
cd /Users/alexkgm/Desktop/Jitsi
cp env.example .env
```

### Core Environment Settings (`.env`):
```ini
# Domain & Protocol
HTTP_PORT=8080
HTTPS_PORT=8443
PUBLIC_URL=https://localhost:8443
DOCKER_HOST_ADDRESS=127.0.0.1

# Security & Token Authentication
ENABLE_AUTH=1
ENABLE_GUESTS=0
AUTH_TYPE=jwt
JWT_APP_ID=my_jitsi_app
JWT_APP_SECRET=your_super_secret_jwt_key_here
JWT_ACCEPTED_ISSUERS=my_jitsi_app
JWT_ACCEPTED_AUDIENCES=my_jitsi_app

# Performance & UI Lockdowns
ENABLE_WELCOME_PAGE=0
ENABLE_RECORDING=0
ENABLE_LIVESTREAMING=0
```

---

## 🔐 Step 2: SSL Certificate Generation (TLS 1.3 SAN)

Modern Chrome and Safari require certificates with explicit `Subject Alternative Names` (SAN) and `keyUsage = critical, digitalSignature, keyEncipherment` to prevent `ERR_SSL_KEY_USAGE_INCOMPATIBLE`.

Run the automated certificate generator:
```bash
./scripts/setup.sh
```

Or generate manually:
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

---

## 📦 Step 3: Build Next.js Web Portal

```bash
cd web-app
npm install
npm run build
cd ..
```

---

## 🚀 Step 4: Launch Containers

Use the included management script:

```bash
./manage.sh start
```

Or using Docker Compose:
```bash
docker compose up -d
```

---

## 🧪 Step 5: Verification & Browser Access

1. Open your browser and navigate to:
   👉 **`http://localhost:3000`**
2. On your first HTTPS connection to Jitsi (`https://localhost:8443`), accept the self-signed SSL certificate in Chrome by clicking **Advanced ➔ Proceed to localhost (unsafe)** or typing `thisisunsafe` on the page.
3. Click **"Start Instant Meeting"** from the portal — you will enter the secure, encrypted room instantly!
