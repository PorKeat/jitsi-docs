# 10. Real-Time Synchronization & Server-Sent Events (SSE) Architecture

This guide details the **Real-Time Push Synchronization** architecture between **Unity Meet** (`meet.unity-workspace.com`) and **Unity Calendar** (`calendar.unity-workspace.com`), utilizing **Valkey Pub/Sub**, **HTTP/2 Server-Sent Events (SSE)**, and **Multi-Layer Zero-Cache Invalidation**.

---

## ⚡ 1. The Real-Time Challenge & Solution

### The Problem with Traditional Polling
* In distributed microservice deployments with edge reverse proxies (Cloudflare, Caddy / Tiyi WAF, and Nginx Ingress), standard HTTP GET requests can easily be held in browser in-memory caches or edge CDN buffers.
* Polling at fixed intervals causes latency (up to the poll period) and burns unnecessary network and CPU cycles across replicas.
* Users previously had to manually refresh the page (`F5` / `Cmd+R`) to see newly scheduled meetings or deleted events.

### The Real-Time Push Architecture
With **Server-Sent Events (SSE)** coupled to **Valkey Pub/Sub**, any modification (creation, update, rescheduling, RSVP, or deletion) made on either application is pushed immediately to all connected browsers in **< 50 milliseconds** without page refreshes.

```
 [ Unity Meet Web ]            [ Unity Calendar Web ]
        │                               │
        ▼ (SSE /api/.../stream)          ▼ (SSE /api/events/stream)
 ┌──────────────────────┐        ┌──────────────────────┐
 │   unity-meet-web     │        │  unity-calendar-api  │
 │  (Next.js App Proxy) │        │      (Go API)        │
 └──────────┬───────────┘        └──────────┬───────────┘
            │                               │
            └───────────────┬───────────────┘
                            ▼
              ┌───────────────────────────┐
              │  Valkey 8.0 In-Memory     │
              │  Channel: unity:events:   │
              │          stream           │
              └───────────────────────────┘
```

---

## 📡 2. Core Components

### A. Valkey Pub/Sub Engine
When an event mutation occurs in the Calendar Go API (`CreateEvent`, `UpdateEvent`, `RescheduleEvent`, `UpdateEventRsvp`, `DeleteEvent`):
1. The database row is committed to PostgreSQL.
2. The cached list keys (`cal:events:list*`) are purged from Valkey.
3. A JSON event notification is published to the `unity:events:stream` channel:
   ```json
   {
     "action": "CREATE",
     "event_id": "evt_1a26dcaa",
     "timestamp": 1788753600000
   }
   ```

### B. Calendar Go API Streaming Endpoint (`/api/events/stream`)
* Connects directly to Valkey via `h.valkey.Subscribe(ctx, "unity:events:stream")`.
* Emits HTTP/2 SSE chunks with headers:
  * `Content-Type: text/event-stream`
  * `Cache-Control: no-cache, no-store, must-revalidate`
  * `Connection: keep-alive`
  * `X-Accel-Buffering: no` (disables buffering in Nginx and Caddy)
* Heartbeat: Dispatches a lightweight `ping` every 15 seconds to prevent edge proxy timeouts.

### C. Unity Meet Next.js Proxy Route (`/api/calendar/events/stream`)
* Streams directly from cluster internal DNS:
  `http://unity-calendar-api.calendar.svc.cluster.local:8000/api/events/stream`
* Configured with:
  * `export const dynamic = "force-dynamic";`
  * `export const revalidate = 0;`
  * Passthrough streaming `ReadableStream` to browser clients.

### D. Zero-Cache Invalidation & Client EventSource Listeners
* Browser clients instantiate standard `EventSource`:
  ```typescript
  const es = new EventSource("/api/calendar/events/stream");
  es.addEventListener("event_update", () => {
    fetchCalendarMeetings(false);
  });
  ```
* All fetch calls append a cache-busting timestamp `?_t=${Date.now()}` and send `cache: "no-store"` with `Cache-Control: no-cache` headers to prevent browser heuristic caching.
* Tab-local synchronization via `BroadcastChannel("unity_workspace_sync")` propagates updates across multiple tabs on the same origin instantly.

---

## 🛡️ 3. Ingress & Proxy Buffering Configuration

To ensure SSE streams are delivered with zero buffering through Kubernetes Nginx Ingress and Tiyi WAF:
1. **`X-Accel-Buffering: no`** is emitted on all SSE response headers.
2. **Ingress timeouts**: `proxy-read-timeout` and `proxy-send-timeout` set to `3600` (1 hour).
3. **Keepalive Heartbeats**: Sent every 15 seconds so Cloudflare never closes idle streams.
