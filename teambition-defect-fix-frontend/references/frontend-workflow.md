# Teambition 前端缺陷单驱动修复

本文是 `teambition-defect-fix-frontend` 的流程事实源，适用于本机任意前端项目。项目自身的 `AGENTS.md`、`llms.txt`、`CLAUDE.md`、`GEMINI.md`、`README.md`、模块说明、构建配置和测试规范优先级高于本文。

拉取缺陷单必须调用 `$teambition-query-defects`。放行契约见 `teambition-query.md`；精简清单见 `frontend-defect-fix.md`；硬修决策见 `evidence-gate.md`。

## Teambition 查询门禁

修改前端代码前，必须先调用 `$teambition-query-defects`：

1. 只接收已核验项目、负责人和 Bug/缺陷类型的任务。
2. 多条结果由用户选定；只有用户明确列出多个缺陷 ID 时才批量。
3. 仅候选 ID 或详情不足时停止，不读源码、不改代码。
4. 确认前端后检查 Git 状态与分支；分支不明则询问。

## 核心判断

- 仅前端缺陷进入本流程；后端/DB/部署/网关/定时任务/服务端逻辑不默认改前端。
- 不要求统一 bug 报表；TB 描述、附件、评论、截图、录屏、日志、钉钉文档、本地文件或用户补充只要足够支撑复现与验收即可。
- 非前端或不确定时，先说明依据并询问是否继续。
- 保持目标项目已有框架与风格；不为单缺陷引入新框架或大范围重构。
- 不硬修：证据不足走 Wait；无法保证修复走 Suggest（见 `evidence-gate.md`）。

## 已集成能力

| 能力 | 时机 | 约束 |
| --- | --- | --- |
| Teambition 查询 | 查询/诊断/修复 | 真实任务 MCP；门禁见查询 skill |
| 钉钉文档 | 原型/PRD/验收/交互链接 | 见 `dingtalk-docs.md`；读不到 → 提示用户、跳过、总结提醒 |
| 设计稿 | UI/视觉/布局 | 见 `design-sources.md`：Lanhu 不可用则截图+TB；Figma 有链才用 |
| YApi | 参数/字段/枚举/适配 | 默认只读 |
| 浏览器验证 | 可见/交互/样式/路由/状态 | 见 `verification.md` |
| UI 回放截图 | UI/样式/布局/交互 | 见 `ui-replay.md` |
| GitLab 交付 | 验证通过且已授权提交 | 见 `gitlab-mr.md`：当前分支 commit；默认不开 MR；禁止自动合并 |
| TB 回填 | 修复/失败/建议/缺证 | 见 `teambition-writeback.md` |
| 无人值守轮询 | 用户明确要求 | 见 `polling.md` |
| 批量 | 用户明确多个 ID | 同分支一 Bug 一 commit |

## 前置阅读

1. 仓库规则：`AGENTS.md`、`llms.txt`、`CLAUDE.md`、`GEMINI.md`、`README.md`、`docs/README.md`
2. 模块说明与测试说明
3. 构建入口：`package.json` 与框架配置
4. 源码入口：路由、页面、组件、API、request、store、权限/字典

## 前端判定

属于前端：页面/路由/导航/表单表格弹窗、布局样式文案、交互与状态同步、前端权限显隐与字典、请求/响应字段映射与客户端适配。

不属于前端：后端接口错误、DB 异常、部署、网关、服务端权限源、定时任务、后端业务规则。

## 上下文来源

| 类型 | 必读 | 缺信息时 |
| --- | --- | --- |
| UI/样式 | 钉钉原型 / Lanhu / Figma；否则截图描述 | 可排查，最终说明未对照设计/文档 |
| 交互/状态 | 步骤 + 文档/设计；必要时 YApi | 期望不明 → Wait |
| 接口/字段 | YApi 或样例 | 多候选让用户选；未读不可声称已对照 |
| 混合 | 先 YApi 再展示规则文档 | 缺任一来源说明缺口 |

## 标准处理流程

1. `$teambition-query-defects` 过门禁；提取步骤、期望/实际、截图、日志、钉钉/设计/YApi 链接。
2. Git 状态；确认当前分支（或用户指定 `base_branch`）；有脏改动先问；默认不另开 `ai-bug/*`。
3. 分类前端/非前端/不确定；只修确认前端。
4. Outcome Triage（`evidence-gate.md`）：Wait/Suggest 则回填后停止。
5. **若需登录复现/验证：按 `test-credentials.md` 提示用户提供启动方式、测试账号、密码、角色与测试数据**；未提供则 Wait，不假装已验证。
6. 读钉钉文档、设计稿、YApi、浏览器证据（按类型）。
7. 复现定位最小路径；UI 类先建回放并尽量拍 before。
8. 按项目框架闭环（RuoYi：页面→路由→API→request→数据更新→页面验证）。
9. 最小改动；聚焦测试 + 浏览器验证；UI 做 after 对比。
10. 验证通过 → 在**当前分支**提交（授权则 push）。**默认不创建 GitLab MR**；仅用户明确要求时再开 MR。验证**失败 → 必须自动回滚到该单修改前**（见 `gitlab-mr.md`）；批量时一 Bug 一 commit，失败项回滚不得误伤已通过项。
11. TB 回填：原因、改动、验证、风险、SHA（无 MR 则写明「未开 MR」）；失败用「已回滚」模板；缺证用暂不硬修模板（可写测试用户名，禁止写密码）。同一条评论必须 `mentionUserIds` @ Bug 修复人（见 `teambition-writeback.md`）。
12. 最终报告：按缺陷列出 pass/fail、分支、SHA、动作；**必须单列「因钉钉文档跳过、需用户处理」清单**（若有）。

## 无人值守

仅用户明确要求且填齐 `poll_config` 后创建。试跑用 `/loop` + `dry-run`/`fix-one`；长期用 Automations。每次 tick：查询 → 去重/冷却/二次补证 → ≤N 条闭环 → 失败项选择性回滚。细节见 `polling.md`。

## 禁止事项

- 写入任何密钥/Token/证书到仓库、skill、日志、MR、TB、automation prompt
- 验证失败却不回滚、把失败改动留在分支上
- 验证失败或未执行却流转为已修复
- 默认关闭/归档缺陷
- 自动批量修查询出的多条候选
- 删测试/降断言过关
- 未读文档/设计/YApi 却声称已对照
- 自动合并 GitLab MR；在用户未要求时主动创建 MR
- 证据不足时硬改并包装成根因
