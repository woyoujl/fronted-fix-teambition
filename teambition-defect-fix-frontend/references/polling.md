# 无人值守轮询

仅在用户明确要求“定时拉取 TB Bug 并修复”时创建。轮询不是默认行为。

## 目标 / 边界 / 非目标

- **目标**：按固定节奏拉取已确认前端 Bug，跑完整闭环；验证通过后在当前分支提交（可选 push）；验证失败**自动回滚到修改前**；TB 回填。
- **边界**：每次 tick 仍必须过查询门禁、证据门禁、验证、Git 提交或失败回滚、回填；凭据只读环境变量。
- **非目标**：默认不开 MR；不自动 merge；不把验证失败的改动留在分支；不默认改 TB 完成态；不清理他人脏工作区；不在 prompt 里写 token。

## 调度载体（选一）

| 载体 | 适用 | 限制 |
| --- | --- | --- |
| **本机 `/loop`（本仓库默认）** | 当前 IDE 会话值守 | 关 Cursor/休眠即停；Windows 用 PowerShell sleep；**启动命令可带测试账号密码** |
| Cursor Automations | 仅云端 GitLab/MCP 已打通时 | 本仓库内网 GitLab 曾 `unauthenticated`；**禁止**在永久 prompt 写密码 |
| 系统计划任务 | 用户明确指定 | 必须能安全注入凭据，且有人值守日志 |

**默认选本机 `/loop`。** 用户确认真修后可直接 `fix-one` / `fix-batch`，不强制先 `dry-run`。

## 启动前必填配置（缺一则只查询不改代码）

向用户确认并写入本次轮询上下文（**本机 `/loop` 启动命令**可含测试账号密码；**不要**写入 skill 文件或 TB）：

```text
poll_config:
  projectId: <required>
  assignee: <指定操作者姓名或 userId>   # 查单负责人 + TB @ 默认对象
  operator: <同 assignee，或显式操作者> # @ 修复人默认用此人
  status_scope: isDone=false
  repo_path: <本地仓库绝对路径>
  base_branch: <当前工作分支>
  frontend_scope: <可选，如 device-web>
  # 指定仓库后可直接 git push 当前分支（需本机已有 GitLab 凭据/权限）
  mode: fix-one | fix-batch          # 用户确认真修后可直接用；dry-run 仅调试时
  max_per_tick: 1
  priority: P0>P1>P2>updated_desc
  allow_commit: true                 # 真修默认 true
  allow_push: true                   # 仓库已指定且可 push 时 true
  allow_mr: false
  allow_tb_comment: true
  empty_tick: quiet
  secondary_pass: true
  skip_if_dirty: true
  cooldown_minutes: 120
  interval: 30m
  # 测试账号：写在 /loop 启动命令里（见下），不要写进 skill 文件
```

未确认 `projectId` / `repo_path` / `base_branch` / `mode` / **指定操作者** 时：**禁止**创建轮询。  
真修还需在启动命令中提供测试账号密码（或已配 `FRONTEND_TEST_*`），否则需登录的单只能 Wait。  
兼容旧字段：`allow_commit_mr` 仅视为 `allow_commit`，不得据此开 MR。

## 模式

| mode | 行为 |
| --- | --- |
| `dry-run` | 只查询+分诊+报告候选；不改代码、不提交、不开 MR；TB 评论默认关闭 |
| `fix-one` | 每 tick 最多 1 条可 Fix 缺陷；验证通过后在 `base_branch` 提交 |
| `fix-batch` | 每 tick 最多 `max_per_tick` 条；均在当前分支；**一 Bug 一 commit**；失败项**必须自动回滚到该单修改前**，不影响已通过 commit |

## 选单策略（每 tick）

1. `$teambition-query-defects`（`projectId` + 负责人 + 状态范围）。
2. 只保留**已确认** Bug/缺陷，并初筛前端/可疑前端。
3. **去重与冷却**（命中则跳过，不重复硬修）：
   - 已有本 skill 回填且结论为「已修复待审」且附带有效 commit SHA（或用户要求时的 MR URL），且无新失败反馈
   - 仍在 `cooldown_minutes` 内的 Wait / 验证失败 / Suggest（**除非** `secondary_pass` 发现新证据）
   - 同分支上已有未完成的同 task 修复进行中（避免并行打架）
4. **二次补证优先**：`secondary_pass=true` 时，优先选「上次 Wait/失败 + 评论/附件/文档有更新」的 task。
5. **排序**：用户指定优先级字段 > P0/P1/P2 > `updated` 降序。
6. 取前 `max_per_tick` 条；`fix-one` 强制截断为 1。
7. 非前端 / 不确定：Suggest 或 Wait 回填（若允许评论），不改代码。

## Tick 执行流水线

```text
query → dedupe/cooldown → pick ≤N → (per task)
  evidence + triage → Fix|Suggest|Wait
  Fix: 记 pre_fix_sha → 留在 base_branch 改代码 → verify →
       pass: commit(+push)+TB成功回填（默认不开 MR）
       fail: **必须**自动回滚到 pre_fix_sha/修改前 + TB失败回填
  Suggest/Wait: TB模板回填（若允许）→ 记入 cooldown
→ tick summary（有变化才对外说；quiet 时空跑沉默）
```

硬规则：

1. 脏工作区且 `skip_if_dirty=true`：整 tick 跳过并报告阻塞（不 `git reset --hard`）。
2. **在 `base_branch`（当前工作分支）上直接修复并提交**；不要为默认流程另开 `ai-bug/*` 或创建 MR。
3. Git：需要 push 时读 `GITLAB_TOKEN`/`GL_TOKEN` + `GITLAB_HOST`；**禁止 merge**；`allow_mr=false` 时禁止开 MR。
4. 验证失败项按 `gitlab-mr.md` **自动回滚到修改前**，不得把失败 diff 留在分支；不得动已通过项。
5. 单 task 耗时过长（建议 >25min）或验证阻塞：停止该 task，**回滚已改动（若有）**，回填阻塞原因，继续下一条（若还有配额）。
6. 401/403/推送失败：本 tick 中止后续写操作，报告可观测错误（不打印 token）。

## 推荐节奏

| 阶段 | interval | max_per_tick | mode |
| --- | --- | --- | --- |
| 首次联调 | 15～30m | 1 | `dry-run` → 再 `fix-one` |
| 日常值守 | 30～60m | 1 | `fix-one` |
| 高峰清积压 | 30m | 2～3 | `fix-batch`（需用户明确） |
| 工作日 cron 例 | `0 10-19 * * 1-5`（按团队时区） | 1 | `fix-one` |

避免过密（<10m）：易触发 TB/GitLab 限流，且浏览器验证来不及。

## 空跑与汇报

- `empty_tick=quiet`：无候选/全跳过 → 不刷屏（默认）。
- `empty_tick=digest`：每 `digest_every` 个 tick 发一行摘要：`checked=… skipped=… fixed=… waited=… errors=…`。
- 有实际 Fix/Wait/Suggest/回滚 → 当 tick 必须简短汇报。

## 本机 `/loop`（Windows 试跑）

用户确认 `poll_config` 后：

1. 立即跑一轮（冷启动）。
2. 再臂固定间隔 loop（sentinel 唯一，如 `AGENT_LOOP_TICK_tb_bug`）。
3. 停用时杀 loop PID，不再 arm。

提示词每 tick 固定调用本 skill + 嵌入 `poll_config`；测试账号密码仅来自本次 `/loop` 启动命令（或环境变量），禁止写入 skill/TB/git。

## Automation / loop 提示词模板

**本机 `/loop`（默认）** — 把测试账号密码写在启动命令里；interval 例：`30m`。

```text
使用 $teambition-defect-fix-frontend 执行无人值守 tick。
先读 references/polling.md，严格按 poll_config：

projectId: <PROJECT_ID>
assignee: <指定操作者>
operator: <指定操作者>
status_scope: isDone=false
repo_path: <本地仓库绝对路径>
base_branch: <当前工作分支>
frontend_scope: <可选，如 device-web>
mode: fix-batch
max_per_tick: 2
priority: P0>P1>P2>updated_desc
allow_commit: true
allow_push: true
allow_mr: false
allow_tb_comment: true
empty_tick: quiet
secondary_pass: true
skip_if_dirty: true
cooldown_minutes: 120

FRONTEND_TEST_USER: <账号>
FRONTEND_TEST_PASSWORD: <密码>
FRONTEND_TEST_BASE_URL: <可选>
FRONTEND_START_CMD: <可选>

步骤：
1) 调用 $teambition-query-defects
2) 去重/冷却/二次补证后选 ≤max_per_tick
3) Outcome Triage；不硬修则按模板 TB 回填并 @ 指定操作者
4) Fix：记 pre_fix_sha；只改 frontend_scope 指定的前端范围；留在 base_branch；验证通过后 commit+push；默认不开 MR；验证失败必须自动回滚到修改前
5) TB 回填 + mentionUserIds @ 指定操作者（operator/assignee）；isDingtalkPM=false
6) 空跑安静；有结果短表：taskId|结论|SHA|动作|是否已@操作者
7) 若有因钉钉文档跳过的单：tick/会话结束必须提醒用户补证（taskId+链接+原因）
禁止：开 MR、merge、force-push 共享分支、验证失败不回滚、密码写入 TB/git/skill、关闭 TB 任务、钉钉读不到却硬猜验收
```

启动示例（用户自行替换账号密码后发送）：

```text
/loop 30m <粘贴上面整段 poll 提示词，并填好 FRONTEND_TEST_USER/PASSWORD>
```

先立即跑一轮冷启动，再臂 30m loop。

## 停用

用户说停止时：关 Automation / 杀 loop / 禁用计划任务，并确认不会再写 TB、开 MR。汇报最后一次成功 tick 时间与当前分支上未完成修复（如有）。
