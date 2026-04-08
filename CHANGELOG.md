# Changelog

All notable changes to the Release Orchestration Framework are documented here.
Format: `[version] — [date] — [summary of changes]`

---

## [Unreleased]

- Added CHANGELOG.md (this file) for version tracking
- Added "Learning Resources" section to README.md linking to Book 4, techstream-learn labs, and techstream.app
- docs/best-practices.md: Completed practice 45 body text; added Approval Workflow Design Patterns section covering risk-tiered approval design (3-tier model with GitHub Actions implementation), emergency change process, and approval latency monitoring metrics (2026-04-07)
- [2026-04-08] Created docs/dora-reporting-guide.md — DORA metrics collection, calculation, and executive reporting guide: methodology for each of the four metrics (deployment frequency, lead time for changes P50/P90, change failure rate, failed deployment recovery time) with data sources and exclusion rules; DORA performance band reference table (Elite/High/Medium/Low); reporting cadence and audience guide (engineering weekly, engineering monthly, CISO monthly, executive quarterly, annual board); narrative framing templates for each metric; DORA metrics and security controls correlation patterns (which controls increase/decrease which metrics and why); tooling reference table (DORA four-keys, Sleuth, LinearB, Jellyfish, custom GitHub API); five common reporting mistakes (team vs. aggregate, incident count vs. CFR, P50 vs. P90 confusion, lagging indicator attribution, single-day snapshots)

## [1.0.0] — 2024-01-15

- Initial public release of the Release Orchestration Framework
- Core framework documentation: introduction, architecture, framework, implementation, best-practices, roadmap
- GitOps architecture reference for declarative release management
- Progressive delivery patterns covering canary releases, blue-green deployments, and feature flags
- Multi-region deployment architecture and coordination patterns
- Database migration safety guide for zero-downtime deployments
- Release incident playbook for managing deployment failures and rollbacks
- Apache 2.0 license and contribution guidelines
