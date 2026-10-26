# Code Wiki — mattpocock/skills

> **仓库**：https://github.com/mattpocock/skills
> **版本**：1.1.0 · **许可**：MIT · **语言构成**：Markdown（主体）+ Bash（脚本）
> **定位**：Matt Pocock 日常使用的一套 **AI Agent Skills（智能体技能）** 集合——面向"真实工程"而非"氛围编码"。技能以 Markdown 文件（`SKILL.md`）为载体，被 Claude Code、Codex 等编码智能体以斜杠命令（`/xxx`）或自动触发方式消费。

---

## 目录

1. [项目概述](#1-项目概述)
2. [项目整体架构](#2-项目整体架构)
3. [目录结构](#3-目录结构)
4. [主要模块职责](#4-主要模块职责)
5. [关键"类与函数"说明](#5-关键类与函数说明)
6. [依赖关系](#6-依赖关系)
7. [项目运行方式](#7-项目运行方式)
8. [版本演进](#8-版本演进)
9. [术语表](#9-术语表)

---

## 1. 项目概述

### 1.1 这个项目解决什么问题

本仓库是一组**可组合、可改造的智能体技能**，针对 AI 辅助开发的四大常见失败模式给出纪律化修法：

| # | 失败模式 | 修法（技能） |
|---|---------|-------------|
| 1 | Agent 没做你想要的（需求错位） | `/grill-me`、`/grill-with-docs`——构建前的严酷访谈 |
| 2 | Agent 太啰嗦（缺少共享语言） | `/domain-modeling` 维护的 `CONTEXT.md` 术语表 + ADR |
| 3 | 代码跑不起来（反馈回路缺失） | `/tdd`（红-绿-重构）、`/diagnosing-bugs`（诊断循环） |
| 4 | 造出一团泥球（软件熵加速） | `/to-spec`、`/codebase-design`、`/improve-codebase-architecture` |

### 1.2 项目形态

这不是传统的可执行代码项目，而是一个**内容即代码（Docs-as-Code）仓库**：

- **核心资产**是 38 个技能目录（22 个 promoted + 16 个非 promoted），每个目录至少含一个 `SKILL.md`（提示词/行为契约），可选含附属参考文件（格式模板、脚本）。
- **无构建产物、无测试套件、无应用入口**。`package.json` 仅有 changesets 版本管理依赖。
- 技能通过 `npx skills@latest add mattpocock/skills`（skills.sh 安装器）分发给终端用户，或通过 `.claude-plugin/plugin.json` 作为 Claude 插件分发。

### 1.3 核心设计理念（源自 `.agents/` 与 `writing-great-skills`）

1. **小、易改、可组合**：拒绝 GSD/BMAD/Spec-Kit 式"接管整个流程"的重框架，每个技能只做一件事，靠组合成完整工作流。
2. **调用轴二分**：每个技能要么 **user-invoked**（仅人键入 `/name` 触达，`disable-model-invocation: true`，零上下文开销），要么 **model-invoked**（模型可自主触发，description 常驻上下文）。user-invoked 技能可调用 model-invoked 技能，但永远无法触达另一个 user-invoked 技能。
3. **依赖即散文调用**：技能间依赖不用文件相对路径交叉引用，而是 `/skill` 式散文调用（"Run the `/grilling` skill"）。
4. **硬/软依赖分离**（ADR-0001）：依赖 per-repo 配置的技能分两档——硬依赖技能（`to-tickets`/`to-spec`/`triage`）带显式 setup 指针；软依赖技能（`tdd`/`diagnosing-bugs` 等）只用模糊散文提及领域文档，缺配置也能降级工作。

---

## 2. 项目整体架构

### 2.1 架构总览图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          分发层（Distribution）                       │
│   npx skills add mattpocock/skills    ·    .claude-plugin/plugin.json │
└──────────────────────────────────┬──────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          技能层（skills/）                            │
│                                                                       │
│  ┌── engineering/（17 个目录，16 个进入插件与顶层 README）────────┐    │
│  │  路由: ask-matt                                               │    │
│  │  主流程: grill-with-docs → to-spec → to-tickets → implement    │    │
│  │  匝道: triage · diagnosing-bugs · wayfinder                    │    │
│  │  设施: setup-matt-pocock-skills（一次性配置）                   │    │
│  │  纪律内核(model-invoked): grilling* · tdd · code-review        │    │
│  │    domain-modeling · codebase-design · prototype · research    │    │
│  │  维护: improve-codebase-architecture                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│  ┌── productivity/（5 个，进入插件与 README）────────────────────┐    │
│  │  grill-me · handoff · teach · writing-great-skills             │    │
│  │  grilling（质询纪律内核，被 engineering 跨桶调用）              │    │
│  └────────────────────────────────────────────────────────────────┘    │
│  ┌── misc/(4) · personal/(2) · in-progress/(6) · deprecated/(4)──┐    │
│  │  不进插件、不进顶层 README（成熟度桶）                          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└──────────────┬──────────────────────────────────────┬───────────────┘
               ▼                                      ▼
┌──────────────────────────┐          ┌───────────────────────────────┐
│  文档层（docs/）          │          │  治理层（.agents/）            │
│  engineering/ productivity│          │  invocation.md  调用轴规范     │
│  与 skills 树镜像，       │          │  writing-docs.md 文档页规范    │
│  发布到 aihero.dev        │          │  adr/          架构决策记录   │
└──────────────────────────┘          └───────────────────────────────┘
               ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    基础设施（scripts/ · .changeset/ · .github/）      │
│  link-skills.sh（本地符号链接安装） · list-skills.sh（清单）          │
│  changesets 版本管理 · GitHub Actions Release 工作流                  │
└─────────────────────────────────────────────────────────────────────┘
                                        * grilling 实际位于 productivity/
```

### 2.2 运行时架构：技能如何被消费

技能不是被"运行"的代码，而是被**加载进智能体上下文**的行为契约：

1. **Frontmatter 解析**：harness（Claude Code 等）读取 `SKILL.md` 头部的 YAML frontmatter。`disable-model-invocation: true` 决定该技能是否进入模型的自动触发候选集；`description` 在 model-invoked 技能中常驻系统上下文（这是唯一的"常驻成本"）；`argument-hint` 提示用户传参。
2. **触发**：user-invoked 由人键入 `/name`；model-invoked 由模型依据 description 中的触发语自主调用。
3. **渐进披露**：正文步骤可指向附属参考文件（如 `tdd` 的 `tests.md`、`prototype` 的 `LOGIC.md`），按需加载，控制上下文消耗。
4. **组合**：一个技能正文可以散文式调用另一个技能（如 `implement` 调 `/tdd`、收尾调 `/code-review`）。

### 2.3 主工作流（idea → ship）

```
                      ┌──────────────────────────────────────────┐
                      │  /setup-matt-pocock-skills（每 repo 一次）│
                      │  写入 tracker/labels/domain 配置          │
                      └───────────────────┬──────────────────────┘
                                          ▼
   /grill-with-docs ──调用──▶ /grilling + /domain-modeling
   （同一上下文窗口内连续进行，约 120k token "smart zone"，快满则 /handoff）
          │
          ├─ 需要可运行答案？──▶ /handoff ↔ /prototype ──▶ /handoff
          ▼
   /to-spec ──── 发布 spec issue（打 ready-for-agent）
          ▼
   /to-tickets ─ 发布 tracer-bullet tickets（声明阻塞边，打 ready-for-agent）
          ▼
   /implement（每张 ticket 一个全新上下文窗口）
          ├──▶ /tdd（红-绿切片）
          └──▶ /code-review（双轴审查后提交）

   匝道汇入：/triage（外部 issue/PR）──▶ agent-ready issue ──▶ /implement
            /wayfinder（超大工程）──▶ 决策清晰后 ──▶ /to-spec 或直接 /implement
```

---

## 3. 目录结构

```
/workspace
├── skills/                        # ★ 核心资产：技能库（按成熟度分桶）
│   ├── engineering/               #   日常代码工作技能（15 个，promoted）
│   │   ├── ask-matt/              #     技能路由器
│   │   ├── grill-with-docs/       #     访谈 + 领域建模
│   │   ├── triage/                #     issue 分诊状态机（+AGENT-BRIEF/OUT-OF-SCOPE）
│   │   ├── improve-codebase-architecture/  # 架构深化（+HTML-REPORT）
│   │   ├── setup-matt-pocock-skills/       # 一次性 per-repo 配置（5 个种子模板）
│   │   ├── to-spec/  to-tickets/  implement/  wayfinder/
│   │   ├── prototype/             #     +LOGIC.md / UI.md 两分支
│   │   ├── diagnosing-bugs/       #     +scripts/hitl-loop.template.sh
│   │   ├── research/  tdd/        #     tdd +mocking.md / tests.md
│   │   ├── domain-modeling/       #     +ADR-FORMAT.md / CONTEXT-FORMAT.md
│   │   ├── codebase-design/       #     +DEEPENING.md / DESIGN-IT-TWICE.md
│   │   ├── code-review/           #     双轴 diff 审查
│   │   ├── resolving-merge-conflicts/  # 冲突解决循环（⚠️ 未进 plugin.json）
│   │   └── README.md              #     分 User/Model 两组列出
│   ├── productivity/              #   通用工作流技能（5 个，promoted）
│   │   ├── grill-me/ grilling/ handoff/ teach/ writing-great-skills/
│   │   └── README.md
│   ├── misc/                      #   低频工具（4 个，不进插件）
│   │   ├── git-guardrails-claude-code/  # +scripts/block-dangerous-git.sh
│   │   ├── migrate-to-shoehorn/ scaffold-exercises/ setup-pre-commit/
│   │   └── README.md
│   ├── personal/                  #   个人环境绑定（2 个，不进插件）
│   │   ├── edit-article/ obsidian-vault/
│   │   └── README.md
│   ├── in-progress/               #   开发中草稿（6 个，不进插件）
│   │   ├── claude-handoff/ loop-me/ wizard/   # wizard +template.sh
│   │   ├── writing-beats/ writing-fragments/ writing-shape/
│   │   └── README.md
│   └── deprecated/                #   弃用（4 个）
│       ├── design-an-interface/ qa/ request-refactor-plan/ ubiquitous-language/
│       └── README.md
├── docs/                          # 人向文档页（镜像 promoted 桶），发布到 aihero.dev
│   ├── engineering/               #   17 个技能各一页（含 resolving-merge-conflicts）
│   └── productivity/              #   5 个技能各一页
├── .agents/                       # 维护者治理文档
│   ├── invocation.md              #   user/model 调用轴规范
│   ├── writing-docs.md            #   docs 页写作模板与 Done 清单
│   └── adr/0001-*.md              #   硬/软依赖分离决策
├── scripts/
│   ├── link-skills.sh             # 把技能符号链接进本地 harness 目录
│   └── list-skills.sh             # 列出全部 SKILL.md
├── .claude-plugin/plugin.json     # Claude 插件清单（21 个技能路径）
├── .changeset/                    # changesets 版本管理配置
├── .github/workflows/release.yml  # main 推送触发的版本 PR / tag 流程
├── .out-of-scope/                 # triage 技能写入的"拒绝知识库"（repo 自用示例）
├── CONTEXT.md                     # 本仓库自身的领域术语表（自反示范）
├── CLAUDE.md                      # 面向 agent 的仓库导航与维护规则
├── README.md                      # 顶层 README（技能总索引，User/Model 分组）
├── CHANGELOG.md · LICENSE · package.json · package-lock.json
└── .gitignore
```

---

## 4. 主要模块职责

### 4.1 engineering 桶（17 个技能）— 日常代码工作

#### User-invoked（用户键入触发，9 个）

| 技能 | 职责 | 输入 → 产物 | 关键机制 |
|------|------|------------|---------|
| **ask-matt** | 技能路由器；记不住用哪个技能时的地图 | 用户处境描述 → 技能建议 | 纯路由不执行；维护主流程图 + 匝道 + 独立技能清单 |
| **grill-with-docs** | 主流程第一站：把模糊计划磨锋利并沉淀文档 | 对话 + codebase → `CONTEXT.md`、`docs/adr/` | 一行封装：`/grilling` + `/domain-modeling` |
| **triage** | 把外部 issue/PR 推过分诊状态机 | tracker 条目 → 标签、Agent Brief、`.out-of-scope/*.md` | 2 类别角色 + 5 状态角色状态机；每条 AI 评论须带免责前缀；`AGENT-BRIEF.md` 定义耐久 brief 四原则；`OUT-OF-SCOPE.md` 定义拒绝知识库去重 |
| **improve-codebase-architecture** | 扫描浅模块 → HTML 报告 → grilling 深挖 | codebase → 临时目录 HTML 报告 + CONTEXT/ADR 更新 | 三阶段（Explore→Report→Grill）；依赖最重：调 `/codebase-design`、`/grilling`、`/domain-modeling`；`HTML-REPORT.md` 规定报告规格与术语纪律 |
| **setup-matt-pocock-skills** | 一次性 per-repo 配置引导 | 问答 → `CLAUDE.md`/`AGENTS.md` 的 `## Agent skills` 块 + `docs/agents/{issue-tracker,triage-labels,domain}.md` | 探索→逐节确认（tracker/labels/domain 布局）→写入；5 个种子模板文件即配置产物原型 |
| **to-spec** | 把已有对话合成为 spec 并发布 | 对话 + repo → tracker 上的 spec issue | 明令不再访谈；3 步流程含"勾勒测试 seam"；spec 模板禁写文件路径/代码 |
| **to-tickets** | 拆解为 tracer-bullet 纵向切片 ticket | spec/对话 → `tickets.md`（本地）或带阻塞边的原生 issues | 5 步流程含 prefactoring 与 expand-contract 宽重构例外；Quiz 迭代至批准 |
| **implement** | 执行 spec/ticket/agent brief 描述的工作 | 工作单元 → 代码 + 测试提交 | 仅 15 行：驱动 `/tdd`、收尾 `/code-review` |
| **wayfinder** | 规划超出单 session 容量的巨型工程 | 模糊想法 → map issue + 子调查 tickets | "产出决策而非交付物"；4 类 ticket（Research/Prototype/Grilling/Task）；雾区（Not yet specified）逐块毕业；每 session 只解析一张 |

#### Model-invoked（模型可自动触发，8 个）

| 技能 | 职责 | 关键机制 |
|------|------|---------|
| **prototype** | 用一次性代码回答设计问题 | 双分支：`LOGIC.md`（逻辑问题→纯模块+TUI）与 `UI.md`（外观问题→单路由多变体 `?variant=` 切换）；用完即删或吸收 |
| **diagnosing-bugs** | 疑难 bug/性能回退的诊断纪律 | 6 阶段循环；核心信条"先建紧反馈回路（red-capable、确定性、秒级）"；10 种回路按序尝试；`[DEBUG-xxx]` 插桩收尾清光；架构性根因修后移交 improve-codebase-architecture；附 `hitl-loop.template.sh` |
| **research** | 后台 agent 对一手来源调研并逐条溯源 | 3 步：调查→带引用 Markdown→按仓库惯例保存；不采信二手转述 |
| **tdd** | 红-绿-重构 TDD 纪律 | 一次一个 seam、一个测试；重构不在循环内（归 code-review）；三反模式（实现耦合/同义反复/水平切片）；`tests.md` 好坏测试对照、`mocking.md` 边界 mock 纪律 |
| **domain-modeling** | 主动构建与锐化领域模型 | 5 个会话动作（挑战/锐化/压力测试/交叉验证/即时更新）；CONTEXT.md 纯词汇表、惰性创建；ADR 三条件门槛；`CONTEXT-FORMAT.md`/`ADR-FORMAT.md` 定格式 |
| **codebase-design** | "深模块"设计词汇与原则 | 词汇表（module/interface/seam/depth/adapter…）+ 术语纪律；`DEEPENING.md` 依赖四分类（in-process/local-substitutable/remote-but-owned/true external）与 "replace, don't layer"；`DESIGN-IT-TWICE.md` 并行子代理出多接口方案 |
| **code-review** | 双轴 diff 审查 | Standards 轴（仓库标准 + 内置 Fowler 12 坏味道基线）与 Spec 轴（忠实实现 issue/PRD）各一个并行子代理，聚合不合并不跨轴重排 |
| **resolving-merge-conflicts** | 按意图解决进行中的 git merge/rebase 冲突 | 5 步循环：看当前状态 → 溯源每边的 primary source（commit/PR/issue）→ 逐 hunk 解决（保留双方意图；不兼容时按合并目标取舍并记录权衡；**永不发明新行为、永不 `--abort`**）→ 发现并运行项目自动检查（typecheck→tests→format）→ 完成合并/rebase。完全独立，无技能依赖 |

### 4.2 productivity 桶（5 个技能）— 通用工作流

| 技能 | 调用类型 | 职责 |
|------|---------|------|
| **grilling** | model-invoked | 质询纪律内核：逐分支走设计树、一次一问、每问附推荐答案、事实自查/决策归人。被 grill-me、grill-with-docs、triage、wayfinder、loop-me 等复用 |
| **grill-me** | user-invoked | `/grilling` 的薄用户入口（一行转发），是"按调用拆分"模式的活例 |
| **handoff** | user-invoked | 把对话压缩成交接文档写入 OS 临时目录：含 suggested skills 节、引用而非复述工件、脱敏 |
| **teach** | user-invoked | 有状态跨会话教学工作区：`MISSION.md` 锚点 + `learning-records/`（教学版 ADR，算最近发展区）+ `lessons/` + `reference/` + `GLOSSARY.md`；知识-技能-智慧三层哲学；4 个 FORMAT 文件定义各状态文件格式 |
| **writing-great-skills** | user-invoked | 元技能：技能写作参考（调用轴、三级信息层级、leading word、六大失败模式）；`GLOSSARY.md` 承载全部定义——本仓库所有 frontmatter 设计的理论出处 |

### 4.3 其他桶（不进插件）

| 桶 | 技能 | 说明 |
|----|------|------|
| **misc/** | git-guardrails-claude-code、migrate-to-shoehorn、scaffold-exercises、setup-pre-commit | 低频工具；git-guardrails 部署 PreToolUse hook 拦截危险 git 命令（`scripts/block-dangerous-git.sh`） |
| **personal/** | edit-article、obsidian-vault | 绑定个人环境（Obsidian 库路径硬编码） |
| **in-progress/** | claude-handoff、loop-me、wizard（+`template.sh`）、writing-beats、writing-fragments、writing-shape | 开发中草稿；写作三件套构成 explore（fragments）→ exploit（beats/shape）流水线 |
| **deprecated/** | design-an-interface、qa、request-refactor-plan、ubiquitous-language | 已弃用；其职责被 codebase-design（DESIGN-IT-TWICE）、triage、domain-modeling（CONTEXT-FORMAT）等继任者吸收 |

### 4.4 支撑目录

| 目录/文件 | 职责 |
|-----------|------|
| `docs/` | **22 页**人向文档页（engineering 17 + productivity 5），目录树镜像 `skills/` 的两个 promoted 桶，与 SKILL.md 一一对应；发布 URL 一律 `https://aihero.dev/skills-<name>`；页面模板与 Done 清单见 `.agents/writing-docs.md`（详见 4.5） |
| `.agents/` | 维护者规范：`invocation.md`（调用轴）、`writing-docs.md`（文档页模板）、`adr/`（本仓库自身的 ADR） |
| `.out-of-scope/` | `triage` 技能的拒绝知识库在**本仓库自身**上的实例——自反示范（3 个文件详见下文） |
| `CONTEXT.md` | 本仓库自身的领域术语表（Issue tracker / Issue / Triage role），同样是自反示范 |
| `CLAUDE.md` | 面向 agent 的维护规则：桶的晋升规则（promoted 必须进 README 与 plugin.json）、docs 页同步规则、ask-matt 路由器同步规则 |
| 杂项 | `.gitignore` 仅忽略 `node_modules`；`LICENSE` 为 MIT（Copyright (c) 2026 Matt Pocock）；`.changeset/README.md` 为 changesets 官方脚手架说明 |

**`.out-of-scope/` 拒绝知识库的 3 条记录**（每条 = 结论 + 理由 + 逃生口 + 先例 issue）：

| 文件 | 拒绝的请求 | 核心理由 | 先例 |
|------|-----------|---------|------|
| `mainstream-issue-trackers-only.md` | 为小众/新兴 tracker 添加后端 | 每个 tracker 后端都把 CLI 形状硬编码进技能，是永久维护面；"主流"判据是典型工程师是否认识该工具（GitHub/GitLab/Backlog.md 算，几百 stars 的新工具不算） | #99（dex，~300 stars） |
| `question-limits.md` | 给 grilling 加问题数硬上限 | 开放性是设计使然；硬上限会混淆两个失败模式——计划欠明确（正常）vs 问题质量差（应修 prompt 而非加计数器） | #44（"Codex 问了 200 个问题"） |
| `setup-skill-verify-mode.md` | 为 setup 技能加 verify 模式或兄弟技能 | prompt 驱动的技能用自然语言即可 scoped 到验证趟；两个技能会随种子模板演化而漂移 | #106 |

### 4.5 docs/ 发布文档系统

22 页与 22 个 promoted `SKILL.md` 严格一一对应（双向无遗漏）。所有页面遵循统一框架（`.agents/writing-docs.md`）：

1. **Quickstart**：`npx skills add mattpocock/skills --skill=<name>` + `npx skills update <name>`；
2. **Source**：指向 GitHub 源目录的绝对链接；
3. **What it does**：一句话定位 + **定义性约束**（defining constraint）——全页最值钱的一句，如 code-review 的"两轴永不合并"、diagnosing-bugs 的"No red-capable loop, no diagnosis"、wayfinder 的"只规划不执行"；
4. **When to reach for it**：调用模式（人键入 or 模型自动）+ 触发边界 + 易混技能的分流指引；
5. **Prerequisites**（可选，8 页有）；
6. **自由中间章节**（1-3 个，必须浮现技能的 leading word）；
7. **It's working if**（可选，仅 engineering 桶 10 页有）；
8. **Where it fits**（必有）：角色定位 + 邻居技能 + 指向 ask-matt 的收尾句——每页是分布式路由的一个节点。

**纪律**：全部链接为绝对链接（`https://aihero.dev/skills-<name>` / `github.com`），因为发布后相对链接会断；页面无 H1（标题由发布层 slug 提供）。**特征章节** `## Pulled out on purpose`（"刻意单源"声明）仅出现在三个词汇单源技能：codebase-design、domain-modeling、grilling。

### 4.6 仓库已知一致性问题（如实记录）

CLAUDE.md 规定"engineering/ 的每个技能必须进顶层 README 与 plugin.json"，但当前存在同步漂移：

| 问题 | 详情 |
|------|------|
| `resolving-merge-conflicts` 未收录 | 有 SKILL.md（engineering 桶）+ docs 页，但**缺席** plugin.json、顶层 README、桶 README 三处 |
| `implement` 漏列桶 README | 在 plugin.json、顶层 README、docs 页齐全，但 `skills/engineering/README.md` 漏列 |
| docs 悬空链接 | `docs/engineering/research.md` 链接到 `https://aihero.dev/skills-to-prd`——to-prd 已在 1.1.0 更名为 to-spec，目标页不存在 |
| docs 残留标记 | `docs/engineering/to-spec.md` 与 `to-tickets.md` 文件末尾残留 `</content>` 字样 |

---

## 5. 关键"类与函数"说明

本项目没有传统类与函数；对应的"接口"是 **SKILL.md frontmatter 规范**与 **Bash 脚本**。

### 5.1 SKILL.md frontmatter（技能的"类定义"）

每个技能目录的 `SKILL.md` 头部是 YAML frontmatter，字段即"接口签名"：

```yaml
---
name: tdd                          # 技能名（= 目录名 = 斜杠命令名）
description: >                     # model-invoked: 面向模型的富触发语，常驻上下文
  Test-driven development ...      # user-invoked: 面向人的一行摘要
  Use when the user wants ...
disable-model-invocation: true     # 可选。出现 = user-invoked（退出模型自动触发候选集）
argument-hint: "What will ..."     # 可选。斜杠命令的参数提示
---
# 正文 = 行为契约（步骤、纪律、模板、反模式）
```

**调用轴判定规则**（见 `.agents/invocation.md`）：

- 含 `disable-model-invocation: true` → user-invoked：零 context load，代价是 cognitive load（人脑当索引）；正文不可被其他 user-invoked 技能触达。
- 缺省 → model-invoked：description 常驻上下文，模型可自主触发，其他技能可散文调用它。
- 判据："模型能否有用地自主触达它？"——复用是提取技能的理由，不是选 model-invoked 的理由。

### 5.2 `scripts/link-skills.sh` — 本地安装器（维护者专用）

```bash
# 伪代码化的执行流程
REPO = 脚本上级目录
DESTS = [~/.claude/skills, ~/.agents/skills]        # 两个 harness 的技能目录
names/srcs = find skills/ -name SKILL.md             # 排除 node_modules 与 deprecated/*
for DEST in DESTS:
    if DEST 是指向本 repo 的符号链接: 报错退出       # 防止把 per-skill 链接写回 repo 自身
    for (name, src) in zip(names, srcs):
        if target 存在且不是符号链接: rm -rf target   # 清理非链接的旧安装
        ln -sfn src target                            # 幂等符号链接：git pull 即更新
```

- **安全设计**：`set -euo pipefail`；符号链接进 repo 的自指检测；幂等（重复运行安全）。
- 效果：本地安装的技能随 `git pull` 自动保持最新，重跑脚本仅需在增删改名技能后。

### 5.3 `scripts/list-skills.sh` — 技能清单

一行式：`find . -name SKILL.md -not -path '*/node_modules/*' | sed 's|^\./||' | sort`。用于审计仓库当前拥有的全部技能。

### 5.4 `skills/misc/git-guardrails-claude-code/scripts/block-dangerous-git.sh` — 危险命令拦截 hook

- 被 Claude Code 的 PreToolUse hook 调用；从 stdin 读 JSON，`jq` 提取 `tool_input.command`。
- 用 `grep -qE` 匹配 9 类危险模式：`push` 全部变体、`reset --hard`、`clean -f(d)`、`branch -D`、`checkout .`、`restore .`。
- 命中 → stderr 输出 BLOCKED 说明并 `exit 2`（Claude Code 语义：拦截并告知 agent 无权执行）；否则 `exit 0` 放行。

### 5.5 `skills/engineering/diagnosing-bugs/scripts/hitl-loop.template.sh` — HITL 反馈回路模板

第 10 种诊断回路的最后手段（确需人手操作时）。提供两个助手函数：

- `step "<指令>"` — 显示指令，等用户回车确认完成；
- `capture VAR "<问题>"` — 向用户提问并读入环境变量。

脚本结尾打印 `KEY=VALUE` 行，供 agent 解析回流结果。agent 复制模板后编辑标记区间的步骤即可运行。

### 5.6 `skills/in-progress/wizard/template.sh` — 向导脚本生成模板

`STAGES` 标记之上是禁止手改的共享 bash 库：终端颜色/banner、带进度与剩余时间的 stage 执行器、`say/step/note/warn`、跨平台（含 WSL）`open_url`、`pause/confirm`、以现有 `.env` 值为默认的幂等 `ask/ask_secret`、幂等 `write_env`、经 `gh` 容错写 secret 的 `set_secret/set_var`、收尾汇总 `finish`。标记之下是技能作者按流程撰写的 stages 区。

### 5.7 配置文件（"常量定义"）

| 文件 | 内容 |
|------|------|
| `.claude-plugin/plugin.json` | 插件清单：`name: mattpocock-skills` + **21 个技能路径**（engineering 16 + productivity 5）；注意 `resolving-merge-conflicts` 有 SKILL.md 与 docs 页但未收录（见 4.6） |
| `package.json` | `name: mattpocock-skills`、`private: true`；scripts 仅 `changeset`/`version`；devDependencies 仅 `@changesets/cli` + `@changesets/changelog-github`；`packageManager: npm@10.9.4` |
| `.changeset/config.json` | changelog 用 `@changesets/changelog-github`（指向 mattpocock/skills）；`privatePackages: {version: true}`（私有包照样打版本）；baseBranch main |
| `.github/workflows/release.yml` | main 推送触发；Node 22 + `npm ci`；`changesets/action` 创建 "chore: version skills" 版本 PR，合并后 `changeset tag` 打 tag |

---

## 6. 依赖关系

### 6.1 技能调用图（散文调用边）

```
【配置底座】
setup-matt-pocock-skills ──写入──▶ docs/agents/{issue-tracker,triage-labels,domain}.md
                                    CLAUDE.md/AGENTS.md 的 "## Agent skills" 块
        │ 读取方分档（ADR-0001）：
        ├─ 硬依赖（输出错而不仅是糊）：triage、to-spec、to-tickets
        ├─ 半硬：code-review（issue-tracker.md 缺失即被指示先跑 setup）
        └─ 软（降级可用）：tdd、diagnosing-bugs、improve-codebase-architecture、
                          wayfinder（无配置回退本地 markdown tracker）

【硬调用链（仅两条）】
grill-with-docs ──▶ /grilling + /domain-modeling（惰性产出 CONTEXT.md + docs/adr/）
implement      ──▶ /tdd（红绿切片）──▶ /code-review（收尾审查）

【条件调用】
triage    ──需要展开时──▶ /grilling + /domain-modeling
wayfinder ──▶ /grilling + /domain-modeling（命名目的地）、/prototype（prototype 类 ticket）
improve-codebase-architecture ──▶ /codebase-design（词汇+DESIGN-IT-TWICE）、
                                  /grilling、/domain-modeling
diagnosing-bugs ──Phase 6 架构性根因，修后移交──▶ improve-codebase-architecture
tdd ──重构归 review 阶段──▶ code-review

【纯被调用方 / 完全独立】
domain-modeling：不调用他人，被 grill-with-docs / improve-codebase-architecture / wayfinder 调用
prototype、research、resolving-merge-conflicts：互不调用，自成一体
```

### 6.2 跨桶依赖

- `grilling` 位于 productivity 桶，但被 engineering 桶的 grill-with-docs、triage、wayfinder、improve-codebase-architecture 调用——这是仓库中最重要的跨桶依赖。
- 调用规则约束：user-invoked（grill-with-docs 等）可调用 model-invoked（grilling），反之不成立；user-invoked 之间互不可达（所以才需要 ask-matt 这个路由器）。

### 6.3 数据流（文件与系统）

```
grill-with-docs/domain-modeling ──写──▶ CONTEXT.md · CONTEXT-MAP.md · docs/adr/NNNN-*.md
setup-matt-pocock-skills ──写──▶ docs/agents/*.md · CLAUDE.md 的 ## Agent skills 块
to-spec / to-tickets / triage / wayfinder ──读写──▶ Issue tracker
        ├─ GitHub：gh CLI（issues、labels、sub-issues、dependencies API）
        ├─ GitLab：glab CLI（issues/epics、/blocked_by）
        └─ 本地：.scratch/<feature>/{PRD.md, issues/NN-*.md} 或根目录 tickets.md
triage ──写──▶ .out-of-scope/<concept>.md（仅 enhancement 被拒为 wontfix 时）
prototype ──写──▶ 紧邻使用位置的原型代码 + NOTES.md/ADR（答案持久化）
improve-codebase-architecture ──写──▶ $TMPDIR/architecture-review-<ts>.html（不落仓库）
handoff ──写──▶ OS 临时目录交接文档（明确不进 workspace）
research ──写──▶ 仓库既有笔记约定处的带引用 Markdown
diagnosing-bugs ──写──▶ 反馈回路命令、[DEBUG-xxx] 插桩（收尾清除）、回归测试
code-review ──读──▶ git diff <fixed-point>...HEAD、CODING_STANDARDS.md 等、spec 来源
```

### 6.4 构建期与外部依赖

| 依赖 | 用途 |
|------|------|
| `@changesets/cli` + `@changesets/changelog-github` | 版本与 CHANGELOG 管理（唯一 npm 依赖，均为 devDependencies） |
| `changesets/action`（GitHub Action） | main 分支自动开版本 PR、打 tag |
| skills.sh（`npx skills`） | 面向终端用户的技能安装器（外部服务） |
| Node 22 / npm 10.9.4 | CI 与包管理 |
| Tailwind CDN + Mermaid 11 ESM CDN | 仅 improve-codebase-architecture 生成的 HTML 报告内引用（运行时按需，非仓库依赖） |

### 6.5 治理一致性约束（隐式依赖）

CLAUDE.md 规定的同步义务——违反即"撒谎的路由器/过期的索引"：

1. promoted 桶技能 ⇔ 顶层 README 条目 ⇔ `plugin.json` 条目，三者必须一致；
2. 技能增删改名/行为变化 ⇔ docs 页同步（`writing-docs.md`）；
3. 用户可达技能变化 ⇔ ask-matt 路由图同步；
4. 各桶 README 列出桶内全部技能并按 User/Model 分组（promoted 桶）或平铺（非 promoted）。

---

## 7. 项目运行方式

### 7.1 作为用户使用（主要方式）

```bash
# 1. 安装（skills.sh 安装器，交互式选择技能与目标 agent）
npx skills@latest add mattpocock/skills

# 或只装单个技能
npx skills add mattpocock/skills --skill=<name>
npx skills update <name>

# 2. 在你的项目 repo 里，务必选中并运行一次配置技能
#    （在 Claude Code 等 agent 中键入）
/setup-matt-pocock-skills
#    它会依次询问：issue tracker（GitHub/GitLab/本地文件）、
#    triage 标签词汇、领域文档布局（single/multi-context），
#    并写入 docs/agents/*.md 与 CLAUDE.md 的 ## Agent skills 块

# 3. 开始使用，推荐主流程：
/grill-with-docs     # 访谈 + 建立领域文档
/to-spec             # 对话合成 spec，发布到 tracker
/to-tickets          # 拆成纵向切片 tickets
/implement           # 逐 ticket 实现（内部驱动 /tdd，收尾 /code-review）

# 常用匝道与独立技能
/triage                      # 分诊外部 issue/PR
/diagnosing-bugs             # 疑难 bug 诊断
/wayfinder                   # 超大工程规划
/improve-codebase-architecture  # 每隔几天跑一次的架构维护
/ask-matt                    # 不知道用哪个时问路由器
```

### 7.2 作为维护者开发

```bash
git clone https://github.com/mattpocock/skills && cd skills
npm ci                      # 安装 changesets（唯一依赖）

# 本地安装全部技能到 harness（符号链接，git pull 即更新）
./scripts/link-skills.sh

# 审计技能清单
./scripts/list-skills.sh

# 修改技能时的同步义务（见 CLAUDE.md）：
#   - promoted 桶技能 → 同步顶层 README + .claude-plugin/plugin.json + 桶 README
#   - 行为变化        → 同步 docs/<bucket>/<name>.md（模板见 .agents/writing-docs.md）
#   - 用户可达技能变化 → 重读并更新 skills/engineering/ask-matt/SKILL.md

# 版本管理：改动后创建 changeset
npx changeset               # 选择 patch/minor/major 并写摘要
git push                    # CI 自动开 "chore: version skills" 版本 PR
# 合并该 PR 后 CI 自动 npx changeset tag 打 tag
```

### 7.3 CI/CD 流程（`.github/workflows/release.yml`）

1. push 到 `main` 触发，workflow 级并发去重；
2. Node 22 + `npm ci`；
3. `changesets/action`：若有未消费的 changesets → 开/更新版本 PR；版本 PR 被合并 → `npx changeset tag` 给对应 commit 打版本 tag；
4. 仓库为 private 包，不发布 npm，tag 即发布单元。

### 7.4 环境要求

- 任意支持 Agent Skills 规范的 harness（Claude Code、pi 及 `~/.agents/skills` 兼容 harness）；
- 维护者：Node 22+、npm 10.9.4、bash（link/list 脚本）、`jq`（仅 git-guardrails hook）；
- 各技能运行时按需调用外部工具：`gh`/`glab`（tracker）、浏览器/无头浏览器（diagnosing-bugs 回路）、平台 `open` 命令（HTML 报告）。

---

## 8. 版本演进（CHANGELOG）

CHANGELOG 由 changesets 自动生成（`@changesets/changelog-github`），每条带 PR/commit 链接。三个版本完整记录了技能集的演化路径，是理解当前命名与结构的最佳背景材料：

| 版本 | 关键变化 |
|------|---------|
| **1.0.0**（架构定型） | ① 新增 **ask-matt** 路由器（Breaking：要求安装全部 user-invoked 技能）；② 从 improve-codebase-architecture/tdd 中抽出**共享技能** `codebase-design`（原 LANGUAGE.md）与 `domain-modeling`（Breaking：新增依赖）；③ `diagnose` → **diagnosing-bugs** 改名；④ `write-a-skill` → **writing-great-skills** 替换；⑤ **grilling** 独立为 model-invoked 技能（grill-me/grill-with-docs 的共享内核）；⑥ 新增 **resolving-merge-conflicts**（独立无依赖）；⑦ 术语分类从 Commands/Skills 更名为 **User-invoked/Model-invoked**；⑧ 移除 caveman、zoom-out |
| **1.0.1** | `teach` 改为 **reuse-first**：课程从 `./assets/` 可复用组件构建，禁止内联未来会重复的代码 |
| **1.1.0**（大整合） | ① **ask-matt 路由补全** 5 个缺失技能（tdd、diagnosing-bugs、domain-modeling、codebase-design、grilling）；② `in-progress/review` 毕业 → **code-review**，新增 Fowler 12 坏味道基线；③ **grilling** 加确认门（用户确认共识前不动手）+ **事实/决策分离**（修复 grilling agent 自问自答）；④ writing-great-skills 新增 **Negation** 与 **Negative Space** 失败模式；⑤ **prototype** 改为 model-invoked；⑥ 新增 **research**；⑦ `to-prd` → **to-spec** 更名；`to-plan` + `to-issues` 合并为 **to-tickets**（阻塞边 + expand–contract 宽重构支持）；⑧ `in-progress/decision-mapping` 毕业 → **wayfinder**（确立 fog of war / frontier / map 主导词框架、四类 ticket、HITL/AFK 分类、原生 blocking、map=共享 issue）；⑨ **tdd** 重塑为纯 reference（红→绿；重构移交 code-review；新增同义反复测试反模式）；⑩ **triage** 支持外部 PR（"PR = 带代码的 issue"，setup 增加开关）；⑪ wayfinder 修复硬编码 tracker 路径（改走 `## Agent skills` 指针的间接层） |

**演化规律**：技能命名持续向 leading word 收敛（diagnose→diagnosing-bugs、decision-mapping→wayfinder、to-prd→to-spec）；共享纪律不断从单技能抽出为独立技能（codebase-design、domain-modeling、grilling）；in-progress 桶是毕业通道（wayfinder、code-review 均由此晋升）。

---

## 9. 术语表

摘自本仓库 `CONTEXT.md` 与 `writing-great-skills`（阅读其余文档前建议先掌握）：

| 术语 | 定义 | 禁用 |
|------|------|------|
| **Issue tracker** | 承载 repo issue 的工具——GitHub Issues、Linear、本地 `.scratch/` 约定等 | backlog manager、backlog backend |
| **Issue** | tracker 内单个工作单元——bug、任务、spec 或 to-tickets 产出的切片 | ticket（仅引用外部系统时可用） |
| **Triage role** | 分诊时施加于 Issue 的规范状态机标签（`needs-triage` 等），经 `docs/agents/triage-labels.md` 映射为真实 label 字符串 | — |
| **User-invoked / Model-invoked** | 调用轴两端：仅人触发 vs 模型可自主触发 | — |
| **Context load / Cognitive load** | description 常驻占用的模型上下文 vs 用户需记住技能存在的心智负担 | — |
| **Leading word** | 技能的"主导词"（tight、seam、tracer bullet…），用预训练中已有的紧凑概念锚定一片行为 | — |
| **Seam** | 不改此处代码即可改变行为的位置（Feathers） | boundary |
| **Deep / shallow module** | 接口处每单位学习成本换到大量行为 vs 接口几乎和实现一样复杂 | component、service、layer |
| **Tracer bullet** | 窄但完整贯穿所有层的纵向切片，可独立演示验证 | — |
| **Progressive disclosure** | 沿"步骤 → 技内参考 → 外部参考"信息层级下移，按需加载 | — |

---

*本文档基于仓库当前状态（2026-10，v1.1.0）生成；技能清单与调用关系以 `skills/` 各 `SKILL.md`、`.claude-plugin/plugin.json` 与顶层 `README.md` 为准，4.6 节记录的漂移点已如实标注。*
