# Frontend Defect Fix Checklist

Use after the defect has actionable context and is confirmed frontend.

## 0. Query gate

- Invoke `$teambition-query-defects` before inspecting or modifying defect-related code.
- Continue only with a confirmed Bug/defect.
- Stop if candidate-only, multiple unselected defects, or unclear branch.

## 1. Outcome triage

- Read `evidence-gate.md`.
- **Fix** / **Suggest** / **Wait**. Suggest and Wait → Teambition comment only, no hard-fix.

## 2. Evidence pack

- Extract environment, route, account, steps, expected/actual, screenshots, console/network.
- **If login/role/data is required: prompt the user for start commands, test username/password, role, and fixtures** (`test-credentials.md`) before browser repro. Do not store passwords in git/TB/MR.
- Read DingTalk docs/prototypes when linked (`dingtalk-docs.md`); use `RenderTaskRtfV3` for rich notes.
- Read design sources per `design-sources.md`: Lanhu down → TB screenshots/description; Figma if link present, else TB description.
- If DingTalk doc is required but unreadable → prompt user, Wait/skip this bug, record for final summary (`dingtalk-docs.md`).
- YApi when API-shaped.
- If expectation is still unclear, or auth credentials were requested but not provided → Wait write-back, stop.

## 3. Branch

- Check Git state; stay on the **current / agreed base branch**.
- Do not create `ai-bug/*` or open an MR unless the user explicitly asks.
- Do not close/archive TB tasks unless asked.

## 4. Reproduce / localize

- Smallest faithful path; map failing step to code.
- UI/style: replay script + before screenshot (`ui-replay.md`).

## 5. Fix narrowly

- Smallest component/hook/store/style/adapter change.
- No unrelated refactors; preserve API contracts unless ticket proves otherwise.

## 6. Verify

- Focused test/lint/typecheck/build as relevant (`verification.md`).
- Browser automation for visible defects (`cursor-ide-browser` in Cursor).
- UI/style: after screenshot + compare.
- Pass → may transition status only if unambiguous; fail/block → **auto-rollback to pre-fix**, no “fixed” status.

## 7. GitLab commit + auto-rollback

- Record `pre_fix_sha` before editing each task.
- Prefer verify **before** commit. After verify pass: **commit on current branch** (+ push when authorized). **No MR by default** (`gitlab-mr.md`).
- On verify fail: **must** restore that task to pre-fix state (discard uncommitted changes or `git revert`); never leave failed diffs; never reset away other passed commits.
- Commit only when user authorized (or polling `allow_commit: true`).
- Batch: one commit per defect (`Teambition: <taskId>`).

## 8. Teambition write-back

- Always comment: cause/uncertainty, changes or recommendation, verification, risk, SHA (`teambition-writeback.md`); note「未开 MR」when following default delivery.
- Same comment **must** `@` Bug 修复人: resolve fixer `userId` → `mentionUserIds`; keep `isDingtalkPM: false` unless user explicitly enables DingTalk PM (no official DWS by default).
- Use rollback template when verify failed and code was restored to pre-fix.
- Wait/Suggest templates when not hard-fixing.
- On new evidence later → secondary pass from query gate.

## 9. Report

- Per defect: task ID, pass/fail, branch, SHA, action (committed / **auto-rolled-back to pre-fix**), TB write-back result.
