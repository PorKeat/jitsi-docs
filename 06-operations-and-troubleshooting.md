# 06. Operations, CLI & Troubleshooting

Unity Meet is managed via standard Docker Compose commands.

---

## 🚀 Standard Service Operations

| Action | Command | Description |
| :--- | :--- | :--- |
| **Start All** | `docker compose up -d` | Starts all microservices and Next.js portal in the background. |
| **Stop All** | `docker compose down` | Safely stops and shuts down all containers. |
| **Restart All** | `docker compose restart` | Reboots all active services. |
| **View Live Logs** | `docker compose logs -f` | Streams real-time container logs. |
| **Container Status**| `docker compose ps` | Displays container health, uptime, and port mappings. |

---

## 🔍 Log Inspection by Service

```bash
# View Next.js application logs
docker compose logs -f portal

# View Nginx / Web frontend logs
docker compose logs -f web

# View XMPP Prosody signaling logs
docker compose logs -f prosody

# View Jicofo focus daemon logs
docker compose logs -f jicofo

# View JVB (Videobridge) media stream logs
docker compose logs -f jvb
```

---

## 🛠️ Common Troubleshooting

### 1. Port 3000, 8443, or 8080 Conflict
If a port is already bound by another process:
```bash
lsof -i :3000
lsof -i :8443
# Kill conflicting PID
kill -9 <PID>
```

### 2. Force Container Rebuild
To refresh container volumes and settings:
```bash
docker compose up -d --force-recreate
```

### 3. Rebuild Next.js App
```bash
cd web-app
npm run build
cd ..
docker compose restart portal
```
