# minecraft-k8s-server
# Minecraft on Kubernetes

A Paper Minecraft server running on a local multi-node `kind` cluster, fronted by a
Velocity proxy, with Dynmap for live world mapping.


## Architecture

```
Minecraft client
  -> localhost:30565            (kind extraPortMapping)
  -> mc-nodeport Service        (nodePort 30565 -> targetPort 25577)
  -> mc-proxy pod               (Velocity, listening on 25577)
  -> mc-0.mc-headless...:25565  (Velocity's own connection, from velocity.toml)
  -> mc-0 pod                   (Paper 1.21.4)
```

The proxy is the only externally reachable component. Backend servers run in offline
mode and are only reachable from inside the cluster.

| Component | Kind | Notes |
|---|---|---|
| `mc` | StatefulSet | Paper 1.21.4, 1 replica, 5Gi PVC per replica |
| `mc-proxy` | Deployment | Velocity, stateless, no PVC |
| `mc-headless` | Service (headless) | DNS only, gives `mc-0` a stable name |
| `mc-nodeport` | Service (NodePort) | External entry point, selects `app: mc-proxy` |
| `proxy-service` | Service (ClusterIP) | Internal only, currently unused |
| `mc-config` | ConfigMap | Game settings as env vars |
| `mc-proxy-config` | ConfigMap | `velocity.toml` as a mounted file |
| `mc-secret` | Secret | RCON password |
| `proxy-secret` | Secret | Velocity forwarding secret |

## Prerequisites

- Docker Desktop with at least 6 GiB allocated (`docker info --format "{{.MemTotal}}"`)
- `kind`
- `kubectl`
- A Minecraft Java client on **1.21.4** — the server version is pinned

## Setup

### 1. Create the cluster

```powershell
kind create cluster --config kind-config.yaml
kubectl get nodes
```

Port mappings are fixed at creation time and cannot be added later. Confirm they exist:

```powershell
docker ps --format "{{.Names}}`t{{.Ports}}"
```

You should see `30565` and `30123` on the worker node.

### 2. Create the secrets

The real secret files are gitignored. Copy the examples and fill in your own values:

```powershell
cp mc-secret.example.yaml mc-secret.yaml
cp proxy-secret.example.yaml proxy-secret.yaml
```

Generate values with:

```powershell
-join ((48..57) + (65..90) + (97..122) | Get-Random -Count 32 | ForEach-Object {[char]$_})
```

### 3. Apply the manifests

Order matters — the namespace is defined in `mc-statefulset.yaml`:

```powershell
kubectl apply -f mc-statefulset.yaml
kubectl apply -f mc-config.yaml
kubectl apply -f mc-secret.yaml
kubectl apply -f mc-nodeport.yaml
kubectl apply -f proxy-secret.yaml
kubectl apply -f mc-proxy-config.yaml
kubectl apply -f mc-proxy.yaml
```

Wait for the server to finish generating its world:

```powershell
kubectl logs -n minecraft mc-0 -f
```

Look for `Done (XX.XXXs)! For help, type "help"`.

### 4. Manual step — `paper-global.yml`

**This step is not captured in any manifest and must be done by hand after the first
server start.**

`paper-global.yml` lives on the PersistentVolumeClaim at `/data/config/paper-global.yml`.
It is not stored in a ConfigMap for two reasons:

- It contains the Velocity forwarding secret, which must not be committed to git
- It carries a schema `_version` field that changes between Paper releases; freezing a
  copy in a ConfigMap would break future Paper upgrades

Edit it in place:

```powershell
kubectl exec -n minecraft mc-0 -- cp /data/config/paper-global.yml /data/config/paper-global.yml.bak
kubectl exec -it -n minecraft mc-0 -- nano /data/config/paper-global.yml
```

In nano: `Ctrl+W` to search for `velocity`, edit, `Ctrl+O` then Enter to save, `Ctrl+X` to exit.

Set the `velocity:` section (around line 112) to:

```yaml
  velocity:
    enabled: true
    online-mode: true
    secret: 'YOUR_FORWARDING_SECRET_HERE'
```

The secret must match the value in `proxy-secret.yaml` exactly. Keep the four leading
spaces — YAML is whitespace-sensitive and nano will not warn you.

Verify and restart:

```powershell
kubectl exec -n minecraft mc-0 -- grep -n -A 5 "velocity:" /data/config/paper-global.yml
kubectl delete pod -n minecraft mc-0
```

This survives pod restarts because it lives on the PVC. It must be redone if the PVC is
deleted.

### 5. Connect

Minecraft client -> Multiplayer -> Add Server -> `localhost:30565`

## Common operations

```powershell
# Stop everything without losing data
kubectl scale statefulset mc -n minecraft --replicas=0
kubectl scale deployment mc-proxy -n minecraft --replicas=0

# Start again
kubectl scale statefulset mc -n minecraft --replicas=1
kubectl scale deployment mc-proxy -n minecraft --replicas=1

# Server console
kubectl exec -it -n minecraft mc-0 -- rcon-cli

# Logs
kubectl logs -n minecraft mc-0 -f
kubectl logs -n minecraft -l app=mc-proxy -f
kubectl logs -n minecraft mc-0 --previous    # after a crash

# The proxy pod has a random name; capture it first
$proxy = kubectl get pod -n minecraft -l app=mc-proxy -o jsonpath='{.items[0].metadata.name}'
kubectl exec -n minecraft $proxy -- cat /server/velocity.toml

# Backup the world
kubectl cp minecraft/mc-0:/data ./mc-0-backup
```

## Known issues

- **`paper-global.yml` is manual.** See step 4.
- **Only one backend server exists.** Velocity is configured for a single `survival`
  backend. Multiple worlds require separate StatefulSets, not `--replicas=3`, because a
  StatefulSet has one pod template and all replicas would be configured identically.
