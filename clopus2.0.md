# Clopus 2.0 - Action Plan

A comprehensive roadmap to evolve Clopus Watcher from a CronJob-based tool into a production-grade Kubernetes controller.

---

## Security

| Issue                           | Risk                                         | Recommendation                                                               | Status |
|---------------------------------|----------------------------------------------|------------------------------------------------------------------------------|--------|
| pods/exec cluster-wide          | Watcher can exec into ANY pod in the cluster | Scope to specific namespaces with RoleBindings instead of ClusterRoleBinding | Done   |
| API key in environment variable | Visible in pod spec, logs                    | Mount as file from secret, use workload identity if available                | Pending |
| No audit logging                | No record of what fixes were applied         | Log all exec commands and changes to a separate audit log                    | Pending |
| Autonomous mode by default      | AI making unsupervised changes to production | Default to report mode, require explicit opt-in for autonomous               | Pending |

---

## Reliability

| Issue                       | Recommendation                                                                    | Status |
|-----------------------------|-----------------------------------------------------------------------------------|--------|
| SQLite on PVC               | Single point of failure, no HA. Consider PostgreSQL or external DB for production | Pending |
| No health checks on cronjob | Add liveness/readiness probes, or use a sidecar to report job health              | Pending |
| 62-minute stuck job we saw  | Add activeDeadlineSeconds to kill hung jobs                                       | Pending |
| No retry logic              | If Claude API fails, run fails. Add exponential backoff                           | Pending |

---

## Observability

| Gap                         | Recommendation                                                              | Status |
|-----------------------------|-----------------------------------------------------------------------------|--------|
| No metrics                  | Expose Prometheus metrics (runs, errors found, fixes applied, API latency)  | Pending |
| Basic logging               | Structured JSON logging with correlation IDs                                | Pending |
| No alerting                 | Integrate with AlertManager for failed runs or high error rates             | Pending |
| Dashboard lacks detail      | Show actual fix diffs, before/after state                                   | Pending |
| **Truncated reasoning logs** | Dashboard only captures final report, not Claude's intermediate reasoning   | Pending |

### Full Reasoning Chain Logging (New)

**Problem:** When Claude attempts multiple fixes (e.g., fix image typo → pod crashes → fix SCC issue), only the final summary is stored. The intermediate steps are lost.

**Impact:**
- Hard to debug failed fixes
- No audit trail of attempted actions
- Can't understand AI decision process

**Solution:**
- Capture full Claude conversation output
- Store in separate `reasoning_log` column in SQLite
- Display expandable "Show reasoning" in dashboard

---

## Functionality

| Missing Feature         | Value                                                                           | Status |
|-------------------------|---------------------------------------------------------------------------------|--------|
| Multi-namespace support | Single TARGET_NAMESPACE is limiting. Support comma-separated or label selectors | Pending |
| Exclude patterns        | Skip certain pods (e.g., kube-system, jobs, init containers)                    | Pending |
| Dry-run mode            | Show what would be fixed without applying                                       | Pending |
| Rollback capability     | Undo a fix if it made things worse                                              | Pending |
| Fix approval workflow   | Queue fixes for human approval before applying                                  | Pending |

---

## Architecture

### Current
```
CronJob → runs every 5 min regardless of state
```

### Target: Kubernetes Controller
```
Controller watches pod events → real-time detection → Claude analyzes → applies fixes
```

| Current                           | Better                                                 |
|-----------------------------------|--------------------------------------------------------|
| Polling (CronJob)                 | Event-driven (Operator/Controller)                     |
| Monolithic entrypoint             | Separate concerns: detector, analyzer, fixer, reporter |
| Tight coupling to Claude Code CLI | Abstract AI interface, support multiple backends       |

### Building a Controller with Kubebuilder

```bash
# Initialize project
kubebuilder init --domain example.com

# Create a custom resource + controller
kubebuilder create api --group apps --version v1 --kind PodWatcher

# This generates:
# - api/v1/podwatcher_types.go      (define your CRD schema)
# - controllers/podwatcher_controller.go (your reconciliation logic)
# - config/crd/                (CRD YAML manifests)
# - config/rbac/               (RBAC manifests)
```

The controller reconciliation loop:
```go
func (r *PodWatcherReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Watch for pod events
    // 2. Detect failures (CrashLoopBackOff, ImagePullBackOff, etc.)
    // 3. Call Claude API for analysis
    // 4. Apply fixes or queue for approval
    // 5. Record results
}
```

---

## Code Quality

| Issue                              | Fix                                                                      | Status |
|------------------------------------|--------------------------------------------------------------------------|--------|
| Claude Code writes to /home/claude | Container user may not own that dir. Fix Dockerfile or use --config flag | Done   |
| No tests visible                   | Add unit tests for detection logic, integration tests for API            | Pending |
| Hardcoded paths                    | Make configurable via env vars                                           | Pending |

---

## Priority Recommendations

### High Priority (Security/Stability)
1. ~~Scope RBAC to specific namespaces~~ ✅ Done
2. Add `activeDeadlineSeconds: 300` to jobs
3. Default to report mode, not autonomous

### Medium Priority (Operability)
4. Add Prometheus metrics endpoint
5. Structured logging
6. Multi-namespace support
7. **Full reasoning chain logging** (New)

### Lower Priority (Features)
8. Dry-run mode
9. Fix approval workflow
10. Move to controller pattern

---

## Completed Items

| Item | Date | Notes |
|------|------|-------|
| Scoped RBAC for microservice-app-dev | 2026-01-02 | Role + RoleBinding for write ops only in target namespace |
| OpenShift SCC compatibility | 2026-01-02 | Fixed Dockerfile: group-writable dirs for arbitrary UID |
| ArgoCD GitOps deployment | 2026-01-01 | Full GitOps workflow with SealedSecrets |
| API key sealed secret fix | 2026-01-02 | Removed conflicting plain secret.yaml |

---

## Test Cases Verified

| Scenario | Result |
|----------|--------|
| CrashLoopBackOff (intentional test pod) | Detected, correctly identified as unfixable |
| ImagePullBackOff (typo: nignx → nginx) | Fixed automatically |
| SCC permission issue (nginx rootless) | Fixed: /tmp dirs, port 8080 |
| Multi-step fix (image + SCC) | Both issues resolved in single run |
