# GitOps Release Architecture

GitOps is a release orchestration pattern where the desired state of all deployment environments is stored as declarative configuration in Git, and a reconciliation controller continuously converges the actual environment state toward that declared state. This document defines the Techstream GitOps reference architecture, security controls, and integration patterns for the Release Orchestration Framework.

---

## Core GitOps Principles

**1. Git as the single source of truth.**
Every deployed configuration — Kubernetes manifests, Helm values, Kustomize overlays — lives in a Git repository. The running state of any environment is derivable from a specific commit.

**2. Declarative desired state.**
Deployments are described as the desired end state, not as imperative steps. The GitOps controller handles convergence.

**3. Pull-based deployment.**
The deployment controller running inside the cluster pulls configuration from Git. No external system has write access to the cluster. This eliminates the need for cluster credentials in CI pipelines.

**4. Automated reconciliation.**
The controller continuously detects and corrects drift between the declared state in Git and the actual cluster state.

---

## Repository Structure

GitOps requires a deliberate repository strategy. The two primary patterns are single-repository (monorepository) and multi-repository. Techstream recommends the **multi-repository pattern** for organizations with more than one team or more than two deployment environments.

### Recommended: Multi-Repository Layout

```
Application repositories (owned by development teams)
├── app-payments/
│   ├── src/                     # Application source code
│   └── .github/workflows/       # CI pipeline — builds image, updates deployment repo
│
└── app-auth/
    ├── src/
    └── .github/workflows/

GitOps configuration repositories (owned by platform/ops teams)
├── gitops-config/                # Cluster configuration, team namespaces, RBAC
│   ├── clusters/
│   │   ├── production/
│   │   │   ├── apps.yaml        # ArgoCD ApplicationSet or Flux HelmRelease
│   │   │   └── cluster-config/  # RBAC, NetworkPolicies, LimitRanges
│   │   ├── staging/
│   │   └── dev/
│   └── platform/                # Shared platform components (cert-manager, ingress-nginx)
│
└── gitops-apps/                  # Per-environment Helm values / Kustomize overlays
    ├── base/
    │   ├── app-payments/
    │   │   └── kustomization.yaml
    │   └── app-auth/
    │       └── kustomization.yaml
    └── overlays/
        ├── production/
        │   ├── app-payments/
        │   │   └── values.yaml  # Production-specific values
        │   └── app-auth/
        │       └── values.yaml
        ├── staging/
        └── dev/
```

**Why separate repos for config and apps:**
- Allows platform team to control cluster configuration without blocking application teams
- Limits blast radius — a misconfigured application overlay cannot affect cluster-level settings
- Enables different access control policies: application teams can merge app overlays; only platform team can merge cluster-config changes

---

## Tooling: ArgoCD vs. Flux

| Capability | ArgoCD | Flux |
|-----------|--------|------|
| **Architecture** | Server-side UI + API; agents connect to central ArgoCD server | Controller-per-cluster; no central server |
| **Multi-cluster management** | ArgoCD Hub-and-Spoke (single ArgoCD, multiple managed clusters) | Flux multi-tenancy via separate Flux instances or tenant controllers |
| **UI / dashboard** | Rich web UI with application topology view | No built-in UI; use Weave GitOps (OSS) |
| **ApplicationSet (dynamic app generation)** | Native ArgoCD ApplicationSet controller | Flux `HelmRelease` + Kustomization with generators |
| **Progressive delivery** | Argo Rollouts (separate controller) | Flagger (integrated with Flux) |
| **Secret management** | ArgoCD Vault Plugin / External Secrets Operator | SOPS encryption or External Secrets Operator |
| **OCI artifact support** | ArgoCD 2.6+ OCI Helm charts | Flux OCI repository source (native) |
| **RBAC** | ArgoCD RBAC (Casbin-based) | Kubernetes RBAC on Flux CRDs |
| **Notification** | ArgoCD Notifications (Slack, webhook, PagerDuty) | Flux Alert controller |
| **GitLab integration** | Full | Full |
| **GitHub integration** | Full | Full |

**Recommendation:**
- Use **ArgoCD** when a centralized visibility dashboard is important, when you need a UI-driven approval workflow, or when you manage a large number of clusters from a single control plane.
- Use **Flux** when you need a fully declarative, GitOps-native setup with no server-side components, or when you require native OCI artifact support and Flagger-based progressive delivery.

---

## Security Controls for GitOps Repositories

### Branch Protection Requirements

All GitOps configuration repositories must enforce the following on the default branch and environment branches:

- At least two required reviewers for changes to `clusters/production/` and `overlays/production/`
- At least one required reviewer for staging overlays
- CODEOWNERS file defining ownership boundaries (platform team owns cluster config; application teams own app overlays)
- Status checks required: YAML lint, Kustomize build dry-run, Helm template validation
- No direct pushes to protected branches — all changes via pull request
- Branch deletion protection on `main`, `production`, `staging`

```
# .github/CODEOWNERS example for gitops-apps
# Platform team approves all cluster-level changes
/clusters/                      @org/platform-team
/platform/                      @org/platform-team

# Application teams approve their own overlays
/overlays/*/app-payments/       @org/team-payments
/overlays/*/app-auth/           @org/team-auth

# Production overlays require platform team co-approval
/overlays/production/           @org/platform-team @org/security-team
```

---

### Image Reference Policy

All image references in GitOps manifests must use digest references, not mutable tags. Mutable tags can be updated without a Git commit, bypassing the GitOps audit trail.

```yaml
# Incorrect: mutable tag — can change without a Git commit
containers:
- name: app
  image: myregistry.io/app:latest

# Correct: digest reference — immutable, traceable to a specific build
containers:
- name: app
  image: myregistry.io/app@sha256:abc123def456...
```

Enforce this with a Kyverno policy:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-image-digest
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Image references must use digest (@sha256:...), not mutable tags."
      pattern:
        spec:
          containers:
          - image: "*@sha256:?"
```

---

### Secrets in GitOps

Never store plaintext secrets in GitOps repositories. The two recommended approaches:

**Option A: External Secrets Operator (ESO)**
Secrets are stored in a cloud-native secrets manager (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager). A `ExternalSecret` CR in the GitOps repo references the secret by name; ESO fetches and syncs it to a Kubernetes `Secret` at runtime.

```yaml
# ExternalSecret pointing to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: payments
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: /production/payments/db-password
```

**Option B: SOPS + Age or KMS Encryption (Flux-native)**
Secrets are encrypted in the repository using SOPS and decrypted by Flux at sync time using a KMS key or Age key that the Flux controller has access to.

```bash
# Encrypt a Kubernetes secret YAML with SOPS
sops --encrypt --age <recipient-public-key> \
  k8s-secret.yaml > k8s-secret.enc.yaml
```

**Recommendation:** Use ESO for production workloads where secrets rotate frequently or where audit logging of secret access is required. Use SOPS for platform configuration secrets where change control via Git history is the primary requirement.

---

## Promotion Pipeline

The promotion pipeline automates the progression of a tested artifact from dev → staging → production through the GitOps repository, while preserving the audit trail.

```
CI Pipeline (app repo)                    GitOps Config Repo
┌─────────────────────────────┐           ┌─────────────────────────────┐
│ 1. Build and test           │           │                             │
│ 2. Scan image               │           │  dev/app-payments/          │
│ 3. Sign image               │──────────▶│    image: myapp@sha256:abc  │
│ 4. Generate SBOM            │  Update   │                             │
│ 5. Push to registry         │  digest   │  staging/app-payments/      │
│ 6. Open PR to GitOps repo   │  in dev   │    image: myapp@sha256:xyz  │
└─────────────────────────────┘  overlay  │                             │
                                          │  production/app-payments/   │
                                          │    image: myapp@sha256:def  │
                                          └─────────────────────────────┘
                                                      │ ArgoCD/Flux sync
                                                      ▼
                                          ┌─────────────────────────────┐
                                          │  Clusters (pull-based)      │
                                          │  dev cluster → staging →    │
                                          │  production (with gates)    │
                                          └─────────────────────────────┘
```

### Automated Promotion with Health Gates

Use ArgoCD ApplicationSet or Flux Kustomization `dependsOn` to sequence promotions with automated health checks.

```yaml
# Flux: stage dependency — staging only syncs after dev is healthy
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-payments-staging
spec:
  dependsOn:
  - name: app-payments-dev
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: app-payments
    namespace: payments-staging
  interval: 5m
  path: ./overlays/staging/app-payments
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-apps
  timeout: 3m
```

Production promotion requires a manual pull request approval — no automated promotion to production without a human reviewer.

---

## Drift Detection and Alerting

Configure alerts for any detected drift between the GitOps declared state and the cluster actual state. Drift indicates either an unauthorized change or a GitOps controller failure.

```yaml
# ArgoCD: Application with drift detection enabled
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-payments-production
spec:
  syncPolicy:
    automated:
      prune: true     # Remove resources deleted from Git
      selfHeal: true  # Auto-correct drift (caution: only after testing)
  # For production, consider selfHeal: false and alert on drift instead
```

**Alert on drift:**
```yaml
# ArgoCD Notification: alert when app is out-of-sync
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  trigger.on-out-of-sync: |
    - when: app.status.sync.status == 'OutOfSync'
      oncePer: app.metadata.name
      send: [drift-alert]
  template.drift-alert: |
    message: |
      Production app {{.app.metadata.name}} has drifted from desired state.
      Commit: {{.app.status.sync.revision}}
      Reason: {{.app.status.conditions | map(attribute='message') | join(', ')}}
```

---

## Audit Trail Integration

Every GitOps deployment produces an immutable audit trail by design — the Git commit history is the audit log. Supplement this with runtime deployment events:

| Event | Source | Destination |
|-------|--------|-------------|
| PR opened for promotion | GitHub/GitLab webhook | SIEM / ticket system |
| PR approved and merged | GitHub/GitLab audit log | Compliance evidence store |
| ArgoCD sync completed | ArgoCD Notification | SIEM / PagerDuty |
| Drift detected | ArgoCD Notification | SIEM / PagerDuty / on-call |
| Manual sync override | ArgoCD audit log | SIEM + security alert |
| Cluster resource modified outside GitOps | Kubernetes audit log | SIEM + security alert |

---

## Integration with Release Orchestration Framework

GitOps is the deployment mechanism; the Release Orchestration Framework provides the governance layer on top of it.

| Release Framework Concern | GitOps Implementation |
|--------------------------|----------------------|
| Approval workflows | Pull request reviews and merge requirements |
| Environment promotion gates | Branch protection + CI status checks on GitOps PRs |
| Rollback | `git revert` of the promotion commit; ArgoCD/Flux re-syncs previous state |
| Change window enforcement | OPA/Kyverno admission policy blocking syncs outside change windows |
| Compliance evidence | Git commit history + ArgoCD sync events + Kubernetes audit logs |
| DORA deployment frequency | Count of merged promotion PRs per time period |
| DORA change failure rate | Count of rollback reverts per total promotions |

---

## Kubernetes Security Hardening for GitOps Clusters

GitOps manages cluster state declaratively, which makes it the correct place to enforce security configuration. All the following controls should be expressed as manifests in the GitOps configuration repository and reconciled automatically.

### Pod Security Standards

Enforce Kubernetes Pod Security Standards (PSS) at the namespace level. Declare labels in namespace manifests in the GitOps config repo:

```yaml
# clusters/production/namespaces/payments.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

PSS `restricted` profile enforces:
- No privilege escalation
- Non-root user and group
- Seccomp profile required (RuntimeDefault or Localhost)
- All capabilities dropped; only `NET_BIND_SERVICE` allowed
- Read-only root filesystem (required for most workloads)
- No host networking, PID, or IPC namespaces

For workloads that cannot immediately meet `restricted`, use `baseline` as a stepping stone. Never use `privileged` in production without explicit justification and security team approval.

### Network Policies

Default-deny network policies must be present in every namespace. Apply them via GitOps:

```yaml
# Default deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow only necessary communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payments-ingress
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payments-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - port: 8080
      protocol: TCP
---
# Allow DNS egress only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

### Admission Control Policy

Deploy Kyverno as the admission controller and commit all policies to the GitOps config repository:

```yaml
# clusters/production/policies/require-labels.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-labels
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-for-labels
    match:
      any:
      - resources:
          kinds: [Deployment, StatefulSet, DaemonSet]
    validate:
      message: "Deployments must include 'team' and 'app' labels for ownership tracking."
      pattern:
        metadata:
          labels:
            team: "?*"
            app: "?*"
---
# clusters/production/policies/block-privileged-containers.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-privileged
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Privileged containers are not allowed."
      pattern:
        spec:
          containers:
          - =(securityContext):
              =(privileged): "false"
```

### Resource Quotas and Limit Ranges

Prevent resource exhaustion attacks by enforcing limits at the namespace level:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "50"
    services: "10"
    secrets: "30"
    configmaps: "30"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-limits
  namespace: payments
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4"
      memory: "8Gi"
```

---

## Progressive Delivery

Progressive delivery reduces deployment risk by gradually shifting traffic to new versions while continuously evaluating health metrics.

### Argo Rollouts (with ArgoCD)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payments-api
  namespace: payments
spec:
  replicas: 10
  selector:
    matchLabels:
      app: payments-api
  template:
    metadata:
      labels:
        app: payments-api
    spec:
      containers:
      - name: payments-api
        image: myregistry.io/payments-api@sha256:abc123
  strategy:
    canary:
      canaryService: payments-api-canary
      stableService: payments-api-stable
      trafficRouting:
        nginx:
          stableIngress: payments-ingress
      steps:
      - setWeight: 10          # 10% of traffic to canary
      - pause: {duration: 5m}  # Wait 5 minutes
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 25
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
  analysis:
    successCondition: result[0] >= 0.99
    failureCondition: result[0] < 0.95
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: payments
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    failureCondition: result[0] < 0.95
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc:9090
        query: |
          sum(rate(http_requests_total{app="payments-api",status=~"2.."}[2m]))
          /
          sum(rate(http_requests_total{app="payments-api"}[2m]))
```

### Flagger (with Flux)

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payments-api
  namespace: payments
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payments-api
  progressDeadlineSeconds: 120
  service:
    port: 8080
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 5        # Max number of failed analysis checks before rollback
    maxWeight: 50       # Maximum canary traffic percentage
    stepWeight: 10      # Traffic increment per analysis interval
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500        # Max 500ms p99 latency
      interval: 1m
    webhooks:
    - name: integration-tests
      type: pre-rollout
      url: http://test-runner.test/run
      timeout: 30s
      metadata:
        type: "bash"
        cmd: "npm run test:integration"
```

**Security gates for progressive delivery:**
- Signature verification is performed before a rollout begins — unsigned images are rejected by Kyverno admission control
- Analysis metrics must show no increase in 5xx error rates during traffic shifting
- Automatic rollback triggers on analysis failure — no manual intervention required
- Rollback events are logged to SIEM and trigger an incident notification

---

## GitOps with Non-Kubernetes Deployment Targets

GitOps principles apply beyond Kubernetes. For organizations with heterogeneous environments — VMs, serverless functions, managed services, and Terraform-managed infrastructure — GitOps patterns can be adapted using target-appropriate reconciliation tools.

### Virtual Machines (EC2, Azure VMs, GCP Compute Engine)

Ansible AWX (the upstream of Red Hat Ansible Automation Platform) acts as the reconciliation engine:

1. Store Ansible playbooks and variable files in the Git config repo.
2. Configure AWX to watch the config repo and trigger job template runs on merge to the environment branch.
3. Run job templates in `check` mode first; apply only if no unexpected changes are detected.
4. Export AWX job run results to the audit trail (SIEM, S3 compliance archive).

```yaml
# AWX GitOps configuration (stored in config repo, managed by AWX itself)
# Job template is version-controlled; AWX pulls the latest on each run
---
name: "Configure Production VMs"
description: "Reconcile VM configuration from Git"
project: "infra-config-repo"
playbook: "playbooks/production-hardening.yml"
job_type: run
limit: "production-vms"
credentials:
  - "production-ssh-key"
  - "vault-prod-token"
extra_vars:
  check_mode: false  # Set to true for drift detection runs
```

**Drift detection:** Schedule a separate AWX job template in `--check` mode on a 15-minute interval. Alert on any run that reports changes — this indicates drift from the declared configuration.

---

### Serverless Functions (AWS Lambda, Azure Functions, GCP Cloud Functions)

Pure pull-based GitOps is not well-suited to serverless — Lambda and Functions are push-deployed. Use a restricted push model instead:

1. Store Lambda function configurations (handler code, environment variables, layers, event sources) in the config repo as CDK, SAM, or Serverless Framework manifests.
2. A deployment bot (GitHub Actions) monitors the config repo for changes to the production environment; deploys using an IAM role scoped to Lambda operations only.
3. Use Lambda aliases and weighted routing for progressive delivery (similar to Kubernetes canary patterns).
4. Treat the config repo merge as the governance gate (same CODEOWNERS + required review rules apply).

```yaml
# GitHub Actions: serverless GitOps push-deployment
name: Deploy Lambda — Production
on:
  push:
    branches: [main]
    paths: ['overlays/production/lambda/**']

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires GitHub Environment approval
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/lambda-deploy-role
          aws-region: us-east-1

      - name: Deploy Lambda function
        run: |
          aws lambda update-function-code \
            --function-name payment-processor \
            --image-uri ${{ env.IMAGE_DIGEST }}

      - name: Shift traffic to new version (canary: 10%)
        run: |
          aws lambda update-alias \
            --function-name payment-processor \
            --name production \
            --routing-config AdditionalVersionWeights={"${{ env.NEW_VERSION }}"=0.1}
```

---

### Infrastructure as Code (Terraform)

Use Atlantis or Spacelift as the GitOps reconciliation engine for Terraform:

```yaml
# atlantis.yaml — Atlantis configuration for GitOps Terraform
version: 3
projects:
  - name: payment-infrastructure-production
    dir: ./infrastructure/production
    workspace: production
    autoplan:
      when_modified: ["**/*.tf", "**/*.tfvars", "../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - mergeable      # PR must be mergeable (no conflicts)
      - approved       # At least one approval required
      - undiverged     # Plan must be applied after the most recent commit
    workflow: production-workflow

workflows:
  production-workflow:
    plan:
      steps:
        - init
        - plan:
            extra_args: ["-var-file=production.tfvars"]
    apply:
      steps:
        - apply
```

**Drift detection for Terraform:** Schedule a nightly Atlantis plan run or use `terraform plan -detailed-exitcode` in a GitHub Actions scheduled workflow. Alert on exit code 2 (changes detected).

---

## Change Window Enforcement in GitOps

Production deployments should only occur within authorized change windows. In a GitOps model, the change window gate is applied at the PR merge step, not at the cluster reconciliation step.

### GitHub Actions Change Window Gate

```yaml
# .github/workflows/production-change-window.yml
# This workflow runs on PRs targeting the production overlay
name: Change Window Enforcement
on:
  pull_request:
    branches: [main]
    paths: ['overlays/production/**', 'clusters/production/**']

jobs:
  check-change-window:
    runs-on: ubuntu-latest
    steps:
      - name: Verify production change window
        run: |
          # Change window: Tuesday–Thursday, 22:00–02:00 UTC
          HOUR=$(date -u +%H)
          DOW=$(date -u +%u)  # 1=Mon, 7=Sun

          IN_WINDOW=false
          if [[ $DOW -ge 2 && $DOW -le 4 ]]; then
            if [[ $HOUR -ge 22 || $HOUR -lt 2 ]]; then
              IN_WINDOW=true
            fi
          fi

          if [[ "$IN_WINDOW" == "false" ]]; then
            echo "::error::Production change window not open."
            echo "::error::Changes to production allowed: Tue–Thu, 22:00–02:00 UTC"
            echo "::error::Current UTC time: $(date -u)"
            echo ""
            echo "If this is an emergency change, use the emergency approval process:"
            echo "1. Tag a release manager in this PR"
            echo "2. They must apply the 'emergency-change' label to bypass this check"
            echo "3. Emergency changes are reviewed retrospectively in the next CAB meeting"
            exit 1
          fi

          echo "Change window is open. Current UTC time: $(date -u)"
```

Add an emergency bypass via label:

```yaml
      - name: Check for emergency override label
        id: check-label
        uses: actions/github-script@v7
        with:
          script: |
            const labels = context.payload.pull_request.labels.map(l => l.name);
            const isEmergency = labels.includes('emergency-change');
            if (isEmergency) {
              console.log('Emergency change label detected — bypassing change window');
              console.log('NOTE: This bypass will be reviewed in the next CAB meeting');
            }
            return isEmergency;
```

---

## GitOps Adoption Checklist

Use this checklist before relying on GitOps for production deployments:

```
Repository Setup
[ ] Config repo separate from application source repo
[ ] CODEOWNERS configured: production overlay → release managers; staging → tech leads
[ ] Branch protection: required reviews, no direct push, CI checks required
[ ] All environment overlays use image digest references (not mutable tags)

Security Controls
[ ] Kyverno policy: require image digest references in all Pods
[ ] Kyverno policy: require image signature verification (Cosign)
[ ] No plaintext secrets in config repo
[ ] ESO, Sealed Secrets, or SOPS deployed and tested
[ ] GitOps controller uses read-only Git credentials
[ ] GitOps controller Kubernetes RBAC scoped to minimum required permissions
[ ] Pod Security Standards enforced (Restricted profile for production namespaces)
[ ] Default-deny NetworkPolicy in all namespaces

Operations
[ ] ArgoCD/Flux deployed and reconciling all environments
[ ] Health checks configured for all applications
[ ] Drift detection alerts operational (sync failure → PagerDuty/Slack)
[ ] Change window enforcement active for production overlay PRs
[ ] Rollback procedure documented and tested (git revert → auto-reconcile)
[ ] Disaster recovery tested: cluster can be bootstrapped from Git from scratch

Governance and Audit
[ ] Promotion workflow documented and communicated to all engineering teams
[ ] ArgoCD/Flux sync events shipped to SIEM
[ ] Git commit history preserved as audit evidence (no force-push to config repo)
[ ] Compliance evidence package: config repo history + sync logs = deployment audit trail
```

---

## Related Documents

- [Release Orchestration Framework](framework.md) — Governance model, approval workflows, rollback strategies
- [Release Best Practices](best-practices.md) — 45 release practices including GitOps-specific guidance
- [Secure Pipeline Templates](../../secure-pipeline-templates/docs/azure-devops-pipeline.md) — CI pipeline that produces signed artifacts for GitOps promotion
- [Cloud Security DevSecOps](../../cloud-security-devsecops/docs/architecture.md) — Kubernetes security architecture
- [API Security Integration Guide](../../devsecops-framework/docs/api-security.md) — API security controls relevant to GitOps-managed services
- [Troubleshooting Guide](../../techstream-docs/docs/troubleshooting-guide.md) — Common GitOps implementation problems and solutions
