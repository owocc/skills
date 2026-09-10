---
name: pixel-perfect-restore
description: 100% 还原设计稿到代码。当用户要求把设计稿（Ardot / Figma 等设计工具的 MCP）还原、复刻或实现为前端页面时使用，强调像素级一致、资源完整导出、截图对比验证的闭环流程。
author: owocc
version: "0.1.0"
tags:
  - design
  - ardot
  - mcp
  - pixel-perfect
  - frontend
---

# pixel-perfect-restore — 100% 还原设计稿

> 待完善：以下为骨架，按实际流程逐步补充。

## 目标

把设计稿 1:1 还原为可运行的前端代码：布局、尺寸、颜色、字体、圆角、阴影、图片资源全部对齐，最终以截图对比验证通过。

## 流程骨架

### 1. 获取设计稿信息
- 连接设计工具 MCP（如 `ardot-remote`），用 `fetch_file_info` / `fetch_editor_state` 确认文件与画板。
- 用 `batch_read` / `capture_layout` 读取画板树、节点属性（尺寸、颜色、字体、间距）。
- 用 `capture_screenshot` 拿基准截图（整画板 + 关键局部）。

### 2. 导出资源
- **必读引用文件：[references/asset-export.md](references/asset-export.md)** —— 背景组（`背景` / `BG` 等命名的组，内部多为多图拼接+蒙版）必须整组作为单一素材导出，禁止拆子节点，详见该文件的判定与自检清单。
- `scan_exportable_resources` 扫描可导出节点（图片 / 图标 / 插画），并按引用文件规则剔除背景组子节点。
- `export_nodes` 导出切图（PNG/SVG/WEBP），`download_source_media` 拿原始素材。
- 颜色 / 字体 token：`fetch_variables`、`fetch_styles`、`export_variables`。

### 3. 写代码
- 按 token 建立设计变量（颜色、字体、间距）。
- 按画板顺序逐屏实现，严格使用设计稿数值，不凭感觉改尺寸。

### 4. 验证闭环（还原 100% 的关键）
- 对实现结果截图，与设计稿截图逐屏对比。
- 差异（尺寸、颜色、字重、间距、图片）逐项修正后复测，直到通过。

## 环境依赖

- 设计工具 MCP 已配置并授权（如 Claude Code / OpenCode 中的 `ardot-remote` → `https://ardot.tencent.com/mcp`，scope `mcp:use`）。
- 目标文件的 fileUrl：`https://ardot.tencent.com/file/{fileId}`。

## 经验记录

<!-- 实战中沉淀：易错点、token 与 px 的换算、图片缩放规则、逐屏验收标准等 -->

### 工具坑（Ardot MCP）

- `batch_read` 传了 `properties` 参数后响应会**丢弃 name/bounds**，只要填充信息时才传；要全量节点信息时不传该参数。
- `capture_layout` 深层节点拿不全（约 4 层以下丢失）；深层结构用 `batch_read`（`readDepth` 拉高、不传 `properties`）。
- 图标字体字形（如 `iostgico`，空名 TEXT 节点 + PUA 码点）**不在** `scan_exportable_resources` 结果里，属正常；记下码点/字号/颜色，用图标字体或近似字形渲染，不要找图。
- GROUP 嵌套时子节点 x/y 混合相对坐标（GROUP 子级似页面坐标、FRAME 子级相对父级），不要手工累计；卡片内部几何以「导出参考图 + 截图互相关」校准更可靠。
- 设计稿里可能存在**被画板/面板裁切而不可见**的节点（如动态 3），还原时必须保持不可见（置于裁切区），不能凭节点树把它画出来。

### 截图对比（验证闭环）

- Chrome headless 最小窗口宽 500px：`--window-size=402` 无效，`#page` 会被居中偏移。做法：大窗（600×2000）截图，再按偏移（(600-402)/2=99px）裁剪出页面。
- 无头截图需等字体/图片加载：`--virtual-time-budget≥6000`。
- 验收方法：同尺寸逐行/逐列平均差扫描定位问题带 + 互相关找最优位移（先对齐结构，再对齐字形），比目测可靠（见 tests/.../tools/pngdiff.py）。
- 字体度量差异（设计稿 SF Pro vs 浏览器 SF NS）会导致**换行点不同**：按设计稿断行位置用 `<br>` 硬断 + 负字距（约 -0.55px）微调；每行内字形残差属字体渲染噪声，可接受。
- 背景 PNG 由 @2x 缩到 @1x 显示时，浏览器重采样与设计稿渲染有轻微软化差异，纹理密集区残差偏高属正常。
- 组件实例（勋章/VIP 标签等小图标）不在扫描清单里，但可按节点 ID 直接 `export_nodes` 导出。
