# Contributing to the Techstream Release Orchestration Framework

Thank you for your interest in contributing. This repository provides governance patterns, approval workflow designs, deployment strategy guidance, and operational best practices for managing production releases securely and reliably. Contributions that improve technical depth, reflect current tooling (GitOps, Argo Rollouts, Flagger, ITSM integrations), and expand coverage of deployment patterns are welcome.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [What We Welcome](#what-we-welcome)
- [What We Do Not Accept](#what-we-do-not-accept)
- [How to Contribute](#how-to-contribute)
- [Documentation Standards](#documentation-standards)
- [Review Process](#review-process)
- [License](#license)

---

## Code of Conduct

All contributors are expected to engage professionally and constructively. Contributions that are dismissive, personal, or unprofessional will not be reviewed.

---

## What We Welcome

- **GitOps release pattern guidance** — Argo CD, Flux, and GitOps-native release orchestration patterns are an evolving area where additional depth is valuable.
- **Kubernetes-native deployment strategy examples** — Argo Rollouts, Flagger, and Helm-based release patterns with working configuration examples.
- **ITSM integration patterns** — ServiceNow, Jira Service Management, and PagerDuty integration guidance for change management workflows.
- **Multi-cloud and multi-region release patterns** — coordinated release strategies for organizations deploying across multiple cloud providers or regions.
- **Metrics and SLO-based promotion guidance** — integrating observability platforms (Prometheus, Datadog, New Relic) with automated promotion decisions.
- **Regulatory compliance guidance** — change control requirements for PCI DSS, SOX, HIPAA, and FedRAMP, mapped to the framework's approval workflow model.
- **Rollback procedure documentation** — concrete, tested rollback procedures for specific deployment technologies.

---

## What We Do Not Accept

- Vendor promotional content or marketing language.
- Guidance for scope areas covered by other Techstream repositories (CI/CD pipeline security → secure-ci-cd-reference-architecture, cloud infrastructure → cloud-security-devsecops).
- Major structural changes without prior issue discussion.

---

## How to Contribute

### Reporting Issues

Use GitHub Issues to report: outdated tool references, missing deployment strategy coverage, inaccurate ITSM integration guidance, or gaps in compliance mapping.

### Submitting Pull Requests

1. Fork and branch from `main`.
2. Make changes following the documentation standards below.
3. Verify that any workflow configuration examples (YAML, Groovy, JSON) are syntactically correct.
4. Open a pull request with a clear description and references for technical claims.

---

## Documentation Standards

- Professional, technical tone for a practitioner audience (release engineers, platform engineers, DevOps leads).
- Configuration examples must be syntactically valid and use placeholder values.
- Mermaid diagrams for approval workflows, deployment stage flows, and promotion pipelines.
- ATX headers, fenced code blocks with language identifiers, relative internal links.

---

## Review Process

Pull requests are reviewed for technical correctness, scope alignment, and documentation standards. Initial responses within 5 business days.

---

## License

By contributing, you agree your contributions will be licensed under the [Apache License 2.0](LICENSE).
