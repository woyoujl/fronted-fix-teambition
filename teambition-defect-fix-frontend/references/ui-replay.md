# UI / 样式缺陷：复现回放与截图对比

适用于布局、样式、文案位置、组件视觉、交互反馈等可见问题。

## 修复前

1. 提取：URL/路由、视口、账号、数据前置、复现步骤、期望结果。
2. 若需登录：先按 `test-credentials.md` 向用户索取测试账号/密码与启动方式，再启动本地环境并登录。
3. 用浏览器走到复现终点。
4. 保存**修复前截图**（关键控件完整入镜；必要时全页 + 局部；避免截到密码框已填明文）。
5. 把步骤写成可回放脚本说明（可不上仓，但报告必须可重放）；**replay 中可写 username 占位，禁止写真实密码**：

```text
replay:
  entry: <URL or route>
  auth: prompt-user | env:FRONTEND_TEST_* | none
  viewport: <width x height>
  steps:
    - navigate ...
    - login with session credentials (not stored in repo)
    - click ...
    - type ...
    - assert visible ...
  screenshot_at: final / after-step-N
```

6. 拍不到修复前状态时，记录原因，改用缺陷原截图 / 设计稿 / 钉钉原型作基线。

## 修复后

1. **同一入口、同一视口、同一步骤**回放。
2. 保存修复后截图，命名区分 before/after。
3. 对比：问题是否消失；是否与钉钉原型 / Lanhu / Figma / 验收文案一致；有无遮挡、错位、溢出等回归。
4. 自动化结论与截图一并写入验证结果和 TB 回填。

## Cursor 操作要点

- 用 `cursor-ide-browser`：navigate → lock → 逐步操作 → snapshot/screenshot → unlock。
- 截图文件路径写入回填；不要把含敏感数据的全屏截图提交进公开仓库，除非团队允许。
- 动画/时序问题：在关键帧额外截图或短录屏说明。

## 通过标准

- 回放可重复
- after 显示原缺陷点已修复
- 无明显新视觉回归
- 有设计/原型链接时，关键差异已对照说明

## 降级

| 情况 | 处理 |
| --- | --- |
| 浏览器不可用 | 给手工回放步骤；不声称自动对比完成 |
| 只有 after | 对照 TB 原截图/设计稿，注明缺 before |
| 数据依赖强 | 固定测试账号/fixture；做不到则 Wait 回填 |

只修缺陷相关选择器/样式/结构，不做大面积“好看化”重构。
