# Database Migration Safety

Database migrations are among the highest-risk operations in a release. Unlike application code, a botched database migration can cause irreversible data loss, corrupt production data, or lock tables long enough to cause cascading failures. This guide defines the classification system, safety patterns, tooling requirements, and runbooks for executing database schema changes safely within the Techstream release orchestration model.

---

## Migration Risk Classification

Not all migrations carry the same risk. A classification system allows teams to apply controls proportional to the actual risk rather than applying maximum caution to every `ALTER TABLE`.

### Class A — Safe (Low Risk)

Class A migrations are non-destructive, reversible, and backward-compatible with the running application version. They can be deployed independently of an application deployment.

**Examples:**
- Adding a nullable column
- Creating a new index (created `CONCURRENTLY` to avoid table locks)
- Creating a new table
- Adding a new foreign key to a new table
- Modifying a column default where the old default is no longer used

**Release requirements:**
- Can be deployed in a standard release window
- No application-level deployment required before or after
- Single approval (automated gates + deployer)

### Class B — Conditional (Medium Risk)

Class B migrations require coordination with the application deployment. They are backward-compatible for a transition period but must be followed by a cleanup migration.

**Examples:**
- Renaming a column (requires: add new column → deploy app using both → backfill → deploy app using only new column → drop old column)
- Making a nullable column non-nullable (requires existing NULLs to be backfilled first)
- Changing a column type with an implicit cast (e.g., `VARCHAR(255)` → `TEXT`)
- Adding a `NOT NULL` constraint without a default

**Release requirements:**
- Explicitly sequenced with application deployment (migration may precede or follow, depending on pattern)
- Break migration into multiple steps if needed
- Expanded change window
- Second approver required (senior engineer or DBA)

### Class C — Destructive (High Risk)

Class C migrations delete or permanently alter data. They are irreversible and require a full backup and explicit data safety review.

**Examples:**
- Dropping a column
- Dropping a table
- Truncating a table
- Running a `DELETE` or `UPDATE` that modifies a large fraction of rows
- Changing a column type with a lossy cast

**Release requirements:**
- Pre-migration full backup mandatory (verified restorable)
- Database lead and release manager approval required
- Deployment in a dedicated extended change window
- Data recovery runbook prepared before migration begins
- Staging dry run with production-sized dataset (or production data copy) required

### Class D — Dangerous (Critical Risk)

Class D migrations affect locking, replication, or schema consistency at a level that can cause production outages regardless of correctness.

**Examples:**
- `ALTER TABLE` operations that acquire exclusive locks on large tables (depends on engine — see engine-specific guidance below)
- Modifying primary key type
- Changing character set or collation on large tables
- Adding a foreign key constraint on a large existing table (unless deferred)
- Adding a check constraint on an existing table (unless `NOT VALID` + `VALIDATE CONSTRAINT`)

**Release requirements:**
- All Class C requirements apply
- Platform engineering + DBA mandatory review
- Full load test in staging using production-scale data
- Online DDL tooling required (pt-online-schema-change, gh-ost, or engine-native online DDL)
- Traffic reduction or off-hours execution required

---

## Engine-Specific Lock Behavior

Understanding which operations acquire locks — and for how long — is essential for Class D migration planning.

### PostgreSQL

| Operation | Lock Acquired | Duration | Safe Alternative |
|-----------|--------------|----------|-----------------|
| `ADD COLUMN` (nullable, no default) | `AccessShareLock` | Instant metadata change | — |
| `ADD COLUMN NOT NULL DEFAULT <value>` | `AccessExclusiveLock` (PG < 11) | Duration of full rewrite | PG 11+: instant metadata change |
| `DROP COLUMN` | `AccessExclusiveLock` | Short (metadata change) | Mark as "droppable", do in off-hours |
| `CREATE INDEX` | `ShareLock` (blocks writes) | Duration of index build | `CREATE INDEX CONCURRENTLY` |
| `ALTER COLUMN TYPE` | `AccessExclusiveLock` | Duration of full rewrite | Add new column + migrate data + swap |
| `ADD FOREIGN KEY` | `ShareRowExclusiveLock` | Full table scan for validation | `ADD FOREIGN KEY NOT VALID` + `VALIDATE CONSTRAINT` |
| `ADD CHECK CONSTRAINT` | `AccessShareLock` (PG 14+) | Short | `ADD CONSTRAINT ... NOT VALID` then validate separately |

**PostgreSQL online index creation:**
```sql
-- Always use CONCURRENTLY for production index creation
-- This avoids the ShareLock that blocks writes during the full index build
CREATE INDEX CONCURRENTLY idx_orders_customer_id
  ON orders (customer_id)
  WHERE status != 'archived';

-- Non-concurrent (only for new tables or maintenance windows):
-- CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

**PostgreSQL foreign key with NOT VALID:**
```sql
-- Step 1: Add constraint without validation (no full table scan)
ALTER TABLE order_items
  ADD CONSTRAINT fk_order_items_orders
  FOREIGN KEY (order_id) REFERENCES orders(id)
  NOT VALID;

-- Step 2: Validate in a separate transaction (ShareUpdateExclusiveLock — does not block reads/writes)
ALTER TABLE order_items VALIDATE CONSTRAINT fk_order_items_orders;
```

### MySQL / Aurora MySQL

| Operation | Lock Behavior |
|-----------|--------------|
| `ADD COLUMN` (8.0+) | Online if `ALGORITHM=INSTANT` supported |
| `ADD INDEX` | Online (`ALGORITHM=INPLACE`) — does not block reads/writes |
| `DROP COLUMN` | `ALGORITHM=COPY` — full table copy, blocks writes |
| `CHANGE COLUMN` (type change) | Often requires full copy |
| `ALTER TABLE ... ENGINE=InnoDB` | Full table copy |

**MySQL gh-ost for large tables:**
```bash
# gh-ost: Online schema change tool for MySQL without triggers
gh-ost \
  --host=db-primary.internal \
  --database=orders_db \
  --table=orders \
  --alter="ADD COLUMN archived_at TIMESTAMP NULL" \
  --chunk-size=500 \
  --max-load=Threads_running=50 \
  --throttle-additional-flag-file=/tmp/gh-ost.pause \
  --panic-flag-file=/tmp/gh-ost.panic \
  --execute
```

---

## Safe Migration Patterns

### Expand-Contract (Rename Column)

Never rename a column in a single step. The running application will fail to read/write the column by its new name.

**Step 1 — Expand: Add new column**
```sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
```
**Step 2 — Backfill**
```sql
UPDATE users SET full_name = display_name WHERE full_name IS NULL;
```
**Step 3 — Deploy application using both columns**
(Application reads `full_name`, falls back to `display_name` if null)

**Step 4 — Verify: Zero reads from old column**
(Monitor application metrics to confirm old column is no longer queried)

**Step 5 — Contract: Drop old column (Class C)**
```sql
ALTER TABLE users DROP COLUMN display_name;
```

### Add NOT NULL Column

```sql
-- Step 1: Add nullable with default
ALTER TABLE orders ADD COLUMN tenant_id VARCHAR(64) NULL;

-- Step 2: Backfill existing rows
UPDATE orders SET tenant_id = 'default' WHERE tenant_id IS NULL;

-- Step 3: Add NOT NULL constraint after backfill confirmed
ALTER TABLE orders ALTER COLUMN tenant_id SET NOT NULL;
```

### Large Table Batched Update

Avoid locking large tables with a single `UPDATE`. Batch updates in small transactions:

```python
# Python: Batched UPDATE for large tables
import psycopg2

def backfill_tenant_id(conn, batch_size=1000):
    with conn.cursor() as cur:
        while True:
            cur.execute("""
                UPDATE orders
                SET tenant_id = 'default'
                WHERE id IN (
                    SELECT id FROM orders
                    WHERE tenant_id IS NULL
                    LIMIT %s
                )
                RETURNING id
            """, (batch_size,))
            updated = cur.rowcount
            conn.commit()

            if updated == 0:
                break

            # Brief pause to reduce lock contention on busy tables
            import time
            time.sleep(0.1)
```

---

## Pre-Migration Safety Checklist

Complete this checklist before every Class B, C, or D migration:

### Planning
- [ ] Migration classified (A/B/C/D) with classification rationale documented
- [ ] Migration tested in staging environment against a recent production data copy
- [ ] Estimated execution time measured in staging — if > 5 minutes, reclassify as D and plan accordingly
- [ ] Rollback procedure documented and tested
- [ ] Application backward compatibility confirmed — old app version compatible with post-migration schema

### Backup and Recovery (Class C/D)
- [ ] Full database backup completed and backup completion verified
- [ ] Backup restoration tested (or restoration time estimated from backup size + IOPS)
- [ ] Point-in-time recovery (PITR) enabled and retention period confirmed
- [ ] Recovery time objective (RTO) documented for this migration

### Approvals
- [ ] Class A: Automated pipeline gate passed
- [ ] Class B: Senior engineer or DBA reviewed sequencing plan
- [ ] Class C: DBA reviewed; data safety confirmed; backup verified
- [ ] Class D: Platform engineering + DBA + release manager; load test passed

### Execution Readiness
- [ ] Change window confirmed and communicated to affected teams
- [ ] On-call engineer notified and available during migration
- [ ] Database monitoring dashboard open (query latency, lock wait time, replication lag)
- [ ] Rollback trigger criteria defined (e.g., migration > 2x estimated time, replication lag > 60s)
- [ ] Incident channel open

---

## Migration Execution Runbook

### Pre-Execution (T-30 minutes)

```bash
# 1. Verify backup is recent and complete
aws rds describe-db-snapshots \
  --db-instance-identifier prod-orders-db \
  --query 'DBSnapshots[?Status==`available`] | sort_by(@, &SnapshotCreateTime) | [-1]'

# 2. Confirm replication lag is within normal range
psql $DB_URL -c "
  SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
         (sent_lsn - replay_lsn) AS replication_lag_bytes
  FROM pg_stat_replication;
"

# 3. Record baseline performance
psql $DB_URL -c "
  SELECT count(*), wait_event_type, wait_event
  FROM pg_stat_activity
  WHERE state != 'idle'
  GROUP BY wait_event_type, wait_event;
"

# 4. Confirm migration is idempotent / has been reviewed
cat db/migrations/<migration_file>.sql
```

### During Execution

Monitor the following during Class C/D migrations:

```bash
# In a separate terminal — monitor long-running queries and locks
watch -n 5 "psql \$DB_URL -c \"
  SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
  FROM pg_stat_activity
  WHERE (now() - pg_stat_activity.query_start) > interval '30 seconds'
  ORDER BY duration DESC;
\""

# Monitor lock waits
watch -n 5 "psql \$DB_URL -c \"
  SELECT blocked_locks.pid AS blocked_pid,
         blocking_locks.pid AS blocking_pid,
         blocked_activity.query AS blocked_statement,
         blocking_activity.query AS current_statement_in_blocking_process
  FROM  pg_catalog.pg_locks blocked_locks
  JOIN  pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
  JOIN  pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  JOIN  pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
  WHERE NOT blocked_locks.granted;
\""
```

### Rollback Procedure

Define rollback before execution. For each migration type:

| Migration Type | Rollback Approach | Estimated Rollback Time |
|---------------|------------------|------------------------|
| Add nullable column | `DROP COLUMN` (Class C) | Seconds (metadata) |
| Add index (CONCURRENTLY) | `DROP INDEX CONCURRENTLY` | Seconds to minutes |
| Backfill data | Re-run backfill with original values; requires original values available | Depends on data volume |
| Drop column | Restore from backup (if PITR available, use PITR) | Minutes to hours |
| Class D (any) | Restore from pre-migration backup | RTO per backup size/IOPS |

**Rollback trigger criteria (example):**
- Migration exceeds 2x the staging-measured execution time
- Replication lag exceeds 120 seconds
- P99 query latency increases > 300% during migration
- Any application error rate increase > 200% during migration window

### Post-Execution Verification

```bash
# Verify schema matches expected state
psql $DB_URL -c "\d orders"  # Or use your migration tool's status command

# Verify application health
curl -s https://api.internal/healthz | jq '.database'

# Verify replication lag returned to baseline
psql $DB_URL -c "
  SELECT client_addr,
         (sent_lsn - replay_lsn) AS lag_bytes,
         CASE
           WHEN (sent_lsn - replay_lsn) < 10000 THEN 'OK'
           WHEN (sent_lsn - replay_lsn) < 1000000 THEN 'LAGGING'
           ELSE 'CRITICAL'
         END AS status
  FROM pg_stat_replication;
"

# Verify query performance (compare to pre-migration baseline)
psql $DB_URL -c "
  SELECT query, calls, total_exec_time, mean_exec_time, rows
  FROM pg_stat_statements
  WHERE query LIKE '%orders%'
  ORDER BY mean_exec_time DESC
  LIMIT 20;
"
```

---

## CI/CD Integration

### Migration Gating in Pipelines

```yaml
# GitHub Actions: migration safety gate
- name: Classify and gate database migrations
  run: |
    CHANGED_MIGRATIONS=$(git diff --name-only origin/main | grep 'db/migrations/' || true)

    if [ -z "$CHANGED_MIGRATIONS" ]; then
      echo "No migrations in this PR"
      exit 0
    fi

    echo "Migrations in this PR:"
    echo "$CHANGED_MIGRATIONS"

    # Run migration risk classifier
    python3 scripts/classify_migrations.py \
      --files "$CHANGED_MIGRATIONS" \
      --output migration-risk-report.json

    RISK_CLASS=$(jq -r '.max_risk_class' migration-risk-report.json)
    echo "Maximum migration risk class: $RISK_CLASS"

    case "$RISK_CLASS" in
      "D")
        echo "::error::Class D migration detected. Requires platform engineering review and load test."
        exit 1
        ;;
      "C")
        echo "::warning::Class C migration detected. DBA review and backup verification required."
        # Allow pipeline to continue but require specific approval environment
        echo "REQUIRES_DBA_APPROVAL=true" >> $GITHUB_ENV
        ;;
    esac
```

### Migration Dry Run in CI

```yaml
- name: Run migration dry run against staging
  run: |
    # Dry run verifies syntax and sequence without committing
    alembic --dry-run upgrade head  # Python/Alembic
    # flyway -dryRunOutput=migration-dry-run.sql migrate  # Java/Flyway
    # rails db:migrate:dry_run  # Ruby on Rails
```

---

## Related Techstream Resources

| Topic | Document |
|-------|---------|
| Release governance and approval | [Release Orchestration Framework](framework.md) |
| Progressive delivery strategies | [Progressive Delivery](progressive-delivery.md) |
| Rollback and incident response | [Release Incident Playbook](release-incident-playbook.md) |
| Change management evidence | [Compliance Automation — Evidence Collection](../../compliance-automation-framework/docs/evidence-collection-automation.md) |
| GitOps promotion patterns | [GitOps Architecture](gitops-architecture.md) |

*Part of the Techstream Release Orchestration Framework. Licensed under Apache 2.0.*
