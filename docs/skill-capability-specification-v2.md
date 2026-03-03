# ppio-ui-skill v3.0 — 全链路设计规范 Skill 能力体系规范

> **版本**: v3.0 (规划)  |  **基线**: v2.0.1 (当前发布) + refactor/dual-design-system (未合入)
> **目标**: 自然语言 → 设计稿 → 可发布代码 → 自动化校验 → 存档管理 全链路闭环
> **适用团队**: 产品经理 / UI 设计师 / 前端研发
> **日期**: 2026-03-03

---

## 一、Skill 定位与价值

### 一句话定位

**强规范绑定的全链路设计工程 Skill**——以设计系统 Token 为单一真相源（Single Source of Truth），驱动「自然语言描述 → 结构化设计稿 → 可发布前端代码 → 自动化合规校验 → 版本化存档回溯」五阶段闭环。

### 解决的团队痛点

| 痛点 | 当前状态 | v3.0 目标 |
|------|---------|----------|
| **设计规范执行靠人脑** | Token 存在但 AI 生成时不一定遵循 | 强制绑定：每个输出节点可溯源到 Token |
| **PM 描述模糊导致返工** | 自由文本 → AI 自由发挥 → 偏差大 | 结构化指令：缺字段自动补问 |
| **设计稿与代码双线漂移** | 手动比对，无自动校验 | 双向同步 + 自动差异报告 |
| **无法回溯设计决策** | Git commit 仅记录二进制差异 | 结构化变更日志 + 语义 diff |
| **新人上手成本高** | 需要逐个学 Token/组件/布局规则 | 自然语言输入 → Skill 自动查询规范 |
| **跨产品规范不隔离** | Novita/PPIO 混用 | 产品路由引擎严格隔离 |

---

## 二、核心设计原则

### 2.1 规范优先（Spec-First）

```
用户指令 ──→ Skill 查询 Token ──→ Token 约束下生成 ──→ 验证是否合规
                 ↑                                         │
                 └─────────── 不合规则自动修正 ←────────────┘
```

- 所有颜色必须引用 `novita.tokens.json` 中定义的值，禁止使用裸色值
- 所有字体必须使用 SCSS Mixin 对应的 Token（如 `font-h4` → 24px/38px/600）
- 所有间距必须使用 spacing Token（0/4/8/12/16/20/24/32/40/48/64px）
- 所有圆角必须使用 radius Token（0/2/4/6/8/12/9999px）

### 2.2 结构优先（Structure-First）

- 输入必须结构化（见第三节），拒绝完全模糊的描述
- 输出必须结构化：Pencil 节点树 / React 组件树 / 校验报告均有固定 schema
- 中间产物可序列化、可缓存、可 diff

### 2.3 工程对齐（Code-Aligned）

- 生成的设计稿节点命名与 `novita-home/src/app/` 目录结构对齐
- 生成的代码直接遵循项目 CLAUDE.md 中的约定（SCSS Modules + Tailwind + shadcn/ui）
- 文件路径、组件引用、样式变量名与实际代码库一致

### 2.4 可验证可回溯（Verifiable & Traceable）

- 每次生成附带「规范溯源清单」（哪些 Token 被引用）
- 每次变更附带「语义 diff」（不仅记录节点变化，还记录设计意图）
- 所有校验结果可导出为结构化报告

### 2.5 可扩展（Extensible）

- 新产品：复制 `data/novita/` 目录结构 → 填写 CSV → 注册到 `core.py`
- 新组件：定义 Token → `sync_tokens.py` 生成 CSV → Pencil 组件同步
- 新校验规则：添加到 `validate_tokens.py` 的验证层
- 新搜索域：在 `core.py` 的 `CSV_CONFIG` 中注册

---

## 三、结构化自然语言指令规范

### 3.1 指令 Schema

用户输入可以是自然语言，但 Skill 会将其解析为以下结构化字段。**缺失的必填字段会触发自动补问**。

```yaml
# 指令解析 Schema（v3.0）
instruction:
  # ── 必填字段 ──
  product: novita | ppio              # 产品线（从上下文推断或补问）
  action: generate | update | review | export | archive
  target: design | code | both        # 输出目标
  page_type: homepage | console | landing | legal | auth

  # ── 推荐字段（有默认值）──
  page_name: string                   # 页面名称，如 "benchmarks", "pricing"
  output_path: string                 # 输出路径，默认 ../novita-design/{page_name}-page.pen
  template: string                    # 基础模板，默认按 page_type 自动选择

  # ── 可选字段 ──
  modules:                            # 页面模块列表
    - type: hero | table | card-grid | form | list | chart | cta | faq
      data_source: string             # 数据来源文件路径
      interaction: sortable | filterable | expandable | static
      columns: number                 # 网格列数

  state:                              # 页面状态
    - loading | empty | error | success | default

  responsive:                         # 响应式需求
    breakpoints: [375, 768, 1024, 1440]  # 默认全部
    priority: mobile-first | desktop-first

  permissions:                        # 权限要求
    auth_required: boolean
    roles: [admin, member, viewer]

  reference:                          # 参考来源
    code_path: string                 # 源代码路径
    screenshot_url: string            # 参考截图
    existing_design: string           # 已有设计稿路径
```

### 3.2 指令解析流程

```
用户输入（自然语言）
    │
    ▼
┌─────────────────────────────────┐
│  Step 1: 意图识别               │
│  识别 action + target            │
│  "生成设计稿" → generate/design  │
│  "写代码" → generate/code        │
│  "检查一下" → review/both        │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Step 2: 上下文补全              │
│  从 cwd 推断 product            │
│  从页面名推断 page_type          │
│  从已有代码推断 modules          │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Step 3: 缺失字段补问            │
│  必填字段缺失 → 提出选项式问题   │
│  "你的目标产品是？               │
│   A) Novita AI  B) PPIO Cloud"  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Step 4: 规范预加载              │
│  --design-system 查询 Token     │
│  --domain sidebar 查询菜单结构   │
│  --domain header 查询头部规范    │
│  选择页面模板                    │
└──────────────┬──────────────────┘
               │
               ▼
         开始执行任务
```

### 3.3 指令示例

#### 示例 1: PM 用自然语言描述新页面

```
输入：
  "给 Novita 做一个模型对比页面，要有筛选侧边栏和模型卡片网格，每行3个"

解析结果：
  product: novita
  action: generate
  target: design
  page_type: homepage        # 自动推断（无 sidebar → homepage 类）
  page_name: model-comparison
  modules:
    - type: card-grid
      columns: 3
      interaction: filterable
    - type: hero              # 自动补充标准 hero 区域
  template: homepage-shell    # 自动选择 Homepage 外壳模板
```

#### 示例 2: 研发从代码生成设计稿

```
输入：
  "根据 src/app/pricing/page.tsx 生成对应设计稿"

解析结果：
  product: novita             # 从项目 CLAUDE.md 推断
  action: generate
  target: design
  page_type: homepage         # 从代码结构推断（有 Header + Footer）
  page_name: pricing
  reference:
    code_path: src/app/pricing/page.tsx
  modules:                    # 从代码 AST 推断
    - type: hero
    - type: table
      interaction: static
    - type: faq
      interaction: expandable
    - type: cta
```

#### 示例 3: 设计师审阅已有设计稿

```
输入：
  "检查 benchmarks-page.pen 是否符合设计规范"

解析结果：
  product: novita
  action: review
  target: design
  reference:
    existing_design: ../novita-design/benchmarks-page.pen
```

---

## 四、完整核心能力（表格）

### 4.1 当前已有能力（v2.0.1 + dual-design-system 分支）

| 场景 | 能力说明 | 输入 | 输出 | 规范绑定点 | 自动化校验 |
|------|---------|------|------|-----------|-----------|
| **代码 → 设计稿** | 读取 Next.js 页面代码，生成 Pencil .pen 文件 | page.tsx + 组件代码 | .pen 设计稿 | Token 颜色/字体 | 截图人工比对 |
| **自然语言 → 设计稿** | 用中/英文描述需求，AI 生成设计稿 | 自然语言描述 | .pen 设计稿 | Token 颜色/字体 | 截图人工比对 |
| **自然语言 → 代码** | 参考 Token 生成前端代码 | 自然语言描述 | React 组件代码 | SCSS Mixin + Tailwind | 无 |
| **设计稿截图审阅** | 生成截图供人工检查 | .pen 文件 | PNG 截图 | 无 | 无 |
| **设计系统查询** | BM25 搜索 14 个域，返回规范数据 | `--domain X --product Y` | CSV 搜索结果 | 完全绑定 | 无 |
| **Token 验证** | 4 层验证 Token 完整性 | `validate_tokens.py` | 验证报告 | Token schema | 自动化 |
| **Token 同步** | JSON → CSV → SKILL.md | `sync_tokens.py` | 更新后的 CSV + SKILL.md | Token 源 | 自动化 |
| **设计系统生成** | 5 域并行查询 + 推理规则 | `--design-system` | 完整设计系统推荐 | 推理规则 | 无 |

### 4.2 v3.0 新增能力

| 场景 | 能力说明 | 输入 | 输出 | 规范绑定点 | 自动化校验 |
|------|---------|------|------|-----------|-----------|
| **结构化指令解析** | 将自然语言解析为结构化 schema，缺字段自动补问 | 自然语言 | 结构化指令 YAML | 指令 schema | 字段完整性检查 |
| **模板优先生成** | 基于页面模板库生成，不从零构建 | 指令 + 模板 | .pen 设计稿 | 模板规范 | 模板一致性检查 |
| **设计稿 → 代码** | 从 .pen 设计稿反向生成 React 代码 | .pen 文件 | React + SCSS 代码 | Token ↔ SCSS Mixin 映射 | 代码 lint + Token 合规 |
| **代码 ↔ 设计稿双向同步** | 检测代码变更，更新设计稿（或反向） | 代码 diff / 设计 diff | 同步后的设计稿/代码 | Token 映射表 | 差异报告 |
| **自动化合规校验** | 校验设计稿/代码是否符合 Token 规范 | .pen 文件 / 代码文件 | 合规报告 JSON | 全部 Token + 布局规范 | 完全自动化 |
| **还原度评分** | 对比线上页面与设计稿，输出分数 | URL + .pen 文件 | 还原度评分报告 | 评分维度标准 | Playwright + SSIM |
| **语义化版本存档** | 每次变更生成语义 diff + 变更日志 | 变更操作 | changelog.md + 语义 diff | 版本规范 | 自动生成 |
| **规范溯源清单** | 每个输出节点标注引用了哪个 Token | 生成结果 | 溯源清单 JSON | Token 引用链 | 自动生成 |
| **页面结构查询** | `--domain layout/footer/template` 新搜索域 | CLI 查询 | CSV 搜索结果 | 完全绑定 | 无 |
| **组件映射查询** | Token ↔ Pencil 组件 ID ↔ shadcn 组件映射 | `--component-map` | 映射表 | 三方对齐 | 一致性检查 |

### 4.3 能力成熟度路线图

```
                 v2.0.1 (当前)          v2.5 (分支合入)        v3.0 (全链路)
                 ───────────           ──────────────        ──────────────
代码→设计稿       ██████░░░░ 70%       █████████░░ 85%       ██████████ 95%
自然语言→设计稿   ██████░░░░ 65%       ████████░░░ 80%       █████████░ 93%
自然语言→代码     ███████░░░ 75%       ████████░░░ 80%       █████████░ 90%
设计稿→代码       ░░░░░░░░░░ 0%        ███░░░░░░░░ 30%       ████████░░ 80%
双向同步          ░░░░░░░░░░ 0%        ░░░░░░░░░░░ 0%        ███████░░░ 70%
自动化校验        ██░░░░░░░░ 20%       ████░░░░░░░ 40%       █████████░ 90%
版本存档          █░░░░░░░░░ 10%       ███░░░░░░░░ 30%       ████████░░ 80%
规范绑定强度      ████░░░░░░ 40%       ███████░░░░ 70%       █████████░ 95%
```

---

## 五、与项目 CLI 架构对齐规则

### 5.1 目录结构对齐

```
novita-home/src/app/                    novita-design/
├── benchmarks/                         ├── benchmarks-page.pen
│   ├── page.tsx                        │   ├── [设计系统组件]
│   ├── components/                     │   └── [页面设计]
│   │   ├── BenchmarkTable.tsx          │
│   │   └── benchmarkData.ts            │
│   └── page.module.scss                │
├── pricing/                            ├── pricing-page.pen
│   ├── page.tsx                        │
│   └── components/                     │
├── models/                             ├── model-library-page.pen
└── gpus-console/                       ├── console-gpus-page.pen
                                        ├── templates/           ← 模板库
                                        │   ├── homepage-shell.pen
                                        │   └── console-shell.pen
                                        └── docs/                ← 文档
```

**命名规则**：
| 代码路径 | 设计稿文件名 | 规则 |
|---------|------------|------|
| `src/app/benchmarks/page.tsx` | `benchmarks-page.pen` | `{路由名}-page.pen` |
| `src/app/pricing/page.tsx` | `pricing-page.pen` | `{路由名}-page.pen` |
| `src/app/gpus-console/page.tsx` | `console-gpus-page.pen` | `console-{功能名}-page.pen` |
| `src/app/models/page.tsx` | `model-library-page.pen` | 语义化命名 |
| `src/app/mainpage/page.tsx` | `homepage.pen` | 特殊：首页 |

### 5.2 组件引用对齐

```
Pencil 设计系统组件                     代码组件来源
─────────────────                     ──────────────
Button/Default                  ←→    @/components/ui/button.tsx (variant="default")
Button/Outline                  ←→    @/components/ui/button.tsx (variant="outline")
Input/Default                   ←→    @/components/ui/input.tsx
Select/Default                  ←→    @/components/ui/select.tsx
Table/Container + Table Row     ←→    @/components/ui/table.tsx
Badge/Default                   ←→    @/components/ui/badge.tsx
Dialog (待建)                   ←→    @/components/ui/dialog.tsx
Tabs (待建)                     ←→    @/components/ui/tabs.tsx
```

### 5.3 样式变量对齐

| 代码中的样式 | 设计 Token | Pencil 设计值 | 对齐状态 |
|------------|-----------|-------------|---------|
| `var(--brand-0)` / `#23d57c` | `colors.brand.0` | `fill: "#23D57C"` | ✅ 已对齐 |
| `var(--dark-1)` / `#292827` | `colors.dark.1` | `fill: "#292827"` | ✅ 已对齐 |
| `var(--gray-2)` / `#e7e6e2` | `colors.gray.2` | `stroke: "#E7E6E2"` | ✅ 已对齐 |
| `@include font-h1` | `font-h1 (80px/74px/600/-1.6)` | `fontSize:80, fontWeight:600` | ⚠️ 未录入 CSV |
| `@include font-h4` | `font-h4 (24px/38px/600/0)` | `fontSize:24, fontWeight:600` | ⚠️ 未录入 CSV |
| `@include font-body` | `font-body (16px/24px/400/0)` | `fontSize:16, fontWeight:400` | ⚠️ 未录入 CSV |
| `@include font-menu` | `font-menu (14px/18px/400/0)` | `fontSize:14, fontWeight:400` | ✅ 已对齐 |
| `@include font-subtle` | `font-subtle (14px/20px/400/0)` | `fontSize:14, lineHeight:"20px"` | ⚠️ 未录入 CSV |
| `@include font-small` | `font-small (12px/20px/400/0)` | `fontSize:12` | ⚠️ 未录入 CSV |
| `padding: var(--spacing-console-16)` | `spacing.16: 16px` | `padding: 16` | ✅ 已对齐 |
| `.max_width_container` → 1280px | 无 | `width: 1280` | ❌ 缺失 |
| Header 高度 80px (Homepage) | 无 (Console 54px 有) | `height: 80` | ❌ 缺失 |
| `@include screen-md` → 768px | 无 | — | ❌ 缺失 |

### 5.4 构建适配规则

生成的代码必须满足：
1. **框架**: Next.js App Router（使用 `"use client"` 标注客户端组件）
2. **样式**: SCSS Modules（`.module.scss`）+ Tailwind 工具类
3. **组件**: 优先使用 `@/components/ui/` 中的 shadcn/ui 组件
4. **图标**: 优先 Lucide React → 备选 iconfont → 备选 `@/lib/icons/`
5. **路径**: `@/*` 别名指向 `./src/*`
6. **状态**: Redux Toolkit（`@/store/slice/`）
7. **请求**: Axios（`@/api/`）
8. **表单**: react-hook-form + zod

---

## 六、设计稿 & 代码存档 & 版本管理

### 6.1 存储结构

```
novita-design/                          ← 设计稿仓库（独立 Git 仓库）
├── README.md
├── .design-meta/                       ← [v3.0 新增] 元数据目录
│   ├── registry.json                   ← 设计稿注册表（文件 → 版本 → 关联代码）
│   ├── changelog/                      ← 变更日志
│   │   ├── 2026-03-03-benchmarks.md
│   │   └── 2026-03-04-pricing.md
│   └── reports/                        ← 校验报告
│       ├── 2026-03-03-benchmarks-compliance.json
│       └── 2026-03-03-benchmarks-fidelity.json
│
├── templates/                          ← [v3.0 新增] 模板库
│   ├── homepage-shell.pen              ← Homepage 通用外壳
│   ├── console-shell.pen              ← Console 通用外壳
│   └── landing-page.pen              ← 营销着陆页模板
│
├── benchmarks-page.pen                 ← 页面设计稿
├── pricing-page.pen
├── homepage.pen
│
└── docs/                               ← 文档
    ├── tutorial-ai-design-workflow.md
    ├── gap-analysis-95-percent-fidelity.md
    └── skill-capability-specification-v2.md  ← 本文档
```

### 6.2 设计稿注册表 (registry.json)

```json
{
  "version": "1.0",
  "designs": {
    "benchmarks-page.pen": {
      "created": "2026-03-02T10:30:00Z",
      "updated": "2026-03-02T15:45:00Z",
      "product": "novita",
      "page_type": "homepage",
      "source_code": "novita-home/src/app/benchmarks/page.tsx",
      "design_system_version": "Novita Design System-2",
      "component_count": 24,
      "tokens_referenced": ["colors.brand.0", "colors.dark.1", "spacing.16", "..."],
      "fidelity_score": 75,
      "last_review": "2026-03-02T15:45:00Z",
      "commits": [
        {
          "sha": "a057301",
          "date": "2026-03-02",
          "message": "feat: fix header height and rebuild footer",
          "changes": ["header: 64→80px height", "footer: rebuilt 4-column layout"]
        }
      ]
    }
  }
}
```

### 6.3 变更日志格式 (changelog)

每次设计稿变更自动生成：

```markdown
# benchmarks-page.pen — 变更日志 2026-03-02

## 变更概要
- **操作**: update
- **触发**: 用户指令 "头部字体样式不同，页面底部不同"
- **影响范围**: Header + Footer

## 语义 Diff
| 区域 | 属性 | 变更前 | 变更后 | 引用 Token |
|------|------|-------|-------|-----------|
| Header | height | 64px | 80px | layout.header-height-homepage |
| Header/Logo | fontWeight | 700 | 600 | font-h5 (weight) |
| Header/Nav | textColor | #292827 | #4f4e4a | colors.dark.2 |
| Footer | 结构 | 单行版权文字 | 4 列完整 Footer | footer-template |

## 规范溯源
- colors.dark.2 (#4f4e4a) ← novita.tokens.json → primitives.colors.dark.2
- font-h5 (20px/24px/600) ← theme.scss → @include font-h5
```

### 6.4 版本关联关系

```
                    设计稿                        代码
                    ─────                         ────
benchmarks-page.pen ◄──── 生成自 ────► src/app/benchmarks/page.tsx
       │                                       │
       │  commit: a057301                      │  commit: 3b1630f
       │  message: "fix header/footer"         │  message: "add benchmarks page"
       │                                       │
       └──── registry.json 记录关联 ────────────┘
```

### 6.5 回溯方式

| 需求 | 命令/操作 |
|------|---------|
| 查看设计稿历史版本 | `git log --oneline benchmarks-page.pen` |
| 回退到某次设计 | `git checkout <sha> -- benchmarks-page.pen` |
| 查看某次变更详情 | `cat .design-meta/changelog/2026-03-02-benchmarks.md` |
| 查看设计稿关联的代码 | `cat .design-meta/registry.json \| jq '.designs["benchmarks-page.pen"].source_code'` |
| 查看最近的合规报告 | `cat .design-meta/reports/2026-03-02-benchmarks-compliance.json` |
| 对比两次设计的差异 | Pencil MCP `batch_get` 读取 + 节点树 diff |

---

## 七、Agent 自动化验证体系

### 7.1 验证层次

```
┌─────────────────────────────────────────────────────────────────┐
│                    验证体系（4 层 + 2 层新增）                      │
├─────────────────────────────────────────────────────────────────┤
│  Layer 1: Token 结构验证 (已有)                                   │
│  ├── novita.tokens.json 结构完整性                                │
│  ├── 必填字段：$meta, primitives, composites, interactions       │
│  └── semver 版本检查                                              │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2: Token 引用完整性 (已有)                                  │
│  ├── composites 中的 {ref} 是否解析到 primitives                  │
│  └── 无悬空引用                                                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3: 交互态完备性 (已有)                                      │
│  ├── Button: default, hover, disabled                            │
│  ├── Input: default, hover, focus, disabled                      │
│  └── Select: default, hover, focus, disabled                     │
├─────────────────────────────────────────────────────────────────┤
│  Layer 4: CSS 变量一致性 (已有)                                    │
│  ├── 无重复 cssVar                                                │
│  └── Hex 色值格式校验                                             │
├─────────────────────────────────────────────────────────────────┤
│  Layer 5: 设计稿合规校验 (v3.0 新增)                               │
│  ├── 颜色合规：所有 fill/stroke 值必须存在于 Token 体系            │
│  ├── 字体合规：fontSize/fontWeight/lineHeight 匹配 Token 定义      │
│  ├── 间距合规：padding/gap/margin 使用标准间距档位                  │
│  ├── 圆角合规：cornerRadius 使用标准圆角档位                       │
│  ├── 组件合规：使用设计系统中的可复用组件，不自建同功能节点          │
│  └── 布局合规：max-width、header 高度、z-index 符合规范            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 6: 代码合规校验 (v3.0 新增)                                 │
│  ├── 样式变量：使用 var(--dark-1) 而非硬编码 #292827              │
│  ├── Mixin 使用：使用 @include font-h4 而非手写 font-size: 24px   │
│  ├── 组件引用：使用 @/components/ui/ 而非自建                     │
│  ├── Tailwind 约束：使用精确值 text-[14px] 而非 text-sm           │
│  └── 文件命名：符合项目约定（PascalCase 组件、.module.scss 样式）  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 校验规则详细表

#### Layer 5: 设计稿合规校验

| 校验项 | 校验规则 | 严重度 | 自动修复 |
|--------|---------|--------|---------|
| **颜色 — 品牌色** | fill/stroke 值必须在 `novita.tokens.json` 的 primitives.colors 中 | ERROR | 是：匹配最近色值 |
| **颜色 — 文本色** | textColor 必须为 `#292827`/`#4F4E4A`/`#9E9C98`/`#BBB9B6`/`#FFFFFF` 之一 | ERROR | 是：匹配最近色值 |
| **颜色 — 背景色** | 背景 fill 必须为标准灰阶（`#FFFFFF`/`#FAFAFA`/`#F5F5F5`/`#E7E6E2`）或品牌色 | WARN | 是 |
| **字体 — 字号** | fontSize 必须属于 Token 定义的字号集（12/13/14/16/18/20/24/36/48/56/80） | ERROR | 是：匹配最近档位 |
| **字体 — 行高** | lineHeight 必须与对应 fontSize Token 的定义匹配 | WARN | 是 |
| **字体 — 字重** | fontWeight 必须为 400/500/600 之一 | ERROR | 是：匹配最近档位 |
| **间距 — padding** | padding 值必须属于 spacing Token 集（0/4/8/12/16/20/24/32/40/48/64） | WARN | 是 |
| **间距 — gap** | gap 值必须属于 spacing Token 集 | WARN | 是 |
| **圆角** | cornerRadius 必须属于 radius Token 集（0/2/4/6/8/12/9999） | WARN | 是 |
| **组件引用** | 设计稿中的 Button/Input 等组件必须引用设计系统 ref，不可自建 | ERROR | 否：需人工确认 |
| **布局 — 最大宽度** | 内容区 width 不超过 1280px（homepage）或计算值（console） | WARN | 是 |
| **布局 — Header 高度** | Homepage Header = 80px，Console Header = 54px | ERROR | 是 |

#### Layer 6: 代码合规校验

| 校验项 | 校验规则 | 严重度 | 自动修复 |
|--------|---------|--------|---------|
| **硬编码颜色** | 不得出现裸 hex 值（如 `color: #292827`），应使用 `var(--dark-1)` | ERROR | 是：替换为变量 |
| **Tailwind 近似** | 不得使用 `text-sm`（14px/20px），应使用 `text-[14px] leading-[18px]` | ERROR | 是：替换为精确值 |
| **SCSS Mixin** | 如果 mixins.scss 中存在对应 mixin，必须使用而非手写属性 | WARN | 否：提示建议 |
| **组件来源** | 如果 `@/components/ui/` 中存在同功能组件，必须引用而非自建 | ERROR | 否：需人工确认 |
| **文件命名** | 组件 PascalCase，样式 `.module.scss`，页面 `page.tsx` | WARN | 是 |
| **import 路径** | 使用 `@/` 别名，不使用相对路径跨模块引用 | WARN | 是 |

### 7.3 校验报告格式

```json
{
  "file": "benchmarks-page.pen",
  "timestamp": "2026-03-02T15:45:00Z",
  "product": "novita",
  "summary": {
    "total_checks": 142,
    "passed": 128,
    "warnings": 10,
    "errors": 4,
    "compliance_score": 90.1
  },
  "issues": [
    {
      "layer": 5,
      "rule": "color-text",
      "severity": "ERROR",
      "node_id": "abc123",
      "node_path": "Header/NavItem[3]",
      "current_value": "#333333",
      "expected_values": ["#292827", "#4F4E4A", "#9E9C98"],
      "suggested_fix": "#292827",
      "auto_fixable": true,
      "token_reference": "primitives.colors.dark.1"
    },
    {
      "layer": 5,
      "rule": "spacing-padding",
      "severity": "WARN",
      "node_id": "def456",
      "node_path": "Main/Hero",
      "current_value": 30,
      "expected_values": [24, 32],
      "suggested_fix": 32,
      "auto_fixable": true,
      "token_reference": "primitives.spacing.32"
    }
  ],
  "auto_fix_available": 12,
  "manual_review_required": 2
}
```

### 7.4 自动修复能力

| 修复类型 | 实现方式 | 触发条件 |
|---------|---------|---------|
| **颜色修正** | Pencil `batch_design` → `U(nodeId, {fill: correctColor})` | 色值不在 Token 集中 |
| **间距修正** | Pencil `batch_design` → `U(nodeId, {padding: nearestToken})` | 间距值不在标准档位 |
| **圆角修正** | Pencil `batch_design` → `U(nodeId, {cornerRadius: nearestToken})` | 圆角不在标准档位 |
| **代码变量替换** | `Edit` 工具替换硬编码值为 CSS 变量 | 硬编码色值/字号 |
| **Tailwind 修正** | `Edit` 工具替换近似类为精确值 | 使用了 `text-sm` 等近似类 |

---

## 八、可扩展点设计

### 8.1 新产品扩展

```
扩展步骤：
1. 复制 data/novita/ → data/{new-product}/
2. 编辑所有 CSV 文件填入产品数据
3. 创建 tokens/{new-product}.tokens.json
4. 在 core.py VALID_PRODUCTS 中注册
5. 运行 sync_tokens.py --product {new-product} --all
6. SKILL.md 自动注入新产品约束

预计工时: 8-16h（视产品复杂度）
```

### 8.2 新组件扩展

```
扩展步骤：
1. 在 novita.tokens.json 中定义组件 Token（状态 × 属性）
2. 运行 validate_tokens.py 确保结构合规
3. 运行 sync_tokens.py 生成 CSV + 更新 SKILL.md
4. 在 Pencil 设计系统中创建对应组件（reusable: true）
5. 在 shadcn/ui 中创建对应 React 组件（如果不存在）
6. 更新组件映射表

预计工时: 1-2h / 组件
```

### 8.3 新校验规则扩展

```python
# validate_tokens.py 扩展点

class TokenValidator:
    def __init__(self):
        self.layers = [
            StructureLayer(),      # Layer 1: 结构
            ReferenceLayer(),      # Layer 2: 引用
            StateLayer(),          # Layer 3: 状态
            CSSVarLayer(),         # Layer 4: CSS 变量
            # ── v3.0 新增 ──
            DesignComplianceLayer(),  # Layer 5: 设计稿合规
            CodeComplianceLayer(),    # Layer 6: 代码合规
        ]

    def add_layer(self, layer: ValidationLayer):
        """扩展点：插入新的验证层"""
        self.layers.append(layer)
```

### 8.4 新搜索域扩展

```python
# core.py 扩展点

CSV_CONFIG = {
    # 已有域...
    "sidebar": {"file": "sidebar.csv", "search_cols": [...], "output_cols": [...]},

    # ── v3.0 新增域 ──
    "layout": {"file": "layout.csv", "search_cols": ["Property", "Context", "Description"], ...},
    "footer": {"file": "footer.csv", "search_cols": ["Section", "Items", "Description"], ...},
    "template": {"file": "page-templates.csv", "search_cols": ["PageType", "Modules", "Description"], ...},
    "scss-mapping": {"file": "scss-mapping.csv", "search_cols": ["Mixin", "Token", "Values"], ...},
    "icon-mapping": {"file": "icons-mapping.csv", "search_cols": ["Icon", "Usage", "Pencil"], ...},
}

# 域分类更新
PRODUCT_DOMAINS.update({"layout", "footer", "template", "scss-mapping", "icon-mapping"})
```

### 8.5 新模板扩展

```
模板注册：
1. 在 novita-design/templates/ 中创建 .pen 文件
2. 在 data/novita/page-templates.csv 中注册模板元数据
3. 模板包含：
   - 外壳结构（Header + Footer / Sidebar + Content）
   - 占位内容区（placeholder: true）
   - 设计系统变量引用
4. Skill 根据 page_type 自动选择模板

模板 CSV 格式：
PageType,TemplateName,PenFile,Includes,ContentArea,Notes
homepage,Homepage Shell,templates/homepage-shell.pen,"Header(80px)+Footer(4col)",Main,标准 Homepage 布局
console,Console Shell,templates/console-shell.pen,"Header(54px)+Sidebar(250px)",Content,Console 三栏布局
landing,Landing Page,templates/landing-page.pen,"Header(80px)+Footer(4col)",Main,营销着陆页
```

### 8.6 扩展点总结

| 扩展维度 | 触碰文件 | 工序 | 自动化程度 |
|---------|---------|------|-----------|
| 新产品 | `data/{product}/` + `core.py` + `sync_tokens.py` | 5 步 | 半自动（sync 自动，数据手动） |
| 新组件 | `tokens.json` + Pencil .pen + shadcn/ui | 6 步 | 半自动（sync 自动，组件手动） |
| 新校验规则 | `validate_tokens.py` | 1 步 | 手动（写 Python 类） |
| 新搜索域 | `core.py` + 新 CSV 文件 | 2 步 | 手动 |
| 新页面模板 | `templates/` + `page-templates.csv` | 2 步 | 半自动 |
| 新 SCSS Mixin | `scss-mapping.csv` + `typography.csv` | 2 步 | 手动 |
| 新图标映射 | `icons-mapping.csv` | 1 步 | 手动 |

---

## 九、与原有能力的差异总结

### 9.1 逐项对比

| 维度 | v2.0.1 现有能力 | v3.0 补齐内容 | 价值提升 |
|------|---------------|-------------|---------|
| **指令体系** | 完全自由文本输入，AI 自由解读 | 结构化 schema + 自动补问 + 模板选择 | PM 输入更精准，减少 50%+ 返工 |
| **规范绑定** | Token 存在但 AI 不一定遵循；SKILL.md 有约束但无运行时强制 | 生成时强制查 Token → 输出后自动校验 → 不合规自动修正 | 规范遵循率从 ~70% 提升到 ~95% |
| **页面模板** | 无模板，每次从零构建 Header/Footer/Sidebar | 模板库（homepage-shell / console-shell / landing-page） | 生成速度提升 3-5x，一致性保障 |
| **组件覆盖** | Pencil 24 个组件，Token 有但 Pencil 缺 | Pencil 扩展到 ~40 个，与 Token 1:1 对应 | 组件直接引用，不再自建 |
| **布局规范** | 缺失 max-width、断点、header 高度等 | 新增 layout.csv / footer.csv / scss-mapping.csv | 布局参数不再靠猜 |
| **设计稿→代码** | 不支持 | .pen → React + SCSS，遵循项目约定 | 打通反向链路 |
| **双向同步** | 不支持 | 代码变更 → 设计更新 / 设计变更 → 代码更新 | 设计-代码永不漂移 |
| **自动化校验** | 仅 Token 结构验证（4 层） | 新增设计稿合规（Layer 5）+ 代码合规（Layer 6） | 交付前自动 QA |
| **还原度度量** | 人工目测 | Playwright 截图 + SSIM 对比 + 分维度评分 | 客观可量化 |
| **版本管理** | Git commit 仅记录二进制 diff | 语义 changelog + 注册表 + 关联代码 + 合规报告 | 可回溯设计决策 |
| **三路同步** | 手动复制 data → src → cli | 待自动化（脚本 or CI） | 消除不一致风险 |
| **PPIO 产品** | 骨架（仅品牌色和产品定义） | 待完善（不在本阶段范围） | 未来双产品对齐 |

### 9.2 核心差距可视化

```
                      v2.0.1 (当前)                v3.0 (目标)
                      ─────────────               ──────────────
结构化指令体系         ░░░░░░░░░░  0%              ██████████  100%
规范强制绑定           ████░░░░░░  40%             █████████░  95%
页面模板库             ░░░░░░░░░░  0%              ████████░░  80%
组件覆盖完整度         ██████░░░░  60%             █████████░  90%
布局规范数据           ██░░░░░░░░  20%             █████████░  90%
设计稿→代码           ░░░░░░░░░░  0%              ████████░░  80%
双向同步              ░░░░░░░░░░  0%              ███████░░░  70%
自动化合规校验         ██░░░░░░░░  20%             █████████░  90%
还原度度量            ░░░░░░░░░░  0%              ████████░░  80%
版本化存档            █░░░░░░░░░  10%             ████████░░  80%
三路同步自动化         █░░░░░░░░░  10%             ████████░░  80%
```

### 9.3 实施优先级建议

| 阶段 | 内容 | 工时 | 效果 |
|------|------|------|------|
| **Phase 0** (立即) | 合入 dual-design-system 分支并发布 | 2h | 其他成员可使用 |
| **Phase 1** (Week 1-2) | 新增 layout/footer/scss-mapping CSV + 页面模板库 | 15h | 还原度 75% → 85% |
| **Phase 2** (Week 3-4) | 补全 Pencil 组件 + Token 对齐 + 组件映射表 | 12h | 还原度 85% → 90% |
| **Phase 3** (Week 5-6) | 结构化指令解析 + Layer 5/6 校验 + 语义 changelog | 20h | 全链路闭环 |
| **Phase 4** (Week 7-8) | 还原度评分系统 + 双向同步 + 三路同步自动化 | 16h | 还原度 90% → 95% |

**总计约 65h 工作量，4-8 周可完成全链路建设。**

---

## 附录 A: 需要新增/更新的数据文件清单

### ppio-ui-skill 数据文件

| 文件 | 状态 | 预计行数 | 内容 |
|------|------|---------|------|
| `data/novita/layout.csv` | 新建 | ~30 | max-width, header-height, 断点, z-index, spacing-layout |
| `data/novita/footer.csv` | 新建 | ~40 | Homepage Footer 完整结构（4 列菜单项 + 社交 + 版权） |
| `data/novita/scss-mapping.csv` | 新建 | ~25 | SCSS Mixin ↔ Token ↔ CSS 值映射（font-h1 ~ font-small） |
| `data/novita/page-templates.csv` | 新建 | ~20 | 页面模板注册表（page_type → template → pen file） |
| `data/novita/icons-mapping.csv` | 新建 | ~40 | Lucide Icon → 用途 → Pencil icon_font 映射 |
| `data/novita/component-map.csv` | 新建 | ~50 | Token ID ↔ Pencil 组件 ID ↔ shadcn 组件路径 |
| `data/novita/typography.csv` | 更新 | +10 | 补全 font-h1 ~ font-small 完整映射 |
| `data/novita/header.csv` | 更新 | +20 | 补充 Homepage Header 规范 |
| `data/novita/design-system.csv` | 更新 | +15 | 补充 CSS 变量色值映射 |

**总计新增 ~250 行 CSV 数据**

### novita-design 模板文件

| 文件 | 内容 |
|------|------|
| `templates/homepage-shell.pen` | Header(80px) + 内容占位 + Footer(4 列完整) |
| `templates/console-shell.pen` | Console Header(54px) + Sidebar(250px) + 内容占位 |
| `templates/landing-page.pen` | Header + Hero + Features + CTA + Footer |

### 脚本增强

| 文件 | 增强内容 |
|------|---------|
| `scripts/core.py` | 新增 layout/footer/template/scss-mapping/icon-mapping/component-map 搜索域 |
| `scripts/validate_tokens.py` | 新增 Layer 5 (设计稿合规) + Layer 6 (代码合规) |
| `scripts/search.py` | 新增 `--component-map` 参数 |
| `scripts/changelog.py` (新建) | 语义 diff 生成 + changelog 写入 |
| `scripts/fidelity.py` (新建) | 还原度评分（Playwright 截图 + SSIM 对比） |

---

## 附录 B: 术语表

| 术语 | 定义 |
|------|------|
| **Token** | 设计系统中的最小原子值（颜色、字号、间距等），存储在 `novita.tokens.json` |
| **Pencil / .pen** | 加密 JSON 格式的设计文件，只能通过 Pencil MCP 工具读写 |
| **BM25** | 信息检索算法，用于在 CSV 数据中进行全文搜索排序 |
| **哨兵标记** | `<!-- BEGIN/END:AUTO_GENERATED_HARD_CONSTRAINTS -->` 标记，用于 SKILL.md 中自动注入约束 |
| **三路同步** | `data/` → `src/ppio-ui-skill/` → `cli/assets/` 的数据镜像机制 |
| **设计系统生成器** | `design_system.py` 中的 5 域并行搜索 + 推理规则引擎 |
| **语义 diff** | 不仅记录节点变化（宽度从 64 到 80），还记录设计意图（Header 高度对齐规范） |
| **还原度** | 设计稿与线上页面的视觉一致程度，用 SSIM 或分维度评分量化 |
| **规范溯源** | 每个输出值（颜色、字号等）可追溯到具体 Token 定义 |
| **SSIM** | Structural Similarity Index，结构相似性指数，用于图像对比 |
