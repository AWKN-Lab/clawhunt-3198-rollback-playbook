# Acceptance evidence — problem 3198

Run date: 2026-11-23 (UTC). Host: Linux, Docker 26.1.3, Compose v2.27.0.
Paths sanitized (`$REPO` = repository root).

## AC 1 — fault version triggers automatic rollback to previous_tag, smoke passes

Fault image: `ops/app/Dockerfile.bad` sets `READYZ_FORCE_FAIL=1`, so `/readyz` always returns 500.

```bash
bash ops/deploy/deploy.sh --env test --image-tag v2-bad --log-name deploy-3198-baddrill
```

Deploy exit code: **33**. Automatic rollback exit code: **0**.

```text
2026-11-23T12:24:47Z [INFO] STEP=migration rc=0 tag=v2-bad target=head
2026-11-23T12:25:32Z [WARN] health node=app-a attempt=1 FAIL (healthz/readyz not both 200)
...
2026-11-23T12:25:51Z [WARN] health node=app-a attempt=10 FAIL (healthz/readyz not both 200)
2026-11-23T12:25:53Z [ERROR] health node=app-a gate FAILED after 10 attempts
2026-11-23T12:25:53Z [ERROR] auto rollback invoked reason=health_gate_failed_app-a
2026-11-23T12:27:27Z [INFO] STEP=rollback rc=0 reason=health_gate_failed_app-a
```

Rollback log:

```text
2026-11-23T12:25:54Z [INFO] ROLLBACK_PLAN env=test from=v2-bad to=v1 reason=health_gate_failed_app-a with_db_downgrade=0 drain=30s
2026-11-23T12:26:42Z [INFO] node_rollback_end node=app-a ts=2026-11-23T12:26:42Z elapsed=48s (threshold 180s)
2026-11-23T12:27:27Z [INFO] node_rollback_end node=app-b ts=2026-11-23T12:27:27Z elapsed=45s (threshold 180s)
2026-11-23T12:27:27Z [INFO] STEP=rollback_app rc=0 nodes=2 target=v1
2026-11-23T12:27:27Z [INFO] smoke /api/v1/ping -> 200
2026-11-23T12:27:27Z [INFO] smoke /api/v1/orgs -> 200
2026-11-23T12:27:27Z [INFO] smoke /api/v1/sites -> 200
2026-11-23T12:27:27Z [INFO] smoke /api/v1/members -> 200
2026-11-23T12:27:27Z [INFO] smoke /api/v1/resources -> 200
2026-11-23T12:27:27Z [INFO] STEP=rollback_smoke rc=0 endpoints=5
2026-11-23T12:27:27Z [INFO] STEP=rollback_done rc=0 from=v2-bad to=v1 duration=94s
```

State after the run: `current_tag=v1`, `last_rollback_from=v2-bad`, `status=rolled_back`.

Artifacts: `evidence/logs/deploy-3198-baddrill.log`,
`evidence/logs/deploy-3198-baddrill.rollback.log`.

## AC 2 — no downgrade without --with-db-downgrade

```bash
bash ops/deploy/rollback.sh --env test --target-tag v1 --reason manual-drill --log-name drill-3198-step3-rollback-v1
```

Exit code: **0**

```text
2026-11-23T12:20:38Z [INFO] STEP=rollback_app rc=0 nodes=2 target=v1
2026-11-23T12:20:38Z [INFO] DB_DOWNGRADE_SKIPPED (no --with-db-downgrade); 迁移遵循只加不删，旧代码可运行在新 schema 上
2026-11-23T12:20:41Z [INFO] STEP=rollback_done rc=0 from=v2 to=v1 duration=116s
```

No `alembic downgrade` command was executed in this run.

## AC 3 — per-node rollback time ≤ 3 minutes

Manual drill (`drill-3198-step3-rollback-v1.log`):

```text
2026-11-23T12:19:34Z [INFO] node_rollback_end node=app-a ts=2026-11-23T12:19:34Z elapsed=51s (threshold 180s)
2026-11-23T12:20:38Z [INFO] node_rollback_end node=app-b ts=2026-11-23T12:20:38Z elapsed=64s (threshold 180s)
```

Auto rollback (`deploy-3198-baddrill.rollback.log`): 48s and 45s. All values ≤ 180s.

## AC 4 — add-only migration policy and exception approval

Documented in `README.md` ("Rollback behaviour") and `docs/MIGRATION-COMPAT.md`:

- Add-only: new tables/columns/indexes only; destructive changes wait for the next window.
- Rollback does not downgrade by default; old code must run on the new schema.
- Downgrade requires: tested `downgrade()`, `migration_reversible=true`, a fresh backup, and
  `--with-db-downgrade --confirm-token CONFIRM-DOWNGRADE` (otherwise exit 43/44/45).

## Four-step drill (old → new → old + migration round-trip)

| Step | Log file | Exit code | Key line |
|---|---|---|---|
| 1 old image v1 | `drill-3198-step1-old-v1.log` | **0** | `STEP=deploy_done rc=0 env=test tag=v1 duration=189s` |
| 2 new image v2 | `drill-3198-step2-new-v2.log` | **0** | `STEP=deploy_done rc=0 env=test tag=v2 duration=121s` |
| 3 back to v1 | `drill-3198-step3-rollback-v1.log` | **0** | `STEP=rollback_done rc=0 from=v2 to=v1 duration=116s` |
| 4 migration round-trip | `migration-roundtrip.log` | **0 / 0 / 0** | `EXIT_CODE upgrade_head=0`, `downgrade_0001=0`, `upgrade_head_again=0`, final `0002 (head)` |

Migration evidence excerpt:

```text
### upgrade_head: alembic upgrade head
### EXIT_CODE upgrade_head=0
### downgrade_0001: alembic downgrade 0001
### EXIT_CODE downgrade_0001=0
### upgrade_head_again: alembic upgrade head
### EXIT_CODE upgrade_head_again=0
### current: alembic current
0002 (head)
```

## Artifact index

| File | Content |
|---|---|
| `evidence/logs/deploy-3198-baddrill.log` | fault deploy, health gate failure, exit 33 |
| `evidence/logs/deploy-3198-baddrill.rollback.log` | automatic rollback, smoke 5/5, exit 0 |
| `evidence/logs/drill-3198-step1-old-v1.log` | step 1, exit 0 |
| `evidence/logs/drill-3198-step2-new-v2.log` | step 2, exit 0 |
| `evidence/logs/drill-3198-step3-rollback-v1.log` | step 3, skip-downgrade evidence, exit 0 |
| `evidence/logs/migration-roundtrip.log` | upgrade/downgrade/upgrade, 0/0/0 |
| `evidence/logs/recover-baddrill.log` | recovery rollback after the first faulty attempt |

## Repository verification

- `bash scripts/secret-scan.sh` → `scan_result=PASS` (private keys, GitHub/cloud tokens, password
  values, private IPs, local absolute paths, e-mails, bid/agent identifiers, hostname leaks, `.env`
  files: all zero findings).
- `sha256sum -c MANIFEST.sha256` executed on a fresh clone of this repository → 0 failures.
