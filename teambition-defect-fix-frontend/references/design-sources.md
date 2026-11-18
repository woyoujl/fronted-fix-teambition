# 设计稿来源：Lanhu / Figma

UI/视觉/布局类缺陷的设计对照规则。

## Lanhu（蓝湖）

- 当前 **Lanhu MCP 常不可用**（error / needsAuth）时：**不要卡死**。
- 降级顺序：
  1. TB 附件截图 / 描述中的期望图
  2. 缺陷单原截图（实际 vs 期望若有）
  3. 浏览器复现前后截图（`ui-replay.md`）
- 报告写明：`Lanhu MCP 不可用，已用 TB 截图/描述对照`。
- 仅当用户明确要求且 MCP 已恢复时，再读蓝湖 URL。

## Figma

采用顺序：

1. **有 Figma 链接**（任务备注/评论/附件/自定义字段中的 `figma.com` 或含 `node-id` 的链接）→ 用 Figma MCP 读取对应 node/frame。
2. **无 Figma 链接** → 回看 **TB Bug 描述/备注/评论/附件**：
   - 有足够文案、步骤、截图 → 按 TB 证据继续 Fix/Wait
   - 仍不足以定期望 → Wait，TB 回填请求补充设计稿或截图
3. 禁止在无链接时猜测 Figma 文件 ID。

## 与钉钉文档的关系

- 设计稿与钉钉原型可并存；冲突时不硬修，提示用户后跳过并在总结提醒（见 `dingtalk-docs.md` / `evidence-gate.md`）。
- 无钉钉、无 Figma、仅有 TB 截图时：允许按截图+描述硬修（范围仍须前端最小改动）。
