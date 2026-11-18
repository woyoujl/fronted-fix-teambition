# GitLab 提交与可选 Merge Request

验证通过后，在**当前工作分支**上提交（可选 push）。**默认不创建 Merge Request**。仅当用户明确要求开 MR 时，才走 MR 流程；禁止自动合并。

Token **只**从环境变量 / 凭据管理器 / 安全 MCP 读取：`GITLAB_TOKEN` 或 `GL_TOKEN`。可选 `GITLAB_HOST`（自建实例）。永不把 token 写入 skill、仓库、日志、MR、TB、shell 历史、automation prompt。

## 目标 / 边界 / 非目标

- **目标**：验证闭环通过后，在当前分支 `commit`（授权时 `push`）；TB 回填带 SHA。验证失败则**自动回滚到该单修改前**。
- **边界**：改动仅含本缺陷相关文件；一缺陷尽量独立 commit（message 含 `Teambition: <taskId>`）。
- **非目标**：默认不开 MR、不 merge、不强制切 `ai-bug/*` 分支；不把验证失败的改动留在分支上。

## 前置条件

- 仓库有 Git remote（GitLab 时可读 `GITLAB_*`）
- 改动仅含本缺陷相关文件
- **本地验证已通过**后才可按「已修复」口径提交；未验证不得声称已修复
- 用户已授权 commit，或无人值守 `allow_commit: true`
- push 需要权限与 token（仅本地提交亦可，须在报告中说明未 push）

缺 remote、权限或 token：保留本地提交并说明原因；**不要**伪造 MR URL。

## 默认交付：当前分支提交（无 MR）

| 步骤 | 行为 |
| --- | --- |
| 1. 确认分支 | 使用**当前检出分支**（或 `poll_config.base_branch` / 用户指定分支）；不要仅为开 MR 再切新分支 |
| 2. 验证 | 聚焦测试 + 必要的浏览器验证（见 `verification.md`）。**未通过则立即自动回滚，不进入步骤 3** |
| 3. 提交 | 仅验证通过后：`git add` 相关文件 → commit（见下方 message 约定） |
| 4. 推送 | 仓库已指定且可 push（`allow_push`/用户确认）：直接 `git push` 当前分支；禁止 force-push 共享历史 |
| 5. 回填 | TB 写 SHA（无 MR 则写「未开 MR（默认策略）」） |

### 分支策略

| 模式 | 默认行为 |
| --- | --- |
| 单缺陷 / 批量 | **留在当前分支**连续修复；每个缺陷独立 commit |
| 用户明确要求隔离 | 才切 `ai-bug/<taskId>-<slug>`；仍**默认不开 MR**，除非用户再要求开 MR |
| 用户明确要求开 MR | 再按下文「可选：创建 MR」执行 |

批量时禁止把多个无关修复揉进同一 commit。

## 提交

commit message：简体中文 + Conventional Commits，subject 说明为什么改：

```text
fix(ui): 修复订单列表空态文案与原型不一致

Teambition: <taskId>
```

不提交密钥、`.env`、含敏感信息的截图。用户未明确要求 commit 时先问；无人值守且 `allow_commit: true` 时可直接提交。

## 验证失败必须自动回滚到修改前（强制）

验证未通过时，**必须**把本缺陷相关改动恢复为动手修改前的样子，不得把失败改动留在工作区或当前分支上「待审」。无需再问用户是否回滚（用户明确要求保留失败改动时除外）。

### 动手前：记下还原点

每个 task 开始改代码前记录：

```text
pre_fix_sha = git rev-parse HEAD
# 可选：记下即将改动的路径列表 touched_paths
```

批量时每个 task 各自记自己的 `pre_fix_sha`（通常等于上一个已通过 task 提交后的 HEAD）。

### 推荐顺序：先验证、再提交

```text
还原点 → 改代码 → 验证
  ├─ 通过 → commit（+ 可选 push）→ TB 已修复回填
  └─ 失败 → 立即自动回滚到还原点 → TB 失败回填（已回滚）
```

**默认：验证通过之前不要 commit。** 这样失败回滚最干净。

### 回滚动作（按状态选一，必须执行到工作区与本 task 相关 diff 清零）

| 状态 | 自动回滚方式 |
| --- | --- |
| 改动未 commit，且工作区只有本 task 改动 | `git restore .` / `git checkout -- .`，或 `git reset --hard pre_fix_sha`（仅当 HEAD 仍等于 `pre_fix_sha` 且无他人/他 task 未提交改动） |
| 改动未 commit，但工作区还混有其他已通过未提交内容 | **只** `git restore -- <本 task 的 touched_paths>`；禁止 `reset --hard` 误伤其他改动 |
| 已对本 task 做了 commit 后才发现验证失败 | `git revert <该 commit sha> --no-edit`；若已 push 则再普通 push；禁止为回滚而 force-push 共享历史 |
| 隔离分支 `ai-bug/*`（用户要求时） | 切回 `base_branch` 并删除该 bug 分支（未合并）；未 push 可用 `git branch -D` |

回滚完成后自检：

1. `git status`：无本 task 残留改动  
2. 相对 `pre_fix_sha`，本 task 相关路径内容已回到修改前  
3. 已通过 task 的 commit / 文件未被破坏  

回滚失败（冲突、权限、误伤风险）：停止继续改下一单，报告阻塞原因；不得假装「已回滚」。

### 批量

1. **先分账**：每单记录 `taskId`、`pre_fix_sha`、验证结论、最终 SHA（通过）或回滚方式（失败）。  
2. **通过项**：保留其 commit。  
3. **失败项**：**必须**按上表自动回滚到该单修改前。  
4. 禁止：为清失败项而 `reset --hard` 掉已通过提交；禁止 merge。

### TB

失败回滚后用「验证失败 / 已回滚」模板回填，写明：失败现象、`pre_fix_sha`、回滚动作、回滚后确认「代码已恢复修改前」。

## 可选：创建 MR（仅用户明确要求）

用户说「开 MR / 提 Merge Request」时才执行。优先级：

1. `glab`（已安装且已登录）
2. GitLab MCP
3. GitLab REST API：`POST {GITLAB_HOST or https://gitlab.com}/api/v4/projects/:id/merge_requests`  
   Header：`PRIVATE-TOKEN: $env:GITLAB_TOKEN`

不要加 merge / approve / auto-merge。`remove_source_branch` 默认 false。

### glab（bash）

```bash
glab mr create --title "fix: <taskId> <short title>" --description "$(cat <<'EOF'
## Summary
- Teambition: <taskId>
- 根因：...
- 改动：...

## Test plan
- [ ] 聚焦测试：...
- [ ] 浏览器步骤：...

## Risk / Rollback
- 风险：...
- 回滚：revert <SHA> 或关闭本 MR
EOF
)" --target-branch <base>
```

### 若已开 MR 后的失败回滚

1. 关闭 MR 或 revert 失败 commit；TB 回填失败模板。
2. 通过项的 MR/commit 不得误伤。

## REST 常用（仅 MR/回滚场景）

Host：`$GITLAB_HOST`，缺省 `https://gitlab.com`。

| 动作 | 方法 |
| --- | --- |
| 关闭 MR | `PUT .../merge_requests/:iid` body `state_event=close` |
| 删除未合并源分支 | `DELETE .../repository/branches/:branch`（确认未合并；默认策略下通常无此分支） |

## 禁止

- 验证失败却不回滚、把失败改动留给人工清理
- 默认流程里主动创建 MR（用户未要求时）
- 把 token 写进任何文件或 skill
- merge / squash merge / auto-merge
- force-push 覆盖他人提交或改写共享历史
- 为“清掉失败项”而 reset 掉已通过缺陷的提交
- 未 push 成功却声称已推送；未开 MR 却伪造 MR URL
- 批量失败时沉默跳过、不回填 TB

## 成功输出

| taskId | 验证 | 分支 | SHA | MR | 动作 |
| --- | --- | --- | --- | --- | --- |
| … | pass/fail | 当前分支名 | … | 无（默认）/ 仅用户要求时填写 | 已提交 / 已 push / **已自动回滚到修改前** |

通过项与回滚项分开列出；TB 回填状态一并说明。
