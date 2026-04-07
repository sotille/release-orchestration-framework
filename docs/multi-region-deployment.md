# Multi-Region Deployment Orchestration

Deploying applications across multiple geographic regions introduces complexity in deployment sequencing, consistency verification, rollback coordination, and compliance with data residency requirements. This document covers the architecture patterns, sequencing models, and operational controls required for safe, auditable multi-region releases.

---

## Why Multi-Region Releases Fail

Multi-region deployment incidents typically follow one of four patterns:

| Failure Pattern | Root Cause | Prevention |
|----------------|-----------|-----------|
| **Partial deployment** | Region 3 failed mid-deploy; 2 of 4 regions run new version | Health-check gates between region groups; automatic rollback on failure |
| **Region ordering misconfiguration** | Production rollout started before canary region validated | Enforced sequential deployment with promotion gates |
| **Data migration races** | Database schema migration ran in all regions simultaneously; some regions hit schema-dependent code before migration completed | Migration-first deployment order; backward-compatible schema changes |
| **Rollback inconsistency** | Rollback triggered in us-east-1 but not in eu-west-1; split-version state persisted for hours | Automated multi-region rollback; version state dashboard |

---

## Deployment Sequencing Models

### Sequential Regional Rollout (Safest)

Regions are deployed one at a time in a defined order. Each region is validated before the next begins.

```
Region 1 (Canary)  →  validate  →  Region 2  →  validate  →  Region 3  →  validate  →  Region 4
```

**Best for:** High-risk releases; schema migrations; releases with cross-region data dependencies
**Trade-off:** Longest deployment window; regions run different versions for hours

### Wave-Based Rollout (Most Common)

Regions are grouped into waves based on risk profile. Low-traffic regions deploy first; high-traffic or regulated regions deploy last.

```
Wave 1: ap-southeast-2 (canary)
  └── validate (5 min)
Wave 2: us-west-2, eu-central-1
  └── validate (15 min)
Wave 3: us-east-1 (primary production)
  └── validate (15 min)
Wave 4: eu-west-1 (GDPR primary), us-gov-west-1 (FedRAMP)
```

**Best for:** Most production releases in multi-region SaaS
**Trade-off:** Regions within a wave deploy concurrently; wave failure requires regional rollback

### Parallel Rollout (Fastest, Highest Risk)

All regions deploy simultaneously. Rollback must be coordinated across all regions simultaneously.

**Appropriate only for:** Hotfixes where deployment speed outweighs the risk of simultaneous failure; must be paired with automated rollback triggers

---

## Region Selection Strategy for Wave 1 (Canary)

The canary region should be:
- **Low-traffic** — failures affect fewer users
- **Representative** — uses the same infrastructure stack as production regions
- **Recoverable** — can be rolled back independently without cascading effects
- **Not compliance-critical** — not the primary region for regulated workloads (GDPR, FedRAMP)

Common canary region choices: `ap-southeast-2` (Sydney), `eu-north-1` (Stockholm), `sa-east-1` (São Paulo).

Do not use a development or staging region as the canary — it will not reflect production-equivalent traffic patterns.

---

## Health Check Gate Design

Between deployment waves, automated health gates verify the deployed region before proceeding. A failing gate stops the rollout and triggers rollback for the affected wave.

### Gate Criteria

| Gate Type | Description | Failure Threshold |
|-----------|-------------|------------------|
| **Deployment health** | Pod/container readiness probes pass | Any pod stuck in CrashLoopBackOff for > 3 minutes |
| **Application health** | `/health` and `/ready` endpoints return 200 | Health check fails in any region for > 2 consecutive minutes |
| **Error rate** | 5xx error rate compared to pre-deployment baseline | Error rate increases by > 2% above baseline for 5 minutes |
| **Latency** | P99 response time compared to baseline | P99 increases by > 200ms above baseline for 5 minutes |
| **Business metric** | Transaction success rate, conversion rate (if applicable) | Drops by > 5% from baseline |

### Gate Implementation (Prometheus / Grafana)

```promql
# Error rate increase detector — fire if error rate doubles from baseline
100 * (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) > 2

# P99 latency gate
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m])
) > 0.5
```

```bash
#!/bin/bash
# wait-for-gates.sh — checks health gates before proceeding to next wave
# Called by CI/CD pipeline between deployment waves

REGION=$1
PROMETHEUS_URL="https://prometheus.internal"
BASELINE_ERROR_RATE=0.5  # Acceptable error rate %

echo "Checking health gates for region: $REGION"

for i in $(seq 1 18); do  # 18 checks × 10 seconds = 3 minutes
  ERROR_RATE=$(curl -s "${PROMETHEUS_URL}/api/v1/query" \
    --data-urlencode "query=100 * sum(rate(http_requests_total{status=~\"5..\",region=\"${REGION}\"}[2m])) / sum(rate(http_requests_total{region=\"${REGION}\"}[2m]))" \
    | jq -r '.data.result[0].value[1]')

  if (( $(echo "$ERROR_RATE < $BASELINE_ERROR_RATE * 2" | bc -l) )); then
    echo "Gate PASSED: error rate ${ERROR_RATE}% is within threshold"
    exit 0
  fi

  echo "Check $i/18: error rate ${ERROR_RATE}% exceeds threshold — waiting 10s..."
  sleep 10
done

echo "Gate FAILED: error rate exceeded threshold for 3 minutes"
exit 1
```

---

## Database Migration in Multi-Region Deployments

Database schema migrations across multi-region deployments require the most careful sequencing. An incompatible schema change deployed before application rollout — or after — causes errors in one or more regions.

### Expand/Contract Pattern (Required for Zero-Downtime)

Never make a destructive schema change in a single deployment. Use the expand/contract pattern:

```
Phase 1 — Expand (backward-compatible):
  - Add new column/table
  - Old code: ignores new column
  - New code: reads/writes new column

Phase 2 — Migrate data:
  - Backfill data into new column
  - Both old and new code work

Phase 3 — Contract (after all regions run new code):
  - Remove old column/table
  - New code only
```

### Multi-Region Migration Sequencing

```
Step 1: Run database migration in ALL regions (expand only)
  → No application code deployed yet
  → Migration must be backward-compatible with current code

Step 2: Deploy new application code to Wave 1 region
  → New code uses new schema
  → Other regions still run old code against migrated schema
  → Both old and new code must work during this window

Step 3: Validate Wave 1; deploy to Wave 2, Wave 3...

Step 4 (separate release): Contract migration — remove old columns/tables
  → Only after 100% of regions run new code
  → Typically in a subsequent release cycle
```

---

## Multi-Region Rollback Orchestration

Rolling back a deployment that has already reached multiple regions requires coordination across all affected regions simultaneously to avoid a mixed-version state persisting.

### Rollback Trigger Conditions

Define explicit conditions that trigger automated rollback:

| Condition | Scope | Rollback Target |
|-----------|-------|----------------|
| Health gate failure in Wave 1 | Wave 1 only | Roll back Wave 1; abort remaining waves |
| Health gate failure in Wave 2 | Wave 2 regions | Roll back Wave 2; leave Wave 1 at new version |
| Error rate spike across all deployed regions | All deployed regions | Roll back all regions simultaneously |
| Manual rollback trigger (incident declared) | All regions | Roll back all regions simultaneously |

### Rollback Implementation

Rollback should be automated from a central orchestration plane. Never require an engineer to manually trigger rollback in each region individually during an incident.

**ArgoCD ApplicationSet for multi-region rollback:**

```yaml
# Application set manages deployment across regions
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: app-multi-region
spec:
  generators:
    - list:
        elements:
          - region: us-east-1
            cluster: eks-us-east-1
            wave: "3"
          - region: us-west-2
            cluster: eks-us-west-2
            wave: "2"
          - region: ap-southeast-2
            cluster: eks-ap-southeast-2
            wave: "1"   # Canary

  template:
    metadata:
      name: 'app-{{region}}'
      annotations:
        argocd.argoproj.io/sync-wave: '{{wave}}'
    spec:
      destination:
        server: '{{cluster}}'
        namespace: production
      source:
        repoURL: https://github.com/your-org/app-gitops
        targetRevision: HEAD
        path: 'environments/production/{{region}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

```bash
# Rollback all regions to previous version
# Called by CI/CD pipeline when a rollback gate fires

PREVIOUS_TAG=$1

echo "Initiating multi-region rollback to ${PREVIOUS_TAG}"

# Update the GitOps manifests to point to previous version
for REGION in us-east-1 us-west-2 ap-southeast-2 eu-west-1; do
  sed -i "s|image: registry.internal/app:.*|image: registry.internal/app:${PREVIOUS_TAG}|g" \
    environments/production/${REGION}/deployment.yaml
done

git commit -am "rollback: revert all regions to ${PREVIOUS_TAG}"
git push

# ArgoCD detects the commit and synchronizes all regions
echo "Rollback initiated. Monitor ArgoCD for sync status."
```

---

## Data Residency and Compliance in Multi-Region Deployments

Multi-region architectures often span regulatory jurisdictions. Deployment orchestration must respect data residency constraints — some regions may not be permitted to receive certain data components.

### Region Classification Framework

Classify each region by its compliance profile before designing deployment waves:

| Region Tag | Meaning | Deployment Constraints |
|-----------|---------|----------------------|
| `gdpr-primary` | Primary GDPR processing region | EU personal data must not leave this region; encryption keys regional |
| `fedramp-boundary` | FedRAMP authorization boundary | Only FedRAMP-authorized images and configurations |
| `pci-in-scope` | In-scope for PCI DSS | Cardholder data systems only; additional scan requirements before deployment |
| `standard` | No special regulatory constraints | Standard wave deployment; no additional controls |

```yaml
# Kubernetes namespace label — enforced by OPA admission controller
# Prevents deployment of non-compliant components to regulated regions
metadata:
  labels:
    compliance.techstream.io/gdpr-primary: "true"
    compliance.techstream.io/fedramp-boundary: "false"
    compliance.techstream.io/pci-in-scope: "true"
```

### Deployment Gate: Compliance Validation Before Production Regions

Before deploying to any region tagged `gdpr-primary`, `fedramp-boundary`, or `pci-in-scope`, run a compliance validation gate:

```bash
#!/bin/bash
# compliance-gate.sh — validates artifact compliance before regulated region deploy

REGION=$1
IMAGE_DIGEST=$2

# 1. Verify image signature
cosign verify --certificate-identity="..." --certificate-oidc-issuer="..." \
  "registry.internal/app@${IMAGE_DIGEST}"

# 2. Verify SBOM is present and complete
curl -s "https://dependency-track.internal/api/v1/component/identity" \
  -H "X-Api-Key: ${DT_API_KEY}" \
  -G --data-urlencode "purl=pkg:docker/app@${IMAGE_DIGEST}" \
  | jq -e '.components | length > 0' || {
    echo "COMPLIANCE GATE FAILED: No SBOM found for image ${IMAGE_DIGEST}"
    exit 1
  }

# 3. Verify no open Critical/High vulnerabilities past SLA
OVERDUE=$(curl -s "https://dependency-track.internal/api/v1/finding/project/${PROJECT_UUID}" \
  -H "X-Api-Key: ${DT_API_KEY}" \
  | jq '[.[] | select(.vulnerability.severity == "CRITICAL" and .attribution.alternateIdentifier != null)] | length')

if [ "$OVERDUE" -gt "0" ]; then
  echo "COMPLIANCE GATE FAILED: ${OVERDUE} overdue Critical findings. Resolve before deploying to ${REGION}."
  exit 1
fi

echo "Compliance gate PASSED for region ${REGION}"
```

---

## Multi-Region Deployment Audit Trail

Each multi-region deployment must generate an auditable record covering:

- What version was deployed (image digest, not tag)
- Which regions received the deployment and in what order
- Who authorized the deployment
- When each region transitioned between states (deploying → healthy → validated → promoted)
- Any gates that failed and what triggered rollback

**Deployment audit log schema:**

```json
{
  "deployment_id": "deploy-2026-04-07-001",
  "artifact": {
    "image": "registry.internal/app",
    "digest": "sha256:a1b2c3...",
    "tag": "build-1234",
    "sbom_url": "https://artifacts.internal/sboms/build-1234.json",
    "signature_verified": true
  },
  "authorized_by": "release-gate-approver@example.com",
  "authorized_at": "2026-04-07T14:23:00Z",
  "change_request": "CHG-4521",
  "regions": [
    {
      "region": "ap-southeast-2",
      "wave": 1,
      "deployed_at": "2026-04-07T14:25:00Z",
      "health_gate_passed_at": "2026-04-07T14:27:30Z",
      "status": "healthy"
    },
    {
      "region": "us-west-2",
      "wave": 2,
      "deployed_at": "2026-04-07T14:30:00Z",
      "health_gate_passed_at": "2026-04-07T14:35:00Z",
      "status": "healthy"
    },
    {
      "region": "us-east-1",
      "wave": 3,
      "deployed_at": "2026-04-07T14:38:00Z",
      "health_gate_passed_at": "2026-04-07T14:43:00Z",
      "status": "healthy"
    }
  ],
  "total_duration_minutes": 20,
  "outcome": "success"
}
```

---

## Cross-References

| Topic | Document |
|-------|---------|
| Progressive delivery patterns (canary, blue/green) | [Progressive Delivery](progressive-delivery.md) |
| GitOps architecture | [GitOps Architecture](gitops-architecture.md) |
| Release governance and approval workflows | [Framework](framework.md) |
| Cloud security for multi-region | [Cloud Security DevSecOps](../../cloud-security-devsecops/docs/multi-cloud-reference.md) |
| Compliance data residency | [Geographic Compliance Guide](../../compliance-automation-framework/docs/geographic-compliance.md) |
