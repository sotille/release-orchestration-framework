# Release Incident Playbook

Release failures — failed deployments, unintended regressions, botched database migrations, and canary analysis failures — are a category of operational incident with distinct characteristics. Unlike security incidents or infrastructure outages with external causes, release incidents typically have a narrow blast radius, a clear rollback target, and a bounded time window for decision-making.

This playbook defines decision trees, runbooks, and communication procedures for the most common release incident scenarios.

---

## Release Incident Classification

| Class | Description | Time to Decision | Rollback Threshold |
|-------|-------------|-----------------|-------------------|
| **Class 1 — Silent failure** | Deployment completed but the application is not functioning correctly; users affected | < 5 minutes from detection | Any confirmed user impact |
| **Class 2 — Canary rejection** | Automated canary analysis detected regression; traffic shift halted before full rollout | Immediate (automated) | Canary analysis policy fail |
| **Class 3 — Failed deployment** | Deployment did not complete; some or all pods/instances failed to start | Immediate | Any deployment failure |
| **Class 4 — Data migration failure** | Schema migration ran but caused incorrect data state, lock timeout, or partial execution | < 15 minutes | Any data integrity issue |
| **Class 5 — Multi-service cascading failure** | Release caused failures in downstream services | < 10 minutes | Any confirmed cascading impact |

---

## Decision Authority

Clear authority to act is essential during release incidents — delays caused by waiting for approval can extend user impact significantly.

| Action | Authority Required |
|--------|------------------|
| Rollback a single service to previous version | On-call engineer (self-authorized) |
| Pause canary or blue/green traffic shift | On-call engineer (self-authorized) |
| Rollback multiple services simultaneously | Release manager or engineering lead |
| Roll back a database migration (Class C/D) | Engineering lead + DBA |
| Declare major incident (P1) | On-call engineer or SRE lead |
| Communicate to customers | Communications + Legal sign-off |

---

## Incident 1: Failed Kubernetes Deployment

### Symptoms
- `kubectl rollout status deployment/your-app` returns failure
- Pods stuck in `CrashLoopBackOff`, `Error`, `Pending`, or `ImagePullBackOff`
- Health check endpoints returning non-200 or not responding
- Alerting from platform monitoring on pod restart rate or deployment stall

### Diagnostic Runbook

```bash
# 1. Check deployment rollout status
kubectl rollout status deployment/your-app --timeout=60s

# 2. Identify failing pods
kubectl get pods -l app=your-app --sort-by=.metadata.creationTimestamp

# 3. Check pod events for specific failure reason
kubectl describe pod <failing-pod-name> | grep -A 20 "Events:"

# 4. Check application logs from the failing pod
kubectl logs <failing-pod-name> --previous --tail=100

# 5. Check if the new image is accessible
kubectl run debug-image-pull \
  --image=registry.internal/your-app:NEW_TAG \
  --restart=Never --rm -it -- echo "Image pull succeeded"

# 6. Check recent deployment configuration changes
kubectl diff -f deploy/your-app.yaml
```

**Common failure modes and resolutions:**

| Error | Likely Cause | Resolution |
|-------|-------------|----------|
| `ImagePullBackOff` | Image tag does not exist in registry | Verify image was pushed; check tag spelling; check registry credentials |
| `CrashLoopBackOff` | Application fails to start | Check logs; likely config error, missing environment variable, or startup crash |
| `Pending` (no nodes) | Insufficient cluster capacity | Scale cluster or investigate node pressure; rollback if urgent |
| `OOMKilled` | Memory limit too low for new code | Rollback; adjust resource limits in follow-up |
| `Readiness probe failed` | App started but not serving traffic | Check readiness endpoint; may be slow startup or external dependency |

### Rollback Procedure

```bash
# Rollback to previous revision (Kubernetes native)
kubectl rollout undo deployment/your-app

# Verify rollback
kubectl rollout status deployment/your-app

# Confirm the correct previous image is running
kubectl get deployment your-app -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check that health checks pass
kubectl rollout status deployment/your-app --timeout=120s && \
  echo "Rollback successful" || echo "Rollback also failing — escalate"
```

**GitOps rollback (ArgoCD):**
```bash
# Roll back to the last good commit in the GitOps repo
# Step 1: Find the last successful sync hash
argocd app history your-app | head -5

# Step 2: Roll back to that revision
argocd app rollback your-app <HISTORY_ID>

# Step 3: Disable auto-sync to prevent re-apply of the failing commit
argocd app set your-app --sync-policy none
```

---

## Incident 2: Canary Analysis Failure

### Symptoms
- Canary analysis tool (Flagger, Argo Rollouts, or manual analysis) reports regression
- Traffic shift paused at current canary percentage
- Automated rollback triggered (if configured)

### Assessment

Before manual rollback, review the canary analysis results:

```bash
# Argo Rollouts — view canary analysis details
kubectl argo rollouts get rollout your-app --watch

# View specific analysis run results
kubectl get analysisrun -l rollout=your-app --sort-by=.metadata.creationTimestamp
kubectl describe analysisrun <analysis-run-name>

# Check the metrics that triggered rejection
kubectl get analysisrun <analysis-run-name> -o json | \
  jq '.status.metricResults[] | {metric: .name, phase: .phase, value: .measurements[-1].value}'
```

**Interpreting rejection signals:**

| Metric Rejected | Likely Root Cause | Priority |
|----------------|------------------|---------|
| Error rate spike | Bug in new code path | High — immediate rollback |
| P99 latency increase | Performance regression or dependency issue | High — rollback if > 20% degradation |
| Success rate < baseline | Feature-specific failure | High — rollback |
| Custom business metric | Feature logic error | Medium — assess user impact before rollback |

### Rollback Procedure

```bash
# Argo Rollouts: abort and roll back
kubectl argo rollouts abort your-app
kubectl argo rollouts undo your-app

# Flagger: manually trigger rollback
kubectl -n your-namespace annotate canary/your-app \
  flagger.app/canary-weight=0

# Verify rollback completed and stable version is serving 100% traffic
kubectl argo rollouts get rollout your-app
```

### Post-Canary Failure Analysis

```bash
# Pull the analysis window metrics for comparison
# (adjust for your observability stack)
kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant \
  'sum(rate(http_requests_total{app="your-app",status=~"5.."}[5m])) /
   sum(rate(http_requests_total{app="your-app"}[5m]))' \
  --time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

---

## Incident 3: Database Migration Failure

See [Database Migration Safety](database-migration-safety.md) for full migration classification and runbooks. This section covers the incident response aspects.

### Class B/C Migration Partial Failure

A migration that fails partway through may leave the schema in an inconsistent state. Do not attempt to resume a partially-applied migration without understanding its current state.

```bash
# PostgreSQL: Check migration state
# (using Alembic as example — adapt for Flyway, Liquibase, etc.)

# View current applied migrations
alembic history --verbose | head -20

# View current revision in the database
psql $DB_URL -c "SELECT version_num FROM alembic_version;"

# Check for locks that may have stalled the migration
psql $DB_URL -c "
  SELECT pid, query, state, wait_event_type, wait_event,
         now() - query_start AS duration
  FROM pg_stat_activity
  WHERE state != 'idle'
  ORDER BY duration DESC;
"

# If a migration-related transaction is still open, identify it
psql $DB_URL -c "
  SELECT pid, usename, application_name, state, query_start,
         now() - query_start AS duration, query
  FROM pg_stat_activity
  WHERE query LIKE '%alembic%' OR application_name LIKE '%migration%'
"
```

**Decision matrix for partial migration:**

| Scenario | Action |
|----------|--------|
| Migration rolled back atomically (transaction) | Safely re-run or investigate and fix |
| Migration partially applied (non-transactional DDL) | Do NOT re-run; assess schema state; restore from backup for Class C/D |
| Data corruption detected | Immediate restore from pre-migration backup |
| Schema inconsistent but data intact | Manual schema repair by DBA; test thoroughly before re-enabling traffic |

### Migration Rollback Procedure (Class C/D)

```bash
# STOP application writes immediately — prevent further data modification
# (Kubernetes: scale deployments to 0 to prevent writes)
kubectl scale deployment your-app --replicas=0

# Verify all application writes have stopped
psql $DB_URL -c "
  SELECT count(*) FROM pg_stat_activity
  WHERE application_name LIKE '%your-app%' AND state = 'active';
"

# Initiate restore from pre-migration snapshot (AWS RDS example)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier your-db-restored \
  --db-snapshot-identifier pre-migration-snapshot-id \
  --db-instance-class db.r6g.2xlarge \
  --publicly-accessible false

# Monitor restore progress
aws rds describe-db-instances \
  --db-instance-identifier your-db-restored \
  --query 'DBInstances[0].DBInstanceStatus'

# After restore is available, validate schema and data integrity
psql $RESTORED_DB_URL -c "\d your-table"

# Update DNS/connection string to point to restored instance
# (using Route53 CNAME update or connection string update in Secrets Manager)
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch file://dns-cutback.json

# Scale application back up
kubectl scale deployment your-app --replicas=3
kubectl rollout status deployment/your-app
```

---

## Incident 4: Multi-Service Cascading Release Failure

When a release causes failures across multiple dependent services, containment and sequenced recovery are critical.

### Containment (First 10 minutes)

```bash
# 1. Identify the originating service (the release that started the cascade)
# Check deployment history across all affected services
kubectl rollout history deployment --all-namespaces 2>/dev/null | \
  grep -v "^$" | tail -30

# 2. Freeze all pending releases across all services
# GitOps: suspend all Flux kustomizations
kubectl -n flux-system get kustomization -o json | \
  jq -r '.items[].metadata.name' | \
  xargs -I{} kubectl -n flux-system patch kustomization {} \
    --type merge -p '{"spec":{"suspend":true}}'

# 3. Identify service dependency graph to understand cascade path
# (from your service mesh or topology documentation)
```

### Sequenced Recovery

In a cascading failure, rollback in reverse dependency order — recover leaf services before core services:

```
Dependency graph (example):
  user-api → order-service → inventory-service → notification-service
                           ↓
                         payment-service

Recovery sequence:
  1. notification-service (leaf)
  2. payment-service (leaf)
  3. inventory-service
  4. order-service
  5. user-api
```

```bash
# Roll back services in reverse dependency order
for SERVICE in notification-service payment-service inventory-service order-service user-api; do
  echo "Rolling back $SERVICE..."
  kubectl rollout undo deployment/$SERVICE

  echo "Waiting for $SERVICE to stabilize..."
  kubectl rollout status deployment/$SERVICE --timeout=120s

  echo "Verifying $SERVICE health..."
  # Add your service-specific health check here

  echo "$SERVICE recovered. Proceeding to next service."
  sleep 10
done
```

---

## Incident Communication

### Internal Status Updates

During a release incident, post status updates in the incident channel at these intervals:
- P1: Every 15 minutes
- P2: Every 30 minutes
- P3: Single initial + resolution notification

**Status update template:**

```
[RELEASE INCIDENT UPDATE — HH:MM UTC]
Service(s) affected: [list]
Current status: [Investigating / Rolling back / Recovering / Resolved]
User impact: [Description or "No user-visible impact"]
Actions taken in last period:
  - [Action]
ETA to resolution: [Estimate or "Unknown — investigating"]
Next update: [Timestamp]
IC: [Name]
```

### Post-Release Incident Report

Complete within 24 hours for P1 incidents, 72 hours for P2:

| Section | Required Content |
|---------|----------------|
| Timeline | Start time, detection time, rollback time, resolution time |
| Root cause | Specific code change, config change, or process failure |
| Detection | How the incident was detected; was alerting effective? |
| Impact | User-visible impact duration and scope |
| Rollback | What was rolled back; was rollback straightforward? |
| Action items | At least 2-3 specific improvements with owners and due dates |

---

## Related Techstream Resources

| Topic | Document |
|-------|---------|
| Database migration safety | [Database Migration Safety](database-migration-safety.md) |
| Progressive delivery | [Progressive Delivery](progressive-delivery.md) |
| GitOps architecture | [GitOps Architecture](gitops-architecture.md) |
| Multi-region deployment | [Multi-Region Deployment](multi-region-deployment.md) |
| Cross-framework IR | [Cross-Framework Incident Response](../../techstream-docs/docs/cross-framework-incident-response.md) |

*Part of the Techstream Release Orchestration Framework. Licensed under Apache 2.0.*
