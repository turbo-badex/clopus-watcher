# Clopus Watcher

A Kubernetes-native Claude Code watcher that monitors pods, detects errors, and applies hotfixes directly, or just writes a report on its findings.

## Overview

Clopus Watcher runs as a CronJob that:
1. Monitors pods in a target namespace
2. Detects degraded pods (CrashLoopBackOff, Error, etc.)
3. Reads logs to understand the error
4. Execs into the pod, explores and applies a hotfix
5. Records the fix to SQLite & provides a report

A separate Dashboard deployment provides a web UI to view all detected errors and applied fixes.

## Prerequisites

**Cluster:**

- Kubernetes or OpenShift cluster
- Sealed Secrets controller (for encrypting API key)

**Local (to build the images):**

- Docker with buildx (for multi-arch builds)
- kubectl / oc
- Container registry access (quay.io, ghcr.io, etc.)
- kubeseal CLI
- gh CLI (optional, for GitHub operations)

## Configuration

| Environment Variable | Description | Default |
|---------------------|-------------|---------|
| `TARGET_NAMESPACE` | Namespace to monitor | `default` |
| `AUTH_MODE` | Auth method: `api-key` or `credentials` | `api-key` |
| `WATCHER_MODE` | Watcher mode: `autonomous` or `watcher` | `autonomous` |
| `ANTHROPIC_API_KEY` | Claude API key (if AUTH_MODE=api-key) | - |
| `SQLITE_PATH` | Path to SQLite database | `/data/watcher.db` |

## Deployment

### OpenShift with ArgoCD (Recommended)

#### 1. Build and Push Images (for amd64 clusters)

```bash
# Login to registry
docker login quay.io -u <username>

# Build for amd64 (required for most clusters)
docker buildx build --platform linux/amd64 -t quay.io/<org>/clopus-watcher:latest -f Dockerfile.watcher --push .
docker buildx build --platform linux/amd64 -t quay.io/<org>/clopus-watcher-dashboard:latest -f Dockerfile.dashboard --push .
```

**Important:** If building on Apple Silicon (arm64), you MUST use `--platform linux/amd64` or pods will fail with "no image found in image index for architecture amd64".

#### 2. Create Sealed Secrets

```bash
# Fetch sealed secrets certificate
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  > /tmp/sealed-secrets-cert.pem

# Create and seal the API key secret
cat > /tmp/claude-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: claude-auth
  namespace: clopus-watcher
type: Opaque
stringData:
  api-key: sk-ant-xxxxx
EOF

kubeseal --cert /tmp/sealed-secrets-cert.pem \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --format yaml < /tmp/claude-secret.yaml > k8s/sealed-secret.yaml

# Clean up plain text secret
rm /tmp/claude-secret.yaml

# For private registry, create pull secret
kubectl create secret docker-registry quay-pull-secret \
  --namespace clopus-watcher \
  --docker-server=quay.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --dry-run=client -o yaml | \
kubeseal --cert /tmp/sealed-secrets-cert.pem \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --format yaml > k8s/quay-pull-sealed-secret.yaml
```

#### 3. Update Manifests

Edit `k8s/cronjob.yaml`:
- Set `TARGET_NAMESPACE` to your target namespace
- Update image to your registry: `quay.io/<org>/clopus-watcher:latest`

Edit `k8s/dashboard-deployment.yaml`:
- Update image to your registry: `quay.io/<org>/clopus-watcher-dashboard:latest`

#### 4. Deploy via ArgoCD

```bash
# Apply ArgoCD application
kubectl apply -f argocd-application.yaml

# Check sync status
kubectl get application clopus-watcher -n openshift-gitops
```

#### 5. Create Route (OpenShift)

```bash
# Create edge-terminated route
oc create route edge dashboard --service=dashboard --port=http -n clopus-watcher

# Get route URL
oc get route dashboard -n clopus-watcher
```

### Manual Deployment (Kubernetes)

```bash
# 1. Create namespace
kubectl create namespace clopus-watcher

# 2. Create secret with API key
kubectl create secret generic claude-auth \
  --namespace clopus-watcher \
  --from-literal=api-key=sk-ant-xxxxx

# 3. Deploy
./scripts/deploy.sh
```

## OpenShift-Specific Notes

### Security Context Constraints

The cronjob does NOT use init containers that run as root. OpenShift's restricted SCC will block `runAsUser: 0`. The current manifests are compatible with OpenShift's default security policies.

### Resource Limits

If your cluster is resource-constrained, the manifests use minimal CPU requests:
- Dashboard: 10m CPU request, 50m limit
- Watcher: 10m CPU request, 200m limit

### NetworkPolicy

The watcher bypasses NetworkPolicies because it communicates via the **Kubernetes API server**, not direct pod-to-pod traffic. Actions like listing pods, reading logs, and exec all go through the API server, which is not affected by namespace NetworkPolicies.

### RBAC

The watcher uses a ClusterRole (not cluster-admin) with minimal permissions:
- `pods`, `pods/status`: get, list, watch
- `pods/log`: get, list
- `pods/exec`: create, get
- `events`: get, list, watch
- `configmaps`: get, list

### Service Port Naming

OpenShift routes require named service ports. The dashboard service uses:
```yaml
ports:
  - name: http    # Required for route to work
    port: 80
    targetPort: 8080
```

Without the port name, routes will return "Application not available".

## Troubleshooting

### Dashboard shows "no namespace yet"

The watcher cronjob hasn't run yet, or it failed. Check:
```bash
# List jobs
oc get jobs -n clopus-watcher

# Check job logs
oc logs job/<job-name> -n clopus-watcher
```

### Pods stuck in Pending

Likely insufficient CPU. Lower resource requests in the manifests.

### Image pull errors

1. Check if images exist and are for correct architecture (amd64)
2. Ensure pull secret exists: `oc get secret quay-pull-secret -n clopus-watcher`
3. Verify imagePullSecrets in deployment/cronjob spec

### Route returns "Application not available"

1. Ensure service port has a `name` field
2. Recreate route with explicit port: `oc create route edge dashboard --service=dashboard --port=http`

## Triggering a Manual Run

```bash
# Create a one-off job from the cronjob
oc create job --from=cronjob/clopus-watcher manual-run -n clopus-watcher

# Watch logs
oc logs -f job/manual-run -n clopus-watcher

# Clean up
oc delete job manual-run -n clopus-watcher
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        clopus-watcher namespace                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  CronJob    │    │  Dashboard  │    │  PVC (watcher.db)   │  │
│  │  (watcher)  │───►│  (web UI)   │◄───│                     │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
│         │                  ▲                                    │
│         │                  │ Route                              │
└─────────┼──────────────────┼────────────────────────────────────┘
          │                  │
          ▼                  │
┌─────────────────┐          │
│  kube-apiserver │          │
└────────┬────────┘          │
         │                   │
         ▼                   │
┌─────────────────────────┐  │
│  target namespace       │  │
│  (microservice-app-dev) │  │
│  ┌─────┐ ┌─────┐        │  │
│  │ pod │ │ pod │ ...    │  │
│  └─────┘ └─────┘        │  │
└─────────────────────────┘  │
                             │
                        Browser
```
