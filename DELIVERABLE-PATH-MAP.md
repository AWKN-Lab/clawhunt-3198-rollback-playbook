# 交付物路径对照表（DELIVERABLE PATH MAP）

> **为什么有这张表**：投标时提交的第一版提案里写的是"计划中的文件路径"；本仓落地时
> 部分文件做了更准确的命名或换了位置（**同一实现**）。买家若按提案里的路径找会找不到，
> 本表给出**一一对应**。
>
> - 提案原文（第一版）：`proposals/3198-proposal.md`
> - 平台现行提案：2026-11-24 修订版，`sha256=52ffcf602915a7446b7c448956a57378ca2cdebfec8ad0b9b5a6ee6a05bda849`
>   （修订后即按本表"本仓实际路径"书写）
> - 逐条验收对照见 `ACCEPTANCE.md` / `docs/ac-3198.md`；证据见 `ACCEPTANCE-EVIDENCE.md`

## 一、路径对照

| 提案里写的路径 | 本仓实际路径 | 说明 |
|---|---|---|
| `ops/deploy/rollback.sh` | `ops/deploy/rollback.sh` | 同名：读 state → 逐台摘除/切镜像/健康检查/挂回 → 数据库按约定处理 → 配置回退 → 冒烟 → 通知 |
| `.github/workflows/rollback.yml` | `.github/workflows/deploy.yml`（`workflow_dispatch` 的 rollback 模式） | **同一 workflow**：部署失败时自动调用回滚；手动触发时 inputs `env` / 可选 `target_tag` |
| `docs/migration-compat-policy.md` | `docs/MIGRATION-COMPAT.md` | **只加不删**的迁移兼容约定与例外审批流程（AC4） |
| `docs/rollback-drill-log.md` | `evidence/logs/drill-3198-step1-old-v1.log`<br>`evidence/logs/drill-3198-step2-new-v2.log`<br>`evidence/logs/drill-3198-step3-rollback-v1.log` | 演练**原始日志**（三步：坏版本 → 新版本 → 回滚），未包装成 Markdown 叙述 |
| `ops/deploy/state/README.md` | `ops/deploy/state/README.md` + `ops/deploy/state/example.test.json` | 同名 + 一份 state 示例 |
| `tests/test_rollback_script.sh` | **本仓无 `tests/` 目录** ⇒ 见"实测证据" | 脚本级回归以**真实运行日志**交付 |
| `tests/run_rollback_drill.sh` | 同上 ⇒ `evidence/logs/drill-3198-*.log`（演练脚本产出的日志即证据） | 演练脚本本身未入库，演练**产物**入库 |
| `README.md` | `README.md` | 回滚流程 / 兼容约定 / 例外审批 / 权限边界（私钥不入仓） |

## 二、"脚本级回归 / 演练"这条 AC 的实测证据（替代未入库的 `tests/`）

| 证据文件 | 对应场景 |
|---|---|
| `evidence/logs/drill-3198-step3-rollback-v1.log` | 部署 `/readyz` 返回 500 的版本 ⇒ **自动回滚**（AC1 演练主证据） |
| `evidence/logs/deploy-3198-baddrill.log` + `deploy-3198-baddrill.rollback.log` | 回滚到坏地址的故障注入与回滚 |
| `evidence/logs/recover-baddrill.log` | 恢复流程 |
| `evidence/logs/migration-roundtrip.log` | 迁移兼容性核对（只加不删） |

## 三、提案未列、但本仓实际提供

| 路径 | 用途 |
|---|---|
| `ACCEPTANCE.md` / `ACCEPTANCE-EVIDENCE.md` / `docs/ac-3198.md` | 逐条 AC ↔ 实现 ↔ 复跑命令 ↔ 实测输出 |
| `docs/MIGRATION-COMPAT.md` | 迁移兼容约定（AC4） |
| `ops/app/*`（含 `Dockerfile.bad`） | 应用本体与坏镜像对照（故障注入用） |
| `ops/deploy/{deploy.sh,bootstrap.sh,cleanup.sh,loadtest.py}`、`ops/deploy/lib/*`、`ops/deploy/lb/*`、`ops/deploy/compose/*` | 与部署流水线共用的部件（回滚调用它们） |
| `scripts/secret-scan.sh` | 秘密扫描 |

## 四、复跑命令（与提案一致）

```bash
bash ops/deploy/rollback.sh --env test
bash ops/deploy/rollback.sh --env test --target-tag v1
bash ops/deploy/rollback.sh --env test --with-db-downgrade --confirm-token CONFIRM-DOWNGRADE
```

## 五、变更说明

本表为**新增说明文件**，不改动任何实现、脚本或证据；
`MANIFEST.sha256` 已重算并可用 `sha256sum -c MANIFEST.sha256` 复核。
