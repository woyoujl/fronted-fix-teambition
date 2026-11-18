# teambition-defect-fix-frontend

Cursor / Codex Agent Skill：查询已确认的 Teambition 前端缺陷，复现、验证后做最小修复，并回填结论。

只处理前端（页面、路由、交互、样式、状态、权限、客户端接口适配）。证据不足时只分析或等待补证，不硬改。

## 能做什么

- 经 `$teambition-query-defects` 确认 Bug 后再改代码
- 读取缺陷描述、钉钉文档、蓝湖 / Figma、YApi（读不到则降级或跳过）
- 分诊为 Fix / Suggest / Wait
- 浏览器复现与验证；UI 类保留前后对照
- 验证通过后在**当前分支**提交，可选 push；**默认不开 MR**，不自动 merge
- 验证失败自动回滚该单改动
- 向 Teambition 回填原因、改动、验证、风险、SHA，并 @ 指定操作者
- 仅在明确要求时做本机 `/loop` 轮询

## 依赖

同级安装 `teambition-query-defects`（Cursor 路径：`%USERPROFILE%\.cursor\skills\teambition-query-defects\`）。没有它，本 skill 不能查单。

还需要：

- Teambition MCP（查任务、发评论）
- 目标前端仓库可在本地启动
- 需要 push 时：本机环境变量 `GITLAB_HOST`、`GITLAB_TOKEN`（或 `GL_TOKEN`）
- 需要登录复现时：当次会话提供测试账号，或本机 `FRONTEND_TEST_USER` / `FRONTEND_TEST_PASSWORD`

Token、密码、证书只放环境变量或当次会话，不要写入本仓库。

## 安装

把本目录放到：

```text
%USERPROFILE%\.cursor\skills\teambition-defect-fix-frontend\
```

若同时使用 Codex，再复制一份到：

```text
%USERPROFILE%\.codex\skills\teambition-defect-fix-frontend\
```

装好后重启 Cursor，新开 Agent 会话。全新电脑的 MCP、环境变量和验收清单见 [使用说明.md](使用说明.md)。

## 怎么用

在目标前端仓库的 Agent 对话里：

```text
使用 $teambition-defect-fix-frontend 修复前端 Bug。
先调用 $teambition-query-defects。
项目 projectId：<PROJECT_ID>
负责人：<指定操作者>
状态：未完成
确认分支后再改代码。
```

## 文档

| 文件 | 内容 |
| --- | --- |
| [使用说明.md](使用说明.md) | 全新电脑配置与日常用法 |
| [SKILL.md](SKILL.md) | Agent 入口与硬规则 |
| [references/frontend-workflow.md](references/frontend-workflow.md) | 完整修复流程 |
| [references/](references/) | 查询门禁、验证、回填、轮询、凭据规则 |

## 不要提交

- GitLab / Teambition Token、测试密码、证书
- 填好的 `projectId`、真实姓名、本机仓库路径、内部分支名
- `.env` 与凭据文件
