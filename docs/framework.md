# Release Orchestration Framework

## Table of Contents

- [Release Governance Model](#release-governance-model)
- [Environment Promotion Strategy](#environment-promotion-strategy)
- [Approval Workflows](#approval-workflows)
- [Deployment Orchestration Across Heterogeneous Systems](#deployment-orchestration-across-heterogeneous-systems)
- [Rollback Strategies](#rollback-strategies)
- [Change Management Integration](#change-management-integration)
- [Risk-Based Deployment Strategy](#risk-based-deployment-strategy)
- [Blue/Green and Canary Orchestration](#bluegreen-and-canary-orchestration)
- [Release Audit Trails and Compliance Evidence](#release-audit-trails-and-compliance-evidence)
- [Release Scheduling and Change Windows](#release-scheduling-and-change-windows)
- [Multi-Team Coordination Model](#multi-team-coordination-model)

---

## Release Governance Model

Release governance defines who has authority over deployment decisions, what processes must be followed, and how accountability is distributed across the organization. A clear governance model prevents both the failure mode of excessive centralization (where a single release team becomes a bottleneck) and the failure mode of excessive decentralization (where there is no coordination and production stability suffers).

### Governance Principles

1. **Subsidiarity** — decisions should be made at the lowest level with sufficient authority and context. Low-risk changes to non-critical services should not require senior approval.
2. **Proportionality** — governance overhead should be proportional to the risk of the change. A configuration change to a development service warrants far less process than a database schema migration on a business-critical service.
3. **Automation first** — governance controls should be automated wherever possible. Human review should be reserved for decisions that require judgment, not mechanical compliance checking.
4. **Immutable accountability** — every deployment decision must be traceable to an authorized actor, with evidence preserved in the immutable audit trail.

### Roles and Responsibilities

| Role | Authority | Responsibilities |
|---|---|---|
| **Release Manager** | Authorize normal and high-risk production deployments | Release scheduling, cross-team coordination, CAB representation, incident coordination during releases |
| **Release Train Engineer (RTE)** | Authorize release train deployments | PI planning coordination, dependency management, release train schedule management |
| **Service Owner / Tech Lead** | Authorize standard changes and staging promotions for their service | Define service-level deployment policies, approve staging promotions |
| **Platform / DevOps Engineer** | Execute deployments; no unilateral production authorization | Configure orchestration tooling, maintain deployment pipelines |
| **Change Advisory Board (CAB)** | Authorize high-risk or high-impact production changes | Risk assessment, condition setting, post-implementation review |
| **Emergency Change Authority (ECA)** | Authorize emergency changes outside normal windows | On-call escalation, emergency approval with post-facto review |

### Governance Decision Matrix

| Change Type | Development | Test | Staging | Production |
|---|---|---|---|---|
| Patch release (bug fix, security patch) | Auto-approve | Auto-approve (CI gates) | Tech Lead | Release Manager |
| Minor feature release | Auto-approve | Auto-approve (CI gates) | Tech Lead | Release Manager |
| Major version / breaking change | Auto-approve | Auto-approve (CI gates) | Release Manager | Release Manager + CAB |
| Database schema migration | Auto-approve | Tech Lead | Release Manager | Release Manager + CAB |
| Infrastructure change | Auto-approve | Tech Lead | Release Manager | CAB required |
| Emergency fix | Auto-approve | Auto-approve | Auto-approve | ECA |

---

## Environment Promotion Strategy

Environment promotion is the systematic movement of validated artifacts through a defined sequence of environments, with each environment providing higher fidelity validation before the artifact reaches production.

### Promotion Lifecycle

**Development (dev)**

Every commit to a main or integration branch triggers an automatic deployment to the development environment. No human approval is required. The development environment exists to give developers rapid feedback on integration and to provide a persistent target for exploratory testing.

- Artifacts deployed: Every CI-successful build
- Promotion criteria: CI pipeline passes (build + unit tests + static analysis + security scan)
- Deployment strategy: Rolling (fastest feedback)
- Data: Synthetic, seeded from fixtures

**Test / QA**

Automated promotion from dev occurs when the dev deployment is stable for a configurable period (typically 15–30 minutes) and automated post-deployment tests pass. Test is the first environment where functional quality is formally validated.

- Artifacts deployed: Artifacts that passed dev validation
- Promotion criteria: Functional test suite pass, integration test suite pass, no regression from prior test results
- Deployment strategy: Rolling
- Data: Anonymized production data snapshot, refreshed weekly

**Staging**

Staging is the pre-production gate and the most critical environment in the promotion chain. It is configured to be as close to production as operationally feasible, providing the highest-confidence validation that a change is safe for production.

- Artifacts deployed: Artifacts approved for staging by service owner or tech lead
- Promotion criteria: Full regression suite pass, performance benchmark within 10% of baseline, security scan clear, manual exploratory testing sign-off for significant changes
- Deployment strategy: Blue/green (to enable rapid rollback)
- Data: Production mirror (anonymized), refreshed nightly
- Duration: Minimum 24 hours in staging before production promotion (configurable by service tier)

**Production**

Production promotion requires explicit authorization from the release manager and, for high-risk changes, CAB approval. Deployments are subject to change window enforcement.

- Artifacts deployed: Artifacts approved by release manager (and CAB where required)
- Promotion criteria: Staging soak complete, change window active, no competing deployments in flight, pre-deployment checklist complete
- Deployment strategy: Canary → progressive rollout → full deployment (configurable)
- Data: Live production
- Monitoring: Enhanced monitoring active for 2 hours post-deployment

### Promotion Gate Checklist

Each promotion requires a promotion record in the release orchestration system containing:

```
Promotion Record
────────────────────────────────────────
Service:         payment-service
Artifact:        sha256:a3f8c7d9...
From:            staging
To:              production
Requested by:    jane.smith@example.com
Requested at:    2024-03-15T21:30:00Z

Quality Evidence
─ CI run:        PASSED (https://ci/runs/12345)
─ Test coverage: 88.2% (threshold: 80%)
─ Security scan: PASSED (0 critical, 2 high — accepted)
─ Performance:   PASSED (p99 latency: 142ms, baseline: 138ms)
─ Staging soak:  PASSED (48h, zero incidents)

Authorization
─ Approver:      release-manager@example.com
─ Approved at:   2024-03-15T21:45:00Z
─ Change record: CHG0012345
─ Change window: 2024-03-19 22:00 EST (Tuesday window)
```

---

## Approval Workflows

### Automated Gate Definitions

Automated gates are non-negotiable quality checks executed by the orchestration system without human involvement. A gate failure blocks promotion regardless of any manual approval.

| Gate | Description | Failure action |
|---|---|---|
| **CI status** | CI pipeline must have completed successfully for the exact artifact digest | Block promotion |
| **Test coverage** | Code coverage must meet or exceed the configured threshold (default: 80%) | Block promotion |
| **Critical vulnerability** | No unfixed critical-severity vulnerabilities in the artifact | Block promotion |
| **High vulnerability count** | High-severity vulnerabilities must not exceed the configured maximum | Block promotion |
| **Staging soak** | Artifact must have been deployed to staging for the minimum soak period | Block promotion |
| **No active incidents** | No active P1/P2 incidents on the target service | Block until incident resolved |
| **Change window** | Current time must fall within an active change window (production only) | Queue for next window |

### Manual Approval Workflow

When manual approval is required, the workflow follows this sequence:

1. **Notification dispatch** — approvers are notified via configured channels (Slack, Teams, email) with a deep link to the approval interface showing all relevant quality evidence.
2. **Approval interface** — approvers review the deployment package including artifact details, quality gate results, change record details, and deployment plan.
3. **Decision and justification** — approvers record their decision (approve/reject) and mandatory justification text.
4. **Quorum evaluation** — the workflow engine checks whether the required approver quorum has been met.
5. **Conditional approval** — approvers may attach conditions (e.g., "notify on-call team before deploying," "deploy during low-traffic window only").
6. **Timeout and escalation** — if no decision is received within the configured timeout, the request is escalated to the next approval authority level.

### CAB Integration Workflow

For changes requiring Change Advisory Board review:

1. Release orchestration system creates a change request in the ITSM platform (ServiceNow, Jira SM) with full deployment context.
2. The change is automatically added to the next scheduled CAB meeting agenda.
3. CAB members review the change in the ITSM platform's CAB workbench.
4. CAB approval or rejection is synchronized back to the release orchestration system via webhook.
5. If approved, the change is cleared for deployment during the authorized change window.
6. If rejected, the release is blocked and the requesting team receives rejection reason and remediation guidance.

---

## Deployment Orchestration Across Heterogeneous Systems

Enterprise environments are rarely homogeneous. A single release may require coordinated changes across Kubernetes services, virtual machines, serverless functions, managed databases, and third-party SaaS platforms.

### Deployment Target Abstraction

The orchestration engine interacts with deployment targets through a plugin/adapter model, with each target type implementing a common interface:

| Operation | Description |
|---|---|
| `deploy(artifact, config, strategy)` | Deploy the artifact to the target using the specified strategy |
| `status()` | Return current deployment status and health |
| `rollback(version)` | Roll back to the specified previous version |
| `verify(checks)` | Execute post-deployment verification checks |

### Supported Deployment Targets

**Kubernetes / OpenShift**
- Deployment via Helm chart updates, Kustomize overlays, or raw manifest application
- Native support for rolling, blue/green, and canary deployment strategies via Argo Rollouts or Flagger
- Health evaluation via Kubernetes health checks and custom metrics

**Virtual Machines (AWS EC2, Azure VMs, GCP Compute)**
- Deployment via cloud-native deployment services (AWS CodeDeploy, Azure VMSS rolling updates)
- Or via Ansible playbooks executed by the orchestration engine
- Health evaluation via load balancer health checks and application health endpoints

**Serverless (AWS Lambda, Azure Functions, GCP Cloud Functions)**
- Deployment via cloud SDK or Terraform
- Traffic shifting via Lambda aliases and weighted routing
- Health evaluation via invocation error rate monitoring

**Databases**
- Schema migration via Flyway or Liquibase, orchestrated as a deployment step
- Backward-compatible migration enforced before application deployment
- Rollback via migration undo scripts (where supported)

**Third-Party SaaS**
- Configuration deployment via platform API (e.g., LaunchDarkly feature flag configuration, Stripe webhook configuration)
- Verification via API health checks and integration tests

### Cross-System Deployment Sequencing

When a release requires changes across multiple systems, the orchestration engine manages the deployment sequence to ensure consistency:

```yaml
# deployment-plan.yaml
release_id: rel_checkout-platform_v3.0.0_20240315
deployment_sequence:
  - step: 1
    name: database-migration
    target: postgres-checkout-prod
    action: run-migration
    artifact: checkout-migrations:v3.0.0
    wait_for_completion: true
    rollback_on_failure: true

  - step: 2
    name: deploy-api-service
    target: k8s-prod-us-east
    action: helm-upgrade
    artifact: checkout-api:sha256:a3f8...
    strategy: canary
    canary_steps: [5, 25, 50, 100]
    analysis_interval: 10m
    depends_on: [database-migration]

  - step: 3
    name: deploy-frontend
    target: cdn-cloudfront
    action: invalidate-and-update
    artifact: checkout-frontend:sha256:b9d2...
    depends_on: [deploy-api-service]
    wait_for: canary_step_50_complete

  - step: 4
    name: update-feature-flags
    target: launchdarkly
    action: flag-rollout
    config:
      flag_key: checkout-v3-experience
      rollout_percentage: 100
    depends_on: [deploy-frontend]
```

---

## Rollback Strategies

### Automated Rollback Triggers

The orchestration system supports automated rollback based on configurable signal thresholds. When a trigger condition is met, rollback is initiated without human intervention.

| Trigger | Condition | Rollback scope |
|---|---|---|
| **Error rate spike** | HTTP 5xx rate exceeds baseline × 3 for 5 minutes | Service-level rollback |
| **Latency degradation** | p99 latency exceeds SLO threshold for 10 minutes | Service-level rollback |
| **Crash loop** | Container restart count exceeds 5 in 10 minutes | Pod/deployment rollback |
| **Health check failure** | Readiness/liveness probe failures exceed 20% of instances | Service-level rollback |
| **Synthetic monitor failure** | Critical user journey synthetic monitor fails 3 consecutive runs | Service-level rollback |
| **Canary analysis failure** | Kayenta/Argo analysis step determines negative outcome | Canary abort + rollback |

### RTO and RPO Targets

| Service Tier | Recovery Time Objective (RTO) | Recovery Point Objective (RPO) |
|---|---|---|
| **Platinum (mission critical)** | 5 minutes | 0 (stateless) / 1 minute (stateful) |
| **Gold (business critical)** | 15 minutes | 0 (stateless) / 5 minutes (stateful) |
| **Silver (business important)** | 30 minutes | 0 (stateless) / 15 minutes (stateful) |
| **Bronze (non-critical)** | 4 hours | N/A |

### Rollback Execution Model

**Stateless services (Kubernetes, serverless):** Rollback is implemented as a rapid re-deployment of the previous artifact. For Kubernetes deployments with Argo Rollouts, rollback is a single API call that immediately routes traffic back to the previous stable ReplicaSet. Target RTO: under 2 minutes.

**Stateful services with schema changes:** Database schema rollbacks require careful orchestration. Backward-compatible migrations (expand/contract pattern) allow application rollback without schema rollback. Where schema rollback is required, pre-staged undo scripts are executed by the orchestration engine.

**Multi-service rollbacks:** When a cross-service deployment is rolled back, the orchestration engine reverses the deployment sequence — rolling back in reverse order of deployment to maintain system consistency.

### Rollback Decision Framework

```
Automated rollback: triggered when health signals breach thresholds
│
├── Can service be rolled back independently?
│   ├── Yes → Execute service rollback immediately
│   └── No (cross-service dependency) → Alert release manager + escalate
│
├── Were database migrations applied?
│   ├── No → Standard artifact rollback
│   └── Yes → Assess migration type:
│       ├── Expand-only (backward compatible) → Rollback application only
│       └── Non-backward compatible → Emergency CAB + assess impact
│
└── Was this a canary deployment?
    ├── Yes → Abort canary, route 100% to stable version
    └── No → Re-deploy previous version
```

---

## Change Management Integration

### ServiceNow Integration

The release orchestration system integrates with ServiceNow through the ServiceNow REST API and webhook subscriptions.

**Automatic change record creation:** When a deployment to staging or production is requested, the orchestration system creates a change request in ServiceNow with pre-populated fields drawn from the artifact metadata, service catalog, and deployment plan.

**Change record fields populated automatically:**

| ServiceNow Field | Source |
|---|---|
| Short description | Service name + version + environment |
| Description | Auto-generated from release notes + deployment plan |
| Configuration item | Service registry lookup |
| Planned start/end date | Change window assignment |
| Assignment group | Service owner group from CMDB |
| Change type | Risk classifier output |
| Test plan | Link to CI test results |
| Implementation plan | Deployment plan (auto-generated) |
| Backout plan | Rollback plan (auto-generated) |

**Approval synchronization:** CAB approvals recorded in ServiceNow are synchronized to the orchestration system via webhook, allowing deployments to proceed without polling.

**Post-deployment evidence:** Upon deployment completion, the orchestration system updates the ServiceNow change record with implementation evidence including deployment logs, post-deployment test results, and final health status.

### Jira Service Management Integration

For organizations using Jira Service Management (JSM) for ITSM, integration follows the same pattern using the JSM REST API:

- Change request creation via JSM API with Jira issue type "Change"
- Approval workflow driven by JSM approval screens
- Webhook-based status synchronization to orchestration system
- Deployment evidence attached as Jira issue attachments and linked test execution records

---

## Risk-Based Deployment Strategy

Not all deployments carry equal risk. The risk-based deployment strategy framework selects the appropriate deployment approach based on a calculated risk score.

### Risk Score Calculation

```
Risk Score = (Service Criticality × 0.3) + (Change Impact × 0.3) + (Historical Stability × 0.2) + (Dependency Complexity × 0.2)
```

| Factor | Score Range | Low (1-3) | Medium (4-6) | High (7-10) |
|---|---|---|---|---|
| **Service Criticality** | 1-10 | Internal tool | Business-facing | Revenue/safety critical |
| **Change Impact** | 1-10 | Config, patch | Minor feature | Schema, major version, API contract |
| **Historical Stability** | 1-10 | Zero incidents in 90d | 1-2 incidents | 3+ incidents |
| **Dependency Complexity** | 1-10 | Isolated service | A few upstream/downstream | Complex dependency graph |

### Deployment Strategy Selection

| Risk Score | Deployment Strategy | Rollback Capability | Human Approval |
|---|---|---|---|
| 1–30 (Low) | Rolling update | Pod recreation (~2 min) | Automated gate only |
| 31–55 (Medium) | Blue/green | Instant traffic switch (<1 min) | Tech lead or release manager |
| 56–75 (High) | Canary (5% → 25% → 100%) | Abort canary + rollback (<1 min) | Release manager |
| 76–100 (Very High) | Canary with extended analysis window | Abort canary + rollback (<1 min) | Release manager + CAB |

---

## Blue/Green and Canary Orchestration

### Blue/Green Orchestration

In a blue/green deployment, two identical production environments run in parallel. The "green" environment receives the new version while the "blue" environment continues serving production traffic. Once validation is complete, traffic is switched.

**Orchestration steps:**
1. Deploy new artifact to idle slot (green)
2. Execute smoke tests and integration checks against green slot
3. Compare green health metrics to blue baseline
4. If validation passes: switch load balancer to route 100% of traffic to green
5. Monitor blue slot for configurable period (default: 30 minutes)
6. If rollback needed: switch load balancer back to blue (< 30 seconds)
7. On final confirmation: decommission old blue slot or hold as warm standby

**Advantages:** Zero-downtime deployment, instant rollback capability, full production validation before traffic switch.

**Trade-offs:** Requires 2× resource capacity during deployment; database schema changes must be backward-compatible with both versions running.

### Canary Orchestration

Canary deployments route a small percentage of production traffic to the new version while the majority remains on the stable version. Traffic is progressively shifted as analysis confirms the new version is healthy.

**Orchestration steps:**
1. Deploy new artifact alongside stable version
2. Route configurable percentage (e.g., 5%) of traffic to canary
3. Execute automated analysis for configurable period (default: 10 minutes)
4. If analysis passes: increase traffic to next step (e.g., 25%)
5. Continue stepping until 100% or abort if analysis fails
6. On abort: route all traffic back to stable version

**Analysis metrics (Kayenta / Argo Rollouts):**

| Metric | Comparison method | Failure threshold |
|---|---|---|
| HTTP 5xx rate | Mann-Whitney U test vs. baseline | P-value < 0.05 |
| p99 latency | Percentage increase vs. baseline | > 20% increase |
| Business transaction success rate | Percentage comparison | > 5% decrease |
| Custom business metrics | Configurable per service | Service-defined |

---

## Release Audit Trails and Compliance Evidence

Every action taken in the release orchestration system generates an immutable audit event. The audit trail serves both operational (incident reconstruction) and compliance (audit evidence) purposes.

### Audit Event Categories

| Category | Events captured |
|---|---|
| **Artifact registration** | Artifact published, quality gate results, metadata recorded |
| **Promotion requests** | Request submitted, requestor identity, target environment, artifact details |
| **Approval decisions** | Approver identity, timestamp, justification, decision outcome |
| **Policy evaluations** | Policy rules evaluated, pass/fail per rule, contextual factors |
| **Deployments** | Deployment started, execution steps, completion status, deployment duration |
| **Verification** | Post-deployment check type, results, pass/fail |
| **Rollbacks** | Rollback trigger, trigger type (automated/manual), execution steps, completion |
| **Window enforcement** | Deployment allowed/denied by change window, exception requests |
| **ITSM actions** | Change record creation, updates, approval sync, closure |

### Compliance Evidence Package

On demand (or automatically at deployment completion), the orchestration system generates a compliance evidence package containing:

- Deployment summary (who, what, where, when)
- Approval chain with digital signatures
- Quality gate results with raw data references
- Change record from ITSM platform
- Deployment execution log
- Post-deployment verification results
- Rollback capability evidence (rollback plan, last tested rollback)

---

## Release Scheduling and Change Windows

### Change Window Management

Change windows define when production deployments are authorized. The orchestration engine enforces change windows as a hard control — deployments initiated outside authorized windows are blocked, not just warned.

**Window categories:**

- **Standard windows:** Recurring weekly or daily periods for routine deployments
- **Special windows:** One-time windows for specific releases (e.g., major platform upgrades)
- **Emergency windows:** Any-time authorization for emergency changes (P1 incident remediation, critical security patches)
- **Blackout periods:** Periods during which all non-emergency deployments are prohibited

### Release Calendar

The release calendar provides a unified view of:
- All scheduled deployments by environment
- Change window availability
- Blackout periods
- Cross-team dependencies for coordinated releases
- ITSM change records linked to scheduled deployments

---

## Multi-Team Coordination Model

### Release Train Model

When multiple teams must coordinate a joint release, the release train model provides a structured coordination mechanism:

- A designated Release Train Engineer (RTE) manages the overall release
- Individual teams commit features and services to the train by a specified cutoff date
- Late features are held for the next train; the train departs on schedule
- The RTE coordinates cross-team dependencies and manages the release plan

### Cross-Service Dependency Management

The orchestration system maintains a dependency graph for all managed services. Before executing a deployment, the engine checks:

1. **Upstream dependencies** — are all upstream services that this service depends on at compatible versions?
2. **Downstream consumers** — will this deployment break any downstream services?
3. **Coordinated release requirements** — is this service flagged as requiring coordinated deployment with other services?

Dependency conflicts generate blocking warnings that must be acknowledged by the release manager before deployment proceeds.

### Release Communication Model

| Audience | Communication | Timing | Channel |
|---|---|---|---|
| Development teams | Deployment scheduled / completed | Real-time | Slack #deployments |
| On-call engineers | Deployment starting in production | 30 min before | PagerDuty + Slack |
| Operations team | All production deployments | Real-time | Ops dashboard + Slack |
| Business stakeholders | Significant releases | Pre-release + post-completion | Email |
| Customer support | Customer-visible changes | Pre-release | Internal wiki + email |
| Customers | Customer-visible changes | Release notes | Product blog + email |

---

## Database Migration Safety

Database schema changes are among the highest-risk operations in the release lifecycle. Unlike application code, schema changes are often irreversible within the rollback window, affect shared infrastructure, and can cause downtime or data corruption if sequenced incorrectly. This section defines the governance model, sequencing patterns, and automated gates required to release schema migrations safely alongside application deployments.

### Why Database Migrations Fail in Production

The most common failure modes for schema migrations at release time:

| Failure Mode | Description |
|---|---|
| Long-table lock | An `ALTER TABLE` on a large table acquires an exclusive lock, causing cascading timeouts and downtime |
| Code/schema version mismatch | Application code referencing a column that does not yet exist (forward deploy) or no longer exists (failed rollback) |
| Missing rollback path | Destructive migration (column drop, table rename) deployed without a verified reverse migration |
| Data truncation | Reducing a column's width or changing its type silently truncates or rejects existing data |
| Index build blocking writes | Foreground index creation blocks writes on the target table during construction |
| Constraint violation on existing data | Adding a NOT NULL constraint or foreign key to a column that contains violating rows fails mid-migration |

### The Expand-Contract Pattern

The expand-contract (also called parallel-change) pattern is the foundational technique for zero-downtime schema migrations when both old and new application versions must coexist during a deployment window.

**Three-phase sequence:**

```
Phase 1: EXPAND
  - Add the new column, table, or constraint (additive only)
  - New application code writes to both old and new structures
  - Old application code continues to function — no breaking change
  - Deploy application version N+1

Phase 2: MIGRATE
  - Backfill data from old to new structure in batches
  - Run concurrently with production traffic; batch size tuned to avoid lock contention
  - Validate row counts and data integrity after each batch

Phase 3: CONTRACT
  - Remove the old column, table, or constraint (destructive step)
  - Only permitted after application version N+1 is verified stable in production
  - Deploy as a separate release from Phase 1 — minimum 24-hour separation recommended
```

**Prohibited patterns (require waiver and compensating controls):**

- Combined expand + contract in a single release
- `ALTER TABLE` with `ADD COLUMN NOT NULL` without a default value on tables with > 100k rows
- `DROP COLUMN` in the same release as the application code stop referencing it
- `RENAME TABLE` or `RENAME COLUMN` without aliasing during transition

### Migration Classification and Gates

Each migration must be classified before it enters the release pipeline:

| Class | Definition | Automated Gate | Approval Requirement |
|---|---|---|---|
| **Class A — Additive** | Adds columns, tables, indexes (non-blocking) | Schema linter pass | Standard release approval |
| **Class B — Transformative** | Backfills, constraint changes, type changes | Schema linter + backfill plan review | Senior engineer sign-off |
| **Class C — Destructive** | Drops columns, tables; renames | Schema linter + expand phase verified complete | Release manager + DBA sign-off |
| **Class D — Locking** | Long-running `ALTER TABLE` estimated > 30s on target table | Must be converted to Class A using online DDL | Architecture review |

**Schema linter checks (automated, required for all migrations):**

```sql
-- Example: automated linter checks in CI
-- 1. Detect ADD COLUMN NOT NULL without default on non-empty tables
SELECT COUNT(*) FROM information_schema.columns
WHERE table_schema = :target_schema
  AND column_name = :new_column
  AND is_nullable = 'NO'
  AND column_default IS NULL;

-- 2. Detect foreground index creation (must use CREATE INDEX CONCURRENTLY)
-- Migration file text scanned for: CREATE INDEX (not CONCURRENTLY)
-- Result: blocking error in CI

-- 3. Detect DROP TABLE or DROP COLUMN in same migration as application deploy
-- Cross-referenced with git diff of application code in same PR
```

### Migration Sequencing in the Release Pipeline

Database migrations must be sequenced explicitly within the deployment pipeline:

```
[CI: Migration Lint] → [Review Gate] → [Deploy to Staging]
        ↓
[Run Migration in Staging] → [Automated Integrity Validation]
        ↓
[Production Deploy Gate] ──────────────────────────────────┐
        ↓                                                   │
[Pre-deploy: Backup verification]                           │
        ↓                                                   │
[Run Migration in Production (Phase 1 or 2)]               │
        ↓                                                   │
[Health check: application + DB connection pool]            │
        ↓                                                   │
[Deploy application code]                                   │
        ↓                                                   │
[Post-deploy validation: row counts, query latency P95]    ←┘
```

**Ordering rule:** Migration runs before application code deployment in forward direction. For rollback, application code reverts first, migration rollback second (unless the migration is additive and backwards-compatible).

### Rollback Strategy for Migrations

| Migration Class | Rollback Approach | Rollback Time |
|---|---|---|
| Class A — Additive | Application rollback only; leave schema change in place (backwards-compatible) | Minutes |
| Class B — Transformative | Reverse migration script required; must be tested in staging | 10–60 min depending on row count |
| Class C — Destructive | **No rollback possible after contract phase.** Rollback window closes permanently when Phase 3 is applied | N/A |
| Class D — Locking | Blocked from reaching production; must be reclassified | N/A |

Class C migrations must be marked with an explicit `POINT_OF_NO_RETURN` annotation in the migration file and require a separate deployment with a minimum 24-hour stabilization window after the expand phase.

### Automated Validation Checks

Run the following checks after every migration in staging and production:

```bash
#!/bin/bash
# post-migration-validation.sh

# 1. Verify row counts are within expected range (no silent data loss)
EXPECTED_COUNT=$(cat migration-manifest.json | jq '.expected_row_count')
ACTUAL_COUNT=$(psql "$DB_URL" -t -c "SELECT COUNT(*) FROM $TARGET_TABLE;")
if [ "$ACTUAL_COUNT" -lt "$((EXPECTED_COUNT * 95 / 100))" ]; then
  echo "ERROR: Row count below 95% of expected. Possible data loss." && exit 1
fi

# 2. Verify no long-running transactions held open during migration
psql "$DB_URL" -c "
  SELECT pid, now() - pg_stat_activity.query_start AS duration, query
  FROM pg_stat_activity
  WHERE (now() - pg_stat_activity.query_start) > interval '30 seconds'
    AND state = 'active';
"

# 3. Verify connection pool health
curl -sf "$APP_HEALTH_ENDPOINT" | jq '.db.pool_available > 0' || exit 1

# 4. Check query latency on migrated table (P95 < baseline + 20%)
psql "$DB_URL" -c "
  SELECT mean_exec_time, calls
  FROM pg_stat_statements
  WHERE query LIKE '%$TARGET_TABLE%'
  ORDER BY mean_exec_time DESC LIMIT 10;
"
```

### Tooling Guidance

| Tool | Use Case | Notes |
|------|----------|-------|
| **Flyway** | SQL-based migration versioning | Strong baseline for teams using raw SQL; integrates with most CI systems |
| **Liquibase** | XML/YAML/SQL migrations with rollback support | Better rollback tooling than Flyway; enterprise license for some advanced features |
| **gh-ost** (GitHub Online Schema Change) | Online DDL for MySQL large tables | Eliminates locking on ALTER TABLE for InnoDB; production-proven at scale |
| **pgroll** | Zero-downtime PostgreSQL migrations with expand-contract automation | Native multi-version schema support; recommended for teams on PostgreSQL |
| **Atlas** | Schema-as-code with automated drift detection | Useful for IaC-style schema management; integrates with Terraform |

### Compliance Considerations

For systems subject to SOX, PCI-DSS, or SOC 2:

- All schema changes must have an associated change record in the ITSM system (linked to the migration file commit SHA)
- Evidence of pre-production testing (migration execution log from staging) must be attached to the change record
- Class C (destructive) migrations require a second approver who is independent of the engineering team that authored the migration
- Migration execution logs must be retained for the compliance evidence retention period (typically 7 years for SOX)
