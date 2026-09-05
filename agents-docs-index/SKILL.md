---
name: agents-docs-index
description: >
  管理项目的 .agents/ 多文档索引结构，并自动生成或更新 AGENTS.md 索引。
  当用户提到"重构 AGENTS.md"、"拆分文档"、"生成文档索引"、"初始化 agents 文档"、
  "迁移 AGENTS.md 到多文档结构"、或想让 AI 自动读取 .agents/wiki/ 目录生成索引时，
  必须使用此 skill。支持两种场景：从零初始化新项目文档，或将已有 AGENTS.md 迁移拆分。
  同时支持 monorepo 多包结构，每个子包可拥有独立的 AGENTS.md 和 .agents/。
  支持"更新文档"触发场景：用户告知新增文件、规范变更时，AI 更新对应文件内容并刷新索引，
  同时保留用户手动标注的 * 必读标记。支持从旧版结构迁移（场景 F）。
---

# Agents Docs Index Skill

将项目文档拆分为 `.agents/` 下的结构化文件，并在 `AGENTS.md` 中维护一份压缩索引。
AI 读取索引后按需检索具体文件，避免每次都加载全量文档，节省 token 与上下文。

支持 **monorepo 两级索引**：根级 `AGENTS.md` 负责全局规范 + 包路由，各子包拥有独立的
`AGENTS.md` 和 `.agents/`，内容聚焦该包自身的技术栈与规范。

索引条目支持 **`*` 必读标记**：带 `*` 的条目表示 AI 每次开始任务前必须读取，
无 `*` 的条目按需读取。用户可手动在任意条目前加 `*`，AI 更新索引时永久保留。

---

## 目录结构总览

```
# 根级
AGENTS.md                      — 压缩索引（仅索引，不含实体内容）
.agents/
  wiki/
    conventions.md             — 全局编码规范、命名约定（必读）
    contributing.md            — 提交规范、PR 要求、质量门控（必读）
    packages.md                — 各包路径、名称、职责（包路由表）
    commands.md                — 自定义命令索引（/命令名 时按需读取）
  docs/
    <adr-or-session>.md        — 架构决策、会议记录等文档（按需）
  commands/
    <command-name>.md          — 每个自定义命令一个文件

# 各子包（结构相同）
apps/app/
  AGENTS.md                    — 子包索引（含项目声明 + 强制前置检查）
  .agents/
    wiki/
      architecture.md          — 包内模块结构（必读）
      conventions.md           — 包内特有规范（必读，覆盖全局时需注明）
      commands.md              — 包级命令索引（/命令名 时按需读取）
      <其他>.md                 — 按包类型扩展，用户可标注 *
    docs/
      <package-specific>.md    — 包内文档记录
    commands/
      <command-name>.md        — 包专属命令文件
```

**目录职责说明：**
- `.agents/wiki/`：结构化规范文档，AI 按索引按需读取
- `.agents/docs/`：非结构化文档记录（ADR、会议记录、设计讨论等），AI 按需查阅
- `.agents/commands/`：自定义命令文件，仅在 `/命令名` 触发时读取

**AGENTS.md 内容边界：**
- `AGENTS.md` 是压缩索引，不是说明书；除强制协议注释、必读说明、索引块外，禁止新增大段正文
- 用户自定义工作约束（如临时输出目录、测试记录位置、专用协作规则）必须沉淀到 `.agents/wiki/<topic>.md`，再作为索引条目加入 `[Global Docs Index]`
- 只有用户明确要求或旧文档已存在对应内容时，才创建这类用户自定义条目；skill 不默认臆造项目约束
- 迁移或更新时，若发现 `## 临时文档说明`、`## 测试输出` 等独立正文段落，应移动到 wiki 文件并在索引中引用，不能继续保留在 `AGENTS.md` 正文

---

## * 必读标记规范

### 标记格式

```
* |architecture:  {architecture.md}  — 包内模块结构（必读）
* |conventions:   {conventions.md}   — 编码规范（必读）
  |workflow-ui:   {workflow-ui.md}   — UI 工作流（按需）
* |DESIGN:        {../../DESIGN.md}  — 设计规范（用户标注必读）
```

- `*` = 必读，空格 = 按需，对齐用空格补位
- `<path>` 可引用任意相对路径，不限于 `.agents/wiki/`

### 默认必读条目

| 级别 | 默认必读 |
|---|---|
| 根级 | `conventions`、`contributing` |
| 子包 | `architecture`、`conventions` |
| 子包跨包引用 | 根级 `contributing`、根级 `conventions` |

### 用户自定义标记

用户可随时手动加 `*`，AI 在**任何更新操作中必须保留所有已有 `*` 标记**，不得移除。

---

## 执行流程

### Step 0 — 判断操作类型

| 用户指令 | 操作类型 |
|---|---|
| "初始化文档" / "生成索引" / "迁移 AGENTS.md" | → 场景 A / B / C |
| "更新文档" / "我新增了 X" / "X 规范变了" | → 场景 D |
| "把 X 标为必读" / "X 加星" | → 场景 E |
| "迁移到新结构" / "升级 agents 目录" | → 场景 F |

### Step 1 — 判断目标范围

| 用户指令 / 上下文 | 操作范围 |
|---|---|
| "初始化整个项目" / "重构根 AGENTS.md" | 根级 + 扫描所有子包 |
| "给 apps/app 加文档" / 任务路径指向特定包 | 仅该子包 |
| 无明确指向 | 询问用户：根级还是特定包？ |

---

### 场景 A — 从零初始化

**根级：**
1. 扫描项目结构：读取根 `package.json` / `pnpm-workspace.yaml` / `turbo.json`
2. 创建 `.agents/wiki/`、`.agents/docs/`、`.agents/commands/`
3. 生成 `wiki/conventions.md`、`wiki/contributing.md`
4. 生成 `wiki/packages.md`：各包路径、名称、一句话职责
5. 生成 `wiki/commands.md`：初始空索引
6. 执行「填充根级索引模板」，默认必读：`conventions`、`contributing`

**子包：**
1. 读取该包 `package.json`，识别技术栈
2. 创建 `<package>/.agents/wiki/`、`.agents/docs/`、`.agents/commands/`
3. 按包类型生成最小模板文档 + 推荐扩展（见「子包类型扩展表」）
4. 生成 `wiki/commands.md`：初始空索引
5. 执行「填充子包索引模板」，默认必读：`architecture`、`conventions`，
   跨包引用：根级 `contributing`、根级 `conventions`

---

### 场景 B — 迁移拆分已有 AGENTS.md

1. 完整读取当前 `AGENTS.md`，提取所有 `*` 标记并记录
2. 按以下映射拆分写入：

   | 章节关键词 | 目标文件 |
   |---|---|
   | 项目结构、模块、apps、packages | `wiki/architecture.md`（子包）/ `wiki/packages.md`（根级）|
   | 命令、dev、build、pnpm、turbo | `wiki/commands.md` 索引 + 各命令写入 `commands/<n>.md` |
   | 规范、命名、TypeScript、Prettier | `wiki/conventions.md` |
   | UI、组件、shadcn、DESIGN.md | `wiki/workflow-ui.md`（子包）|
   | 提交、commit、PR、质量门控 | `wiki/contributing.md` |
   | 临时文档、测试输出、工作区约束、协作规则 | `wiki/<topic>.md`（如 `wiki/temp.md`、`wiki/workspace.md`）|
   | ADR、sessions、文档记录 | `docs/<n>.md` |

3. 每个文件开头加 `# <标题>\n> 此文件由 agents-docs-index skill 管理。`
4. 执行「填充索引模板」，**恢复所有 `*` 标记**
5. 不得把已拆分内容以 `## 临时文档说明`、`## 测试输出`、`## 协作规则` 等正文段落形式重新写回 `AGENTS.md`

---

### 场景 C — 重新扫描更新索引

1. 列出 `.agents/wiki/` 下所有 `.md` 文件
2. 读取现有 `AGENTS.md`，提取并记录所有 `*` 标记
3. 读取每个文件的 `# 标题` 行提取描述
4. 执行「填充索引模板」，恢复所有 `*` 标记

---

### 场景 D — 更新文档内容

触发词：「更新文档」「我新增了 X」「X 规范变了」「把 X 加入文档」

1. 读取现有 `AGENTS.md`，完整提取所有 `*` 标记
2. 判断变更类型：

   | 变更类型 | 操作 |
   |---|---|
   | 新增外部文件（如 `DESIGN.md`） | 索引末尾追加条目，询问用户是否标 `*` |
   | 现有 wiki 文件内容过时 | 更新对应 `.md` 内容，不改动索引结构 |
   | 新增功能模块需要文档 | 创建 `wiki/<n>.md`，追加索引条目 |
   | 新增用户自定义工作约束 | 创建或更新 `wiki/<topic>.md`，追加索引条目，不在 `AGENTS.md` 写正文 |
   | 删除模块/文件 | 移除索引条目，删除对应文件 |

3. 外部文件用相对路径引用：`{../../DESIGN.md}`
4. 写回 `AGENTS.md`，**所有 `*` 标记必须保留**
5. 若用户要求记录“临时文档说明”等操作规则，应将完整说明写入 wiki 文件，`AGENTS.md` 只增加一行索引
6. 向用户确认变更摘要

---

### 场景 E — 仅更新必读标记

触发词：「把 X 标为必读」「X 加星」「X 去掉星」

1. 读取当前 `AGENTS.md`
2. 精确修改对应条目的 `*` 标记
3. 其余内容**一字不改**，写回

---

### 场景 F — 从旧版结构迁移

触发词：「迁移到新结构」「升级 agents 目录」「更新 agents 架构」

旧版特征识别：
- `AGENTS.md` 中存在独立的 `## 斜杠命令协议` / `## Package Index` 段落
- `.agents/commands/README.md` 存在
- `documentation.md` 位于 `.agents/wiki/` 而非 `.agents/docs/`
- `packages.md` 作为独立 wiki 文件而非索引条目
- `AGENTS.md` 中存在独立的 `## 临时文档说明` / `## 测试输出` / `## 协作规则` 等用户自定义工作约束段落

**迁移步骤：**

1. **备份**：读取现有 `AGENTS.md` 完整内容，记录所有 `*` 标记
2. **清理 AGENTS.md 冗余段落**，需移除以下内容：
   - `## 斜杠命令协议` 整个段落（4-5 行的命令查找说明）
   - 独立的 `## Package Index` 段落及其说明文字
   - 子包 AGENTS.md 中的 `> 斜杠命令：收到 /命令名 时…` 说明行
   - `## 临时文档说明`、`## 测试输出`、`## 协作规则` 等可沉淀为 wiki 的用户自定义正文段落
3. **迁移 documentation.md**：
   - 若 `.agents/wiki/documentation.md` 存在 → 移动到 `.agents/docs/documentation.md`
   - 更新 AGENTS.md 中对应索引条目的路径
4. **删除 `.agents/commands/README.md`**（如存在）
5. **将 packages 条目折叠进 Global Docs**：
   - 原来独立的 `## Package Index` + `[Package Index]` 块
   - 改为 Global Docs 索引中的一条：`|packages: {wiki/packages.md} — 各包路径与职责`
   - 包定位说明（路径关键词）迁移进 `wiki/packages.md` 文件内部
6. **迁移用户自定义工作约束**：
   - 例如原文为“测试或记录临时输出时保存到 `.agents/temp/`，该目录已加入 `.gitignore`”
   - 写入 `.agents/wiki/temp.md` 或语义更准确的 `.agents/wiki/workspace.md`
   - 在 `[Global Docs Index]` 中追加 `|temp: {wiki/temp.md} — 临时文档与测试输出存放规范` 等索引条目
   - 若旧段落提到 `.gitignore`，只验证并保留事实；不要替用户新增未确认的项目约束
7. **重新填充索引模板**，恢复所有 `*` 标记
8. 向用户输出迁移摘要：移除了哪些段落、迁移了哪些文件、新的 AGENTS.md 结构

---

## 索引模板

### 根级 AGENTS.md 完整模板

```markdown
# Project Guide — Monorepo Root

<!-- ============================================================
  强制包上下文协议 — 禁止跳过
  在写任何代码或进行任何修改之前，你必须：
  1. 确认本次任务涉及哪个包（路径关键词推断，或直接询问用户）
  2. 读取该包的 AGENTS.md
  3. 读取该包 AGENTS.md 中所有标有 * 的条目对应文件
  4. 完成以上三步后才可以开始执行任务
  无论任务大小，此协议不可跳过。
  /命令名：先查 .agents/wiki/commands.md，找到则执行，找不到则忽略。
============================================================ -->

> 重要：优先通过检索文档来推理，而非依赖训练数据中的预设知识。
> 标有 * 的条目为必读，任务开始前必须读取完毕。无 * 的条目按需读取。

[Global Docs Index]|root: ./.agents
* |conventions:   {wiki/conventions.md}   — <填充：全局编码规范描述>
* |contributing:  {wiki/contributing.md}  — <填充：提交与 PR 规范描述>
  |packages:      {wiki/packages.md}      — <填充：各包路径与职责，包含包定位关键词>
  |commands:      {wiki/commands.md}      — 自定义命令索引（/命令名 时读取）
  |apps/app:              {apps/app/AGENTS.md}               — <填充：包描述>
  |apps/web:              {apps/web/AGENTS.md}               — <填充：包描述>
  |apps/desktop:          {apps/desktop/AGENTS.md}           — <填充：包描述>
  |apps/cli:              {apps/cli/AGENTS.md}               — <填充：包描述>
  |packages/core-server:  {packages/core-server/AGENTS.md}  — <填充：包描述>
  |packages/cloud-server: {packages/cloud-server/AGENTS.md} — <填充：包描述>
  |packages/ui:           {packages/ui/AGENTS.md}            — <填充：包描述>
```

**填充规则：**
- `packages.md` 条目描述写「各包路径与职责」，包定位关键词写入 `packages.md` 文件内部，不在 AGENTS.md 展开
- 用户自定义工作约束按需追加为普通 Global Docs 条目，例如：`|temp: {wiki/temp.md} — 临时文档与测试输出存放规范`
- 这类按需条目不默认标 `*`；只有用户明确要求必读，或原索引已标 `*`，才保留或添加 `*`
- 未初始化的包标注 `— (未初始化)`
- 包路由条目不标 `*`，AI 按任务需要自行定位后读取

---

### 子包 AGENTS.md 完整模板

```markdown
# <包名> 开发指南

<!-- ============================================================
  【项目声明】
  当前位置：<包路径>（如 apps/app / packages/ui）
  ⚠️ 这是一个 Monorepo 子包，不是独立项目。
  - 完整项目根目录位于：../../
  - 根目录 AGENTS.md 位于：../../AGENTS.md
  - 当前子包的技术栈与规范以本文件及 [Docs Index] 为准
  - 全局规范（提交规范、PR 要求、monorepo 构建命令）以根目录 AGENTS.md 为准
  - 其他子包有各自独立的 AGENTS.md，不要混用
  - /命令名：先查本包 .agents/wiki/commands.md，找到则执行，找不到则忽略。
============================================================ -->

<!-- ============================================================
  【强制前置检查 — 禁止跳过】
  你正在读取此包的 AGENTS.md。
  在做任何其他事情之前，你必须读取下方所有标有 * 的条目对应文件。
  在所有 * 条目读取完毕之前，禁止写代码、禁止浏览文件、禁止回答关于此包的任何问题。
  如果某个 * 文件无法访问，立即停下并告知用户。
============================================================ -->

> 重要：优先通过检索文档来推理，而非依赖训练数据中的预设知识。
> 全局规范见根目录 AGENTS.md。标有 * 的条目为必读，任务开始前必须全部读取。

> 停止：请先完成以下清单，再开始任务。
> [ ] 读取 [Docs Index] 中所有标有 * 的条目
> [ ] 读取 [Global Refs] 中所有标有 * 的条目
> 全部完成后才可开始执行。

[Docs Index]|root: ./.agents/wiki
* |architecture:  {architecture.md}  — <填充：包内模块结构描述>
* |conventions:   {conventions.md}   — <填充：包内规范描述（覆盖全局时需注明）>
  |commands:      {commands.md}      — 本包自定义命令索引（/命令名 时读取）

[Global Refs]
* |contributing:       {../../.agents/wiki/contributing.md}  — 提交规范与 PR 要求（全局）
* |conventions-global: {../../.agents/wiki/conventions.md}   — 全局编码规范（子包规范优先）
```

> **关于强制检查块：**
> - HTML 注释块面向 AI，用户通常不可见；`> 停止：` 区块面向 AI 可见，强制执行顺序
> - 斜杠命令协议折叠进 `【项目声明】` 注释块，不在可见区域展开
> - 子包先列 `[Docs Index]`，再列 `[Global Refs]`

---

### commands.md 模板

```markdown
# 自定义命令索引

> 此文件仅在用户输入 /命令名 时触发检索，不是必读项。
> 命令文件存放于 .agents/commands/<command-name>.md

[Commands]
  |<command-name>: {../commands/<command-name>.md} — <一句话描述命令用途>
```

---

### 单个命令文件模板（`.agents/commands/<command-name>.md`）

```markdown
# /<command-name>

> 触发方式：用户输入 /<command-name>
> 作用范围：<根级全局 / 仅限 <包名>>

## 描述
<这个命令做什么>

## 执行步骤
1. <步骤一>
2. <步骤二>

## 参数（可选）
- `<参数名>`：<说明>

## 示例
用户输入：`/<command-name> <参数>`
AI 行为：<描述预期行为>
```

---

## 子包类型扩展表

| 包类型 | 推荐追加（按需标 `*`）|
|---|---|
| React / 前端应用 | `workflow-ui.md`（组件库、设计规范）|
| Astro 站点 | `workflow-ui.md`、`routing.md`（页面路由规则）|
| Node 服务 / CLI | `api.md`（端点或命令设计规范）|
| 共享 UI 库 | `components.md`、`workflow-ui.md` |
| 桌面 shell | `bridge.md`（前后端桥接协议）|
| 任意包 + 外部设计文件 | `{../../DESIGN.md}`（用户标 `*` 后永久保留）|

追加文件时在 `[Docs Index]` 末尾添加条目，若标 `*` 则在 `> 停止：` 清单追加 checkbox：

```markdown
> [ ] 读取 * workflow-ui 条目
```

---

## AI 包定位逻辑

1. **用户明确告知** → 直接读该包 `AGENTS.md`，执行强制检查块
2. **路径关键词推断** → 读 `wiki/packages.md` 中的关键词映射表定位包
3. **多包任务** → 依次读取各包 `AGENTS.md`，依次执行强制检查块
4. **无法判断** → 读 `wiki/packages.md` 后询问用户，**不得在确认前开始任何代码操作**

---

## 注意事项

- **强制检查块永不删除**：`<!-- 强制前置检查 -->` 注释块和 `> 停止：` 区块在任何场景均不得移除或弱化
- **斜杠命令协议归位**：协议说明只写在注释块内，不在 AGENTS.md 可见正文展开
- **`*` 标记永不丢失**：写入前提取，写入后核验
- **`packages.md` 承载关键词**：包定位关键词写在 `packages.md` 文件内，不在 AGENTS.md 展开
- **自定义工作约束归档**：临时输出目录、测试记录位置、协作偏好等用户自定义规则写入 wiki 文件，AGENTS.md 只保留索引条目
- **外部文件引用**：`{../../DESIGN.md}` 可引用项目内任意路径
- **只读目录**：`/mnt/skills/` 下文件不可修改
- **幂等性**：重复执行不产生重复条目
- **子包覆盖全局**：子包 `conventions.md` 顶部注明 `> 以下规范覆盖根级 conventions`
- **语言跟随**：文档语言跟随项目已有文档
- **未初始化的包**：根级索引标注 `(未初始化)`，不阻断其他包正常使用
