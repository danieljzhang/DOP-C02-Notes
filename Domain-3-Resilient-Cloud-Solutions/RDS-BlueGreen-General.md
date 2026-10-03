# RDS Blue/Green Deployments - DOP-C02 Exam Notes

> **Launched:** November 2022 for RDS MySQL, MariaDB, and Aurora MySQL. PostgreSQL support added later. This is a tested resilience and deployment pattern — know it alongside Multi-AZ and read replicas.

## 1. Overview

**What it is:** A managed deployment mechanism that creates a staging environment (green) that mirrors your production database (blue), lets you make and validate changes on green, then switches traffic from blue to green with minimal downtime — typically under 1 minute.

**What problem it solves:** Major database changes (engine upgrades, schema changes, parameter group changes) traditionally require a maintenance window with significant downtime. Blue/green deployments let you prepare and test the change in advance, then cut over quickly.

---

## 2. How It Works

```
Blue (production)          Green (staging — copy of blue)
├── Primary instance   →   ├── Primary instance (changes applied here)
└── Read replicas          └── Read replicas

Logical replication keeps green in sync with blue during preparation.

Switchover:
1. RDS quiesces writes on blue
2. Waits for green to catch up (replication lag → 0)
3. Promotes green to production
4. Renames endpoints (green gets blue's DNS names)
5. Blue becomes the old environment (kept for rollback)
```

The key: **endpoints don't change**. Applications keep the same connection strings — the DNS names are swapped under the hood.

---

## 3. Supported Engines

| Engine | Support |
|---|---|
| RDS MySQL 5.7, 8.0 | ✅ |
| RDS MariaDB | ✅ |
| RDS PostgreSQL | ✅ |
| Aurora MySQL | ✅ |
| Aurora PostgreSQL | ✅ |
| RDS Oracle / SQL Server | ❌ Not supported |

---

## 4. What You Can Change on Green Before Switchover

- **Engine version** (minor and major upgrades)
- **Instance class** (resize up or down)
- **Parameter groups** (change DB parameters)
- **Schema changes** (DDL — add columns, indexes, etc.)
- **Storage type** (e.g., gp2 → gp3)

You cannot change the engine type (e.g., MySQL → PostgreSQL) — that's a migration, not a blue/green deployment.

---

## 5. Switchover

```bash
# Create the green environment
aws rds create-blue-green-deployment \
  --blue-green-deployment-name my-bg-deployment \
  --source arn:aws:rds:us-east-1:123456789012:db:prod-mysql \
  --target-engine-version 8.0.35 \
  --target-db-instance-class db.r6g.xlarge

# Apply changes to green (e.g., parameter group)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-green-instance \
  --db-parameter-group-name new-param-group \
  --apply-immediately

# Validate green (run tests against green endpoint)

# Switchover — RDS handles the cutover
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier bgd-xxxxxxxxx \
  --switchover-timeout 300  # seconds; default 300, max 3600
```

The `--switchover-timeout` controls how long RDS waits for replication lag to reach zero before aborting. If lag doesn't clear within the timeout, the switchover is cancelled and blue remains production.

---

## 6. After Switchover

- Green is now production (has blue's original endpoint DNS names)
- Blue is retained as the old environment — **you can roll back by switching again**
- Blue is not automatically deleted — you pay for it until you delete it
- Delete blue when you're confident the switchover was successful:

```bash
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier bgd-xxxxxxxxx \
  --delete-target  # also deletes the old blue instances
```

---

## 7. Blue/Green vs Other RDS Upgrade Approaches

| Approach | Downtime | Rollback | Complexity |
|---|---|---|---|
| **Blue/Green deployment** | < 1 minute | Easy (switch back) | Low |
| In-place minor upgrade | Minutes (Multi-AZ) | Snapshot restore | Low |
| In-place major upgrade | 10–30 min | Snapshot restore | Medium |
| Read replica promotion | Minutes | Manual | Medium |
| Snapshot restore to new instance | Hours | N/A | High |

**Exam angle:** "Upgrade RDS with minimal downtime and easy rollback" → Blue/Green deployment.

---

## 8. Limitations

- Not supported for RDS Oracle or SQL Server
- Green environment must be in the same Region as blue
- Cannot use blue/green if the source has cross-Region read replicas
- External replication (from outside RDS) is not supported
- The green environment cannot itself be a read replica source during the deployment

---

## 9. Common Exam Scenarios

### Scenario 1: Upgrade MySQL 5.7 to 8.0 with minimal downtime
**Solution:** Create a blue/green deployment targeting engine version 8.0. Test the green environment. Run `switchover-blue-green-deployment`. Downtime is under 1 minute. Keep blue for rollback, delete after validation.

### Scenario 2: Test a schema change before applying to production
**Solution:** Create blue/green deployment. Apply DDL changes to green. Run application integration tests against the green endpoint. Switchover when validated. If tests fail, delete the green environment — blue is untouched.

### Scenario 3: Resize production RDS instance class with rollback option
**Solution:** Blue/green deployment with `--target-db-instance-class` set to the new size. Validate performance on green. Switchover. If performance regresses, switch back to blue.

---

## 10. Exam Tips

- Blue/green is **not the same as Multi-AZ** — Multi-AZ is for HA/failover; blue/green is for deployments/upgrades.
- The **endpoints swap** — applications don't need connection string changes.
- Blue is **kept after switchover** — you pay for it; delete it explicitly.
- `switchover-timeout` is the replication-lag deadline, not the total switchover time.
- Not supported for Oracle or SQL Server — if the exam mentions those engines, blue/green is a distractor.
- "Minimal downtime major version upgrade with rollback" → Blue/Green is the answer over snapshot restore.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
