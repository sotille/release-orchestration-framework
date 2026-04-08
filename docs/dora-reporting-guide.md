# DORA Metrics Reporting Guide

## Overview

DORA metrics are leadership-facing performance indicators, not internal engineering dashboards. The four metrics — Deployment Frequency, Lead Time for Changes, Change Failure Rate, and Failed Deployment Recovery Time — are meaningful only when reported with consistent measurement methodology, appropriate baselines, and enough context to separate signal from noise.

This guide covers the collection, calculation, visualization, and reporting of DORA metrics for organizations using the Release Orchestration Framework. It complements the metric definitions in `implementation.md` and the engineering dashboard in `best-practices.md`.

---

## Measurement Methodology

Consistent measurement is more important than precision. An organization that measures DORA metrics the same way every quarter can track trends reliably. An organization that changes its methodology each quarter produces incomparable data.

Document and version-control your measurement methodology. Any methodology change requires a note in the reporting narrative and, where possible, calculation of prior periods under the new methodology for continuity.

### Deployment Frequency

**Definition:** How often the team deploys to production.

**What to count:** A deployment is a version promotion to the production environment. Count:
- Every production deployment, regardless of size (a single-line change counts the same as a major release)
- Automated and manual deployments
- Hotfixes and emergency releases

**What not to count:** Deployments to staging, pre-production, or canary that have not been promoted to 100% of production traffic.

**Calculation period:** Report as deployments per day (for Elite/High performers) or deployments per week or month (for Medium/Low performers). Use the same granularity consistently.

**Data sources:**
- Git deployment tags (if you tag production deployments in source control)
- CI/CD pipeline execution logs filtered to `environment: production` jobs
- Release tracking systems (PagerDuty releases, JIRA releases, GitHub releases)
- Deployment audit logs from the release orchestration system

**Exclusions:** If your organization excludes certain system types from DORA reporting (e.g., batch jobs, database migration-only releases), document the exclusion explicitly. Apply it consistently.

### Lead Time for Changes

**Definition:** The time from code commit to code running in production.

**What to measure:** For each change that reached production in the measurement period, calculate the elapsed time from the earliest commit in the change (the first commit not previously in production) to the production deployment event. Use the P50 (median) of this distribution as the primary metric; also report P90.

**Commit identification:** In trunk-based development, the first commit of each change is identifiable by examining the deploy diff. In feature-branch workflows, use the first commit to the feature branch as the start time, or the PR creation time if commits predate the PR.

**Calculation period:** Measure over a rolling 30-day or 90-day window, not a snapshot. Single-day measurements are meaningless for lead time.

**Data sources:**
- Git commit timestamps (`git log --format="%H %ai" --first-parent main`)
- Deployment event timestamps from CI/CD logs or deployment tracking
- PR creation and merge timestamps from GitHub/GitLab API

**Automation:** This metric requires tooling to compute. Manual calculation is error-prone and does not scale. Options:
- DORA four-keys BigQuery pipeline (open-source, from Google)
- Linear B, Sleuth, Jellyfish, or similar DORA platforms
- Custom instrumentation using your CI/CD API and git log

### Change Failure Rate

**Definition:** The percentage of deployments that cause a failure in production requiring a hotfix, rollback, or other corrective action.

**What counts as a failure:** A production deployment is a failure if it requires:
- A hotfix or patch release within 24 hours (or within your SLA window)
- A rollback to the prior version
- A feature flag disable triggered by a production defect (not a planned rollout change)
- A customer-impacting incident whose root cause is traced to the deployment

**What does not count as a failure:**
- Planned feature flag toggling as part of the release strategy
- Performance degradation that was expected and within SLA
- Issues discovered in canary that were caught before reaching 100% traffic (the canary prevented the failure)

**Calculation:**
```
Change Failure Rate = (Deployments that resulted in a failure / Total deployments) × 100
```

**Data sources:**
- Incident tracking system (PagerDuty, OpsGenie, Jira) — incidents linked to deployment events
- Rollback events in CI/CD logs
- Hotfix PR labels or emergency release tags in Git

**Measurement period:** Report as a percentage over a rolling 90-day window. A 30-day window is too noisy for CFR; a 90-day window smooths random variation while remaining responsive to trend changes.

### Failed Deployment Recovery Time

**Definition:** The time from failure detection to service restoration. DORA's original term was "Time to Restore Service"; some organizations use "MTTR" — use the terminology consistently in your reports.

**What to measure:** For each production failure caused by a deployment, measure the elapsed time from:
- **Start:** When the failure was first detected (alert fired, customer report received, monitor triggered — whichever is earliest)
- **End:** When service was restored to normal operation at normal traffic levels (not when the incident was closed — those are different events)

**Calculation:** Report P50 and P90 over a rolling 90-day window.

**Data sources:**
- Incident management system: alert time (start) and incident mitigation time (end)
- Deployment event system: rollback completion time
- Status page events: incident start and resolution times

---

## Performance Bands

DORA's research defines four performance bands. Use these for benchmarking and communication:

| Metric | Elite | High | Medium | Low |
|---|---|---|---|---|
| Deployment Frequency | Multiple times per day | Once per day to once per week | Once per week to once per month | Once per month to once every six months |
| Lead Time for Changes | < 1 hour | 1 day to 1 week | 1 week to 1 month | 1 month to 6 months |
| Change Failure Rate | 0–5% | 5–10% | 10–15% | 15–30% |
| Failed Deployment Recovery Time | < 1 hour | < 1 day | 1 day to 1 week | > 1 week |

**Reporting guidance:**
- Report current band alongside the numeric metric — "Lead Time: 3.2 days (High)" communicates more than "Lead Time: 3.2 days"
- Report trend direction: whether the organization is moving toward Elite or toward Low
- Avoid using DORA bands as targets without context. An organization at Low Deployment Frequency because it has a validated quarterly release cycle is different from one that deploys infrequently due to broken pipelines

---

## Reporting Cadence and Audiences

### Engineering Weekly (10 minutes)
Audience: Engineering leads, team tech leads
Metrics: All four metrics for each team/service, current period vs. prior week
Focus: Regressions requiring investigation; which teams are trending positively
Format: Table or scorecard; one slide or dashboard view

### Engineering Monthly (20 minutes)
Audience: Engineering Director, VPE, security team
Metrics: All four metrics, 90-day trend, team-by-team breakdown
Focus: Trend analysis, comparison to prior quarter, identification of systemic improvements or regressions
Format: Dashboard + 2-3 slides with narrative

### Executive Quarterly (5 minutes)
Audience: CTO, CISO, Board (via CTO)
Metrics: Aggregate organization metrics + high-level trend
Focus: Whether the organization is improving its software delivery performance and whether security controls are aiding or impeding delivery
Format: 1-2 slides, narrative framing, comparison to industry bands

### Annual Board Summary
Audience: Board of Directors, Audit Committee
Metrics: Year-over-year progress on all four metrics; correlation with business outcomes (uptime, incident frequency, customer impact)
Focus: Return on DevSecOps investment as demonstrated by delivery performance; risk reduction (CFR and Recovery Time as reliability indicators)
Format: 1 slide, narrative framing in CISO/CTO board letter

---

## Narrative Framing

Raw DORA numbers without narrative context are commonly misinterpreted. Each reporting period should include brief narrative for each metric:

**Deployment Frequency narrative template:**
```
Deployment Frequency this quarter: [N] per [day/week], [Elite/High/Medium/Low] band.
[If improving]: Increased from [prior] to [current] following [change — trunk-based development adoption, CI pipeline optimization, approval gate streamlining].
[If declining]: Declined from [prior] to [current]. Investigating [candidate causes — test suite instability, manual approval bottlenecks, deployment window restrictions].
```

**Lead Time narrative template:**
```
Lead Time P50: [N] [hours/days], P90: [N] [hours/days], [band].
[Context]: The P90 value reflects a tail of [large features with longer review cycles / cross-team dependencies / security review queues]. Median lead time for single-team changes is [N].
[If improving]: Reduction in P50 attributable to [PR review SLA enforcement, parallelized test suites, deployment automation for common change types].
```

**Change Failure Rate narrative template:**
```
Change Failure Rate: [N]%, [band], over [M] deployments.
[If within target]: [N] of [M] deployments required corrective action. Root causes: [summary].
[If elevated]: CFR elevated this quarter due to [specific incident cluster or systemic issue]. Root cause analysis complete; remediation actions: [list].
[Security context]: [N] of the [N] failures had a security component [or: None of the failures had a security component — security controls in the pipeline did not correlate with production failures this quarter].
```

---

## DORA Metrics and Security Controls

A common concern when introducing security controls into CI/CD pipelines is that they will degrade DORA metrics. Tracking DORA before and after security control deployments answers this question with data rather than assumption.

**What good security controls look like in DORA data:**
- SAST/SCA integrated into CI: No change to deployment frequency; modest increase in Lead Time if controls are run sequentially (parallelize to recover). Change Failure Rate decreases if the controls catch defects before production.
- Mandatory security review gates: Lead Time increases in the short term; stabilizes as security review becomes faster (champion programs, automated policy checks). CFR decreases as design-level security issues are caught earlier.
- Signature verification and SBOM requirements: No expected DORA impact; adds 1–2 minutes to pipeline; no change to deployment frequency or lead time at the P50 level.

**Red flags in DORA data indicating security control misconfiguration:**
- Deployment Frequency drops sharply after introducing a security gate: Gate is blocking too many legitimate deployments. Review gate thresholds, exception process, and whether the gate is evaluating findings that are not genuinely blocking.
- Lead Time P90 increases significantly while P50 is unchanged: A subset of changes is being held for manual security review. Investigate whether the review criteria are correctly defined and whether automation can replace the manual step.
- CFR increases after disabling a security control: The control was preventing production failures. Restore and track the correlation explicitly.

---

## Automation and Tooling Reference

| Tool | Metrics supported | Integration effort | Notes |
|---|---|---|---|
| DORA four-keys (Google) | All four | Medium (BigQuery + data pipeline) | Open-source; requires cloud data infrastructure |
| Sleuth | Deployment Frequency, Lead Time, CFR | Low (native CI/CD integration) | SaaS; paid |
| LinearB | All four | Low | SaaS; paid; strong Git integration |
| Jellyfish | All four + team-level breakdown | Medium | SaaS; paid; strong for team reporting |
| Custom (GitHub API + BigQuery/Redshift) | All four | High (build it yourself) | Full control; appropriate for orgs with privacy constraints |
| Grafana + Prometheus | Deployment Frequency, Recovery Time | Medium | Works well if deployments are tracked in Prometheus events |

For organizations already using GitHub Actions, the `release-orchestration-framework/templates/` deployment pipelines emit structured deployment events that can be consumed by any of the above tools or a custom pipeline.

---

## Common Reporting Mistakes

**Measuring teams separately but reporting only the aggregate.** An improving aggregate can hide a team that is regressing. Always decompose the aggregate when investigating trends.

**Conflating incident count with Change Failure Rate.** If you have many incidents but few are caused by deployments, CFR may be excellent while MTTR is poor. Report each metric independently.

**Treating DORA bands as absolute targets.** An organization delivering a regulated financial product may legitimately operate at High rather than Elite deployment frequency due to change control requirements. The goal is continuous improvement from the current baseline, not achieving Elite in all dimensions simultaneously.

**Not distinguishing P50 from P90 for Lead Time and Recovery Time.** The P50 tells you about typical performance; the P90 tells you about the tail. A great P50 with a poor P90 indicates a small number of changes are stuck. A poor P50 with a great P90 is unusual but indicates measurement inconsistency.

**Attributing a CFR change to the wrong cause.** CFR is a lagging indicator — it reflects deployment quality decisions made 1–3 weeks earlier. A CFR improvement this quarter may reflect security controls or quality improvements deployed last quarter. Maintain a deployment change log to correlate metric changes with their causes.
