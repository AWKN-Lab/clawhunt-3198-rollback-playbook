# clawhunt-3198-rollback-playbook

Rollback playbook: **two-node image rollback with the same drain → swap → health → attach flow as
deployment, database downgrade off by default (add-only migrations), controlled downgrade behind a
double confirmation, configuration rollback, post-rollback smoke test and ops notification.**

- Verification record: [ACCEPTANCE.md](ACCEPTANCE.md)
- Evidence: [ACCEPTANCE-EVIDENCE.md](ACCEPTANCE-EVIDENCE.md)
- Original AC mapping: [docs/ac-3198.md](docs/ac-3198.md)
- Migration compatibility policy: [docs/MIGRATION-COMPAT.md](docs/MIGRATION-COMPAT.md)

## Repository layout

```
ops/deploy/rollback.sh            rollback orchestration (state → per-node rollback → DB/config → smoke)
ops/deploy/deploy.sh              used to stage the old/new versions in the drill; also the auto-rollback caller
ops/deploy/bootstrap.sh           first-time environment bring-up
ops/deploy/lib/common.sh          logging, lock, LB drain/attach, health, smoke, backup
ops/deploy/lib/state.py           atomic state/{env}.json read/write
ops/deploy/state/README.md        state file contract (previous/current tag, revision, reversible)
ops/deploy/compose/               PostgreSQL 16, Redis 7, app-a/app-b, nginx LB
ops/app/                          FastAPI + SQLAlchemy 2 + Alembic demo service (0001 init, 0002 add-only)
.github/workflows/deploy.yml      workflow_dispatch: mode=rollback, env, target_tag, with_db_downgrade
scripts/secret-scan.sh            key / private-IP / local-path / email / id scanner
```

## Rollback behaviour

```bash
bash ops/deploy/rollback.sh --env test                       # target = state.previous_tag, DB untouched
bash ops/deploy/rollback.sh --env test --target-tag v1       # explicit target
bash ops/deploy/rollback.sh --env test --with-db-downgrade --confirm-token CONFIRM-DOWNGRADE
```

Per node: remove from LB upstream → reload → drain 30s → recreate with the target tag →
`/healthz` and `/readyz` three consecutive 200 → attach back. A node that fails its gate stays
detached; the other node keeps serving. Per-node elapsed time is asserted ≤ 180 s.

Database policy is **add-only** (只加不删): new columns are nullable or have defaults, old code keeps
running on the new schema, and rollback does **not** run `alembic downgrade` unless all four
conditions hold — (1) the migration has a tested `downgrade()`, (2) `state.migration_reversible=true`,
(3) a fresh backup was taken, (4) the operator passes `--with-db-downgrade --confirm-token
CONFIRM-DOWNGRADE`. Missing any of them exits 43/44/45. Details: [docs/MIGRATION-COMPAT.md](docs/MIGRATION-COMPAT.md).

## Exit codes

| Code | Meaning |
|---|---|
| 11 | lock held by another run |
| 12 | no rollback target / target image missing |
| 43 / 44 | DB downgrade refused: not reversible / no second confirmation |
| 45 / 46 | DB backup failed / downgrade failed |
| 47 | post-rollback smoke failed |
| 51-55 | per-node rollback step failed (detach/up/down/health/attach/over 180s) |

## Scope and honest limitations

1. Image tags resolve from the local image store, not a private registry.
2. GitHub Actions workflow is delivered as configuration only; it was not executed here.
3. The target host blocks container-to-container TCP, so services run with `network_mode: host`
   on loopback ports (28100 LB, 28101/28102 nodes, 28379 Redis, 28432 PostgreSQL).
4. Credential rollback is modelled as a version pointer in `state/{env}.secrets.json`; swap for the
   real vault adapter in production.
5. No `.env`, tokens, private keys, private IPs or local paths are included. Run
   `bash scripts/secret-scan.sh`.
