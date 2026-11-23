# Acceptance — ClawHunt problem 3198 (rollback playbook)

Status: **PASS** — all four acceptance criteria plus the mandatory four-step drill verified on 2026-11-23.
Environment: Linux host, Docker 26.1.3, Compose v2.27.0, two application nodes behind nginx 1.27,
PostgreSQL 16, Redis 7; all ports loopback-only.

Each criterion gives the **English scoring text**, the Chinese original, the observed result, the exact
reproduction command with its real exit code, and the artifact to inspect.

`previous_tag` is a first-class object in this repository: it is the `previous_tag` field of
`ops/deploy/state/{env}.json`, written by `deploy.sh` before any change, and is the default rollback
target of `rollback.sh`. A concrete instance is committed at
[ops/deploy/state/example.test.json](ops/deploy/state/example.test.json).

---

## AC 1 — Faulty deployment rolls back to `previous_tag`, smoke passes

**English (scoring text):** In a test drill, after a faulty version is deployed the system automatically
rolls back to `previous_tag` and all smoke tests pass (attach logs).

**中文原文：** test 演练中部署故障版本后自动回滚到 previous_tag，冒烟测试全部通过（附日志）

**Result: PASS.** Fault image `s4d-api:v2-bad` (`/readyz` always 500) → deploy exit **33**; automatic
rollback exit **0**; `previous_tag` measured as **`v1`**; `ROLLBACK_PLAN ... from=v2-bad to=v1`;
both nodes rolled back (48s / 45s); smoke 5/5 × 200.

**Reproduce:**

```bash
docker build --network host -f ops/app/Dockerfile.bad -t s4d-api:v2-bad ops/app
bash ops/deploy/deploy.sh --env test --image-tag v2-bad --log-name deploy-3198-baddrill
# exit code: 33 (automatic rollback inside the run exits 0)
python3 ops/deploy/lib/state.py ops/deploy/state/test.json get previous_tag
# prints: v1
```

**Evidence:** `evidence/logs/deploy-3198-baddrill.log`, `evidence/logs/deploy-3198-baddrill.rollback.log`.
Full provenance of `previous_tag` (source, verification, measured value): see
[ACCEPTANCE-EVIDENCE.md](ACCEPTANCE-EVIDENCE.md#previous_tag-provenance).

---

## AC 2 — No downgrade unless explicitly requested

**English (scoring text):** When `--with-db-downgrade` is not passed, the script executes no
`alembic downgrade` and the log shows an explicit skip note.

**中文原文：** 未传 `--with-db-downgrade` 时脚本不执行任何 alembic downgrade（日志可见跳过说明）

**Result: PASS.** Rollback exit **0**; `DB_DOWNGRADE_SKIPPED (no --with-db-downgrade)`; no downgrade
command ran.

**Reproduce:**

```bash
bash ops/deploy/rollback.sh --env test --target-tag v1 --reason manual-drill \
  --log-name drill-3198-step3-rollback-v1
# exit code: 0
```

**Evidence:** `evidence/logs/drill-3198-step3-rollback-v1.log`.

---

## AC 3 — Single-node rollback within 3 minutes

**English (scoring text):** A single node rollback takes no more than 3 minutes, measured from the log
timestamps.

**中文原文：** 单台节点回退耗时 ≤ 3 分钟（以日志时间戳为准）

**Result: PASS.** Manual drill 51s / 64s; automatic rollback 48s / 45s; the script asserts a 180s
threshold per node and fails the run above it.

**Reproduce:**

```bash
grep 'node_rollback_end' evidence/logs/drill-3198-step3-rollback-v1.log
# elapsed=51s / elapsed=64s (threshold 180s)
```

**Evidence:** timestamped begin/end pairs in the rollback logs.

---

## AC 4 — Add-only migration policy and exception approval documented

**English (scoring text):** The README documents the add-only migration compatibility convention and the
exception approval process for destructive changes.

**中文原文：** README 写明只加不删的迁移兼容约定与例外审批流程

**Result: PASS.** `README.md` section "Rollback behaviour" and `docs/MIGRATION-COMPAT.md` state the
add-only rule and the four conditions required to downgrade (tested `downgrade()`,
`migration_reversible=true`, fresh backup, `--with-db-downgrade --confirm-token CONFIRM-DOWNGRADE`).

**Reproduce:**

```bash
grep -n '只加不删' README.md docs/MIGRATION-COMPAT.md
```

**Evidence:** the two documents above.

---

## Mandatory four-step drill (old → new → old → migration round-trip)

| Step | Command | Exit code | Observed |
|---|---|---|---|
| 1 old image up | `deploy.sh --env test --image-tag v1` | **0** | app-a 79s, app-b 78s |
| 2 switch to new image | `deploy.sh --env test --image-tag v2` | **0** | app-a 55s, app-b 48s |
| 3 switch back to old image | `rollback.sh --env test --target-tag v1` | **0** | app-a 51s, app-b 64s |
| 4 migration round-trip | `alembic upgrade head` / `downgrade 0001` / `upgrade head` | **0 / 0 / 0** | final revision `0002 (head)` |

Artifacts: `evidence/logs/drill-3198-step1-old-v1.log`, `...step2-new-v2.log`,
`...step3-rollback-v1.log`, `evidence/logs/migration-roundtrip.log`.

## Rollback behaviour in one paragraph

`rollback.sh` reads `state/{env}.json`, defaults the target to `previous_tag` (overridable with
`--target-tag`), rolls nodes one at a time (detach → 30s drain → swap image → `/healthz` + `/readyz`
three consecutive 200 → attach), skips the database by default, restores the environment snapshot and
credential version pointer, then runs the five-endpoint smoke test and emits an
environment/version/result/duration notification.

## Exit codes

| Code | Meaning |
|---|---|
| 11 | lock held by another run |
| 12 | no usable rollback target / image missing |
| 43 / 44 | DB downgrade refused: not reversible / no second confirmation |
| 45 / 46 | DB backup failed / downgrade failed |
| 47 | post-rollback smoke failed |
| 51-55 | per-node step failed (detach / up / health / attach / over 180s) |

Detailed evidence: [ACCEPTANCE-EVIDENCE.md](ACCEPTANCE-EVIDENCE.md).
Compatibility rules: [docs/MIGRATION-COMPAT.md](docs/MIGRATION-COMPAT.md).
