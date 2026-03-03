# AI 设计稿生成工作流 —— 使用教程

> 本文档为 PPT 演示文稿的内容大纲，可直接导入 PPT 工具生成幻灯片。
> 每个 `## Slide` 标题对应一页幻灯片。

---

## Slide 1: 封面

**AI 设计稿生成工作流**
*用自然语言驱动设计，从代码到设计稿的自动化桥梁*

- 适用团队：产品经理 / 前端研发 / UI 设计师
- 工具链：Claude Code + Pencil MCP + ppio-ui-skill
- 版本：v1.0 | 2026-03

---

## Slide 2: 这套工作流能做什么？

### 核心能力
| 场景 | 说明 |
|------|------|
| **代码 → 设计稿** | 从已有的 Next.js 页面代码自动生成 Pencil (.pen) 设计稿 |
| **自然语言 → 设计稿** | 用中文/英文描述需求，AI 自动生成符合品牌规范的设计稿 |
| **自然语言 → 代码** | 用自然语言描述需求，AI 参考设计系统 Token 生成前端代码 |
| **设计稿审阅** | 生成截图预览，支持逐节点检查和迭代修改 |

### 适用产品
- **Novita AI** — 完整设计系统（24 组件 + 67 变量 + 100 Token）
- **PPIO Cloud** — 基础框架（待补充）

---

## Slide 3: 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                    用户（自然语言指令）                    │
│  "为 benchmarks 页面生成设计稿，保存到 novita-design"     │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Claude Code (CLI)                      │
│  ┌──────────────────┐  ┌──────────┐  ┌───────────────┐ │
│  │  ppio-ui-skill   │  │Pencil MCP│  │Playwright MCP │ │
│  │ 双产品设计系统    │  │设计稿读写 │  │页面截图参考   │ │
│  │ BM25搜索引擎     │  │          │  │              │ │
│  │ Token同步/验证   │  │          │  │              │ │
│  └──────┬───────────┘  └────┬─────┘  └──────┬───────┘ │
│         │                   │                │         │
│  ┌──────▼───────────────────▼────────────────▼───────┐ │
│  │        AI 智能体（分析、设计、验证循环）            │ │
│  └───────────────────────────────────────────────────┘ │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              输出文件 (.pen 设计稿)                       │
│  保存位置: /Users/ivy/work/novita-design/               │
│  提交方式: git commit → git push origin main            │
└─────────────────────────────────────────────────────────┘
```

---

## Slide 3.5: ppio-ui-skill —— 双产品设计系统架构

### 核心改造（refactor/dual-design-system 分支）
**13 个 commit | +11,470 行 | 126 个文件**

### 数据架构
```
ppio-ui-skill/data/
├── shared/              ← 产品无关（charts, landing, ux, icons, react, web）
│   ├── charts.csv, landing.csv, ux-guidelines.csv ...
│   ├── ui-reasoning.csv ← 40 个 UI 品类推理规则
│   └── stacks/          ← React / shadcn 栈指南
│
├── novita/              ← Novita AI 专属（绿 #23D57C）
│   ├── sidebar.csv      ← 111 行，Home/Model APIs/GPUs/Sandbox 4 种菜单
│   ├── header.csv       ← 36 行，Console 顶部导航
│   ├── design-system.csv ← 225 行（自动生成，勿手动编辑）
│   └── tokens/novita.tokens.json ← 源头：100+ 原语 + 组合 + 交互态
│
└── ppio/                ← PPIO Cloud 专属（蓝 #1161fe，骨架待补充）
```

### 新增工具链
| 工具 | 命令 | 用途 |
|------|------|------|
| **搜索引擎** | `python3 search.py "query" --product novita --domain sidebar` | BM25 全文搜索 14 个域 |
| **设计系统生成** | `python3 search.py "query" --product novita --design-system` | 5 域并行 + 推理规则 |
| **Token 验证** | `python3 validate_tokens.py --product novita` | 4 层验证（结构/引用/状态/CSS） |
| **Token 同步** | `python3 sync_tokens.py --product novita --all` | JSON → CSV → SKILL.md |

### 关键约束
- **design-system.csv 是自动生成的**，修改源头是 `novita.tokens.json`
- **sidebar.csv 包含 Figma 节点 ID**，用于截图交叉验证
- **三路同步**：`data/` → `src/ppio-ui-skill/data/` → `cli/assets/data/`

---

## Slide 4: 前置条件 —— 需要安装什么？

### 1. Claude Code CLI
```bash
# macOS 安装
brew install claude-code
# 或通过官方 npm 安装
npm install -g @anthropic-ai/claude-code
```

### 2. Pencil 桌面应用
- 下载安装 Pencil.app（macOS）
- 安装后 MCP Server 自动内置于应用中

### 3. Pencil MCP Server（全局配置）
已配置在 `~/.claude.json` 中：
```json
{
  "mcpServers": {
    "pencil": {
      "command": "/Applications/Pencil.app/Contents/Resources/app.asar.unpacked/out/mcp-server-darwin-arm64",
      "args": ["--app", "desktop"],
      "type": "stdio"
    }
  }
}
```

### 4. ppio-ui-skill 插件
已通过 Claude Code 插件系统安装，无需额外配置。

### 5. 设计稿仓库
```bash
git clone git@github.com:ivyrao-novita/novita-design.git
cd novita-design
```

---

## Slide 5: 环境检查清单

在开始之前，请确认以下几项：

| 检查项 | 如何验证 | 预期结果 |
|--------|---------|---------|
| Claude Code 已安装 | 终端输入 `claude --version` | 显示版本号 ≥ 2.1 |
| Pencil 应用已打开 | 检查 Dock 栏 | Pencil 图标显示运行中 |
| MCP 连接正常 | Claude Code 启动时观察日志 | 显示 `pencil` MCP Server 已连接 |
| 设计仓库已克隆 | `ls ~/work/novita-design/` | 目录存在且包含 .pen 文件 |
| Git 权限正常 | `cd novita-design && git push --dry-run` | 无权限错误 |

> **重要**：启动 Claude Code 前必须先打开 Pencil 桌面应用，否则 MCP 连接会失败。

---

## Slide 6: 快速上手 —— 3 步完成设计稿生成

### Step 1: 启动 Claude Code
```bash
cd ~/work/novita-home    # 进入代码项目目录
claude                    # 启动 Claude Code
```

### Step 2: 用自然语言下达指令
```
将 benchmarks 页面生成一个 pencil 格式的设计稿，保存到 ../novita-design 目录
```

或者更具体的指令：
```
根据 src/app/pricing/page.tsx 的代码，生成对应的 Pencil 设计稿，
使用 Novita 品牌色(#23D57C)，保存为 ../novita-design/pricing-page.pen
```

### Step 3: 等待生成 + 审阅
- AI 会自动读取代码、分析结构、调用 Pencil MCP 创建设计
- 生成过程中会产生截图供你预览
- 如有不满意的地方，直接用自然语言描述修改需求

---

## Slide 7: 工作流程详细步骤

```
1. 打开 Pencil 桌面应用
       │
       ▼
2. 启动 Claude Code (cd 到代码项目 → 输入 claude)
       │
       ▼
3. 输入自然语言指令描述需求
       │
       ▼
4. AI 自动执行：
   ├── 读取源代码（page.tsx, 组件, 样式文件）
   ├── 查询 ppio-ui-skill 获取设计 Token
   ├── 调用 Pencil MCP 创建/打开 .pen 文件
   ├── 构建设计系统组件（按钮、输入框、表格等）
   ├── 逐层构建页面：Header → 主体内容 → Footer
   └── 生成截图验证
       │
       ▼
5. 审阅设计截图
   ├── 满意 → 进入 Step 6
   └── 不满意 → 用自然语言描述修改（如"头部高度改为 80px"）
       │
       ▼
6. 保存文件
   └── AI 自动调用 Pencil 的 File > Save
       │
       ▼
7. 提交到 Git
   └── AI 自动 git add + commit + push
```

---

## Slide 8: 设计稿保存在哪里？

### 文件位置
```
~/work/novita-design/
├── README.md                  # 仓库说明
├── benchmarks-page.pen        # Benchmarks 页面设计稿
├── pricing-page.pen           # Pricing 页面设计稿（示例）
├── docs/                      # 文档目录
│   ├── tutorial-*.md          # 使用教程
│   └── gap-analysis-*.md      # 分析文档
└── ...
```

### 命名规范
| 页面 | 文件名 |
|------|--------|
| 首页 | `homepage.pen` |
| 价格页 | `pricing-page.pen` |
| Benchmarks | `benchmarks-page.pen` |
| 模型库 | `model-library-page.pen` |
| Console 仪表盘 | `console-dashboard.pen` |

### .pen 文件特点
- **加密的 JSON 格式**，只能通过 Pencil 应用或 Pencil MCP 工具读写
- 不能用普通文本编辑器打开
- 支持 Git 版本管理（二进制差异追踪）
- 每个文件包含：设计系统组件 + 页面设计

---

## Slide 9: 如何提交设计稿？

### 方式一：让 AI 自动提交（推荐）
在 Claude Code 中直接说：
```
请将刚才的设计稿 commit 并 push 到远程仓库
```
AI 会自动执行：
```bash
cd ~/work/novita-design
git add benchmarks-page.pen
git commit -m "feat: add benchmarks page design"
git push origin main
```

### 方式二：手动提交
```bash
cd ~/work/novita-design
git add .
git commit -m "feat: 新增/更新 XXX 页面设计稿"
git push origin main
```

### 提交规范
| 前缀 | 场景 |
|------|------|
| `feat:` | 新增设计稿 |
| `fix:` | 修复设计问题（布局、颜色、字体等）|
| `refactor:` | 重构设计（如更换设计系统组件）|
| `docs:` | 文档更新 |

---

## Slide 10: 常用自然语言指令示例

### 生成设计稿
```
将 pricing 页面生成 pencil 设计稿，保存到 ../novita-design/pricing-page.pen
```

### 从零设计新页面
```
设计一个 AI 模型对比页面，包含：
- Novita 品牌标准头部导航
- 模型卡片网格（每行3个）
- 筛选侧边栏
- 标准页脚
保存到 ../novita-design/model-comparison.pen
```

### 修改已有设计
```
打开 ../novita-design/benchmarks-page.pen
将头部高度改为 80px
将表格行的 hover 背景色改为 #FAFAFA
```

### 检查设计
```
给我看一下 benchmarks-page.pen 的全页截图
```

### 批量操作
```
将设计稿中所有 #000000 颜色替换为 #292827
```

---

## Slide 11: 设计系统 Token 查询（高级用法）

### 调用 ppio-ui-skill 查询设计规范
```
/ppio-ui-skill 查询 Novita 的按钮设计规范
```

### 在终端直接使用搜索脚本
```bash
# 查询完整设计系统
python3 scripts/search.py "Novita dashboard" --product novita --design-system

# 查询侧边栏结构
python3 scripts/search.py "sidebar GPUs menu" --product novita --domain sidebar -n 20

# 查询颜色 Token
python3 scripts/search.py "brand color primary" --product novita --domain design-system

# 查询组件规范
python3 scripts/search.py "button states" --product novita --domain design-system -n 10
```

---

## Slide 12: 设计稿审阅技巧

### 截图验证
AI 在生成过程中会自动截图。你也可以主动要求：
```
截图 header 节点让我看看
截图整个页面的全貌
```

### 节点检查
```
读取 header 节点的详细属性
检查 footer 的子节点结构
```

### 布局检查
```
检查页面的布局结构，看是否有溢出或重叠
```

### 对比参考
可以让 AI 同时打开线上页面和设计稿对比：
```
打开 https://novita.ai/pricing 页面，
对比设计稿中 pricing-page.pen 的头部和底部是否一致
```

---

## Slide 13: 常见问题排查

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| "failed to connect to Pencil app" | Pencil 应用未启动 | 先打开 Pencil 桌面应用，再启动 Claude Code |
| "WebSocket not connected" | MCP Server 断开 | 重启 Claude Code（退出后重新输入 `claude`）|
| 设计稿文件大小没变 | Pencil 未保存到磁盘 | 让 AI 执行 `File > Save`，或手动在 Pencil 中 Cmd+S |
| 颜色显示全黑 | 变量未正确解析 | 检查设计系统变量定义，必要时使用直接色值替代 |
| git push 失败 | 无远程仓库写权限 | 检查 SSH Key 和仓库权限设置 |
| 生成的设计与代码不匹配 | AI 未读取完整源代码 | 明确指定要参考的文件路径 |

---

## Slide 14: 团队协作最佳实践

### 分工建议
| 角色 | 职责 |
|------|------|
| **产品经理** | 用自然语言描述页面需求 → 审阅设计截图 → 确认提交 |
| **前端研发** | 用代码路径指定参考源 → 校验设计与代码一致性 → 调整细节 |
| **UI 设计师** | 在 Pencil 中直接编辑微调 → 维护设计系统组件 |

### Git 工作流
```
main (主分支)
  └── 所有确认的设计稿直接提交到 main
  └── 大版本迭代可以用分支：
      └── feature/redesign-pricing
      └── feature/new-dashboard
```

### Code Review
- 每次提交后在 GitHub 上查看 diff（.pen 文件为二进制）
- 通过截图对比确认变更内容
- 可以让 AI 生成变更说明作为 commit message

---

## Slide 15: 进阶 —— 自定义设计系统

### 设计系统存储位置
```
novita-design/
└── benchmarks-page.pen
    ├── Novita Design System-2    # 设计系统框架（24个可复用组件）
    │   ├── Button/Default         # 主按钮
    │   ├── Button/Outline         # 边框按钮
    │   ├── Input/Default          # 输入框
    │   ├── Badge/Default          # 徽标
    │   ├── Table/Container        # 表格容器
    │   ├── Table Row              # 表格行
    │   └── ...                    # 共 24 个组件
    └── Benchmarks Page            # 具体页面设计
```

### 设计变量（67个）
- 颜色变量：`--primary`, `--foreground`, `--background`, `--border` 等
- 字体变量：`--font-family-primary`, `--font-size-base` 等
- 间距变量：`--spacing-0` ~ `--spacing-16`
- 圆角变量：`--radius-sm` ~ `--radius-pill`

### 扩展组件
需要新组件时，可以告诉 AI：
```
在设计系统中添加一个 Modal/Dialog 组件，
参考 shadcn/ui 的 Dialog 样式，使用 Novita 品牌色
```

---

## Slide 16: 总结

### 关键记忆点

1. **启动顺序**：先开 Pencil → 再开 Claude Code
2. **设计稿位置**：`~/work/novita-design/*.pen`
3. **提交方式**：`git add → commit → push` 或让 AI 自动完成
4. **修改方式**：用自然语言描述，AI 会自动定位和修改
5. **截图验证**：随时可以要求 AI 截图查看当前设计

### 一句话总结
> 打开 Pencil，启动 Claude Code，说出你要什么，审阅截图，确认提交。

---

## Slide 17: Q&A

**常见问题**

Q: 这个流程需要会代码吗？
A: 不需要。产品经理可以完全用自然语言描述需求。

Q: 生成一个页面设计稿需要多久？
A: 简单页面约 3-5 分钟，复杂页面（如 Dashboard）约 10-15 分钟。

Q: 能不能修改设计稿的局部？
A: 可以。直接用自然语言描述要修改的部分，AI 会精准定位修改。

Q: 设计稿和代码怎么保持同步？
A: 目前为手动同步。修改代码后重新生成设计稿覆盖即可。

**联系方式**
- 技术问题：在 Claude Code 中随时提问
- 仓库地址：`github.com/ivyrao-novita/novita-design`
