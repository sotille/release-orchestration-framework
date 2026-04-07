# Progressive Delivery in the Release Orchestration Framework

Progressive delivery is a set of deployment techniques that reduce the blast radius of changes by controlling the scope of exposure at rollout time. Rather than deploying to all production traffic simultaneously, progressive delivery strategies route a controlled percentage of traffic to the new version, monitor for regressions, and either automate promotion or trigger rollback before the full user base is affected.

This guide covers canary deployments, blue/green deployments, feature flag-driven releases, and shadow traffic testing — including implementation patterns, automation integration, and the governance model for progressive delivery within the Techstream Release Orchestration Framework.

---

## Table of Contents

- [Why Progressive Delivery](#why-progressive-delivery)
- [Deployment Strategy Comparison](#deployment-strategy-comparison)
- [Canary Deployments](#canary-deployments)
- [Blue/Green Deployments](#bluegreen-deployments)
- [Feature Flag-Driven Releases](#feature-flag-driven-releases)
- [Shadow Traffic Testing](#shadow-traffic-testing)
- [Progressive Delivery Automation](#progressive-delivery-automation)
- [Rollback Orchestration](#rollback-orchestration)
- [Governance and Approval Integration](#governance-and-approval-integration)
- [Observability Requirements](#observability-requirements)
- [Tool Reference](#tool-reference)

---

## Why Progressive Delivery

The DORA research program consistently identifies that elite engineering organizations deploy more frequently, recover from failures faster, and have lower change failure rates than lower-performing peers. These outcomes are not contradictory — they are causally linked. Progressive delivery is a key mechanism that enables high deployment frequency while keeping change failure rates low.

The traditional trade-off between deployment speed and deployment risk is a false dilemma. It persists because organizations conflate "deployment" with "full exposure." When deployment means exposing 100% of traffic to a change simultaneously, caution is rational. When deployment means exposing 1% of traffic first, with automated promotion gated on success metrics, speed and safety align.

**Business case for progressive delivery:**

| Without Progressive Delivery | With Progressive Delivery |
|---|---|
| Incidents affect 100% of users | Incidents affect only the canary population |
| Mean Time to Detect (MTTD): hours to days | MTTD: minutes (automated metric checks) |
| Mean Time to Recover (MTTR): hours (rollback, redeploy) | MTTR: < 5 minutes (instant traffic shift) |
| Deployment frequency limited by risk tolerance | Deployment frequency decoupled from blast radius |
| Feature releases require large synchronization efforts | Features can be deployed dark and enabled independently |

---

## Deployment Strategy Comparison

| Strategy | Traffic Split | Infrastructure Cost | Rollback Speed | Best For |
|---|---|---|---|---|
| **Canary** | Percentage-based (1% → 100%) | Low (shared infrastructure with routing layer) | Seconds (routing rule update) | Stateless services, API-heavy workloads |
| **Blue/Green** | Full switch at promotion | Medium (two full environments) | Seconds (DNS/load balancer flip) | Stateful services, batch jobs, database migrations |
| **Feature flags** | User/segment-based | Low (single deployment, conditional logic) | Milliseconds (flag evaluation) | Feature graduation, A/B testing, kill switches |
| **Shadow** | Mirrored (no user impact) | Medium (processing all traffic twice) | N/A (no user impact) | Pre-production validation, algorithm replacement |
| **Rolling update** | Gradual instance replacement | Low (no additional infrastructure) | Minutes (wait for rollback rollout) | Stateless services without traffic control requirements |

---

## Canary Deployments

A canary deployment routes a small fraction of production traffic to the new version while the majority continues to use the stable version. Automated analysis of observability data determines whether the canary version is healthy enough to expand to higher traffic percentages.

### Canary Architecture

```
                    ┌─────────────────────────────────┐
                    │         Load Balancer /          │
                    │       Ingress Controller         │
                    └──────┬──────────────────┬────────┘
                           │                  │
                    5% weight           95% weight
                           │                  │
                    ┌──────▼──────┐    ┌──────▼──────┐
                    │  Canary     │    │   Stable    │
                    │  v2.1.0     │    │   v2.0.3    │
                    │  (new)      │    │  (current)  │
                    └──────┬──────┘    └──────┬──────┘
                           │                  │
                    ┌──────▼──────────────────▼──────┐
                    │   Observability Stack           │
                    │   (Prometheus, Grafana, APM)    │
                    └─────────────────────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │   Canary Analysis Controller  │
                    │   (Argo Rollouts / Flagger)   │
                    └───────────────────────────────┘
```

### Canary Progression Stages

A standard canary progression follows a weighted traffic increase with automated analysis gates between stages:

| Stage | Canary Traffic Weight | Analysis Window | Success Criteria |
|---|---|---|---|
| Initial canary | 5% | 10 minutes | Error rate < baseline + 0.5%; P99 latency < baseline × 1.2 |
| Expansion | 25% | 15 minutes | Same criteria at higher traffic volume |
| Majority | 50% | 20 minutes | Same criteria; compare error absolute counts, not just rates |
| Promotion | 100% | Continuous | Stable version decommissioned after 30 minutes of clean 100% canary |

**Automated analysis criteria (Argo Rollouts / Flagger compatible):**

```yaml
# Argo Rollouts — AnalysisTemplate for HTTP error rate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: http-error-rate
spec:
  metrics:
  - name: error-rate
    interval: 60s
    successCondition: result[0] < 0.005   # Less than 0.5% error rate
    failureLimit: 3                        # Fail after 3 consecutive failures
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(http_requests_total{
            job="{{args.service-name}}",
            status=~"5.."
          }[2m]))
          /
          sum(rate(http_requests_total{
            job="{{args.service-name}}"
          }[2m]))
  - name: latency-p99
    interval: 60s
    successCondition: result[0] < 0.500   # P99 latency under 500ms
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{
              job="{{args.service-name}}"
            }[2m])) by (le)
          )
```

### Canary Rollout Configuration (Argo Rollouts)

```yaml
# Argo Rollouts — Rollout resource with canary strategy
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 20
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: registry.example.com/payment-service:2.1.0
  strategy:
    canary:
      analysis:
        templates:
        - templateName: http-error-rate
        args:
        - name: service-name
          value: payment-service
      steps:
      - setWeight: 5
      - pause: {duration: 10m}
      - setWeight: 25
      - pause: {duration: 15m}
      - setWeight: 50
      - pause: {duration: 20m}
      - setWeight: 100
      canaryService: payment-service-canary
      stableService: payment-service-stable
      trafficRouting:
        istio:
          virtualService:
            name: payment-service-vsvc
            routes:
            - primary
```

---

## Blue/Green Deployments

Blue/green deployments maintain two identical production environments — the currently live environment (blue) and the new version environment (green). Traffic switches from blue to green atomically once the green environment passes validation.

### Blue/Green Architecture

```
         ┌─────────────────────────────────────────────┐
         │           Load Balancer / DNS               │
         │        (Routes 100% to one slot)            │
         └───────────────────┬─────────────────────────┘
                             │
              Currently: pointing to Blue
              After promotion: pointing to Green
                             │
         ┌───────────────────┴─────────────────────────┐
         │                                             │
  ┌──────▼──────┐                             ┌────────▼────┐
  │  Blue Slot  │                             │ Green Slot  │
  │  v2.0.3     │                             │  v2.1.0     │
  │  (current)  │                             │  (staging)  │
  │  LIVE       │                             │  WARMING UP │
  └─────────────┘                             └─────────────┘
```

### When to Choose Blue/Green Over Canary

Blue/green is preferred when:

1. **The service manages stateful sessions** that cannot be easily split across two versions.
2. **Database schema migrations** are involved that are not backwards-compatible. The green environment is fully validated with the new schema before any traffic is shifted.
3. **Validation requires the complete production environment** (not a traffic sample), such as full integration test suites against live dependencies.
4. **Regulatory requirements** mandate a clear point of promotion with pre-promotion validation evidence.
5. **Rollback semantics matter** — blue/green provides an immediately available rollback target (the still-running blue environment) without requiring a fresh deployment.

### Database Migration Pattern for Blue/Green

Database schema migrations require the expand/contract pattern to be compatible with blue/green deployments:

```
Phase 1 (Expand):   Add new schema elements; both old and new code work
  - Blue v2.0.3:    Reads old columns; ignores new columns
  - Green v2.1.0:   Reads both old and new columns; writes both

Phase 2 (Contract): After full green promotion; remove old schema elements
  - This becomes a separate release, deployable independently
  - Validate that no code paths reference the removed elements before deployment
```

**This pattern is mandatory for zero-downtime blue/green database migrations.** Attempting to migrate schema and switch traffic simultaneously will cause failures when the old code version (blue) encounters the new schema.

### Blue/Green Promotion Gate

```yaml
# Example GitHub Actions job for blue/green promotion gate
name: Blue-Green Promotion Gate
on:
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment slot to promote
        required: true
        type: choice
        options: [green-staging, green-production]

jobs:
  promotion-gate:
    runs-on: ubuntu-latest
    environment: production-promotion  # Requires manual approval
    steps:
    - name: Run pre-promotion validation suite
      run: |
        # Integration tests against green slot
        ./scripts/run-integration-tests.sh --target=${{ inputs.environment }}

    - name: Verify SBOM and artifact signatures
      run: |
        cosign verify \
          --certificate-identity=https://github.com/${{ github.repository }}/.github/workflows/build.yml@refs/heads/main \
          --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
          registry.example.com/${{ env.SERVICE }}:${{ env.VERSION }}

    - name: Check compliance gate
      run: |
        # Verify OPA policies pass for this deployment
        opa eval --data policies/ \
          --input deployment-manifest.json \
          "data.release.allow"

    - name: Execute traffic switch
      if: success()
      run: |
        ./scripts/switch-traffic.sh --from=blue --to=green

    - name: Record promotion event
      if: success()
      run: |
        # Write immutable audit record
        ./scripts/record-deployment-event.sh \
          --action=blue-green-promote \
          --actor=${{ github.actor }} \
          --version=${{ env.VERSION }}
```

---

## Feature Flag-Driven Releases

Feature flags decouple deployment from release. Code is deployed to production in an inactive state (dark deployment) and enabled for specific users, segments, or environments through a flag management system.

### Feature Flag Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Application Code                      │
│                                                         │
│   if featureFlags.isEnabled("new-checkout-flow"):       │
│       renderNewCheckout()                               │
│   else:                                                 │
│       renderLegacyCheckout()                            │
└─────────────────────────────────┬───────────────────────┘
                                  │ SDK evaluates flag
                                  ▼
┌─────────────────────────────────────────────────────────┐
│              Feature Flag Management System             │
│    (LaunchDarkly / Unleash / OpenFeature provider)      │
│                                                         │
│  Flag: new-checkout-flow                                │
│  Targeting rules:                                       │
│    - Internal users: 100% enabled                       │
│    - Beta group: 25% enabled                            │
│    - All users: 0% enabled (default)                    │
└─────────────────────────────────────────────────────────┘
```

### OpenFeature Standard

Techstream recommends adopting the OpenFeature standard for feature flag evaluation. OpenFeature provides a vendor-neutral SDK that allows swapping flag management backends without application code changes.

```go
// OpenFeature SDK usage (Go)
import (
    "github.com/open-feature/go-sdk/pkg/openfeature"
)

client := openfeature.NewClient("payment-service")

// Evaluate a flag with context
newCheckoutEnabled, _ := client.BooleanValue(
    ctx,
    "new-checkout-flow",
    false, // default value
    openfeature.NewEvaluationContext(
        userID,
        map[string]interface{}{
            "plan":    user.Plan,
            "country": user.Country,
            "beta":    user.IsBetaEnrolled,
        },
    ),
)
```

### Flag Lifecycle Governance

Feature flags accumulate as technical debt when not managed. Implement a lifecycle policy:

| Flag State | Criteria | Action |
|---|---|---|
| **Experimental** | Newly created; less than 30 days old | Active development; no review required |
| **Rollout** | Actively ramping traffic | Weekly review with product/engineering owner |
| **Stable** | Enabled for ≥ 95% of users for 14+ days | Scheduled for removal (flag debt cleanup) |
| **Deprecated** | Scheduled for cleanup; owner notified | Remove from code within 30 days or flag escalation |
| **Removed** | Code and flag deleted | Archive flag in management system for audit trail |

Enforce flag lifecycle through CI checks:

```yaml
# CI job: fail if deprecated flags are still referenced in code
- name: Check for deprecated feature flags
  run: |
    DEPRECATED_FLAGS=$(curl -s https://flags.example.com/api/v1/flags?state=deprecated \
      | jq -r '.[].key')

    for flag in $DEPRECATED_FLAGS; do
      if grep -r "\"$flag\"" src/; then
        echo "ERROR: Deprecated flag '$flag' still referenced in code"
        exit 1
      fi
    done
```

---

## Shadow Traffic Testing

Shadow mode routes a copy of live production traffic to the new service version without the response being served to users. The new version's responses are captured and compared to the stable version's responses.

**Use cases:**

- Validating machine learning model replacements where correctness must be verified against real-world inputs
- Pre-production validation of stateless computation changes (pricing engines, recommendation algorithms)
- Performance benchmarking of new service versions under realistic production load

**Shadow testing tools:**

| Tool | Language | Use Case |
|---|---|---|
| Diffy (Twitter, OSS) | JVM | HTTP API response comparison |
| GoReplay | Go | TCP-level traffic replay and mirroring |
| Istio Traffic Mirroring | Platform | Service mesh-native request mirroring |
| Envoy Mirror | Platform | Proxy-level traffic duplication |

**Istio shadow configuration:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: recommendation-engine
spec:
  hosts:
  - recommendation-engine
  http:
  - route:
    - destination:
        host: recommendation-engine
        subset: stable
      weight: 100
    mirror:
      host: recommendation-engine
      subset: shadow-v2
    mirrorPercentage:
      value: 100.0  # Mirror all traffic; responses discarded from user path
```

---

## Progressive Delivery Automation

### Automated Promotion Decision Logic

A sound automated promotion system evaluates a composite health score across multiple signal types before advancing a canary. Relying on a single metric (e.g., error rate only) produces false confidence when other dimensions regress.

**Composite analysis signals:**

| Signal Category | Metrics | Weight |
|---|---|---|
| **Error rate** | HTTP 5xx rate, application exception rate | 40% |
| **Latency** | P50, P95, P99 response times | 30% |
| **Business metrics** | Conversion rate, transaction success rate, business KPIs | 20% |
| **Infrastructure** | CPU, memory, GC pause time | 10% |

**Automated analysis failure modes to guard against:**

1. **Insufficient traffic** — canary analysis at 1% traffic on a low-volume service may have too few samples for statistical significance. Set minimum request count thresholds before analysis runs.

2. **Baseline drift** — if the stable version is degrading simultaneously, the canary may appear healthy relative to baseline. Use absolute thresholds in addition to comparative thresholds.

3. **Time-of-day effects** — deploy canaries during representative traffic periods. Avoid initiating automated analysis during off-peak hours when traffic patterns are unrepresentative.

4. **Partial rollout of dependencies** — if the canary depends on a new database schema or downstream API not yet available, failure is environmental rather than code-related. Validate dependency readiness before initiating canary analysis.

### Release Pipeline Integration

```yaml
# GitHub Actions — Progressive delivery pipeline
name: Progressive Delivery

on:
  push:
    branches: [main]

jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build and push image
      uses: docker/build-push-action@v5
      with:
        push: true
        tags: registry.example.com/${{ env.SERVICE }}:${{ github.sha }}
    - name: Sign image with Cosign
      uses: sigstore/cosign-installer@v3
      run: |
        cosign sign --yes registry.example.com/${{ env.SERVICE }}:${{ github.sha }}

  deploy-canary:
    needs: build-and-sign
    runs-on: ubuntu-latest
    steps:
    - name: Update Argo Rollout image
      run: |
        kubectl argo rollouts set image ${{ env.SERVICE }} \
          ${{ env.SERVICE }}=registry.example.com/${{ env.SERVICE }}:${{ github.sha }}

    - name: Monitor canary progress
      run: |
        # Wait for rollout to complete or fail (max 2 hours)
        kubectl argo rollouts status ${{ env.SERVICE }} \
          --timeout=2h \
          --watch

  notify-on-failure:
    needs: deploy-canary
    if: failure()
    runs-on: ubuntu-latest
    steps:
    - name: Notify release manager
      run: |
        curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
          -H 'Content-type: application/json' \
          --data '{"text":"Canary deployment of ${{ env.SERVICE }}:${{ github.sha }} failed. Automatic rollback triggered."}'
```

---

## Rollback Orchestration

Every progressive delivery strategy must define its rollback semantics before deployment begins, not after an incident occurs.

### Rollback Decision Matrix

| Strategy | Rollback Trigger | Rollback Mechanism | RTO |
|---|---|---|---|
| Canary | Analysis failure; manual abort | Route 100% traffic to stable; decommission canary pods | < 30 seconds |
| Blue/Green | Post-promotion incident | DNS/LB switch back to blue slot; keep green available | < 60 seconds |
| Feature flag | Any time | Disable flag in management system | < 1 second (flag evaluation) |
| Rolling update | Kubectl rollout undo | Replace new pods with previous ReplicaSet | 2–10 minutes |

### Automated Rollback Conditions

Automate rollback initiation for the following conditions without requiring human intervention:

1. Error rate exceeds `baseline_error_rate * 3` for more than 5 consecutive minutes
2. P99 latency exceeds `500ms` (or service-specific SLO threshold) for 3 consecutive analysis periods
3. Any security scanner alert on the running canary (anomalous process execution, unexpected network connections)
4. Canary pod crash loop (2+ OOMKilled or CrashLoopBackOff events within 10 minutes)
5. Business metric (conversion, transaction success) drops more than 10% relative to stable version baseline

### Post-Rollback Actions

After automated rollback, the following actions are required before the release is retried:

1. **Automated incident creation** — the rollback event triggers a P2 incident in the ITSM system
2. **Artifact quarantine** — the failed image is tagged as `quarantined` in the registry and blocked from future deployments by admission control policy
3. **Root cause investigation** — required within 24 hours (P2 SLA) before the service version may be redeployed
4. **Approval gate reset** — subsequent release of the same service requires explicit release manager sign-off, bypassing automatic promotion

---

## Governance and Approval Integration

### Progressive Delivery as a Governance Control

Progressive delivery integrates with the Release Governance Model described in [framework.md](framework.md). The canary or blue/green promotion event replaces the manual "deploy to production" approval for standard changes. Instead of a human approving a production deployment, the automated analysis results serve as the approval decision.

| Change Type | Governance Model |
|---|---|
| Standard change (low risk) | Automated canary analysis; no human approval if criteria met |
| Significant change (medium risk) | Canary analysis required + Release Manager review of analysis results before promotion to 100% |
| Major change (high risk) | Canary analysis required + CAB review of analysis report + Release Manager sign-off |
| Emergency change | ECA approval; canary may be bypassed with post-facto review required |

### Compliance Evidence from Progressive Delivery

Progressive delivery tooling generates deployment evidence that satisfies change management and compliance requirements. Ensure the following artifacts are captured and stored in your immutable audit trail:

| Evidence Type | Source | Compliance Requirement |
|---|---|---|
| Canary analysis report | Argo Rollouts / Flagger metrics | Change failure rate measurement (DORA); SOC 2 CC8.1 |
| Traffic weight transition log | Load balancer / ingress logs | Deployment timeline evidence; ISO 27001 A.8.32 |
| Rollback decision record | CI/CD audit log | Change management evidence; PCI-DSS 6.4 |
| Promotion approval record | Release orchestration system | Segregation of duties evidence; SOC 2 CC6.3 |

---

## Observability Requirements

Progressive delivery without observability is inoperable. The following observability capabilities are prerequisites:

### Minimum Viable Observability for Canary Analysis

1. **Request-level error rate** — labeled by service version (canary vs. stable). Most service meshes and ingress controllers provide this automatically if pods are labeled with version.

2. **Latency histograms** — P50, P95, P99 per service version. Requires Prometheus histogram instrumentation in the application, or service mesh telemetry (Istio, Linkerd).

3. **Deployment version labels on all metrics** — all application metrics must include a `version` label populated from the pod's container image tag or a downward API field.

4. **Business metric instrumentation** — at minimum, transaction success rate and business-critical conversion events. These must be tagged with the serving version.

### Grafana Dashboard for Canary Monitoring

The recommended Grafana dashboard layout for canary monitoring includes:

- **Top panel**: Current traffic weight (canary % vs. stable %)
- **Error rate panel**: Side-by-side comparison: canary error rate vs. stable error rate, with threshold lines
- **Latency panel**: Overlapping P99 time series for canary and stable
- **Request volume panel**: Total requests per second by version (ensures analysis has sufficient traffic)
- **Business metrics panel**: Conversion rate or primary business KPI by version
- **Rollout stage panel**: Current canary stage and time remaining in analysis window

---

## Tool Reference

| Tool | Role | Integration |
|---|---|---|
| **Argo Rollouts** | Kubernetes-native canary and blue/green controller | kubectl plugin; integrates with Prometheus, Datadog, New Relic for analysis |
| **Flagger** | GitOps-native progressive delivery controller | Works with Flux; supports Istio, Linkerd, Nginx, Gloo Edge |
| **Istio / Linkerd** | Service mesh for traffic weight control | VirtualService / TrafficSplit for weighted routing |
| **Nginx Ingress** | Ingress-level traffic splitting | Annotations for canary weight, header-based routing |
| **LaunchDarkly** | Enterprise feature flag management | SDK for all major languages; OpenFeature provider available |
| **Unleash** | Open-source feature flag platform | Self-hosted; OpenFeature provider available |
| **Prometheus + Grafana** | Metrics collection and analysis | Required for automated canary analysis |
| **Flagger Load Tester** | Load generation for canary warm-up | Ensures canary has sufficient traffic for analysis |

---

## Related Techstream Resources

- [Release Orchestration Framework — Core Framework](framework.md)
- [Release Orchestration Framework — GitOps Architecture](gitops-architecture.md)
- [Secure CI/CD Reference Architecture — Threat Model](../../secure-ci-cd-reference-architecture/docs/threat-model.md)
- [DevSecOps Maturity Model — Metrics and KPIs](../../devsecops-maturity-model/docs/metrics-kpis.md)
- [Cloud Security DevSecOps — Kubernetes Security Hardening](../../cloud-security-devsecops/docs/framework.md)
