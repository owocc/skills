---
name: ardot-guide
apply_to: pixel-perfect-restore
---

# Ardot MCP 工具调用错误记录

> 实战中踩过的 Ardot MCP 工具坑，按工具名与现象查找。主流程在 `SKILL.md`。

## 工具坑

- `batch_read` 传了 `properties` 参数后响应会**丢弃 name/bounds**，只要填充信息时才传；要全量节点信息时不传该参数。
- `capture_layout` 深层节点拿不全（约 4 层以下丢失）；深层结构用 `batch_read`（`readDepth` 拉高、不传 `properties`）。
- 所有文件工具的第一个必选参数都是 `fileUrl`，格式 `https://ardot.tencent.com/file/{数字ID}[?node_id=xxx]`；MCP 没有列出文件的能力，token scope 仅 `mcp:use`，不能访问网页版文件列表 API。
- 图标字体字形（如 `iostgico`，空名 TEXT 节点 + PUA 码点）**不在** `scan_exportable_resources` 结果里，属正常；记下码点 / 字号 / 颜色，用图标字体或近似字形渲染，不要找图。
- GROUP 嵌套时子节点 x/y 混合相对坐标（GROUP 子级似页面坐标、FRAME 子级相对父级），不要手工累计；卡片内部几何以「导出参考图 + 截图互相关」校准更可靠。
- 设计稿里可能存在**被画板 / 面板裁切而不可见**的节点，还原时必须保持不可见（置于裁切区），不能凭节点树把它画出来。
- 组件实例（勋章 / VIP 标签等小图标）不在扫描清单里，但可按节点 ID 直接 `export_nodes` 导出。
- `capture_screenshot` / `export_nodes` 批量调用（一次传多个 nodeIds）返回签名 URL，逐个下载；截图属较贵资源，不要无目的地批量截。
