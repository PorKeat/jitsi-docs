# 06. Operations, CLI & Troubleshooting Runbook

This guide contains operational procedures, monitoring commands, and solutions to common infrastructure and application incidents for Unity Meet.

---

## 🏛️ Part 1: Production Kubernetes & Helm Operations

Connect to the primary control plane node:
```bash
ssh -i ~/.ssh/unity-workspace-key root@10.1.18.10
```

### 1. Cluster Status & Pod Health
```bash
# View all nodes and ready state
kubectl get nodes -o wide

# Check all pods in the 'jitsi' namespace
kubectl get pods -n jitsi -o wide

# Check for any non-running pods cluster-wide
kubectl get pods -A | grep -v -E "Running|Completed"
```

### 2. Live Log Inspection
```bash
# Go API microservice logs
kubectl logs -l app.kubernetes.io/component=api -n jitsi -c api -f --tail=100

# Next.js 16 Web UI logs
kubectl logs -l app.kubernetes.io/component=web -n jitsi -c web -f --tail=100

# Jitsi WebRTC Web gateway & Nginx logs
kubectl logs -l app.kubernetes.io/name=web -n jitsi -f --tail=100

# JVB (Videobridge) media streams
kubectl logs -l app.kubernetes.io/component=jvb -n jitsi -f --tail=100

# Prosody XMPP signaling & JWT auth
kubectl logs -n jitsi unity-meet-jitsi-meet-prosody-0 -f --tail=100

# Valkey In-Memory datastore
kubectl logs -l app.kubernetes.io/component=valkey -n jitsi -f --tail=100
```

### 3. Safe GitOps Deployment / Upgrade
All Helm configuration is strictly maintained in `values-prod.yaml`:
```bash
cd /root/unity-meet-helm

# Lint chart before applying
helm lint . -f values-prod.yaml

# Atomic rolling upgrade
helm upgrade unity-meet . -n jitsi -f values-prod.yaml

# Zero-downtime rolling restart of microservices
kubectl rollout restart deployment/unity-meet-api deployment/unity-meet-web -n jitsi
kubectl rollout status deployment/unity-meet-api -n jitsi
kubectl rollout status deployment/unity-meet-web -n jitsi
```

---

## 🛠️ Part 2: Production Incident Runbooks

### Incident 1: `longhorn-driver-deployer` CrashLoopBackOff & Stuck `Terminating` Pods

* **Symptom:** `longhorn-driver-deployer` crashes or remains in `CrashLoopBackOff`, and several pods across namespaces are stuck in `Terminating`.
* **Root Cause:** When a physical node (e.g. `jitsi-meet1` / `10.1.18.9`) becomes `NotReady` or runs an incompatible runtime, kubelet does not send finalizers. DaemonSets and CSI discovery pods (such as `discover-proc-kubelet-cmdline`) stay in `Terminating` state, blocking Longhorn's deployer.
* **Resolution:**
  ```bash
  # 1. List stuck terminating pods
  kubectl get pods -A | grep Terminating

  # 2. Force delete stuck pods with zero grace period
  kubectl delete pod <stuck-pod-name> -n <namespace> --force --grace-period=0

  # 3. Specifically verify longhorn-system CSI pods are cleared:
  kubectl delete pod -n longhorn-system --force --grace-period=0 -l app=longhorn-driver-deployer

  # 4. Longhorn deployer will instantly spawn fresh and reach 1/1 Running:
  kubectl get pods -n longhorn-system -l app=longhorn-driver-deployer
  ```

---

### Incident 2: JVB Videobridge Pods Stuck in `Pending` (`hostPort` Conflict)

* **Symptom:** A JVB pod remains in `Pending` with event `0/3 nodes are available: 1 node(s) had untolerated taint, 2 node(s) didn't have free ports for the requested pod ports (10000)`.
* **Root Cause:** JVB uses `useHostNetwork: true` / `hostPort: 10000/UDP` for direct kernel-level WebRTC media routing. Each physical Kubernetes node can bind port 10000 exactly once.
* **Resolution:**
  * If your cluster has 2 active worker nodes (`jitsi-meet2` & `jitsi-meet3`), set `jitsi-meet.jvb.replicaCount: 2` in `values-prod.yaml`.
  * Never set `replicaCount` higher than the number of active physical worker nodes.
  ```bash
  # Scale deployment to match available nodes
  kubectl scale deployment unity-meet-jitsi-meet-jvb-0 -n jitsi --replicas=2
  ```

---

### Incident 3: Lib-Jitsi-Meet `TypeError: Cannot read properties of undefined (reading 'type')`

* **Symptom:** Chrome developer console reports:
  ```text
  ConnectionQuality.ts:213 Uncaught TypeError: Cannot read properties of undefined (reading 'type')
      at On.<anonymous> (ConnectionQuality.ts:213:29)
  ```
* **Root Cause:** In upstream `lib-jitsi-meet`, the stats callback expects an event object `t`, but an undefined event payload triggers `"stats" === t.type`, throwing an uncaught TypeError.
* **Resolution (GitOps Automated):**
  A permanent `lifecycle.postStart` hook is configured in `values-prod.yaml`:
  ```yaml
  jitsi-meet:
    web:
      lifecycle:
        postStart:
          exec:
            command:
            - /bin/sh
            - -c
            - perl -pi -e "s/\"stats\"===t\.type/t&&\"stats\"===t\.type/g" /usr/share/jitsi-meet/libs/lib-jitsi-meet.min.js
  ```
  Every time `unity-meet-jitsi-meet-web` starts or restarts, the script safely patches the minified bundle automatically.

---

### Incident 4: Screen Share Aspect Ratio Glitching & Jagged Zoom

* **Symptom:** When sharing a high-resolution or ultra-wide screen, the video jumps between cropped (`cover`) and padded (`contain`), or lags during stage zooming.
* **Resolution:**
  1. **GPU Acceleration:** Stage viewport uses `transform: scale(...)` and CSS `will-change: transform; transform: translateZ(0);` for 60 FPS hardware acceleration.
  2. **1-Click Fit / Fill Toggle:** Attendees can switch between **Fit to Screen** (preserve full text clarity without cropping) and **Fill Stage** (immersive full screen) via the floating stage pill controls.
  3. **Floating Controls Capsule:** Stage controls float cleanly with `backdrop-blur-md` without obscuring presentations or bottom toolbars.

---

### Incident 4: HashiCorp Vault Agent Sidecar & Secret Rotation Runbook

#### 1. Inspecting Vault Agent Injected Credentials
To verify that credentials are mounting purely into in-memory `tmpfs` without persistence:
```bash
# Verify pod has 2/2 containers running (api + vault-agent)
kubectl get pods -n jitsi -l app.kubernetes.io/component=api

# Inspect in-memory secrets file inside running pod
kubectl exec -n jitsi deploy/unity-meet-api -c api -- ls -la /vault/secrets
kubectl exec -n jitsi deploy/unity-meet-api -c api -- cat /vault/secrets/credentials.env
```

#### 2. Rotating Secrets in HashiCorp Vault (Zero Git Changes)
To rotate the JWT signing key or database password without modifying Helm charts or Git:
```bash
# Connect to Vault Host (10.1.18.8)
ssh -i ~/.ssh/unity-workspace-key root@10.1.18.8

# Authenticate with Vault CLI
export VAULT_TOKEN="<admin-token>"
export VAULT_CACERT="/opt/vault/tls/vault-ca.crt"
export VAULT_ADDR="https://127.0.0.1:8200"

# Put new secret values
vault kv put secret/unity-workspace/meet/api \
  jwt_app_id="unity_meet_enterprise" \
  jwt_app_secret="<new-secure-secret-key>" \
  database_url="postgres://postgres:<new-password>@unity-meet-postgres:5432/unity_meet?sslmode=disable"
```

#### 3. Triggering Zero-Downtime Rolling Update
Once Vault is updated, restart the deployment so Vault Agent fetches the new version:
```bash
# On K8s Control Plane (10.1.18.10)
kubectl rollout restart deployment/unity-meet-api -n jitsi
kubectl rollout status deployment/unity-meet-api -n jitsi
```

#### 4. Troubleshooting Vault Agent Webhook / Auth Failures
If API pods show `Init:0/1` or `CrashLoopBackOff`:
```bash
# Check init container logs (retrieves secret at startup)
kubectl logs deploy/unity-meet-api -n jitsi -c vault-agent-init

# Check sidecar logs (keeps secrets refreshed)
kubectl logs deploy/unity-meet-api -n jitsi -c vault-agent

# Verify Kubernetes API TokenReview proxy on 10.1.18.10:8443
systemctl status k8s-apiserver-proxy.service
```

---

## 💻 Part 3: Local Docker Compose Operations

```bash
# Start all local microservices
docker compose up -d

# Check status of local containers
docker compose ps

# Force recreate local containers
docker compose up -d --force-recreate

# View real-time logs
docker compose logs -f

# Safely stop local containers
docker compose down
```
