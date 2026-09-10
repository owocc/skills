---
name: pixel-perfect-restore
description: 100% 还原设计稿到代码。当用户要求把设计稿（Ardot / Figma 等设计工具的 MCP）还原、复刻或实现为前端页面时使用，强调像素级一致、资源完整导出、截图对比验证的闭环流程。
author: owocc
version: "0.3.0"
tags:
  - design
  - ardot
  - figma
  - mcp
  - pixel-perfect
  - frontend
---

# pixel-perfect-restore — 100% 还原设计稿

把设计稿1:1
还原为可运行的前端代码：布局、尺寸、颜色、字体、圆角、阴影、图片资源全部对齐，最终以截图对比验证通过。

## 流程骨架

### 1. 询问交互逻辑（先于一切设计稿操作）

- 在连接设计工具 MCP 之前，先向用户索要当前设计稿的**交互逻辑说明**：各屏之间的关系、点击/滑动等交互行为、状态切换、页面跳转流程。
- 设计稿只呈现静态画面，交互行为必须由人补充；没有交互说明，还原只能做到「看起来一样」，无法做到「用起来一样」。
- 若用户始终不提供交互说明，不要反复追问阻塞流程：遵循用户当前请求，按设计稿直接生成页面（纯静态还原），并在交付时注明哪些交互行为是推测的、需要用户后续补充。

### 2. 获取设计稿信息

- 连接设计工具 MCP，支持两类：
  - **Ardot MCP**：用 `fetch_file_info` / `fetch_editor_state` 确认文件与画板，`batch_read` / `capture_layout` 读取画板树与节点属性，`capture_screenshot` 拿基准截图（整画板 + 关键局部）。
  - **Figma MCP**：用 `get_metadata` 读取画板结构，`get_code` / `get_screenshot` 拿代码与基准截图，`get_variable_capsules` 拿颜色/字体 token。
- 后续流程以「画板树 + 节点属性 + 基准截图」三样为准，与所用 MCP 无关。

### 3. 导出资源

- `scan_exportable_resources` 扫描可导出节点（图片 / 图标 / 插画）。
- `export_nodes` 导出切图（PNG/SVG/WEBP），`download_source_media` 拿原始素材。
- 颜色 / 字体 token：`fetch_variables`、`fetch_styles`、`export_variables`。

#### 背景组整组导出（强制）

结论先行：**遇到表示“背景”的组，必须把整组作为单一素材导出，禁止导出组内子节点。**

设计稿里的背景经常不是一张图，而是**多张图片 + 蒙版 + 渐变拼接**的组。拆开导出会丢失组级的合成关系，还原结果必然错。

真实案例（文件 `722223308803000`，节点 `1:375`）：

```
1:375 「背景」 (GROUP, 362×221)
├─ 1:376 「Mask group」 (362×96)   ← 与 1:379 重叠
│  ├─ 1:3205 「image 964」 线性渐变（灰 85%→45%，透明度渐变）
│  └─ 1:3206 「image 965」 IMAGE 填充 VECTOR
└─ 1:379 「Mask group」 (362×149)  ← 与 1:376 重叠
   ├─ 1:3207 「image 964」 线性渐变（灰 85%→45%，透明度渐变）
   └─ 1:3208 「image 965」 IMAGE 填充 VECTOR（与 1:3206 同一 imageHash）
```

对 `1:375` 执行 `scan_exportable_resources`，工具返回的是**子节点级清单**：`image 965` ×2 作为两张独立图片。如果照这个清单导出：背景被拆成碎片、渐变蒙版与重叠关系丢失、两块拼图实际是同一张原图的不同裁切+蒙版（imageHash 相同）无法正确拼合。

**`scan_exportable_resources` 的子节点清单只代表“这些节点各自可导出”，不代表“应该单独导出”。是否原子导出，由下面的组名规则决定。**

在拿到节点树后，先做背景识别，再决定导出单位。命中任一条件即视为背景组：

1. **组名命中背景语义**（不区分大小写、中英文都查）：
   - 中文：`背景`、`底图`、`底`、`场景`、`环境`、`光效`
   - 英文：`bg`、`background`、`backdrop`、`scene`、`wallpaper`
   - 组合形式：`bg_1`、`BG-home`、`background-image` 等前缀/后缀变体
2. **结构特征指向背景拼接**（组名未命中时，用结构特征兑底）：
   - 组内包含名为 `Mask group` / `蒙版` 的子组，且子组之间 bounds 重叠；
   - 同一 `imageHash` 的图片以不同裁切/变换出现多次；
   - 子节点带渐变（`GRADIENT_*`）填充且透明度渐变（gradientStops 中有 alpha < 1），或 `blendMode` 非 `NORMAL`。

满足条件 1 的组**无条件整组导出**；仅满足条件 2 的组也整组导出（宁可多导一张背景，不可拆坏）。强制执行顺序：

1. **先看树，再扫描**：对目标画板先跑 `capture_layout`（`problemsOnly: false`），用上述规则标记所有背景组。
2. **背景组剔除出扫描清单**：跑 `scan_exportable_resources` 后，把位于背景组内部的子节点从导出清单中删除，改用背景组自身的节点 ID。
3. **整组导出**：`export_nodes` 传背景组的节点 ID（如 `1:375`），格式默认 `png@2x`（背景带渐变/蒙版时禁用 SVG）。
4. **截图验收**：对背景组 `capture_screenshot`，与导出的 PNG 对比——如果拆过组，这两者会出现明显差异（缺渐变、断层），可直接发现。
5. **命名落盘**：背景素材命名沿用组名，如 `bg-home.png` / `背景-家族首页.png`，放 `assets/` 目录，不做二次加工。

导出前自检（任一条不满足，禁止进入下一步写代码）：

- [ ] 所有命中背景组名的 GROUP 都以组 ID 出现在 `export_nodes` 参数里？
- [ ] 导出清单里有没有背景组的子孙节点 ID？有 → 删掉，换成组 ID。
- [ ] 背景组带渐变/蒙版时是否用了 PNG（而非 SVG）？
- [ ] 每个背景组的截图与导出图是否一致？

### 4. 写代码
- 按 token 建立设计变量（颜色、字体、间距）。
- 按画板顺序逐屏实现，严格使用设计稿数值，不凭感觉改尺寸。

### 5. 验证闭环（还原 100% 的关键）

- 对实现结果截图，与设计稿截图逐屏对比。
- 差异（尺寸、颜色、字重、间距、图片）逐项修正后复测，直到通过。
- 验收方法：同尺寸逐行/逐列平均差扫描定位问题带 + 互相关找最优位移（先对齐结构，再对齐字形），比目测可靠（见 tests/.../tools/pngdiff.py）。
- Chrome headless 最小窗口宽 500px：`--window-size=402` 无效，`#page` 会被居中偏移。做法：大窗（600×2000）截图，再按偏移（(600-402)/2=99px）裁剪出页面。
- 无头截图需等字体/图片加载：`--virtual-time-budget≥6000`。
- 字体度量差异（设计稿 SF Pro vs 浏览器 SF NS）会导致**换行点不同**：按设计稿断行位置用 `<br>` 硬断 + 负字距（约 -0.55px）微调；每行内字形残差属字体渲染噪声，可接受。
- 背景 PNG 由 @2x 缩到 @1x 显示时，浏览器重采样与设计稿渲染有轻微软化差异，纹理密集区残差偏高属正常。

## 工具坑（Ardot MCP）

- `batch_read` 传了 `properties` 参数后响应会**丢弃 name/bounds**，只要填充信息时才传；要全量节点信息时不传该参数。
- `capture_layout` 深层节点拿不全（约 4 层以下丢失）；深层结构用 `batch_read`（`readDepth` 拉高、不传 `properties`）。
- 图标字体字形（如 `iostgico`，空名 TEXT 节点 + PUA 码点）**不在** `scan_exportable_resources` 结果里，属正常；记下码点/字号/颜色，用图标字体或近似字形渲染，不要找图。
- GROUP 嵌套时子节点 x/y 混合相对坐标（GROUP 子级似页面坐标、FRAME 子级相对父级），不要手工累计；卡片内部几何以「导出参考图 + 截图互相关」校准更可靠。
- 设计稿里可能存在**被画板/面板裁切而不可见**的节点（如动态 3），还原时必须保持不可见（置于裁切区），不能凭节点树把它画出来。
- 组件实例（勋章/VIP 标签等小图标）不在扫描清单里，但可按节点 ID 直接 `export_nodes` 导出。

## 环境依赖

- 设计工具 MCP 已配置并授权：
  - Ardot：Claude Code / OpenCode 中的 `ardot-remote` → `https://ardot.tencent.com/mcp`，scope `mcp:use`；目标文件 fileUrl 为 `https://ardot.tencent.com/file/{fileId}`。
  - Figma：Dev Mode MCP Server（桌面端启用或远程端点），需可访问目标文件。
