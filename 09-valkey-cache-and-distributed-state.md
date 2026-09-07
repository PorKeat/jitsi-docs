# 09. Valkey High-Performance In-Memory Cache & Distributed State Guide

This guide details the **Valkey 8.x** in-memory datastore architecture deployed across the Unity Workspace cluster (Unity Meet and Unity Calendar) for distributed caching, sub-millisecond room lookups, cluster-wide bans, and atomic rate limiting.

---

## ⚡ 1. Why Valkey?

Following Redis Inc.'s licensing change away from BSD open source, the **Linux Foundation** founded **Valkey** with broad industry backing from AWS, Google Cloud, Oracle, and Percona.

* **100% Drop-in Wire Compatibility:** Implements the Redis RESP2 / RESP3 protocol natively (`go-redis/v9` connects without modifications).
* **True Open Source:** Released under the permissive BSD-3-Clause license.
* **Extreme Multi-Threaded Throughput:** Delivers sub-millisecond lookup latency across multi-replica microservices.
* **Graceful Fallback:** If Valkey is temporarily offline during maintenance, all API pods automatically fall back to direct PostgreSQL / in-memory storage without service interruption.

---

## 🏛️ 2. Cluster Topology & Kubernetes Architecture

Valkey runs as a persistent service within the `jitsi` Kubernetes namespace and is shared across all workspace microservices over the Cilium overlay network.

```
                                  [ User Requests ]
                                          │
                        ┌─────────────────┴─────────────────┐
                        ▼                                   ▼
             [ Unity Meet Go API ]               [ Unity Calendar Go API ]
                (Replicas: 3)                       (Replicas: 3)
                        │                                   │
                        └─────────────────┬─────────────────┘
                                          │
                        ┌─────────────────┴─────────────────┐
                        ▼                                   ▼
          [ Valkey Service: valkey:6379 ]         [ PostgreSQL: 5432 ]
             ClusterIP: 10.233.61.3:6379             ClusterIP: 5432
                        │
                        ▼
             [ Longhorn CSI Storage ]
               (5Gi AOF Persistence)
```

---

## 🔑 3. Datastore Key Schemas

### A. Unity Meet Keys
| Key Pattern | Type | TTL | Purpose |
| :--- | :--- | :---: | :--- |
| `meet:room:<room_id>` | `String (JSON)` | 48 Hours | Cached room metadata, hashed passcode, host secret, and expiry timestamp. Eliminates database queries on every heartbeat. |
| `meet:bans:<room_id>` | `Set` | Session | Instant cluster-wide ban enforcement. Banned participant fingerprints are blocked across all API pods within <1ms. |
| `meet:lock:<key>` | `String` | 5–10 Sec | Distributed atomic concurrency locks (`SETNX`). Prevents race conditions during room initialization. |
| `meet:ratelimit:<action>:<ip>` | `String (Int)` | 1 Minute | Sliding-window atomic rate limiting for `/api/create-room` (10 req/min) and `/api/token` (30 req/min). |

### B. Unity Calendar Keys
| Key Pattern | Type | TTL | Purpose |
| :--- | :--- | :--- | :--- |
| `cal:holidays:<year>` | `String (JSON)` | 24 Hours | Complete Cambodian public holiday calendar for the year. |
| `cal:date:<YYYY-MM-DD>` | `String (JSON)` | 24 Hours | Full Khmer lunisolar Chhankitek details (Buddhist era, lunar phase, animal year, holy day status). |
| `cal:events:list:*` | `String (JSON)` | 10 Minutes | Cached user events. Automatically invalidated via prefix deletion on any event create, update, reschedule, RSVP, or delete. |
| `cal:calendars:list` | `String (JSON)` | 10 Minutes | Cached calendar categories. Automatically invalidated on calendar mutations. |

---

## ⚙️ 4. Helm & Kubernetes Configuration

In `values-prod.yaml` (`unity-meet-helm` and `unity-calendar-helm`):

```yaml
valkey:
  enabled: true
  image:
    repository: valkey/valkey
    tag: "8.0-alpine"
    pullPolicy: IfNotPresent

  service:
    type: ClusterIP
    port: 6379

  persistence:
    enabled: true
    storageClass: "longhorn"
    accessModes:
      - ReadWriteOnce
    size: 5Gi

  resources:
    limits:
      cpu: 1000m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi
```

### Microservice Environment Variables:
```yaml
- name: VALKEY_HOST
  value: "valkey.jitsi.svc.cluster.local"
- name: VALKEY_PORT
  value: "6379"
```

---

## 🔍 5. Verification & Troubleshooting Runbook

### 1. Test Valkey Health & Response Time
```bash
kubectl exec -n jitsi deploy/unity-meet-valkey -- valkey-cli ping
# Output: PONG
```

### 2. View Active Cached Keys
```bash
# List all active meeting and calendar keys
kubectl exec -n jitsi deploy/unity-meet-valkey -- valkey-cli keys "*"
```

### 3. Check Memory & Hit Rate Statistics
```bash
kubectl exec -n jitsi deploy/unity-meet-valkey -- valkey-cli info stats
kubectl exec -n jitsi deploy/unity-meet-valkey -- valkey-cli info memory
```

### 4. Verify API Pod Connection in Logs
```bash
kubectl logs -l app.kubernetes.io/component=api -n jitsi --tail=30 | grep "Valkey"
# Output: 🚀 Connected to Valkey datastore at valkey:6379
```
