# 本地接入流程

记录 `teambition-defect-fix-frontend` 在本机的落盘、依赖命名和验证步骤。  
**全新电脑的完整配置与使用步骤请优先阅读同目录 `../使用说明.md`（或 skill 根目录 `使用说明.md`）。**

本文偏维护者向；不描述具体缺陷修复。该 skill 依赖同级 `$teambition-query-defects`。

## 目标目录（双副本保持一致）

```text
%USERPROFILE%\.codex\skills\teambition-defect-fix-frontend
%USERPROFILE%\.cursor\skills\teambition-defect-fix-frontend
```

推荐结构：

```text
teambition-defect-fix-frontend/
├─ SKILL.md
├─ agents/
│  └─ openai.yaml
└─ references/
   ├─ local-setup.md
   ├─ teambition-query.md
   ├─ frontend-workflow.md
   ├─ frontend-defect-fix.md
   ├─ evidence-gate.md
   ├─ dingtalk-docs.md
   ├─ verification.md
   ├─ ui-replay.md
   ├─ gitlab-mr.md
   ├─ teambition-writeback.md
   ├─ polling.md
   └─ test-credentials.md
```

同步时以任一目录改完后复制到另一目录，避免 Cursor/Codex 行为漂移。

## 配置流程

### 1. 目录

```powershell
$roots = @(
  "$env:USERPROFILE\.codex\skills\teambition-defect-fix-frontend",
  "$env:USERPROFILE\.cursor\skills\teambition-defect-fix-frontend"
)
foreach ($r in $roots) {
  New-Item -ItemType Directory -Force -Path "$r\references","$r\agents" | Out-Null
}
```

### 2. 放置文件

写入 `SKILL.md`、`agents/openai.yaml`、`references/*.md`。长流程放 `references/`，`SKILL.md` 只保留入口与硬规则。

### 3. 对齐依赖名

| 能力 | 期望 | 本机常见缺口 |
| --- | --- | --- |
| TB 任务 | `teambition` / `teambition-user` / `teambition-app` | Token 过期 |
| TB OpenAPI 文档 | `teambition-docs` | 勿当业务任务源 |
| 钉钉文档 | `dingtalk-docs` 或等价；否则浏览器 | **常未配置专用 MCP** |
| 钉钉听记 | `GetDingTalkAIMinutesTranscript` | 仅听记，非通用文档 |
| YApi | `yapi_auto_mcp` / `user-yapi-auto-mcp` | 项目映射 |
| Lanhu | `lanhu` | 常需重新认证 |
| Figma | Framelink / `user-figma-mcp` | 需具体 node 链接 |
| 浏览器 | Cursor: `cursor-ide-browser`；其它: chrome-devtools | 本地站点未起 |
| GitLab | `glab` → GitLab MCP → REST + `GITLAB_TOKEN`/`GL_TOKEN` | **常未装 glab / 无 token** |
| 回填 | `CreateTaskCommentV3` | 评论权限 |
| 轮询 | Cursor Automations 或 `/loop` | 需用户显式开启 |

实际 MCP 名不同时，改 `agents/openai.yaml` 的 `value`，不要改业务仓。

确认查询 skill 存在：

```text
%USERPROFILE%\.codex\skills\teambition-query-defects\SKILL.md
%USERPROFILE%\.cursor\skills\teambition-query-defects\SKILL.md
```

### 4. 验证

重开会话后触发：

```text
使用 $teambition-defect-fix-frontend 修复前端 bug；先 $teambition-query-defects，过门禁后再确认分支。
```

检查点：

- [ ] 能识别 skill，并先走查询门禁
- [ ] 能读 `references/*`
- [ ] 有钉钉链接时走文档读取或明确报告缺口
- [ ] 证据不足走 Wait 回填，不硬修
- [ ] 修复后跑聚焦测试 + 浏览器验证
- [ ] UI 缺陷有回放与前后截图或说明缺失
- [ ] 验证失败时自动回滚到修改前，工作区无该 task 残留 diff
- [ ] 验证通过后能在当前分支 commit（可 push）；默认不开 MR
- [ ] TB 回填含原因/改动/验证/风险/SHA（无 MR 则注明；失败注明已回滚）与 @ 修复人
- [ ] 仅在用户要求时创建轮询

## 需协调支持清单（环境探测结果）

优化 skill 文档后，以下项仍依赖人工协调；缺一项则对应能力降级，不得假装完成：

1. **钉钉文档 MCP 或可读账号**：配置专用文档 MCP，或提供可登录的文档权限 / 导出正文。当前常见仅有听记接口，通用 alidocs 需浏览器或 MCP。
2. **本地复现账号（修 Bug 时由 agent 按 `test-credentials.md` 提示索取）**：各前端的启动命令、测试账号/密码、角色、必要测试数据。交互会话中用户当次提供；轮询则预先配置 `FRONTEND_TEST_USER` / `FRONTEND_TEST_PASSWORD`（勿写入 skill 或聊天长期存档）。密码禁止进 TB/MR/git。
3. **GitLab CLI 或 Token（用户本机自行配置，禁止把 token 贴进聊天或写入 skill）**：
   权限建议：能 push 当前工作分支即可；**默认不依赖**开 MR / merge。Skill 支持同分支批量验证后对失败 commit 做 `revert`，见 `gitlab-mr.md`。若 token 曾出现在聊天中，先在 GitLab 轮换再配置。

```powershell
# 当前会话
$env:GITLAB_TOKEN = '<your-token>'   # 或 GL_TOKEN
$env:GITLAB_HOST = 'https://gitlab.com'  # 自建则改成实际 Host

# 可选：写入用户环境变量（新开终端生效）
[Environment]::SetEnvironmentVariable('GITLAB_TOKEN', $env:GITLAB_TOKEN, 'User')
[Environment]::SetEnvironmentVariable('GITLAB_HOST', $env:GITLAB_HOST, 'User')

# 校验（不应打印 token）
Invoke-RestMethod -Headers @{ 'PRIVATE-TOKEN' = $env:GITLAB_TOKEN } -Uri "$($env:GITLAB_HOST.TrimEnd('/'))/api/v4/user" | Select-Object username, id
```

4. **Lanhu MCP**：可保持不可用；修复时用 TB 截图/描述降级（见 `design-sources.md`）。
5. **本地复现账号（必做约定）**：修 Bug 时 agent 按 `test-credentials.md` **主动提示用户**提供启动方式、测试账号/密码、角色、测试数据。交互会话当次使用；轮询用 `/loop` 启动命令或环境变量。缺账号则 Wait，不假装已验证。
6. **轮询参数（启用前必填）**：见 `polling.md` 的 `poll_config`——至少 `projectId`、指定操作者、`repo_path`、`base_branch`、`mode`、`max_per_tick`、`allow_commit`、`allow_mr=false`、`empty_tick`、`cooldown_minutes`。默认本机 `/loop`。
7. **TB 回填权限**：当前身份可对目标项目 `CreateTaskCommentV3`；回填须 `mentionUserIds` @ 指定操作者；`isDingtalkPM` 默认 `false`。
8. **双副本同步约定**：改 skill 后同时更新 `.codex` 与 `.cursor` 两份（可用同步脚本/拷贝）。

## 更新原则

- 只改本机 skill 目录，不改业务仓库。
- 新能力优先补 `references/`。
- MCP/CLI 名称或验证方式变化时同步本文件与 `agents/openai.yaml`。
