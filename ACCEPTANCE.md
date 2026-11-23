# Acceptance — ClawHunt problem 3198 (rollback playbook)

Status: **PASS** — four acceptance criteria, including a full four-step drill
(old image → new image → old image → migration round-trip), verified on 2026-11-23.

Environment: Linux server, Docker 26.1.3, Compose v2.27.0, two application nodes behind nginx 1.27,
PostgreSQL 16, Redis 7. All ports loopback-only.

| # | Acceptance criterion (verbatim) | Implementation | Result |
|---|---|---|---|
| 1 | test 演练中部署故障版本后自动回滚到 previous_tag，冒烟测试全部通过（附日志） | `deploy.sh` failure branch → `rollback.sh`; fault image `s4d-api:v2-bad` | **PASS** — deploy exit **33**, auto rollback exit **0**, `ROLLBACK_PLAN from=v2-bad to=v1`, nodes 48s/45s, smoke 5/5 × 200 |
| 2 | 未传 `--with-db-downgrade` 时脚本不执行任何 alembic downgrade（日志可见跳过说明） | `rollback.sh::db_rollback` | **PASS** — `DB_DOWNGRADE_SKIPPED (no --with-db-downgrade)` in the rollback log; no downgrade command executed |
| 3 | 单台节点回退耗时 ≤ 3 分钟（以日志时间戳为准） | `rollback.sh::rollback_node` (start/end timestamps, hard assert) | **PASS** — 51s / 64s (manual drill) and 48s / 45s (auto rollback), all ≤ 180s |
| 4 | README 写明只加不删的迁移兼容约定与例外审批流程 | `README.md`, `docs/MIGRATION-COMPAT.md` | **PASS** — policy and four-condition exception approval documented |

## Four-step drill (hard requirement)

| Step | Command | Exit code |
|---|---|---|
| 1 old image up | `deploy.sh --env test --image-tag v1` | **0** (app-a 79s, app-b 78s) |
| 2 switch to new image | `deploy.sh --env test --image-tag v2` | **0** (app-a 55s, app-b 48s) |
| 3 switch back to old image | `rollback.sh --env test --target-tag v1` | **0** (app-a 51s, app-b 64s) |
| 4 migration round-trip | `alembic upgrade head` / `downgrade 0001` / `upgrade head` | **0 / 0 / 0**, final revision `0002 (head)` |

## What a rollback touches

1. Reads `ops/deploy/state/{env}.json`; target defaults to `previous_tag`, overridable with `--target-tag`.
2. Rolls back application nodes one at a time (drain, swap image, health gate, re-attach).
3. Skips database downgrade unless explicitly authorised (see criteria 2 and 4).
4. Restores the environment snapshot and the credential version pointer from the pre-deploy state.
5. Runs the 5-endpoint smoke test and emits an environment/version/result/duration notification.

## Reproduce

```bash
bash ops/deploy/bootstrap.sh
bash ops/deploy/deploy.sh --env test --image-tag v1
bash ops/deploy/deploy.sh --env test --image-tag v2
bash ops/deploy/rollback.sh --env test --target-tag v1
docker run --rm --network host -e DATABASE_URL=... s4d-api:v1 alembic upgrade head
docker run --rm --network host -e DATABASE_URL=... s4d-api:v1 alembic downgrade 0001
docker run --rm --network host -e DATABASE_URL=... s4d-api:v1 alembic upgrade head
```

Exact exit codes, log lines and artifacts: [ACCEPTANCE-EVIDENCE.md](ACCEPTANCE-EVIDENCE.md).

## Limitations

- Local image store instead of a private registry.
- GitHub Actions workflow delivered but not executed in this environment.
- Host networking works around the target host blocking container-to-container TCP.
- The controlled `--with-db-downgrade` end-to-end path is not an AC; only the skip path was exercised,
  plus a standalone migration round-trip (0/0/0).
