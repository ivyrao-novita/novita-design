# 95%+ 设计还原度 —— 能力差距分析与改进方案

> 目标：产品经理和研发人员用自然语言调用 Skill，生成的页面设计稿还原度达到 95% 以上。
> 分析基准：当前 Novita AI 项目的实际代码 vs AI 生成的 Pencil 设计稿。

---

## 零、已完成的基础建设（ppio-ui-skill 双产品架构改造）

> 在分析差距之前，需要先盘点已经完成的重要基础工作。
> 以下改造工作在 `ppio-ui-skill` 仓库的 `refactor/dual-design-system` 分支完成，
> 共 **13 个 commit，+11,470 行代码，涉及 126 个文件**。

### 0.1 架构改造概览

**改造前**（main 分支，单产品架构）：
```
data/
├── colors.csv           ← 单一混合色板
├── styles.csv           ← 混合风格
├── products.csv         ← 2 个产品挤在 1 个文件
├── typography.csv       ← 共享字体
└── tokens/novita.tokens.json
```

**改造后**（refactor/dual-design-system 分支，双产品架构）：
```
data/
├── shared/              ← 产品无关（charts, landing, ux, icons, react, web, ui-reasoning）
├── novita/              ← Novita AI 专属（绿色 #23D57C, TT Interphases Pro）
│   ├── product.csv, colors.csv, styles.csv, typography.csv
│   ├── sidebar.csv      ← 111 行，4 种产品变体的侧边栏结构
│   ├── header.csv       ← 36 行，Console 顶部导航规范
│   ├── design-system.csv ← 225 行，由 sync_tokens.py 自动生成
│   └── tokens/
│       ├── novita.tokens.json  ← 完整 Token（100+ 原语 + 组合 + 交互态）
│       └── schema.json
└── ppio/                ← PPIO Cloud 专属（蓝色 #1161fe, PingFang SC）
    ├── product.csv, colors.csv, styles.csv, typography.csv
    ├── sidebar.csv      ← 骨架（待补充）
    ├── header.csv       ← 骨架（待补充）
    ├── design-system.csv
    └── tokens/
        ├── ppio.tokens.json  ← 骨架（仅基础色）
        └── schema.json
```

### 0.2 新增的核心能力

| 能力 | 改造内容 | 文件 | 行数 |
|------|---------|------|------|
| **产品路由引擎** | `resolve_data_dir()` 函数，按 `--product` 参数路由到对应产品目录 | `core.py` | +300 |
| **CLI 产品参数** | `--product novita\|ppio` 全链路传递 | `search.py` | +113 |
| **设计系统生成器** | 5 域并行搜索 + 推理规则，支持产品上下文 | `design_system.py` | +1,068 |
| **Token 同步工具** | JSON → CSV 自动展平 + SKILL.md 哨兵注入 | `sync_tokens.py` | +481 |
| **Token 验证工具** | 4 层验证（结构/引用/状态/CSS 变量） | `validate_tokens.py` | +378 |
| **SKILL.md 约束注入** | `<!-- BEGIN/END:AUTO_GENERATED_HARD_CONSTRAINTS:novita -->` 哨兵 | `sync_tokens.py` | — |
| **模板系统** | `templates/base/` + `templates/platforms/` (Claude/Cursor 配置) | templates/ | +393 |
| **参考截图库** | 15 张 Novita Console 截图（对话框/筛选器/布局/悬停态等） | docs/screenshots/ | 15 张 |

### 0.3 Novita 专属数据资产（已创建）

| 数据文件 | 行数 | 内容 |
|---------|------|------|
| `novita/design-system.csv` | 225 行 | 自动生成：20+ 品牌色 + 语义色 + 排版 + 圆角/阴影/间距原语 |
| `novita/sidebar.csv` | 111 行 | 4 个产品变体（Home/Model APIs/GPUs/Agent Sandbox）完整菜单结构 |
| `novita/header.csv` | 36 行 | Console 顶部导航（标题/Credits/团队切换/通知/头像） |
| `novita/tokens/novita.tokens.json` | ~430 行 | 完整层级：primitives → composites → interactions |
| `novita/colors.csv` | 1 行 | 品牌主色 #23D57C + 辅助色 |
| `novita/typography.csv` | 1 行 | TT Interphases Pro + Inter 字体配置 |
| `novita/styles.csv` | 1 行 | Flat Design + Minimalism 风格推荐 |
| `novita/product.csv` | 1 行 | 产品定义：AI API 平台，绿色品牌 |

### 0.4 三路同步机制

```
data/（源头）
  ↓ 手动复制
src/ppio-ui-skill/data/（Skill 插件运行时）
  ↓ 手动复制
cli/assets/data/（npm CLI 发布包）
```

### 0.5 Token 管理工作流

```
1. 编辑 novita.tokens.json（手动或从 Figma 提取）
       ↓
2. python3 validate_tokens.py --product novita（4 层验证）
       ↓
3. python3 sync_tokens.py --product novita --all
   ├── 生成 design-system.csv（BM25 可搜索）
   └── 注入 SKILL.md（哨兵标记间的约束表格）
       ↓
4. 手动同步到 src/ppio-ui-skill/ 和 cli/assets/
```

---

## 一、当前能力评估

### 1.1 已具备的能力

| 能力维度 | 当前状态 | 覆盖度 | 说明 |
|---------|---------|--------|------|
| **双产品架构** | ✅ 已完成 | 95% | Novita 完整，PPIO 骨架 |
| **产品路由引擎** | ✅ 已完成 | 100% | `--product novita\|ppio` 全链路 |
| **Token 同步/验证** | ✅ 已完成 | 95% | JSON → CSV → SKILL.md 自动化 |
| **品牌颜色系统** | ✅ 完备 | 95% | 67 个 CSS 变量，含 light/dark 主题 |
| **排版 Token** | ✅ 完备 | 90% | 字号/行高/字重定义完整，但字体文件映射待补充 |
| **间距系统** | ✅ 完备 | 90% | 0-64px 共 11 级间距 |
| **圆角系统** | ✅ 完备 | 95% | 从 0px 到 9999px (pill) 完整覆盖 |
| **按钮组件 Token** | ✅ 完备 | 95% | 9 种变体 × 4 种状态 = 36 组规范 |
| **表单组件 Token** | ✅ 完备 | 90% | Input/Select/Checkbox/Radio/Switch |
| **标签 Token** | ✅ 完备 | 95% | 7 种语义颜色 |
| **对话框 Token** | ✅ 完备 | 90% | Overlay + Content 规范 |
| **Sidebar 结构 (Console)** | ✅ 详细 | 85% | 111 行 CSV，4 种产品变体，含 Figma 节点 ID |
| **Header 结构 (Console)** | ✅ 详细 | 80% | 36 行 CSV，含 Credits/团队切换/通知组件 |
| **BM25 搜索引擎** | ✅ 可用 | 90% | 14 个搜索域，46+ CSV 数据文件 |
| **设计系统生成器** | ✅ 可用 | 90% | 5 域并行搜索 + 推理规则 + Master/Override 持久化 |
| **Pencil MCP 集成** | ✅ 可用 | 80% | 可创建/编辑/截图 .pen 文件 |
| **Playwright 截图** | ✅ 可用 | 85% | 可截取线上页面作为参考 |
| **参考截图库** | ✅ 已有 | 70% | 15 张 Novita Console 截图 |
| **平台模板** | ✅ 已有 | 80% | Claude/Cursor 配置模板 |

### 1.2 当前还原度评估

以 Benchmarks 页面为例（代码 → 设计稿）：

| 区域 | 代码实际效果 | 设计稿还原 | 差距原因 |
|------|------------|-----------|---------|
| **Header** | 80px 高，SVG Logo，8 个导航项 | ❌ 最初 64px，文字 Logo | 缺少 Header 模板 |
| **Hero 标题** | 36px/600/dark-1 | ❌ 最初 700 字重 | 缺少页面级字体规范映射 |
| **数据表格** | 12 列排序表格 | ✅ 基本还原 | 分数颜色编码正确 |
| **Legend 区域** | 2 列 Grid 布局 | ✅ 基本还原 | — |
| **Footer** | 4 列菜单 + Logo + 社交图标 | ❌ 最初仅单行文字 | 缺少 Footer 模板 |

**当前综合还原度估计：约 70-75%**

---

## 二、差距分析

### 2.1 核心差距一览

```
┌──────────────────────────────────────────────────────────────┐
│                     95% 还原度目标                            │
├──────────────────────────────────────────────────────────────┤
│  ████████████████████████░░░░░░░░  当前水平 ~75%             │
│                          ↑                                   │
│              需要弥补的 20% 差距                               │
├──────────────────────────────────────────────────────────────┤
│  差距构成：                                                   │
│  ├── 页面模板缺失 .............. ~8%                          │
│  ├── 组件深度不足 .............. ~4%                          │
│  ├── 全局布局规范缺失 .......... ~3%                          │
│  ├── 实际代码样式映射缺失 ...... ~3%                          │
│  └── 图标/图片资产缺失 ......... ~2%                          │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 差距详细分析

---

#### 差距 1: 页面级模板缺失（影响 ~8%）

**现状**：
- AI 每次都从零构建 Header、Footer、Sidebar
- 没有预置的页面骨架模板
- 每次生成都可能出现不一致

**缺失内容**：

| 模板 | 需要的内容 | 当前状态 |
|------|-----------|---------|
| **Homepage Header** | Logo(SVG) + 导航项(Model APIs/GPUs/Agent Sandbox/Pricing/Benchmarks/Docs/Blog) + Auth按钮 + 80px 高度 | ❌ 无模板，每次手动构建 |
| **Homepage Footer** | Logo + 系统状态 + 社交图标 + 4 列菜单(CONTACT/RESOURCES/COMPANY/PARTNERS) + 版权信息 + 具体链接项 | ❌ 无模板，每次手动构建 |
| **Console Header** | 页面标题 + Credits + 团队切换 + 通知 + 头像 + 54px 高度 | ⚠️ CSV 有数据，但无 Pencil 组件模板 |
| **Console Sidebar** | Logo + 产品切换 + 导航菜单 + 快捷入口 + Docs | ⚠️ CSV 有详细数据，但无 Pencil 组件模板 |
| **CTA Section** | "Ready to build smarter?" + Get Started + Book a Demo | ❌ 无模板 |
| **空状态页面** | 404 / 空数据 / 加载中 | ❌ 无模板 |

**解决方案**：
1. 在 novita-design 仓库中创建 `templates/` 目录
2. 为每种模板创建独立的 .pen 文件或在主设计系统中添加可复用组件
3. 模板应包含所有实际导航项、链接、图标的精确数据

---

#### 差距 2: 设计系统组件深度不足（影响 ~4%）

**现状**：
- Pencil 设计系统（Novita Design System-2）仅有 24 个组件
- 缺少多种实际项目中常用的组件

**已有 vs 缺失组件对比**：

| 组件 | Pencil 设计系统 | ppio-ui-skill Token | 状态 |
|------|----------------|--------------------|----|
| Button (4变体) | ✅ Default, Outline, Ghost, Destructive | ✅ 9 变体完整 | ⚠️ Pencil 少 5 个变体 |
| Input | ✅ Default, WithLabel | ✅ 5 状态完整 | ✅ 基本够用 |
| Checkbox | ✅ Unchecked, Checked, WithLabel | ✅ 4 状态 | ✅ |
| Radio | ✅ Unchecked, Checked | ✅ 4 状态 | ✅ |
| Switch | ✅ Off, On | ✅ 4 状态 | ✅ |
| Badge | ✅ Default, Secondary, Destructive, Outline | ✅ 7 变体 | ⚠️ Pencil 少 3 个变体 |
| Alert | ✅ Success, Destructive | ✅ 4 变体 | ⚠️ Pencil 少 2 个变体 |
| Select | ✅ Default | ✅ Trigger + Content + Option | ✅ |
| Table | ✅ Container, Row | ✅ 3 状态 | ✅ |
| **Dialog/Modal** | ❌ 缺失 | ✅ Overlay + Content | ❌ 需要添加 |
| **Toast** | ❌ 缺失 | ✅ 3 变体 | ❌ 需要添加 |
| **Tabs** | ❌ 缺失 | ✅ List + Trigger 4 状态 | ❌ 需要添加 |
| **Accordion** | ❌ 缺失 | ✅ Trigger + Content | ❌ 需要添加 |
| **Tooltip** | ❌ 缺失 | ✅ 完整 | ❌ 需要添加 |
| **Popover** | ❌ 缺失 | ✅ 完整 | ❌ 需要添加 |
| **Dropdown Menu** | ❌ 缺失 | ✅ Content + Item + Item-danger | ❌ 需要添加 |
| **Skeleton** | ❌ 缺失 | ✅ 完整 | ❌ 需要添加 |
| **Sidebar Item** | ❌ 缺失 | ✅ Default + Group 状态 | ❌ 需要添加 |
| **Breadcrumb** | ❌ 缺失 | ❌ 缺失 | ❌ 需要补充 |
| **Pagination** | ❌ 缺失 | ❌ 缺失 | ❌ 需要补充 |
| **Avatar** | ❌ 缺失 | ❌ 缺失 | ❌ 需要补充 |
| **Navigation Menu** | ❌ 缺失 | ❌ 缺失 | ❌ 需要补充 |
| **Progress Bar** | ❌ 缺失 | ❌ 缺失 | ❌ 需要补充 |
| **Card** | ❌ 缺失 | ✅ 4 状态 | ❌ 需要添加 |

**需要新增的 Pencil 组件**：至少 15 个（Dialog, Toast, Tabs, Accordion, Tooltip, Popover, Dropdown, Skeleton, Sidebar Item, Card, Breadcrumb, Pagination, Avatar, Navigation Menu, Progress Bar）

---

#### 差距 3: 全局布局规范缺失（影响 ~3%）

**现状**：AI 每次需要猜测或从代码中推断全局布局参数。

**缺失的布局规范**：

| 规范 | 实际代码值 | 当前 Skill 数据 | 状态 |
|------|-----------|----------------|------|
| `--header-height` | 80px (homepage) / 54px (console) | ⚠️ Header CSV 有 54px | ⚠️ Homepage 80px 未明确记录 |
| `--max-width` | 1280px | ❌ 未记录 | ❌ 需要添加 |
| `--spacing-layout-x` | 24px(mobile) / 60px(tablet) / 120px(desktop) | ❌ 未记录 | ❌ 需要添加 |
| 响应式断点 | 640px / 768px / 1024px / 1280px / 1440px | ❌ 未记录 | ❌ 需要添加 |
| 页面最大宽度容器 | `.max_width_container` + `.px-web` | ❌ 未记录 | ❌ 需要添加 |
| Z-index 层级 | 999(header) / 100(modal) / 50(dropdown) | ❌ 未记录 | ❌ 需要添加 |
| Footer 区域高度 | 约 400-500px（完整 Footer） | ❌ 未记录 | ❌ 需要添加 |

---

#### 差距 4: 代码样式 → 设计映射缺失（影响 ~3%）

**现状**：
- ppio-ui-skill 中的 Token 定义和实际代码中的 CSS/SCSS 变量有偏差
- 缺少从 `var(--dark-1)` 到 `#292827` 的快速映射表
- 缺少从 SCSS Mixin 到设计 Token 的映射

**需要建立的映射关系**：

```
实际代码 SCSS Mixin          →  ppio-ui-skill Token         →  Pencil 设计值
@include font-h1             →  font-h1 (80px/74px/600)     →  fontSize:80, fontWeight:600
@include font-body-medium    →  font-body-medium             →  fontSize:16, fontWeight:500
var(--dark-1) #292827        →  --dark-1                     →  fill: "#292827"
var(--gray-2) #e7e6e2        →  --gray-2                     →  stroke: "#e7e6e2"
.max_width_container          →  layout-max-width: 1280px     →  width: 1280
```

**缺失映射表**：

| 代码中的类/变量 | 实际值 | 需要录入到 |
|----------------|-------|-----------|
| `var(--dark-1)` | #292827 | design-system.csv |
| `var(--dark-2)` | #4f4e4a | design-system.csv |
| `var(--dark-3)` | #9e9c98 | design-system.csv |
| `var(--gray-1)` | #cbc9c4 | design-system.csv |
| `var(--gray-2)` | #e7e6e2 | design-system.csv |
| `var(--gray-3)` | #f5f5f5 | design-system.csv |
| `var(--gray-4)` | #fafafa | design-system.csv |
| `@include font-h1` | 80px/74px/600/-1.6px | typography 补充 |
| `@include font-h2` | 56px/56px/600/-1.12px | typography 补充 |
| `@include font-h3` | 48px/48px/600/-0.96px | typography 补充 |
| `@include font-h4` | 24px/38px/600/0 | typography 补充 |
| `@include font-h5` | 20px/24px/600/-0.4px | typography 补充 |
| `@include font-body` | 16px/24px/400/0 | typography 补充 |
| `@include font-body-medium` | 16px/20px/500/0 | typography 补充 |
| `@include font-menu` | 14px/18px/400/0 | ✅ 已有 |
| `@include font-subtle` | 14px/20px/400/0 | typography 补充 |
| `@include font-small` | 12px/20px/400/0 | typography 补充 |

---

#### 差距 5: 图标与图片资产缺失（影响 ~2%）

**现状**：
- 设计稿中无法嵌入实际的 SVG 图标
- Logo 只能用文字模拟
- 社交图标只能用字符替代

**缺失资产**：

| 资产 | 用途 | 当前替代方案 |
|------|------|------------|
| Novita Logo SVG | Header/Footer | 文字 "novita.ai" |
| Lucide Icons | 导航/操作图标 | 无法渲染 |
| 社交图标 (X/YouTube/LinkedIn/Discord) | Footer | 文字字符 |
| "new" 标签图片 | Agent Sandbox 导航项 | 无 |
| 品牌插图 | 空状态/404/加载 | 无 |

**可能的解决方案**：
1. 在 Pencil 中使用 `icon_font` 节点（已支持）
2. 使用 Pencil 的 `G()` 操作生成 AI 图标
3. 在设计系统中预置常用图标组件
4. 将 SVG 转为 Pencil path 节点

---

## 三、从 75% 到 95% 的改进路线图

### Phase 1: 基础补全（预计提升到 ~85%）

**时间估计：1-2 周**

| 任务 | 优先级 | 预计工时 | 效果 |
|------|--------|---------|------|
| 创建 Homepage Header 模板组件 | P0 | 2h | 所有 Homepage 类页面头部一致 |
| 创建 Homepage Footer 模板组件 | P0 | 3h | 所有 Homepage 类页面底部一致 |
| 创建 Console Header 模板组件 | P0 | 2h | Console 页面头部一致 |
| 创建 Console Sidebar 模板组件（4个产品变体） | P0 | 4h | Console 页面侧边栏一致 |
| 补全全局布局规范到 design-system.csv | P0 | 1h | AI 获取正确的宽度/间距 |
| 补全 SCSS Mixin → Token 映射表 | P1 | 2h | 字体样式精准还原 |
| 补全 CSS 变量 → 色值映射表 | P1 | 1h | 颜色精准还原 |

**Phase 1 交付物**：
- `novita-design/templates/homepage-header.pen` — Header 模板
- `novita-design/templates/homepage-footer.pen` — Footer 模板
- `novita-design/templates/console-header.pen` — Console Header 模板
- `novita-design/templates/console-sidebar-*.pen` — 4 个 Sidebar 变体模板
- `ppio-ui-skill/data/novita/layout.csv` — 全局布局规范
- `ppio-ui-skill/data/novita/scss-token-mapping.csv` — 样式映射表

---

### Phase 2: 组件完善（预计提升到 ~90%）

**时间估计：1-2 周**

| 任务 | 优先级 | 预计工时 | 效果 |
|------|--------|---------|------|
| 在 Pencil 设计系统中新增 Dialog 组件 | P1 | 1h | 模态框页面可还原 |
| 新增 Tabs 组件 | P1 | 1h | Tab 切换页面可还原 |
| 新增 Toast 组件 | P1 | 0.5h | 通知提示可还原 |
| 新增 Dropdown Menu 组件 | P1 | 1h | 下拉菜单可还原 |
| 新增 Card 组件 | P1 | 1h | 卡片布局页面可还原 |
| 新增 Tooltip/Popover 组件 | P2 | 1h | 工具提示可还原 |
| 新增 Skeleton 加载态组件 | P2 | 0.5h | 加载状态可还原 |
| 新增 Breadcrumb/Pagination 组件 | P2 | 1h | 导航组件可还原 |
| 新增 Avatar 组件 | P2 | 0.5h | 用户头像可还原 |
| 新增 Progress Bar 组件 | P2 | 0.5h | 进度指示可还原 |
| 补全 Badge 额外 3 个变体 | P2 | 0.5h | 标签完整覆盖 |
| 补全 Alert 额外 2 个变体 | P2 | 0.5h | 提示框完整覆盖 |
| 补全 Button 额外 5 个变体 | P1 | 1h | 按钮完整覆盖 |

**Phase 2 交付物**：
- Pencil 设计系统组件数从 24 个增加到 ~40 个
- ppio-ui-skill Token 与 Pencil 组件 1:1 对应

---

### Phase 3: 精细化调优（预计提升到 ~95%）

**时间估计：2-3 周**

| 任务 | 优先级 | 预计工时 | 效果 |
|------|--------|---------|------|
| 建立「页面截图对比」自动化验证 | P1 | 4h | 客观度量还原度 |
| 创建常见页面模板库（Pricing/Models/Dashboard） | P1 | 8h | 常见页面直接套模板 |
| 建立图标资产映射（Lucide Icon → Pencil icon_font） | P1 | 3h | 图标可正确渲染 |
| 创建响应式变体规范 | P2 | 4h | 移动端/平板设计可还原 |
| 补充 CTA Section 模板 | P2 | 2h | 营销页面底部可还原 |
| 补充空状态/错误页模板 | P2 | 2h | 异常页面可还原 |
| 建立 Novita 品牌图片资产库 | P2 | 3h | Logo/插图可正确引用 |
| 编写自动化 Skill 触发脚本 | P2 | 4h | 一键生成常见页面 |
| 建立页面还原度评分标准 | P3 | 2h | 可量化评估 |

**Phase 3 交付物**：
- `novita-design/templates/` — 8-10 个页面模板
- 图标映射表
- 自动化验证脚本
- 还原度评分标准文档

---

## 四、ppio-ui-skill 分支状态与待办

### 4.0 refactor/dual-design-system 分支（未合入 main）

> **重要**：以下改造工作已完成但尚未合入 `ppio-ui-skill` 的 `main` 分支。
> 当前插件缓存使用的是发布版本（2.0.1），双产品架构功能已通过本地 symlink 生效。

**分支状态**：
```
ppio-ui-skill 仓库
├── main 分支                    ← 上游原始版本（单产品，通用 UI Skill）
└── refactor/dual-design-system  ← 你的改造（双产品，Novita + PPIO 专属）
    ├── 13 个 commit
    ├── +11,470 行 / -1,007 行
    └── 126 个文件变更
```

**改造工作量汇总**：

| 类别 | 文件数 | 新增行数 | 说明 |
|------|--------|---------|------|
| **搜索引擎改造** (core.py) | 1 | +300 | 产品路由、域分类、向后兼容 |
| **CLI 改造** (search.py) | 1 | +113 | `--product` 参数全链路 |
| **设计系统生成器** (design_system.py) | 1 | +1,068 | 5 域并行 + 推理 + 产品上下文 |
| **Token 同步** (sync_tokens.py) | 1 | +481 | JSON → CSV → SKILL.md 哨兵注入 |
| **Token 验证** (validate_tokens.py) | 1 | +378 | 4 层验证框架 |
| **Novita 数据** (CSV + JSON) | 8 | ~600 | 完整 Token + Sidebar + Header |
| **PPIO 骨架** (CSV + JSON) | 8 | ~40 | 最小可用骨架 |
| **共享数据迁移** | 10 | ~600 | charts/landing/ux/icons 等 |
| **文档** (SKILL.md + CLAUDE.md) | 2 | +960 | 完整使用说明 + 架构文档 |
| **截图参考** | 15 | — | Novita Console 各页面/状态截图 |
| **模板系统** | 4 | +393 | Claude/Cursor 平台配置 |
| **CLI/src 同步** | ~70 | ~7,000 | 三路同步（data → src → cli） |

**后续需处理**：
1. **合入 main 或发布新版本** — 当前依赖本地 symlink，其他团队成员无法使用
2. **发布为 npm 包或 Claude 插件更新** — 让安装即用
3. **编写 PR 说明** — 记录架构变更便于 Code Review

### 4.1 ppio-ui-skill 需要补充的 CSV 数据

| 文件 | 需要补充的内容 | 预计行数 |
|------|---------------|---------|
| `novita/layout.csv` (新建) | 全局布局规范：max-width, header-height, spacing-layout-x, 断点, z-index | ~30 行 |
| `novita/scss-mapping.csv` (新建) | SCSS Mixin → Token 映射：font-h1 到 font-small 共 10+ 个 Mixin | ~25 行 |
| `novita/page-templates.csv` (新建) | 页面级模板规范：各页面的区域组成、间距、宽度 | ~50 行 |
| `novita/icons-mapping.csv` (新建) | Lucide Icon → 用途映射：ChevronDown, Bell, User, Menu 等 | ~40 行 |
| `novita/design-system.csv` (更新) | 补充缺失的 CSS 变量色值映射 | ~15 行追加 |
| `novita/typography.csv` (更新) | 补充完整的 font-h1~font-small Mixin 映射 | ~10 行追加 |
| `novita/header.csv` (更新) | 补充 Homepage Header 规范（当前只有 Console Header） | ~20 行追加 |
| `novita/footer.csv` (新建) | Homepage Footer 完整规范 | ~40 行 |

**总计需要新增/更新约 230 行 CSV 数据**

### 4.2 Pencil 设计系统需要新增的组件

| 组件名 | 变体数 | Token 来源 | 优先级 |
|--------|--------|-----------|--------|
| Dialog/Modal | 1 (Overlay+Content) | ✅ novita.tokens.json | P1 |
| Toast | 3 (success/error/info) | ✅ novita.tokens.json | P1 |
| Tabs | 1 (List+Trigger) | ✅ novita.tokens.json | P1 |
| Card | 1 (default 4状态) | ✅ novita.tokens.json | P1 |
| Dropdown Menu | 1 (Content+Item) | ✅ novita.tokens.json | P1 |
| Button/Secondary | 1 | ✅ novita.tokens.json | P1 |
| Button/Link | 1 | ✅ novita.tokens.json | P1 |
| Button/Icon-gray | 1 | ✅ novita.tokens.json | P2 |
| Button/Icon-primary | 1 | ✅ novita.tokens.json | P2 |
| Button/Icon-outline | 1 | ✅ novita.tokens.json | P2 |
| Accordion | 1 (Trigger+Content) | ✅ novita.tokens.json | P2 |
| Tooltip | 1 | ✅ novita.tokens.json | P2 |
| Popover | 1 | ✅ novita.tokens.json | P2 |
| Skeleton | 1 | ✅ novita.tokens.json | P2 |
| Sidebar Item | 2 (default+group) | ✅ novita.tokens.json | P1 |
| Breadcrumb | 1 | ❌ 需定义 Token | P2 |
| Pagination | 1 | ❌ 需定义 Token | P2 |
| Avatar | 1 | ❌ 需定义 Token | P2 |
| Navigation Menu | 1 | ❌ 需定义 Token | P2 |
| Progress Bar | 1 | ❌ 需定义 Token | P3 |
| Badge/Success | 1 | ✅ novita.tokens.json | P2 |
| Badge/Warning | 1 | ✅ novita.tokens.json | P2 |
| Badge/Info | 1 | ✅ novita.tokens.json | P2 |
| Alert/Warning | 1 | ✅ novita.tokens.json | P2 |
| Alert/Info | 1 | ✅ novita.tokens.json | P2 |

**总计需要新增约 25 个组件（含变体）**

### 4.3 需要创建的模板文件

| 模板 | 描述 | 需要包含的内容 |
|------|------|---------------|
| `homepage-shell.pen` | Homepage 通用外壳 | Header(80px) + 内容区占位 + Footer(完整4列) |
| `console-shell.pen` | Console 通用外壳 | Console Header(54px) + Sidebar(250px) + 内容区占位 |
| `console-sidebar-home.pen` | Home 侧边栏 | Logo + [Home/Model APIs/GPUs/Agent Sandbox] + Quick Access |
| `console-sidebar-modelapi.pen` | Model APIs 侧边栏 | Logo + Switcher + Model Library + LLM组 + IMAGE组 |
| `console-sidebar-gpus.pen` | GPUs 侧边栏 | Logo + Switcher + Explore + Templates + Instances组 + Serverless组 |
| `console-sidebar-sandbox.pen` | Agent Sandbox 侧边栏 | Logo + Switcher + Sandbox + Quick Access |
| `pricing-page.pen` | 价格页模板 | Header + Hero + 价格表格 + FAQ + CTA + Footer |
| `model-library.pen` | 模型库模板 | Header + 搜索栏 + 筛选器 + 模型卡片网格 + Footer |
| `landing-page.pen` | 通用着陆页模板 | Header + Hero + Features + Social Proof + CTA + Footer |

---

## 五、技术架构改进建议

### 5.1 当前架构的瓶颈

```
当前流程：
用户指令 → AI 读取代码 → AI 查询 Skill → AI 从零构建 Pencil 节点 → 截图验证
                                                    ↑
                                              这一步最容易出错
                                              每次都从零开始
                                              没有模板复用
```

### 5.2 理想架构

```
改进后流程：
用户指令 → AI 识别页面类型
                │
                ▼
         AI 选择页面模板（homepage-shell / console-shell / landing-page）
                │
                ▼
         AI 从模板库复制基础结构（Header + Footer + Sidebar 已完备）
                │
                ▼
         AI 只需要构建主体内容区域（组件从设计系统中引用）
                │
                ▼
         截图验证 → 微调 → 完成
```

**关键改进点**：
1. **模板优先**：不再从零构建，而是基于模板扩展
2. **组件引用**：主体内容使用设计系统中的可复用组件
3. **布局规范自动应用**：max-width、间距等从 CSV 自动读取

### 5.3 Skill 增强建议

| 增强方向 | 说明 | 实现方式 |
|---------|------|---------|
| **新增 `--domain template` 搜索域** | 支持查询页面模板 | 新建 `novita/templates.csv` |
| **新增 `--domain layout` 搜索域** | 支持查询全局布局规范 | 新建 `novita/layout.csv` |
| **新增 `--domain footer` 搜索域** | 支持查询 Footer 结构 | 新建 `novita/footer.csv` |
| **增强 Header 搜索域** | 补充 Homepage Header 数据 | 更新 `novita/header.csv` |
| **新增组件映射命令** | `--component-map` 列出 Token → Pencil 组件 ID 映射 | 新建映射表 |
| **新增截图对比命令** | 自动对比线上页面与设计稿 | 脚本增强 |

---

## 六、还原度评分标准（建议）

### 评分维度

| 维度 | 权重 | 5分标准（95%+） | 3分标准（85%） | 1分标准（<70%） |
|------|------|----------------|---------------|----------------|
| **布局结构** | 25% | 区域划分完全一致，间距误差 ≤2px | 区域划分正确，间距有偏差 | 区域缺失或错位 |
| **颜色准确性** | 20% | 所有颜色与品牌规范完全一致 | 主要颜色正确，细节有偏差 | 颜色明显不匹配 |
| **字体排版** | 20% | 字号/字重/行高完全匹配 | 字号正确，字重/行高有偏差 | 字体样式明显不同 |
| **组件完整性** | 20% | 所有 UI 组件正确渲染 | 大部分组件正确，个别缺失 | 多个组件缺失或错误 |
| **内容准确性** | 15% | 文案/数据/链接项完全一致 | 主要内容正确，细节有遗漏 | 内容明显不匹配 |

### 自动化验证建议

```
验证脚本流程：
1. Playwright 截图线上页面（或本地 dev server）
2. Pencil MCP 截图设计稿对应区域
3. 图像对比（SSIM 或像素级对比）
4. 输出差异报告和还原度分数
```

---

## 七、总结与行动建议

### 关键发现

1. **ppio-ui-skill 双产品架构改造已完成**（13 commit, +11,470 行），Novita Token 体系完备，但分支未合入 main
2. **当前还原度约 75%**，主要差距在页面模板和组件深度
3. **Token 数据基础扎实**（225 行 design-system.csv + 111 行 sidebar.csv + 36 行 header.csv），但缺少到实际代码 SCSS Mixin 的映射
4. **Pencil 设计系统组件不足**（24 个 vs 需要 ~50 个），且和 Skill Token 未 1:1 对应
5. **缺少页面级模板**（Homepage Header/Footer、Console Shell）是最大的效率瓶颈
6. **图标和图片资产**是精细还原的最后一公里
7. **三路同步（data → src → cli）为手动操作**，存在不一致风险

### 优先行动项（按 ROI 排序）

| 序号 | 行动 | 预计耗时 | 预计还原度提升 | 涉及仓库 |
|------|------|---------|---------------|---------|
| 0 | **合入/发布 dual-design-system 分支** | 2h | 前提条件 | ppio-ui-skill |
| 1 | 创建 Homepage Header/Footer 模板 | 5h | +5% → 80% | novita-design |
| 2 | 补全布局规范 + SCSS Mixin 映射 + Footer CSV | 4h | +3% → 83% | ppio-ui-skill |
| 3 | 创建 Console Shell 模板（Header + Sidebar） | 6h | +3% → 86% | novita-design |
| 4 | 新增 10 个高频组件到 Pencil 设计系统 | 8h | +4% → 90% | novita-design |
| 5 | 创建 5 个常见页面模板 | 10h | +3% → 93% | novita-design |
| 6 | 图标资产映射 + 品牌资产库 | 6h | +2% → 95% | ppio-ui-skill + novita-design |

**总计约 41 小时工作量，可分 3 个 Phase 在 4-6 周内完成。**

### 长期愿景

```
Phase 0 (立即):       ██░░░░░░░░░░░░░░░░░░  — dual-design-system 合入 main 并发布
Phase 1 (Week 1-2):  ████████░░░░░░░░░░░░  85% — 基础模板 + 布局规范 + Footer CSV
Phase 2 (Week 3-4):  █████████████░░░░░░░  90% — 组件完善 + Token ↔ Pencil 组件对齐
Phase 3 (Week 5-6):  ███████████████████░  95% — 精细化 + 自动化验证 + 三路同步自动化
Phase 4 (持续迭代):   ████████████████████  97%+ — 新页面模板 + PPIO 补全
```

---

## 附录 A: 文件清单汇总

### 已有文件（ppio-ui-skill refactor/dual-design-system 分支）
```
ppio-ui-skill/
├── CLAUDE.md                       ✅ 双产品架构文档（144 行改动）
├── .claude/skills/ppio-ui-skill/
│   └── SKILL.md                    ✅ 完整使用指南 + 自动注入约束（+815 行）
├── scripts/
│   ├── core.py                     ✅ 产品路由搜索引擎（+300 行）
│   ├── search.py                   ✅ CLI 入口 --product 参数（+113 行）
│   ├── design_system.py            ✅ 5 域并行生成器（+1,068 行）
│   ├── sync_tokens.py              ✅ Token → CSV → SKILL.md 同步（+481 行）
│   └── validate_tokens.py          ✅ 4 层 Token 验证（+378 行）
├── data/novita/
│   ├── colors.csv                  ✅ 品牌色完整
│   ├── design-system.csv (225 行)  ✅ 自动生成（勿手动编辑）
│   ├── header.csv (36 行)          ⚠️ 仅 Console Header（缺 Homepage Header）
│   ├── product.csv                 ✅ 产品定义
│   ├── sidebar.csv (111 行)        ✅ 4 变体，含 Figma 节点 ID
│   ├── styles.csv                  ✅ 风格推荐
│   ├── typography.csv              ⚠️ 缺 SCSS Mixin 映射
│   └── tokens/
│       ├── novita.tokens.json      ✅ 完整层级（~430 行）
│       └── schema.json             ✅ Token 结构定义
├── data/ppio/                      ⚠️ 骨架状态（仅基础色和产品定义）
├── data/shared/                    ✅ 产品无关数据（charts/landing/ux/icons 等）
├── docs/screenshots/               ✅ 15 张 Novita Console 参考截图
├── templates/                      ✅ Claude/Cursor 平台配置
├── src/ppio-ui-skill/              ✅ Skill 运行时镜像
└── cli/assets/                     ✅ npm CLI 发布包镜像

novita-design/
├── benchmarks-page.pen  (224 KB)   ✅ 含 24 组件设计系统 + 页面设计
└── docs/                           ✅ 文档目录（教程 + 分析）
```

### 需要新建的文件
```
ppio-ui-skill/data/novita/
├── layout.csv           (新建)     全局布局规范（max-width/断点/间距/z-index）
├── footer.csv           (新建)     Homepage Footer 结构（菜单项/社交图标/版权）
├── scss-mapping.csv     (新建)     SCSS Mixin → Token 映射（font-h1~font-small）
├── page-templates.csv   (新建)     页面级模板规范（区域组成/间距/宽度）
└── icons-mapping.csv    (新建)     Lucide Icon → 用途/Pencil icon_font 映射

novita-design/templates/
├── homepage-shell.pen   (新建)     Homepage 通用模板（Header 80px + Footer 4列）
├── console-shell.pen    (新建)     Console 通用模板（Header 54px + Sidebar 250px）
├── console-sidebar-home.pen        Home 侧边栏
├── console-sidebar-modelapi.pen    Model APIs 侧边栏
├── console-sidebar-gpus.pen        GPUs 侧边栏
└── console-sidebar-sandbox.pen     Agent Sandbox 侧边栏
```

---

## 附录 B: ppio-ui-skill 当前覆盖域统计

| 搜索域 | CSV 行数 | 产品范围 | 完善度 |
|--------|---------|---------|--------|
| product | 1 | Novita | ✅ 完善 |
| style | ~10 | Novita | ✅ 完善 |
| color | 1 | Novita | ✅ 完善 |
| typography | ~5 | Novita | ⚠️ 缺映射 |
| sidebar | 110 | Novita | ✅ 完善 |
| header | 30 | Novita (Console) | ⚠️ 缺 Homepage |
| design-system | 100 | Novita | ⚠️ 缺布局规范 |
| chart | ~25 | Shared | ✅ 完善 |
| landing | ~30 | Shared | ✅ 完善 |
| ux | ~40 | Shared | ✅ 完善 |
| icons | ~50 | Shared | ✅ 完善 |
| react | ~30 | Shared | ✅ 完善 |
| web | ~20 | Shared | ✅ 完善 |
| ui-reasoning | ~40 | Shared | ✅ 完善 |
| **footer** | **0** | **N/A** | **❌ 完全缺失** |
| **layout** | **0** | **N/A** | **❌ 完全缺失** |
| **page-templates** | **0** | **N/A** | **❌ 完全缺失** |
