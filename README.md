# TeamVibe

**Decision version control for team vibe coding.**

> Multi-agent tools are everywhere. Multi-human collaboration tools for vibe coding teams? None — until now.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[中文版](#中文版)

---

## The Problem — A Story You'll Recognize

Alice (PM) and Bob (developer) are building a product together. Both use AI coding assistants — Alice uses Cursor, Bob uses Claude Code.

**Monday morning.** Alice tells her AI: *"Remove the Z-shaped block from the game."* The AI does it. She commits and pushes.

**Monday afternoon.** Bob pulls the code, sees some changes in the diff, but doesn't know *why* the Z block was removed. He tells his AI: *"Add back the Z block and remove L instead."* His AI happily does it.

**Alice is furious.** She spent an hour discussing with users before deciding to remove Z. Bob had no idea. His AI had no idea. Nobody had any idea.

**This is the core problem:** In vibe coding teams, **business decisions live and die inside individual AI chat sessions.** They never become shared team knowledge.

### It Gets Worse

- **Your AI doesn't know your teammate's decisions.** When Bob asks his AI to change a feature, the AI can't warn him: *"Hey, Alice explicitly decided this last week."*
- **You can't tell who decided what.** Was it the human who explicitly asked for this? Or did the AI just pick an approach on its own? The distinction matters — overriding a human's deliberate choice requires a conversation; overriding an AI's guess doesn't.
- **Context window compaction loses decisions.** After a long coding session, your AI summarizes the conversation and forgets the specific decisions made early on.
- **Pull requests come too late.** By the time you write a PR description, the vibe coding decisions are already made and pushed. The damage is done.

## The Solution — What TeamVibe Does

TeamVibe turns every `git commit` into a decision checkpoint. Not a changelog — a **decision record**: *what* was decided, *why*, *by whom*, and whether it was a human choice or an AI suggestion.

These records travel with your code via Git. When your teammate pulls, their AI automatically learns about your decisions. When they try to override one, the system intervenes.

**Here's what actually happens — step by step, using our Tetris demo:**

### Step 1: Alice Commits → Decision Record Generated

Alice tells her AI: *"Change all blocks to rounded corners."* The code changes. She runs `git commit`.

**The commit is intercepted.** TeamVibe's hook asks the AI to generate a decision record:

```json
{
  "decision_key": "block-rounded-corners",
  "requirement": { "what": "Change all blocks to rounded corners", "why": "Modern visual style" },
  "decision": { "type": "human", "motivation": "User explicitly requested rounded corners" }
}
```

Alice reviews it, confirms, and the commit proceeds — now carrying both code *and* the decision record.

### Step 2: Bob Pulls → Gets a Team Briefing

Bob runs `git pull`. Instead of just seeing a diff, his AI tells him:

```
## Team Changes Since Your Last Pull

1. [Alice] 2026-03-16
   Decision: Change all blocks to rounded corners
   Type: 👤 Human decision
   Motivation: User explicitly requested rounded corners

⚠️ Alice explicitly decided this. Changing it requires talking to her first.
```

Now Bob's AI *knows* about Alice's decision — and will remember it for the rest of the session.

### Step 3: Bob Tries to Override → Conflict Detected

Bob tells his AI: *"Make the blocks square with sharp corners."*

**Before writing any code**, the AI checks the decision records and responds:

> ⚠️ This conflicts with Alice's decision (2026-03-16): she explicitly chose rounded corners. 
> Please talk to Alice offline first. Reply "confirmed, proceed" after you've discussed it.

Bob can't skip this. Even if he thinks his approach is better, the system requires human-to-human confirmation for overriding another person's explicit decision.

### Step 4: After Discussion → Decision Chain Updated

Bob talks to Alice. They agree on a compromise. Bob tells his AI "confirmed, proceed" and makes the change. The new decision record includes:

```json
{
  "decision_key": "block-rounded-corners",
  "supersedes": { "record_id": "alice's-record", "reason": "Discussed and agreed on compromise" },
  "decision": { "type": "human", "motivation": "Team agreed on updated approach after discussion" }
}
```

The decision chain is preserved. Anyone can trace *why* this evolved.

### Step 5: Push → Completeness Check

Bob runs `git push`. TeamVibe validates:
- ✅ Every commit with code changes has a corresponding decision record
- ✅ Major changes include background context

If anything is missing, the push is blocked until it's fixed.

## Three Decision Types — Why They Matter

Not all decisions are equal. TeamVibe distinguishes:

| Type | Example | When someone conflicts |
|------|---------|----------------------|
| `human` 👤 | *"I want rounded corners"* — Alice explicitly chose this | **Must** talk to Alice before overriding. No exceptions. |
| `human_ai` 👥 | *"The AI suggested streaming upload. I agreed."* — Consensus | Warn the user. They can decide whether to override. |
| `ai` 🤖 | *The AI picked 500ms as the animation speed on its own* | Mention it briefly. Proceed without blocking. |

This is the key insight: **a human's deliberate choice deserves more protection than an AI's arbitrary guess.**

## Five Hooks, Five Jobs

| When | Hook | What it does |
|------|------|-------------|
| 🟢 **Session starts** | `session-init` | Loads team decisions into AI context. Your AI starts every session knowing what the team decided. |
| 🔴 **`git commit`** | `pre-commit-decision` | Intercepts the commit. AI generates a decision record from the conversation + diff. You review and confirm. |
| 🔴 **`git push`** | `pre-push-validate` | Validates every commit has a decision record. Major changes must have background context. |
| 🟡 **`git pull`** | `post-pull-review` | Reads new decision records. Formats a team briefing and injects it into your AI session. |
| 🟡 **Context compacting** | `pre-compact-reminder` | Reminds you to save decision drafts before the AI's context window gets compacted. |

Red = blocks the action. Yellow = injects information. Green = sets up context.

## Real-Time Decision Drafts

What if the AI's context window gets compacted mid-session, before you commit?

TeamVibe's **Rules layer** solves this. The AI is instructed to maintain a draft file (`.teamwork/drafts/current.json`) throughout the conversation:

- You make a decision → AI appends it to the draft
- You and AI reach consensus → AI appends it
- AI makes an autonomous design choice → AI appends it
- Context about to compact → Hook reminds you to verify the draft

When you finally commit, the draft is already mostly complete. The commit hook just refines it against the actual diff.

## Supported Platforms

Both Cursor and Claude Code are supported through a shared compatibility layer:

| | Cursor | Claude Code |
|---|--------|-------------|
| Hooks config | `.cursor/hooks.json` | `.claude/settings.json` |
| Rules | `.cursor/rules/*.mdc` | `.claude/rules/*.md` |
| Hook scripts | `.cursor/hooks/*.py` (shared) | `.cursor/hooks/*.py` (shared) |
| Auto-detection | `cursor_version` in input | Fallback |

The **same Python scripts** run on both platforms. The `compat.py` layer handles input/output format differences automatically.

## Quick Start

**1.** Copy `.cursor/`, `.claude/`, and `.teamwork/` into your project root.

**2.** Create `.teamwork/config.json` with your team:

```json
{
  "version": "1.0",
  "team_members": [
    { "name": "Alice", "role": "PM", "email": "alice@team.com" },
    { "name": "Bob", "role": "Developer", "email": "bob@team.com" }
  ]
}
```

**3.** Commit and push. Done — every team member who pulls gets the hooks automatically.

## Project Structure

```
your-project/
├── .cursor/
│   ├── hooks.json                    # Cursor hooks config
│   └── hooks/
│       ├── compat.py                 # Cross-platform compatibility
│       ├── session-init.py           # → Session start context injection
│       ├── pre-commit-decision.py    # → Commit gate
│       ├── pre-push-validate.py      # → Push validation
│       ├── post-pull-review.py       # → Pull team briefing
│       └── pre-compact-reminder.py   # → Compaction reminder
├── .claude/
│   ├── settings.json                 # Claude Code hooks config
│   └── rules/
│       └── teamwork-decisions.md     # AI behavior rules
├── .teamwork/
│   ├── config.json                   # Team member registry
│   ├── decisions/                    # Decision records (Git-tracked)
│   │   └── 2026-03-16T1430_alice_a1b2c3.json
│   └── drafts/                       # Session drafts (gitignored)
└── tetris.html                       # Demo: Tetris H5 game
```

## The Untapped Power of Hooks

TeamVibe is built entirely on the **hooks systems** of AI coding IDEs — a powerful capability that most developers haven't fully explored yet.

Both [Cursor Hooks](https://cursor.com/docs/hooks) and [Claude Code Hooks](https://code.claude.com/docs/en/hooks) allow you to run custom scripts at key moments in the AI agent's lifecycle: before a command executes, after a file is edited, when a session starts, before context compaction, and more. They can observe, intercept, or modify the AI's behavior — deterministically, not by hoping the LLM remembers your instructions.

**Most people use hooks for simple tasks** — auto-formatting code, running linters, or blocking dangerous commands. But hooks can do far more:

- **Inject shared context** across team members' AI sessions (TeamVibe: `session-init`)
- **Gate workflows** with structured checkpoints (TeamVibe: `pre-commit-decision`)
- **Bridge information gaps** between collaborators (TeamVibe: `post-pull-review`)
- **Protect against context loss** during long sessions (TeamVibe: `pre-compact-reminder`)
- **Enforce team policies** without relying on LLM compliance (TeamVibe: `pre-push-validate`)

TeamVibe is one example of what's possible. The hooks API is an **underutilized extension point** — essentially a programmable middleware layer between humans and AI agents. We believe more creative applications will emerge as teams discover this capability.

> **Further reading:**  
> - [Cursor Hooks Documentation](https://cursor.com/docs/hooks)  
> - [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks)

## Why Not Just Use...

| Tool | Why it doesn't work |
|------|-------------------|
| **Git commit messages** | No structure. No conflict detection. You'd need to read every commit message to find decisions. |
| **Pull Request descriptions** | Too late. In vibe coding, you commit dozens of times. Decisions are made mid-session, not at PR time. |
| **Notion / Confluence** | Completely disconnected from code. Your AI can't read your Notion page. Manual sync breaks instantly. |
| **ADR (Architecture Decision Records)** | Manual-only. Nobody writes ADRs during a fast vibe coding session. No automation, no conflict detection. |
| **CrewAI / AutoGen / LangGraph** | These orchestrate *multiple AI agents*. TeamVibe coordinates *multiple humans, each with their own AI*. Completely different problem. |

## Contributing

Contributions welcome! Interesting directions:

- Support for more AI coding tools (Windsurf, Copilot, etc.)
- Web dashboard for decision visualization
- Decision analytics and team insights
- CLI for non-IDE workflows

## License

MIT

---

# 中文版

# TeamVibe

**团队 Vibe Coding 的决策版本管理工具。**

> 多 Agent 协作工具遍地都是。但多人 + 多 AI 的 Vibe Coding 团队协作工具？没有 —— 直到现在。

## 问题 —— 一个你一定遇到过的场景

Alice（产品经理）和 Bob（开发工程师）在一起做一个产品。Alice 用 Cursor，Bob 用 Claude Code。

**周一上午。** Alice 对 AI 说：*"把游戏里的 Z 形方块移除掉。"* AI 执行了。她提交并推送。

**周一下午。** Bob 拉取代码，在 diff 里看到了一些变化，但不知道**为什么** Z 方块被移除了。他对 AI 说：*"把 Z 方块加回来，改为移除 L 方块。"* AI 愉快地执行了。

**Alice 炸了。** 她跟用户讨论了一个小时才决定移除 Z。Bob 完全不知情。Bob 的 AI 也不知情。没有人知情。

**这就是核心问题：在 vibe coding 团队中，业务决策只存在于各自的 AI 对话里。** 它们永远不会成为团队共享的知识。

### 问题还不止于此

- **你的 AI 不知道队友的决策。** Bob 让 AI 改一个功能时，AI 无法提醒他：*"Alice 上周明确决定了这件事。"*
- **分不清谁做了什么决策。** 是人类明确要求的？还是 AI 自己挑的方案？这个区分很重要 —— 推翻人的深思熟虑需要沟通，推翻 AI 的随机选择不需要。
- **上下文窗口折叠后决策丢失。** 长时间编码后，AI 把对话压缩了，早期的决策细节消失了。
- **PR 描述来得太晚。** 等你写 PR 描述时，vibe coding 的决策早就做完并推送了。木已成舟。

## 解决方案 —— TeamVibe 做了什么

TeamVibe 把每次 `git commit` 变成一个决策检查点。不是 changelog —— 是**决策记录**：决定了*什么*、*为什么*、*谁决定的*、是人的选择还是 AI 的建议。

这些记录通过 Git 跟随代码一起分发。队友拉取代码时，他们的 AI 自动了解你的决策。当他们试图推翻某个决策时，系统会介入。

**以下是实际发生的过程 —— 用我们的俄罗斯方块演示项目逐步说明：**

### 第 1 步：Alice 提交 → 生成决策记录

Alice 对 AI 说：*"将方块全部改为圆角。"* 代码改好了。她执行 `git commit`。

**提交被拦截。** TeamVibe 的 hook 让 AI 从对话内容和代码差异中生成决策记录：

```json
{
  "decision_key": "block-rounded-corners",
  "requirement": { "what": "将所有方块改为圆角样式", "why": "提升视觉美观度" },
  "decision": { "type": "human", "motivation": "用户明确要求圆角" }
}
```

Alice 审阅确认后，提交继续 —— 现在这个 commit 同时携带了代码**和**决策记录。

### 第 2 步：Bob 拉取 → 收到团队变更通报

Bob 执行 `git pull`。他的 AI 不只是展示 diff，而是告诉他：

```
## 团队变更通报（自你上次 pull 以来）

1. [Alice] 2026-03-16
   决策：将所有方块改为圆角样式
   类型：👤 人工明确决策
   动机：用户明确要求圆角

⚠️ Alice 明确决定了这件事。修改需要先与她沟通。
```

现在 Bob 的 AI **知道了** Alice 的决策 —— 并且在整个会话中都会记住。

### 第 3 步：Bob 试图推翻 → 冲突检测

Bob 对 AI 说：*"把方块改回直角。"*

**在写任何代码之前**，AI 检查决策记录并回应：

> ⚠️ 这与 Alice 的决策冲突（2026-03-16）：她明确选择了圆角样式。
> 请先与 Alice 线下沟通确认。沟通后回复"已沟通，可以继续"。

Bob 无法跳过这一步。即使他认为自己的方案更好，系统也要求人与人之间的确认才能推翻另一个人的明确决策。

### 第 4 步：沟通后 → 决策链更新

Bob 跟 Alice 聊了。他们达成共识。Bob 回复"已沟通，可以继续"，然后修改代码。新的决策记录包含：

```json
{
  "decision_key": "block-rounded-corners",
  "supersedes": { "record_id": "alice的记录", "reason": "沟通后达成共识" },
  "decision": { "type": "human", "motivation": "团队讨论后同意更新方案" }
}
```

决策链被完整保留。任何人都可以追溯这个决策**为什么**演变。

### 第 5 步：推送 → 完整性校验

Bob 执行 `git push`。TeamVibe 校验：
- ✅ 每个有代码变更的 commit 都有对应的决策记录
- ✅ 重大变更包含背景说明

任何缺失都会阻止推送，直到补充完整。

## 三种决策类型 —— 为什么区分很重要

不是所有决策都一样重要。TeamVibe 做了区分：

| 类型 | 例子 | 有人冲突时怎么办 |
|------|------|---------------|
| `human` 👤 | *"我要圆角"* —— Alice 明确选择的 | **必须**跟 Alice 沟通后才能推翻。没有例外。 |
| `human_ai` 👥 | *"AI 建议用流式上传，我同意了"* —— 共识 | 提醒用户。由用户决定是否推翻。 |
| `ai` 🤖 | *AI 自己选了 500ms 作为动画速度* | 简要提及。不阻拦。 |

这是核心洞察：**人类的深思熟虑，比 AI 的随机选择，值得更多保护。**

## 五个 Hook，五个职责

| 触发时机 | Hook | 做什么 |
|---------|------|--------|
| 🟢 **会话启动** | `session-init` | 将团队决策加载到 AI 上下文中。AI 从第一句话就知道团队做了什么决策。 |
| 🔴 **`git commit`** | `pre-commit-decision` | 拦截提交。AI 从对话 + diff 中生成决策记录。你审阅确认。 |
| 🔴 **`git push`** | `pre-push-validate` | 校验每个 commit 是否有决策记录。重大变更必须有背景说明。 |
| 🟡 **`git pull`** | `post-pull-review` | 读取新增的决策记录。格式化为团队变更通报注入 AI 会话。 |
| 🟡 **上下文折叠** | `pre-compact-reminder` | 提醒你在 AI 上下文窗口压缩前确认决策草稿。 |

🔴 = 阻拦操作。🟡 = 注入信息。🟢 = 建立上下文。

## 实时决策草稿

如果 commit 前 AI 的上下文窗口就被压缩了怎么办？

TeamVibe 的 **Rules 层**解决了这个问题。AI 被指示在整个对话过程中维护一份草稿文件（`.teamwork/drafts/current.json`）：

- 你做了一个决策 → AI 立即追加到草稿
- 你和 AI 达成共识 → AI 立即追加
- AI 自主做了设计选择 → AI 立即追加
- 上下文即将压缩 → Hook 提醒你检查草稿

当你最终提交时，草稿已经基本完整。commit hook 只需要根据实际 diff 做增补修正。

## 支持的平台

通过共享的兼容层同时支持 Cursor 和 Claude Code：

| | Cursor | Claude Code |
|---|--------|-------------|
| Hooks 配置 | `.cursor/hooks.json` | `.claude/settings.json` |
| Rules | `.cursor/rules/*.mdc` | `.claude/rules/*.md` |
| Hook 脚本 | `.cursor/hooks/*.py`（共享） | `.cursor/hooks/*.py`（共享） |

**同一套 Python 脚本**在两个平台上运行。`compat.py` 兼容层自动处理输入输出格式差异。

## 快速开始

**1.** 将 `.cursor/`、`.claude/`、`.teamwork/` 复制到你的项目根目录。

**2.** 创建 `.teamwork/config.json` 配置团队成员：

```json
{
  "version": "1.0",
  "team_members": [
    { "name": "Alice", "role": "产品经理", "email": "alice@team.com" },
    { "name": "Bob", "role": "开发工程师", "email": "bob@team.com" }
  ]
}
```

**3.** 提交并推送。搞定 —— 团队成员拉取代码后自动获得所有 hooks。

## 项目结构

```
your-project/
├── .cursor/
│   ├── hooks.json                    # Cursor hooks 配置
│   └── hooks/
│       ├── compat.py                 # 跨平台兼容层
│       ├── session-init.py           # → 会话启动上下文注入
│       ├── pre-commit-decision.py    # → 提交守门
│       ├── pre-push-validate.py      # → 推送校验
│       ├── post-pull-review.py       # → 拉取后团队通报
│       └── pre-compact-reminder.py   # → 上下文折叠提醒
├── .claude/
│   ├── settings.json                 # Claude Code hooks 配置
│   └── rules/
│       └── teamwork-decisions.md     # AI 行为规则
├── .teamwork/
│   ├── config.json                   # 团队成员配置
│   ├── decisions/                    # 决策记录（Git 跟踪）
│   └── drafts/                       # 会话草稿（gitignored）
└── tetris.html                       # 演示：俄罗斯方块 H5 游戏
```

## Hooks 的潜力远未被挖掘

TeamVibe 完全构建在 AI 编程 IDE 的 **Hooks 系统**之上 —— 一个大多数开发者还没有充分探索的强大能力。

[Cursor Hooks](https://cursor.com/docs/hooks) 和 [Claude Code Hooks](https://code.claude.com/docs/en/hooks) 都允许你在 AI Agent 生命周期的关键节点运行自定义脚本：命令执行前、文件编辑后、会话启动时、上下文压缩前……它们可以观察、拦截或修改 AI 的行为 —— 是确定性的，不依赖于 LLM "记住"你的指令。

**大多数人只用 hooks 做简单的事** —— 自动格式化代码、运行 linter、拦截危险命令。但 hooks 能做的远不止这些：

- **跨成员注入共享上下文**，让每个人的 AI 从第一句话就了解团队决策（TeamVibe: `session-init`）
- **用结构化检查点把控工作流**，在提交时自动生成决策记录（TeamVibe: `pre-commit-decision`）
- **弥合协作者之间的信息差**，拉取代码后自动通报变更（TeamVibe: `post-pull-review`）
- **防止长会话中的上下文丢失**，在压缩前提醒保存决策草稿（TeamVibe: `pre-compact-reminder`）
- **强制执行团队策略**，不依赖 LLM 的"自觉"而是确定性拦截（TeamVibe: `pre-push-validate`）

TeamVibe 只是 hooks 能力的一个应用示例。Hooks API 本质上是**人与 AI Agent 之间的可编程中间件层** —— 一个被严重低估的扩展点。我们相信随着更多团队发现这个能力，会涌现出更多创造性的应用。

> **延伸阅读：**  
> - [Cursor Hooks 官方文档](https://cursor.com/docs/hooks)  
> - [Claude Code Hooks 官方指南](https://code.claude.com/docs/en/hooks)

## 为什么不用...

| 工具 | 为什么不够 |
|------|----------|
| **Git commit message** | 没有结构。没有冲突检测。你得逐条阅读每个 commit message 才能找到决策。 |
| **PR 描述** | 太晚了。在 vibe coding 中，你会提交几十次。决策在会话中途做出，不是在写 PR 时。 |
| **Notion / Confluence** | 与代码完全脱节。你的 AI 读不了你的 Notion 页面。手动同步一开始就会失效。 |
| **ADR（架构决策记录）** | 纯手动。在快节奏的 vibe coding 会话中没人会写 ADR。无自动化，无冲突检测。 |
| **CrewAI / AutoGen / LangGraph** | 这些工具编排*多个 AI Agent*。TeamVibe 协调*多个人类，每人各带一个 AI*。完全不同的问题。 |

## 参与贡献

欢迎贡献！有意思的方向：

- 支持更多 AI 编程工具（Windsurf、Copilot 等）
- 决策可视化 Web 面板
- 决策分析与团队洞察
- 非 IDE 场景的 CLI 工具

## 许可证

MIT
