# Teambition 回填

修复、验证失败、建议不改、或证据不足停止后，必须回填 Teambition。

## 工具

优先真实任务 MCP 的 `CreateTaskCommentV3`：

| 参数 | 值 |
| --- | --- |
| `taskId` | 已确认缺陷 ID |
| `requestBody.content` | 回填正文 |
| `requestBody.renderMode` | `markdown`（接口支持时） |
| `requestBody.mentionUserIds` | **必填**：解析出的「Bug 修复人」`userId`（见下节）；单次上限 30 |
| `requestBody.isDingtalkPM` | **默认 `false`**（当前无钉钉官方 DWS，不做钉钉私聊）；仅当用户明确要求且企业已绑钉钉时再设 `true` |

也可用 `CreateTaskTraceV3` 写进展，但评论是默认通道。不要用 `teambition-docs` 回填业务任务。

## @ Bug 修复人（强制）

每次回填评论成功创建时，必须在**同一条** `CreateTaskCommentV3` 中通过 `mentionUserIds` @ **Bug 修复人**，不得只写正文不 @。

### 谁是「Bug 修复人」

默认 **@ 指定操作者**（用户在会话或 `poll_config.assignee` 里点名的人）。

解析为唯一 `userId`（能唯一确定即停）：

1. **指定操作者**：`poll_config.assignee` / 用户当次声明的操作者姓名或 userId（经 `GetUsersMe` 或 `PostV3MemberQuery` 解析）。
2. 任务自定义字段名含「修复人」且值为成员/userId（仅当用户未指定操作者时）。
3. 本轮 MCP 身份 `GetUsersMe` → `userId`（指定操作者与登录身份一致时常用）。
4. 任务负责人 `executorId`（仅作最后回退）。

无法解析出任何 `userId` 时：仍发评论正文，并在最终报告写明「未能 @ 修复人：原因说明」；不得伪造 ID。

### 调用约定

```text
CreateTaskCommentV3(
  taskId,
  requestBody: {
    content: <模板正文>,
    renderMode: "markdown",
    mentionUserIds: [<fixerUserId>],
    isDingtalkPM: false
  }
)
```

- 正文不必手写 `@姓名` 占位；Teambition 站内提醒以 `mentionUserIds` 为准。
- **钉钉私聊默认关闭**：无官方 DWS / 未绑钉钉时保持 `isDingtalkPM: false`，不主动开钉钉 PM。
- 可在正文末追加一行：`请修复人关注本条回填并验收/补充。`（可选，不替代 `mentionUserIds`）。
- 禁止把「无人值守」当作不 @ 的理由；旧规则「默认不刷屏 @」已废止。
- 同一 tick 多条缺陷：每条评论各自 @ 该单解析出的修复人，勿把无关人塞进 `mentionUserIds`。

## 何时回填

| 场景 | 回填 | 改状态 |
| --- | --- | --- |
| 验证通过并已提交（当前分支） | 是 | 仅目标状态明确且流程要求 |
| 验证失败 | 是 | 否 |
| 批量中单项回滚 | 是（该 task） | 否 |
| 证据不足 / 不硬修 | 是 | 否 |
| 只给建议 | 是 | 否 |
| 非前端 / 需转交 | 是（说明依据） | 否，除非用户明确要求 |

禁止默认关闭、归档或流转到最终完成态。

## 已修复模板

```text
处理结论：已修复，待审核/待验证
产生原因：...
改动内容：...
验证结果：
- 测试：<command> => <pass/fail>
- 浏览器：<步骤摘要> => <pass/fail>
- UI 证据：修复前 <path/url>；修复后 <path/url>
风险与回滚：...；回滚可 revert <SHA>
提交信息：<commit SHA>
分支：<当前分支名>
MR 地址：未开 MR（默认策略） / <仅用户要求时填写 URL>
参考文档：钉钉/设计稿/YApi <已读链接或“未读+原因”>
```

## 暂不硬修模板（Wait）

```text
处理结论：暂不硬修，需补充证据后二次修改
已核查证据：TB 描述/评论/截图/钉钉文档/设计稿/YApi/本地复现结果
不确定点：...
不能安全修改的原因：...
需要补充：
- [ ] 可复现账号/环境
- [ ] 测试账号与密码（仅私聊/会话提供，勿贴到任务公开字段若公司禁止）
- [ ] 启动方式与访问 URL
- [ ] 完整步骤与期望结果
- [ ] 截图或录屏
- [ ] 必要测试数据（订单/门店/角色等）
- [ ] 钉钉文档权限或正文
- [ ] 设计稿 / YApi 定位
建议排查：...
下次入口：补充后重新走查询门禁 → 证据读取 → 修复/验证
```

## 建议模板（Suggest）

```text
处理结论：暂不直接改代码，给出分析与建议
可能原因：...
建议方案：...
风险与影响面：...
需要确认：产品/后端/负责人 <具体问题>
验证缺口：...
参考链接：...
```

## 验证失败 / 已回滚模板

```text
处理结论：验证未通过，已自动回滚到修改前（未进入待审修复）
验证结果：
- 测试：<command> => <pass/fail>
- 浏览器：<步骤摘要> => <pass/fail>
失败现象：...
还原点：pre_fix_sha=<SHA>
回滚动作：restore 未提交改动 / revert <SHA>（取实际执行项）
回滚确认：本 task 相关路径已恢复修改前；通过项未受影响
下一步：补充证据后二次修改，或按建议方案处理
```

## 字段完整性

已修复评论尽量同时含：产生原因、改动、验证结果、风险、提交 SHA、所在分支。默认策略无 MR；仅用户要求开 MR 时再写 MR 地址。缺哪项写明原因，例如“仅本地提交 `<local SHA>`，未 push”。

## 二次修改

评论区或附件出现新证据 → 再跑 `$teambition-query-defects` → 读新证据 → 从复现重新开始（见 `evidence-gate.md`）。
