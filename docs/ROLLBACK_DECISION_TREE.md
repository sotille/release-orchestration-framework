# Rollback Decision Tree

This document provides a structured decision framework for choosing between rolling back vs. rolling forward when a production deployment exhibits problems.

## Quick reference

```
                   Is the system functional for end users?
                              /              \
                           NO                 YES
                            |                  |
                  [ROLLBACK IMMEDIATELY]      Is the bug a security issue?
                                                 /          \
                                              YES            NO
                                               |              |
                                     Is fix in <2h?     Is impact >5% users?
                                          /    \           /          \
                                       YES    NO        YES            NO
                                        |     |         |              |
                              [ROLL FORWARD]  |    [ROLLBACK]    [ROLL FORWARD]
                                          [ROLLBACK]              with hotfix priority
```

## Decision criteria

### Criterion 1 — Is the system functional?

**Functional = end users can complete primary workflows.**

If NO → immediate rollback. Do not debate.

### Criterion 2 — Security issue?

**Security issue = exposed data, broken auth, escalation of privilege, RCE.**

If YES → rollback unless the fix is trivial (config change, <30 min).

### Criterion 3 — Is the fix in <2 hours?

**Fix in <2h = fix is identified, code is written, testing is done, deployment is ready.**

If NO → rollback while you work on the fix.

### Criterion 4 — Is impact >5% of users?

**Impact = users hitting errors, degraded experience, missed transactions.**

If YES → rollback and prioritize the fix.

## When NOT to rollback

- The database schema changed and rollback would require data migration backward (consider forward-fix or partial rollback)
- The release is the dependency for downstream systems that have also released
- The bug is a known issue with documented workaround (degrade gracefully, fix in next sprint)

## Rollback execution checklist

- [ ] Rollback target version identified (last known good)
- [ ] Stakeholders notified (engineering, support, business)
- [ ] Rollback deployment triggered via standard pipeline (no manual hot-edit of production)
- [ ] Smoke tests passed on rolled-back version
- [ ] Monitoring confirms error rates returning to baseline
- [ ] Post-incident review scheduled within 5 business days
- [ ] Root cause documented in CHANGELOG / RCA

## Post-incident analysis

Every rollback triggers a post-incident review:

1. Why did the issue reach production?
2. Why did the gates not catch it?
3. What change to the pipeline would have caught it?
4. Was the rollback procedure smooth? If not, what improvement?

Document in `docs/incidents/` or your incident registry.
