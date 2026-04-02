# Release Orchestration Implementation Guide

## Table of Contents

- [Implementation Overview](#implementation-overview)
- [Phase 1: Foundation](#phase-1-foundation)
- [Phase 2: Standardization](#phase-2-standardization)
- [Phase 3: Optimization](#phase-3-optimization)
- [Phase 4: Innovation](#phase-4-innovation)
- [Toolchain Selection](#toolchain-selection)
- [Integration Patterns with CI/CD Tools](#integration-patterns-with-cicd-tools)
- [Approval Workflow Configuration](#approval-workflow-configuration)
- [Environment Configuration Management](#environment-configuration-management)
- [Release Train Setup](#release-train-setup)
- [Feature Flag Integration](#feature-flag-integration)
- [Deployment Window Enforcement](#deployment-window-enforcement)
- [Rollback Automation Configuration](#rollback-automation-configuration)

---

## Implementation Overview

Implementing release orchestration is an organizational transformation, not just a tooling deployment. The implementation approach must account for tooling configuration, process changes, team training, and governance model adoption.

This guide presents a four-phase approach that progressively builds capability, validates the model with early adopters, and scales across the organization. Each phase has defined objectives, deliverables, and success criteria before advancing to the next.

**Implementation principles:**

- **Pilot before scale** — implement the framework with one or two teams before broad rollout, using their experience to validate and refine the model
- **Automate the happy path** — prioritize automating common, low-risk deployments so that teams experience the value of orchestration as reduced friction, not increased process
- **Integrate, don't replace** — orchestration layers sit on top of existing CI/CD investments; replace CI/CD tools only when the orchestration integration is insufficient
- **Measure from day one** — establish DORA metric baselines before making changes so that the impact of the framework can be demonstrated objectively

---

## Phase 1: Foundation

**Duration:** 6–8 weeks
**Objective:** Establish the core orchestration infrastructure, integrate with one pilot team's CI/CD pipeline, and demonstrate the basic promotion workflow.

### Deliverables

**1.1 Environment Inventory and Classification**

Document all environments across the organization:

```yaml
# environment-registry.yaml
environments:
  - name: dev
    purpose: Integration development
    owner: platform-team
    kubernetes_cluster: eks-dev-us-east-1
    promotion_target: test
    auto_promote: true
    data_classification: synthetic

  - name: test
    purpose: Functional and regression testing
    owner: qa-team
    kubernetes_cluster: eks-test-us-east-1
    promotion_target: staging
    auto_promote: false
    requires_approval: tech_lead
    data_classification: anonymized

  - name: staging
    purpose: Pre-production validation
    owner: platform-team
    kubernetes_cluster: eks-staging-us-east-1
    promotion_target: production
    auto_promote: false
    requires_approval: release_manager
    minimum_soak_hours: 24
    data_classification: anonymized_production_mirror

  - name: production
    purpose: Live production
    owner: platform-team
    kubernetes_cluster: eks-prod-us-east-1
    auto_promote: false
    requires_approval: release_manager
    requires_change_window: true
    data_classification: production
```

**1.2 Orchestration Platform Deployment**

Deploy the selected orchestration platform (see [Toolchain Selection](#toolchain-selection)) to dedicated infrastructure:

- Kubernetes cluster (recommended: dedicated management cluster or dedicated namespace with strict RBAC)
- High availability configuration (minimum 3 replicas for control plane components)
- Persistent storage for state (PostgreSQL recommended)
- TLS for all internal and external communication
- LDAP/OIDC integration for user authentication
- Role-based access control configuration

**1.3 Artifact Registry Integration**

Configure the artifact registry as the authoritative source for deployable artifacts:

```bash
# Artifactory/Nexus: Configure webhook to notify orchestration system
curl -X POST https://artifactory.example.com/api/webhooks \
  -H "Authorization: Bearer $ARTIFACTORY_TOKEN" \
  -d '{
    "key": "release-orchestration",
    "url": "https://release-orchestration.internal/api/v1/artifacts/events",
    "events": ["artifact.deployed"],
    "criteria": {
      "repoKeys": ["docker-prod-local", "maven-prod-local", "npm-prod-local"]
    }
  }'
```

**1.4 Pilot Team Onboarding**

Onboard the first pilot team through:

1. Service catalog registration — register the service with its metadata (tier, owner, dependencies)
2. CI/CD pipeline integration — add orchestration registration step to existing pipeline
3. Deployment policy definition — document and configure the service's deployment policies
4. Walkthrough and rehearsal — the team performs a full promotion cycle in a training environment before live use

### Phase 1 Success Criteria

- [ ] At least one service is deploying to dev and test via orchestration system
- [ ] Promotion records are being created with full audit trail
- [ ] Pilot team can initiate and approve a production deployment through the orchestration UI
- [ ] DORA metrics baseline has been established
- [ ] Rollback has been tested and documented for the pilot service

---

## Phase 2: Standardization

**Duration:** 8–12 weeks
**Objective:** Extend the framework to all production services, integrate with ITSM, and establish change window enforcement.

### Deliverables

**2.1 ITSM Integration Configuration**

Configure bidirectional integration with the ITSM platform. Example ServiceNow integration:

```python
# servicenow_integration.py
import requests
from dataclasses import dataclass

@dataclass
class ChangeRequestPayload:
    service_name: str
    artifact_version: str
    target_environment: str
    change_type: str
    planned_start: str
    planned_end: str
    implementation_plan: str
    backout_plan: str
    test_plan_url: str

class ServiceNowClient:
    def __init__(self, instance_url: str, username: str, password: str):
        self.base_url = f"https://{instance_url}/api/now"
        self.auth = (username, password)
        self.headers = {"Content-Type": "application/json", "Accept": "application/json"}

    def create_change_request(self, payload: ChangeRequestPayload) -> dict:
        """Create a change request in ServiceNow for a deployment."""
        cr_data = {
            "short_description": f"Deploy {payload.service_name} {payload.artifact_version} to {payload.target_environment}",
            "description": payload.implementation_plan,
            "type": self._map_change_type(payload.change_type),
            "assignment_group": self._lookup_assignment_group(payload.service_name),
            "cmdb_ci": self._lookup_ci(payload.service_name),
            "planned_start_date": payload.planned_start,
            "planned_end_date": payload.planned_end,
            "implementation_plan": payload.implementation_plan,
            "backout_plan": payload.backout_plan,
            "test_plan": f"See CI pipeline: {payload.test_plan_url}",
            "u_orchestration_deployment_id": self._generate_deployment_id(payload)
        }
        response = requests.post(
            f"{self.base_url}/table/change_request",
            json=cr_data,
            auth=self.auth,
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()["result"]

    def update_change_request(self, cr_number: str, state: str, notes: str) -> dict:
        """Update a change request state and work notes."""
        cr_sys_id = self._get_sys_id_by_number(cr_number)
        data = {
            "state": self._map_state(state),
            "work_notes": notes,
            "close_code": "successful" if state == "implemented" else None,
            "close_notes": notes if state == "closed" else None
        }
        response = requests.patch(
            f"{self.base_url}/table/change_request/{cr_sys_id}",
            json={k: v for k, v in data.items() if v is not None},
            auth=self.auth,
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()["result"]
```

**2.2 Service Catalog Expansion**

Extend the service catalog to all production services, capturing:

- Service tier (Platinum / Gold / Silver / Bronze)
- Business owner and technical owner
- CMDB configuration item ID
- Deployment policy (promotion requirements, soak times, approval requirements)
- Dependency map (upstream and downstream services)
- Rollback procedure
- On-call contact

**2.3 Change Window Configuration**

Configure the change window registry based on the organization's operational requirements (see architecture document for YAML schema).

**2.4 Team Training Program**

Deliver training to all engineering teams and release managers:

| Session | Audience | Duration | Content |
|---|---|---|---|
| Release Orchestration Overview | All engineers | 1 hour | Framework overview, promotion workflow, approval process |
| Developer Workflow | Developers | 2 hours | Hands-on: monitoring deployments, approving stage promotions, using UI |
| Release Manager Workflow | Release managers | 4 hours | Hands-on: approving production deployments, managing change records, executing rollbacks |
| Platform Configuration | DevOps engineers | Full day | Tooling configuration, policy authoring, troubleshooting |

### Phase 2 Success Criteria

- [ ] All production services are registered in the service catalog
- [ ] All production deployments flow through the orchestration system
- [ ] ServiceNow change records are created automatically for all normal and emergency changes
- [ ] Change window enforcement is active and tested
- [ ] All engineers have completed release orchestration training

---

## Phase 3: Optimization

**Duration:** 8–12 weeks
**Objective:** Implement advanced deployment strategies, automate rollback, and optimize approval workflows based on operational data.

### Deliverables

**3.1 Progressive Delivery Implementation**

Implement canary deployments for Platinum and Gold tier services using Argo Rollouts:

```yaml
# argo-rollout-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
  namespace: payment
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-stable
      trafficRouting:
        istio:
          virtualService:
            name: payment-service-vsvc
            routes:
              - primary
      steps:
        - setWeight: 5
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: success-rate-analysis
            args:
              - name: service-name
                value: payment-service
        - setWeight: 25
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: success-rate-analysis
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:
        successfulRunHistoryLimit: 5
        unsuccessfulRunHistoryLimit: 5
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-analysis
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.97
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

**3.2 Automated Rollback Configuration**

Configure automated rollback triggers for each service tier:

```yaml
# rollback-policy.yaml
rollback_policies:
  platinum:
    triggers:
      - metric: http_error_rate
        threshold: "3x baseline"
        window: 5m
        consecutive_breaches: 2
      - metric: p99_latency
        threshold: "+50% baseline"
        window: 10m
        consecutive_breaches: 3
      - metric: health_check_failure_rate
        threshold: "20%"
        window: 2m
        consecutive_breaches: 1
    rollback_action: immediate
    notify: [release-manager, on-call-engineer, service-owner]
    post_rollback: create_incident

  gold:
    triggers:
      - metric: http_error_rate
        threshold: "5x baseline"
        window: 10m
        consecutive_breaches: 2
      - metric: p99_latency
        threshold: "+100% baseline"
        window: 15m
        consecutive_breaches: 3
    rollback_action: notify_then_auto_15min
    notify: [release-manager, on-call-engineer]
    post_rollback: create_incident
```

---

## Phase 4: Innovation

**Duration:** Ongoing
**Objective:** Leverage operational data to further automate governance, implement self-service deployment capabilities, and drive continuous improvement.

### Deliverables

**4.1 AI-Assisted Risk Scoring**

Leverage deployment history, change pattern analysis, and incident correlation data to improve risk score accuracy:

- Train a classification model on historical deployment outcomes (success/failure/incident-caused)
- Use model predictions to inform (not replace) human approval decisions
- Surface predicted risk factors in the approval UI with explanation

**4.2 Self-Service Standard Changes**

Based on operational data, identify patterns of changes that have a near-zero incident rate and promote them to self-service standard changes:

- Teams can deploy standard changes without release manager approval
- Automated gates enforce quality requirements
- Real-time monitoring provides rapid rollback capability

**4.3 Deployment Analytics and Continuous Improvement**

Establish a continuous improvement loop based on deployment analytics:

- Monthly review of DORA metrics with team-level breakdowns
- Analysis of bottlenecks in approval workflows (approval wait time distribution)
- Trend analysis of automated rollback frequency
- Quarterly policy reviews to adjust risk scoring and approval requirements based on outcomes

---

## Toolchain Selection

### Decision Framework

| Criterion | Weight | Considerations |
|---|---|---|
| **Kubernetes nativity** | High for cloud-native orgs | Argo CD/Rollouts vs. traditional tools |
| **Heterogeneous target support** | High for mixed environments | Digital.ai, Harness, Spinnaker |
| **ITSM integration depth** | High for regulated industries | Digital.ai, Harness have native ServiceNow integrations |
| **Approval workflow capabilities** | High | Enterprise tools vs. open source |
| **Existing CI/CD investment** | Medium | Choose tools that complement existing pipelines |
| **Operational overhead** | Medium | Spinnaker has high ops overhead; hosted SaaS reduces it |
| **Cost** | Medium | Open source vs. commercial licensing |
| **Community and support** | Medium | Argo has strong CNCF community; commercial tools offer SLAs |

### Tool Comparison Matrix

| Tool | Kubernetes Native | Multi-cloud | ITSM Integration | Approval Workflows | Hosting |
|---|---|---|---|---|---|
| **Argo CD + Rollouts** | Native | Limited | Plugin-based | Basic | Self-hosted |
| **Flux + Flagger** | Native | Limited | Plugin-based | Limited | Self-hosted |
| **Spinnaker** | Good | Excellent | Plugin-based | Good | Self-hosted |
| **Harness CD** | Excellent | Excellent | Native (SN, JSM) | Excellent | SaaS / Self-hosted |
| **Digital.ai Deploy** | Good | Excellent | Native (SN, Jira) | Excellent | Self-hosted / SaaS |
| **Octopus Deploy** | Good | Good | Plugin-based | Good | Self-hosted / SaaS |

### Recommended Toolchain by Organization Profile

**Cloud-native, Kubernetes-first, startup to mid-size:**
- Orchestration: Argo CD + Argo Rollouts
- GitOps: Argo CD
- Progressive delivery: Argo Rollouts + Flagger
- Feature flags: Flagsmith (open source) or LaunchDarkly
- ITSM: Jira Service Management

**Large enterprise, heterogeneous environments, strong compliance requirements:**
- Orchestration: Harness or Digital.ai Deploy
- Progressive delivery: Native platform canary (or Argo Rollouts for Kubernetes)
- Feature flags: LaunchDarkly or Split
- ITSM: ServiceNow

---

## Integration Patterns with CI/CD Tools

### GitHub Actions Integration

```yaml
# .github/workflows/deploy.yml
name: Release Orchestration Integration

on:
  workflow_dispatch:
    inputs:
      target_environment:
        description: Target environment
        required: true
        type: choice
        options: [dev, test, staging, production]
      artifact_digest:
        description: Container image digest to deploy
        required: true

jobs:
  register-deployment:
    runs-on: ubuntu-latest
    steps:
      - name: Request deployment via Release Orchestration
        uses: techstream/release-orchestration-action@v2
        with:
          api-url: ${{ secrets.RO_API_URL }}
          api-token: ${{ secrets.RO_API_TOKEN }}
          service-name: ${{ github.event.repository.name }}
          artifact-digest: ${{ github.event.inputs.artifact_digest }}
          target-environment: ${{ github.event.inputs.target_environment }}
          wait-for-completion: true
          timeout-minutes: 60
```

### Jenkins Integration

```groovy
// Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Build & Test') {
            steps {
                sh 'make build test'
            }
        }
        stage('Register with Release Orchestration') {
            steps {
                script {
                    def response = httpRequest(
                        url: "${env.RELEASE_ORCHESTRATION_URL}/api/v1/deployments",
                        httpMode: 'POST',
                        customHeaders: [[name: 'Authorization', value: "Bearer ${env.RO_TOKEN}"]],
                        requestBody: groovy.json.JsonOutput.toJson([
                            service: env.SERVICE_NAME,
                            artifact_digest: env.IMAGE_DIGEST,
                            target_environment: 'dev',
                            ci_run_url: env.BUILD_URL
                        ]),
                        contentType: 'APPLICATION_JSON'
                    )
                    env.DEPLOYMENT_ID = readJSON(text: response.content).deployment_id
                }
            }
        }
        stage('Await Dev Deployment') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    waitUntil {
                        script {
                            def status = httpRequest(
                                url: "${env.RELEASE_ORCHESTRATION_URL}/api/v1/deployments/${env.DEPLOYMENT_ID}/status",
                                customHeaders: [[name: 'Authorization', value: "Bearer ${env.RO_TOKEN}"]]
                            )
                            return readJSON(text: status.content).state in ['deployed', 'failed']
                        }
                    }
                }
            }
        }
    }
}
```

---

## Approval Workflow Configuration

### Workflow Definition (Harness)

```yaml
# approval-workflow.yaml (Harness Pipeline)
pipeline:
  name: Payment Service Production Deployment
  identifier: payment_service_prod
  stages:
    - stage:
        name: Staging Validation Gate
        type: Approval
        spec:
          execution:
            steps:
              - step:
                  type: HarnessApproval
                  name: Release Manager Approval
                  spec:
                    approvalMessage: |
                      Review the deployment details:
                      - Service: payment-service
                      - Version: <+artifact.tag>
                      - Staging soak: <+pipeline.variables.staging_soak_hours> hours
                      - Test results: <+pipeline.variables.test_results_url>
                    includePipelineExecutionHistory: true
                    approvers:
                      userGroups:
                        - release-managers
                      minimumCount: 1
                    approverInputs:
                      - name: deployment_justification
                        defaultValue: ""
                    isAutoRejectEnabled: false
                  timeout: 1d
```

---

## Environment Configuration Management

Environment-specific configuration should be managed separately from application artifacts, using a combination of:

- **Kubernetes ConfigMaps and Secrets** for runtime configuration
- **External Secrets Operator** for secrets fetched from Vault, AWS Secrets Manager, or Azure Key Vault
- **Environment overlays in Kustomize or Helm values files** for structural configuration differences

### Example Kustomize Overlay Structure

```
services/payment-service/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── deployment-patch.yaml  # replica count: 1, resources: small
│   ├── test/
│   │   ├── kustomization.yaml
│   │   └── deployment-patch.yaml  # replica count: 1, resources: medium
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── deployment-patch.yaml  # replica count: 3, resources: prod-like
│   └── production/
│       ├── kustomization.yaml
│       └── deployment-patch.yaml  # replica count: 10, resources: full
```

---

## Release Train Setup

### Release Train Configuration

```yaml
# release-train.yaml
release_train:
  name: Platform Release Train
  cadence: bi-weekly
  train_days: [Monday]  # Train departs every other Monday
  cutoff_day: Wednesday  # Feature cutoff: Wednesday of the prior week

  participating_services:
    - payment-service
    - order-service
    - inventory-service
    - notification-service
    - api-gateway

  release_train_engineer: rte@example.com
  slack_channel: "#platform-release-train"

  milestones:
    - name: Feature cutoff
      offset_days: -5  # 5 days before departure
      action: freeze_feature_branches
    - name: Integration validation complete
      offset_days: -3
      action: all_services_deployed_to_test
    - name: Staging deployment
      offset_days: -2
      action: deploy_all_to_staging
    - name: CAB review
      offset_days: -1
      action: submit_cab_package
    - name: Production deployment
      offset_days: 0
      action: deploy_to_production
      change_window: Tuesday_22:00
```

---

## Feature Flag Integration

### LaunchDarkly Integration

```python
# feature_flag_service.py
import ldclient
from ldclient.config import Config

class FeatureFlagService:
    def __init__(self, sdk_key: str):
        ldclient.set_config(Config(sdk_key))
        self.client = ldclient.get()

    def execute_release_rollout(self, flag_key: str, target_percentage: int, context: dict) -> bool:
        """Progressively roll out a feature flag to a percentage of users."""
        # Update flag targeting rules via LaunchDarkly management API
        import requests
        response = requests.patch(
            f"https://app.launchdarkly.com/api/v2/flags/{context['project_key']}/{flag_key}",
            headers={"Authorization": context['api_key'], "Content-Type": "application/json"},
            json=[{
                "op": "replace",
                "path": "/environments/production/fallthrough/rollout/variations/0/weight",
                "value": target_percentage * 1000  # LaunchDarkly uses weight in thousandths
            }]
        )
        return response.status_code == 200

    def kill_switch(self, flag_key: str, project_key: str, api_key: str) -> bool:
        """Immediately disable a feature flag (emergency rollback)."""
        import requests
        response = requests.patch(
            f"https://app.launchdarkly.com/api/v2/flags/{project_key}/{flag_key}",
            headers={"Authorization": api_key, "Content-Type": "application/json"},
            json=[{"op": "replace", "path": "/environments/production/on", "value": False}]
        )
        return response.status_code == 200
```

---

## Deployment Window Enforcement

### Enforcement Implementation

The deployment window enforcer is a policy component that intercepts deployment requests and evaluates them against the change window registry:

```python
# window_enforcer.py
from datetime import datetime, timezone
from typing import Optional
import pytz

class ChangeWindowEnforcer:
    def __init__(self, window_registry: ChangeWindowRegistry):
        self.registry = window_registry

    def check_deployment_allowed(
        self,
        environment: str,
        change_type: str,
        requested_at: Optional[datetime] = None
    ) -> tuple[bool, str]:
        """
        Check whether a deployment is allowed at the given time.
        Returns (allowed: bool, reason: str).
        """
        now = requested_at or datetime.now(timezone.utc)

        # Emergency changes bypass window enforcement
        if change_type == "emergency":
            return True, "Emergency change — window enforcement bypassed"

        # Check for active blackout periods
        active_blackout = self.registry.get_active_blackout(environment, now)
        if active_blackout:
            return False, (
                f"Deployment blocked: active blackout period '{active_blackout.name}' "
                f"until {active_blackout.end.isoformat()}. "
                f"Reason: {active_blackout.rationale}"
            )

        # Check for active change window
        active_window = self.registry.get_active_window(environment, now)
        if active_window:
            return True, f"Deployment allowed within change window '{active_window.name}'"

        # Find next available window
        next_window = self.registry.get_next_window(environment, now)
        if next_window:
            return False, (
                f"Deployment blocked: no active change window for {environment}. "
                f"Next window: '{next_window.name}' opens at {next_window.next_start.isoformat()}. "
                f"Request an emergency change authorization if this cannot wait."
            )

        return False, f"No change windows configured for environment {environment}. Contact platform team."
```

---

## Rollback Automation Configuration

### Argo Rollouts Automated Rollback

```yaml
# analysis-template-rollback.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: automated-rollback-analysis
  namespace: production
spec:
  metrics:
    - name: error-rate
      interval: 1m
      failureLimit: 3
      successCondition: result[0] < 0.03
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{
              namespace="{{args.namespace}}",
              service="{{args.service}}",
              status_code=~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              namespace="{{args.namespace}}",
              service="{{args.service}}"
            }[2m]))

    - name: p99-latency
      interval: 1m
      failureLimit: 5
      successCondition: result[0] < 0.5
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99, sum(rate(
              http_request_duration_seconds_bucket{
                namespace="{{args.namespace}}",
                service="{{args.service}}"
              }[2m]
            )) by (le))

    - name: notify-on-rollback
      provider:
        job:
          spec:
            template:
              spec:
                containers:
                  - name: notify
                    image: curlimages/curl:latest
                    command: [sh, -c]
                    args:
                      - |
                        curl -X POST $SLACK_WEBHOOK_URL \
                          -H 'Content-Type: application/json' \
                          -d '{"text":"AUTOMATED ROLLBACK TRIGGERED for {{args.service}} in {{args.namespace}}. Error rate or latency threshold breached. Initiating rollback now."}'
                restartPolicy: Never
```

### Rollback Notification and Incident Creation

```python
# rollback_handler.py
class RollbackHandler:
    def __init__(self, itsm_client, notification_client, orchestration_client):
        self.itsm = itsm_client
        self.notify = notification_client
        self.orchestration = orchestration_client

    def handle_rollback_completion(self, deployment_id: str, rollback_trigger: dict):
        """Handle post-rollback actions: create incident, notify stakeholders, update audit trail."""

        deployment = self.orchestration.get_deployment(deployment_id)

        # Create incident in ITSM
        incident = self.itsm.create_incident({
            "short_description": f"Automated rollback triggered for {deployment.service_name}",
            "description": (
                f"Deployment {deployment_id} of {deployment.service_name} "
                f"v{deployment.artifact_version} to {deployment.environment} "
                f"was automatically rolled back.\n\n"
                f"Trigger: {rollback_trigger['type']} — "
                f"{rollback_trigger['metric']}: {rollback_trigger['observed_value']} "
                f"(threshold: {rollback_trigger['threshold']})\n\n"
                f"Deployment started: {deployment.started_at}\n"
                f"Rollback completed: {rollback_trigger['completed_at']}\n"
                f"Previous version restored: {deployment.previous_artifact_version}"
            ),
            "priority": self._determine_incident_priority(deployment.service_tier),
            "assignment_group": deployment.on_call_group,
            "related_change": deployment.change_request_id
        })

        # Notify on-call and release manager
        self.notify.send_urgent(
            channels=[deployment.slack_alert_channel, "#release-incidents"],
            message={
                "title": f"ROLLBACK: {deployment.service_name} ({deployment.environment})",
                "body": f"Automated rollback completed. Incident: {incident.number}",
                "trigger": rollback_trigger,
                "action_items": [
                    "Review deployment logs and post-rollback metrics",
                    "Update incident with root cause analysis",
                    "Schedule post-mortem if P1/P2"
                ]
            }
        )

        # Record in audit trail
        self.orchestration.record_audit_event(
            event_type="deployment.rolled_back",
            deployment_id=deployment_id,
            actor={"type": "system", "identity": "rollback-automation"},
            details={
                "trigger": rollback_trigger,
                "incident_number": incident.number,
                "rollback_duration_seconds": rollback_trigger.get("duration_seconds")
            }
        )
