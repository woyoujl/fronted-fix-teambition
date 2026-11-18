---
name: teambition-defect-fix-frontend
description: >-
  Reproduce, diagnose, verify, and optionally fix confirmed Teambition frontend
  defects across local projects (web UI, routes, interactions, styles, state,
  permissions, client API adapters). Use when fixing TB/Teambition bugs, reading
  DingTalk prototypes/docs linked from defects, running browser verification,
  committing verified fixes on the current Git branch (no MR by default),
  optionally creating GitLab MRs only when asked, automatically rolling back
  failed fixes to the pre-change state while keeping verified ones, writing results back to
  Teambition, scheduling unattended TB bug polling, or analyzing defects that
  should not be hard-fixed. Before inspecting or modifying defect-related code,
  invoke $teambition-query-defects and continue only with a confirmed Bug/defect;
  stop when the result is only a candidate, the target is ambiguous, or the
  branch is unclear.
---

# Teambition Frontend Defect Fix

Frontend-only, project-agnostic. Prefer analysis + Teambition write-back over unsafe hard-fixes.

## Required Reading

- Invoke `$teambition-query-defects` first. It owns member resolution, Bug type resolution, search, pagination, detail verification, and query failures.
- Read `references/teambition-query.md` for the query→fix handoff gate.
- Read `references/frontend-workflow.md` as the full workflow source of truth.
- Read `references/frontend-defect-fix.md` for the compact checklist.
- Read capability refs on demand:
  - `references/dingtalk-docs.md` — prototypes / PRD；读不到则提示用户、跳过、总结提醒
  - `references/design-sources.md` — Lanhu 降级（截图+TB）；Figma 有链则用，无链看 TB
  - `references/evidence-gate.md` — hard-fix vs suggest vs wait-for-evidence
  - `references/verification.md` — tests + browser auto-verify
  - `references/test-credentials.md` — prompt user for local test account/password before repro/verify
  - `references/ui-replay.md` — UI/style replay + before/after screenshots
  - `references/gitlab-mr.md` — current-branch commit/push after verify; MR only if user asks; selective rollback (no auto-merge)
  - `references/teambition-writeback.md` — TB comment templates
  - `references/polling.md` — unattended scheduled pickup
- Read `使用说明.md` for fresh-PC setup and end-user usage (Chinese).
- Read `references/local-setup.md` for maintainer install/validation checklist.
- Before changing a project, read local nav files when present: `AGENTS.md`, `llms.txt`, `CLAUDE.md`, `GEMINI.md`, `README.md`, `docs/README.md`, module guides, `package.json`, etc.
- If not confidently frontend, explain why and ask before editing.

## Mandatory Query Gate

1. Invoke `$teambition-query-defects` with project, assignee or explicit task ID, and status scope.
2. Accept only confirmed Bug/defect results (task ID, title, assignee, status, priority, updated time, enough detail to classify frontend scope).
3. Multiple confirmed defects → user selects one, unless the user explicitly listed IDs for batch.
4. Stop when query skill/MCP is unavailable, results are candidates-only, or details are insufficient.
5. After selecting a frontend defect, inspect Git state and confirm the target branch before editing.

Do not reimplement Teambition query here.

## Outcome Triage (before coding)

| Outcome | When | Code change | GitLab | Teambition |
| --- | --- | --- | --- | --- |
| **Fix** | Reproducible, expected behavior clear, frontend scope clear | Yes, minimal | Verify → commit on current branch (no MR by default) | Write-back + `@` 修复人 |
| **Suggest** | Likely cause known but fix not safe to guarantee | No (or exploratory only if user insists) | No fix commit | Analysis + recommendations + `@` 修复人 |
| **Wait** | Evidence insufficient / docs inaccessible / conflict | No | No | Clarification comment + `@` 修复人；resume on new evidence |

Details: `references/evidence-gate.md`. Never hard-fix ambiguous bugs.

## Closed-Loop Phases

Execute in order; skip only when the triage outcome is Suggest/Wait.

0. **Query gate** — `$teambition-query-defects` → confirmed frontend Bug + branch.
1. **Evidence pack** — TB note/comments/attachments; `RenderTaskRtfV3`; DingTalk docs (`dingtalk-docs.md`: if needed but unreadable → prompt user, skip bug, remind in final summary); Lanhu/Figma (`design-sources.md`); YApi when API-shaped.
2. **Reproduce** — if the page needs login/role/data, **prompt the user for test account, password, start commands, and fixtures** (`test-credentials.md`) before browser repro; then smallest faithful path; UI/style: before screenshot + replay (`ui-replay.md`).
3. **Fix or stop** — narrow change, or stop per evidence gate.
4. **Verify** — focused tests + browser automation (`verification.md`); UI defects require after screenshot comparison.
5. **Deliver** — on the **current branch**, after verify: commit (+ push when authorized). **Do not create MR by default**; only if the user explicitly asks (`gitlab-mr.md`).
6. **Auto-rollback on verify fail** — restore code to the pre-fix state for that task (discard uncommitted changes or `git revert` if already committed); keep other passed commits untouched (`gitlab-mr.md`). Never leave a failed fix on the branch.
7. **Write back** — Teambition comment with cause, changes, verification, risk, SHA (MR URL only if an MR was requested); **must `@` Bug 修复人 via `mentionUserIds`** on the same `CreateTaskCommentV3` (`teambition-writeback.md`).
8. **Secondary pass** — when TB gains new evidence under a prior Wait/fail comment, re-run query gate → evidence → fix/verify.
9. **Polling (optional)** — only if user explicitly requests unattended pickup (`polling.md`).

## Integrated Capabilities

- Mandatory lookup via `$teambition-query-defects`.
- Teambition task MCP for details/comments (`CreateTaskCommentV3`, `RenderTaskRtfV3`, etc.).
- `teambition-docs` only for OpenAPI/TQL docs, never business tasks.
- DingTalk / product docs: if required but unreadable → prompt user, Wait/skip that bug, list in final summary (`dingtalk-docs.md`). No official DWS assumed.
- Design: Lanhu MCP often broken → TB screenshots/description (`design-sources.md`). Figma only when a link exists; else fall back to TB description. YApi read-only unless user asks to write.
- Browser verification (`cursor-ide-browser` preferred in Cursor; chrome-devtools elsewhere). When login is required, prompt the user for test credentials per `test-credentials.md`; never persist passwords into skill, git, MR, or Teambition.
- UI replay + before/after screenshots.
- GitLab delivery: **default = commit (+ optional push) on current branch after verify; no MR**. Create MR only when the user explicitly asks. Never auto-merge. Token only from `GITLAB_TOKEN`/`GL_TOKEN` (+ optional `GITLAB_HOST`).
- Selective / auto rollback: on verify fail, **must** restore that task to pre-fix state; keep passed commits; never leave failed diffs on the branch.
- Teambition write-back for fix / fail / rollback / wait / suggest; every comment `@`s Bug 修复人 via `mentionUserIds`.
- Unattended polling defaults to local `/loop` (Automations only if cloud GitLab works); see `polling.md`.
- Batch only when user lists multiple defect IDs; stay on current branch with one commit per ID (isolation branches only if user asks).

## Core Rules

- Actionable defect context is enough; no unified `$bug-report` gate.
- Treat linked DingTalk docs/prototypes/PRDs as acceptance evidence when readable. If required but unreadable: prompt user, skip that bug (Wait), remind in the final summary. Always preserve URL in the evidence trail.
- Do not claim design/YApi/DingTalk was checked if it was not successfully read. Lanhu unavailable → use TB screenshots/description; Figma only with an explicit link.
- After selecting a frontend defect that needs local repro/browser verify with auth: ask the user for start commands, test username/password, role, and seed data before claiming reproduction or verification (`test-credentials.md`).
- After code changes, run the most focused verification; browser-visible defects require automation when the app can run locally.
- After verify, commit on the **current branch** (push when authorized). **Do not open a GitLab MR unless the user explicitly asks.** Never merge/squash/auto-merge. If verification fails, **automatically roll back that task to its pre-fix state** without waiting for user confirmation (unless the user asked to keep the failed diff). After batch verification, roll back only failed defects without disturbing passed ones.
- After verify or evidence-block, always write a Teambition comment (templates in `teambition-writeback.md`) and `@` the **指定操作者** via `mentionUserIds` (`isDingtalkPM: false` by default; no DingTalk DWS).
- Create polling only on explicit request with a filled `poll_config` (`polling.md`). **Default runner: local `/loop`** (not Automations). User may go straight to `fix-one`/`fix-batch`. Put test username/password in the `/loop` start prompt (never into skill files, git, or TB). Each tick: query → dedupe → verify → commit+push on current branch (no MR) or auto-rollback → TB `@` operator.
- Do not change backend unless the ticket proves it is required.
- Do not close/archive/transition Teambition tasks unless the user asks or the post-verify target status is unambiguous.

## Safety Rules

- Never write UserToken, App Secret, tokenized MCP URLs, auth headers, passwords (including frontend test passwords), certificates, or GitLab tokens into repos, skill files, logs, MR descriptions, TB comments, shell history, or automation prompts.
- Do not weaken tests or remove assertions to pass.
- Do not batch-fix discovered candidates unless IDs were explicitly listed.
- If a capability is unavailable, report the gap and substitute evidence; never claim it was completed.
- Never force-push over user work or rewrite shared history.

## Local Setup

- 全新电脑配置与使用：见同目录 `使用说明.md`（推荐新人首读）。
- 维护者接入细节：见 `references/local-setup.md`。
