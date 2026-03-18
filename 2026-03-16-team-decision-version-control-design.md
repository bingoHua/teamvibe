# 团队 Vibe Coding 上下文版本管理工具

日期: 2026-03-16

## 概述

基于 Cursor Hooks + Rules 能力，实现一套纯 Git 分发的团队决策版本管理系统。解决多人（产品经理、交互设计师、开发工程师）使用不同 vibe coding 工具进行篝火模式协作时的上下文同步问题。

核心理念：**管理的不是代码变更，而是业务决策链路**。

### 问题域

| # | 问题 | 严重度 |
|---|------|--------|
| 1 | 多人 vibe coding 时业务决策散落在各自对话中，无法追溯 | 高 |
| 2 | 无法区分 AI 自主决策和人工明确决策，导致协作冲突 | 高 |
| 3 | 大的需求变更缺乏背景说明，其他成员无法理解变更动机 | 中 |
| 4 | Pull 代码后不知道其他人做了什么业务变更 | 中 |
| 5 | 修改代码时可能无意推翻其他成员的明确决策 | 高 |
| 6 | AI 上下文窗口折叠后，对话早期的决策信息丢失 | 中 |

### 设计选型

| 维度 | 决策 |
|------|------|
| 变更说明生成方式 | AI 从对话+diff 自动提取，人审阅确认 |
| 决策归属标注 | AI 推断（human/human_ai/ai）+ 人可修正 |
| 大变更判定 | 基于业务语义分析（非代码量） |
| 存储分发 | 纯 Git，结构化 JSON 文件随仓库走 |
| 工具优先级 | Cursor 优先，兼容 Claude Code 扩展 |
| Pull 后呈现 | 直接在对话中展示，不阻拦 |
| 冲突检测时机 | 每次用户描述需求时实时校验 |
| 架构方案 | Rules 引导 + Hooks 守门（方案 B） |
| 草稿维护 | Rule 驱动实时渐进式草稿 |

## 一、数据层

### 目录结构

```
project-root/
├── .cursor/
│   ├── hooks.json                        # Cursor hooks 配置
│   ├── hooks/
│   │   ├── pre-commit-decision.py        # Commit 守门
│   │   ├── pre-push-validate.py          # Push 守门
│   │   ├── post-pull-review.py           # Pull 后呈现变更
│   │   ├── session-init.py               # 会话启动注入上下文
│   │   └── pre-compact-reminder.py       # 上下文折叠提醒
│   └── rules/
│       └── teamwork-decisions.mdc        # AI 冲突检测 + 草稿维护行为规则
├── .teamwork/                            # 工具无关的决策数据（随 Git 分发）
│   ├── config.json                       # 团队配置
│   ├── decisions/                        # 正式决策记录（已提交，随 Git 走）
│   │   ├── 2026-03-16T1430_alice_a1b2c3.json
│   │   └── 2026-03-16T1500_bob_d4e5f6.json
│   └── drafts/                           # 决策草稿（本地工作文件，gitignored）
│       └── current.json
```

`.cursor/` 放 Cursor 特有配置，`.teamwork/` 放工具无关的业务数据。未来 Claude Code 用户只需在自己的 hooks 目录写脚本读取 `.teamwork/decisions/` 即可接入。

### 决策记录格式

每次 commit 生成一个 JSON 文件，存放在 `.teamwork/decisions/`：

```json
{
  "id": "2026-03-16T1430_alice_a1b2c3",
  "version": "1.0",
  "author": {
    "name": "Alice",
    "role": "产品经理"
  },
  "timestamp": "2026-03-16T14:30:00Z",
  "commit_hash": "a1b2c3d4",
  "is_major_change": true,
  "change_background": "用户反馈CSV导入超过1万行时无任何进度提示，多次投诉",

  "entries": [
    {
      "decision_key": "csv-import-progress-display",
      "status": "active",

      "requirement": {
        "what": "支持CSV批量导入时的进度展示",
        "why": "用户导入大文件时不知道进度，体验差，导致重复提交"
      },
      "solution": {
        "approach": "采用流式解析 + 进度条方案，分批处理",
        "not_included": "本次不做导入速度优化，只解决进度可见性"
      },
      "decision": {
        "type": "human",
        "motivation": "产品经理明确要求必须有进度条，参考了竞品做法",
        "decided_by": "Alice"
      }
    },
    {
      "decision_key": "csv-import-error-recovery",
      "status": "active",

      "requirement": {
        "what": "导入失败时支持断点续传",
        "why": "大文件导入中断后需要重头开始，浪费用户时间"
      },
      "solution": {
        "approach": "记录已处理行数，失败后从断点继续",
        "not_included": "不支持跨会话的断点恢复"
      },
      "decision": {
        "type": "human_ai",
        "motivation": "用户提出需要容错能力，AI建议断点续传方案，用户认可采纳",
        "decided_by": "Alice"
      }
    }
  ],

  "affected_modules": ["csv-parser", "upload-ui"],
  "noise_excluded": "过滤了2个AI类型推断错误的修复和1次需求理解偏差的回退"
}
```

### 决策版本链（supersedes 机制）

同一个业务决策点通过 `decision_key` 关联，形成版本链。新记录通过 `supersedes` 字段声明替代旧决策，不需要修改旧文件（避免 Git 合并冲突）：

```json
{
  "id": "2026-03-16T1500_bob_c3d4e5",
  "author": { "name": "Bob", "role": "前端工程师" },
  "timestamp": "2026-03-16T15:00:00Z",
  "commit_hash": "c3d4e5f6",
  "is_major_change": false,

  "entries": [
    {
      "decision_key": "csv-import-progress-display",
      "status": "active",

      "requirement": {
        "what": "CSV导入时的进度展示方式",
        "why": "用户导入大文件时需要感知进度"
      },
      "solution": {
        "approach": "百分比数字 + 预估剩余时间的组合展示",
        "not_included": "不做进度条动画效果"
      },
      "decision": {
        "type": "human",
        "motivation": "与Alice沟通后达成共识，纯进度条不够直观，加上时间预估更实用",
        "decided_by": "Bob"
      },

      "supersedes": {
        "record_id": "2026-03-16T1430_alice_a1b2c3",
        "decision_key": "csv-import-progress-display",
        "reason": "团队沟通后优化方案",
        "resolution_type": "discussed_and_agreed"
      }
    }
  ]
}
```

版本链示意：

```
csv-import-progress-display:
  v1 [Alice, 03-15] "用进度条"              ← human 决策
  v2 [Bob, 03-16]   "百分比+预估时间"        ← supersedes v1, discussed_and_agreed
  v3 [Carol, 03-18] "进度条+百分比+预估时间"  ← supersedes v2, better_solution
                     ↑ 当前 active 版本
```

Rule 冲突检测只看每个 `decision_key` 的最新 active 版本。

### resolution_type 枚举

描述旧决策被替代的原因（记录在 `supersedes` 字段中）：

| 值 | 含义 |
|---|------|
| `discussed_and_agreed` | 与原决策者沟通后达成共识 |
| `requirement_changed` | 业务需求本身发生了变化 |
| `better_solution` | 找到了更优的解决方案 |

注意区分：`resolution_type` 描述"如何替代旧决策"，而 Rule 中的冲突处理逻辑（block/warn/proceed）描述"遇到冲突时的行为策略"，两者是不同维度。

### config.json（团队配置）

```json
{
  "version": "1.0",
  "team_members": [
    { "name": "Alice", "role": "产品经理", "email": "alice@team.com" },
    { "name": "Bob", "role": "前端工程师", "email": "bob@team.com" },
    { "name": "Carol", "role": "交互设计师", "email": "carol@team.com" }
  ],
  "major_change_keywords": ["新增模块", "删除功能", "数据结构变更", "API接口变更"]
}
```

### author 解析规则

决策记录中的 `author` 按以下优先级确定：
1. `CURSOR_USER_EMAIL` 环境变量（Cursor 自动提供）与 `config.json` 的 `team_members[].email` 匹配
2. `git config user.email` 作为 fallback
3. 均无法匹配时，使用 `git config user.name` 作为 author.name，role 留空

### 首次使用 / 空仓库

各 Hook 在 `.teamwork/` 目录不存在时的行为：
- `session-init.py`：返回空上下文，不报错
- `pre-commit-decision.py`：自动创建 `.teamwork/` 目录结构和空 `config.json`，然后正常执行
- `post-pull-review.py`：无决策文件可读，返回空 `additional_context`
- `pre-push-validate.py`：无决策文件视为通过

团队初始化推荐流程：项目负责人手动创建 `.teamwork/config.json` 填写成员列表后提交。

### 字段速查

| 字段 | 用途 |
|------|------|
| `decision_key` | 业务决策点的稳定标识，同一件事的所有版本共享同一个 key |
| `status` | `active` / `deprecated`。被 supersede 的记录文件本身不修改，由 supersedes 引用链推导 |
| `requirement.what` / `requirement.why` | 需求：做什么、为什么做（纯业务） |
| `solution.approach` / `solution.not_included` | 解决方案：怎么做、不做什么（非代码层面） |
| `decision.type` | `human` / `human_ai` / `ai` 三种归属 |
| `decision.motivation` | 为什么选这个方案 |
| `supersedes` | 指向被替代的旧决策，含替代原因和类型 |
| `is_major_change` + `change_background` | 大变更时强制填写的背景说明 |
| `noise_excluded` | 过滤掉的噪声（bug修复、AI误解纠正等） |

## 二、Hooks 守门层

5 个 hook 脚本，各司其职。

### hooks.json 配置

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "command": "python3 .cursor/hooks/session-init.py",
        "timeout": 10
      }
    ],
    "beforeShellExecution": [
      {
        "command": "python3 .cursor/hooks/pre-commit-decision.py",
        "matcher": "git commit",
        "timeout": 30
      },
      {
        "command": "python3 .cursor/hooks/pre-push-validate.py",
        "matcher": "git push",
        "timeout": 15
      }
    ],
    "postToolUse": [
      {
        "command": "python3 .cursor/hooks/post-pull-review.py",
        "matcher": "Shell",
        "timeout": 15
      }
    ],
    "preCompact": [
      {
        "command": "python3 .cursor/hooks/pre-compact-reminder.py",
        "timeout": 5
      }
    ]
  }
}
```

### 各 Hook 职责

| Hook | 触发时机 | 职责 | 拦截方式 |
|------|---------|------|---------|
| `session-init.py` | 新建 Cursor 会话 | 读取最近决策记录，注入为会话上下文 | 不拦截，注入 `additional_context` |
| `pre-commit-decision.py` | 执行 `git commit` 前 | 拦截提交，要求 AI 生成/定稿决策记录 | 硬拦截，deny → 定稿 → 确认 → 放行 |
| `pre-push-validate.py` | 执行 `git push` 前 | 校验决策记录完整性；大变更必须有背景 | 硬拦截，不完整则 deny |
| `post-pull-review.py` | `git pull` 完成后 | 读取新增决策记录，格式化后注入对话 | 不拦截，注入 `additional_context` |
| `pre-compact-reminder.py` | 上下文即将折叠 | 提醒确认决策草稿已更新 | 不拦截，显示 `user_message` |

### session-init.py

```
输入：sessionStart 标准字段（session_id, workspace_roots 等）
逻辑：
  1. 读取 .teamwork/decisions/ 下最近 10 条记录
  2. 对每个 decision_key 取最新 active 版本
  3. 读取 .teamwork/config.json 获取团队成员列表
  4. 格式化为摘要文本
输出：
  {
    "additional_context": "## 团队决策上下文\n\n当前 active 决策：\n...",
    "env": { "TEAMWORK_DIR": ".teamwork" }
  }
```

### pre-commit-decision.py

```
输入：beforeShellExecution 标准字段（command, cwd, conversation_id 等）
逻辑：
  1. 检查 commit 消息是否含 [skip decision]
  2. 检查 /tmp/cursor-hooks/decision-{conversation_id} flag 文件
  3. 若首次拦截：获取 staged diff，设置 flag，deny + agent_message
agent_message 指令：
  1. 读取 .teamwork/drafts/current.json（若存在，作为草稿基础）
  2. 结合 staged diff 增补/修正
  3. 读取已有 decisions/ 匹配 decision_key，需要 supersede 则标注
  4. 判断 is_major_change
  5. 呈现给用户审阅确认
  6. 保存到 .teamwork/decisions/{timestamp}_{author}_{hash}.json
  7. 删除 drafts/current.json
  8. git add 决策文件，重新执行 commit
输出（首次拦截）：
  { "permission": "deny", "agent_message": "..." }
输出（已有 flag 或 skip）：
  { "permission": "allow" }
```

### pre-push-validate.py

```
输入：beforeShellExecution 标准字段
逻辑：
  1. git log origin/HEAD..HEAD 获取待推送 commit 列表
  2. 扫描 commit 中是否包含 .teamwork/decisions/ 文件变更
  3. 对没有 [skip decision] 标记的 commit，检查是否有对应决策记录
  4. 读取决策记录，检查 is_major_change 的记录是否有 change_background
  5. 用关键词 + diff 范围做基础判断：疑似大变更但未标记的也拦截
输出（不完整）：
  {
    "permission": "deny",
    "agent_message": "推送被拦截：检测到大变更缺少背景说明...",
    "user_message": "检测到重大变更未填写背景说明，请补充后再推送"
  }
输出（完整）：
  { "permission": "allow" }
```

### post-pull-review.py

```
输入：postToolUse 标准字段（tool_name, tool_input, tool_output 等）
逻辑：
  1. 解析 tool_input（JSON 对象），检查 tool_input.command 是否匹配
     git pull 的各种形式（git pull / git pull origin main / git pull --rebase 等），
     使用正则 `^git\s+pull` 匹配。非 git pull 命令输出 {} 退出
  2. git log ORIG_HEAD..HEAD --name-only 获取新增 commit
  3. 筛选 .teamwork/decisions/*.json 文件
  4. 读取并排序：is_major_change 优先 → human 决策优先 → 时间倒序
  5. 格式化为分层文本
输出：
  {
    "additional_context": "## 团队变更通报\n\n### 🔴 重大变更\n...\n### 🟡 一般变更\n..."
  }
```

呈现格式示意：

```
## 团队变更通报（自你上次 pull 以来）

### 🔴 重大变更（1 条）
1. [Alice / 产品经理] 2026-03-16 14:30
   需求：支持CSV批量导入时的进度展示
   方案：百分比 + 预估剩余时间的组合展示
   决策类型：👤 人工决策
   动机：参考竞品后产品经理明确要求
   背景：用户反馈CSV导入超过1万行时无任何进度提示

### 🟡 一般变更（2 条）
2. [Bob / 前端工程师] 2026-03-16 15:00
   需求：导入失败时支持断点续传
   方案：记录已处理行数，失败后从断点继续
   决策类型：👥 人+AI共同决策
   ↳ 替代了 Alice 之前的方案

### ⚠️ 与你相关的决策提醒
- Alice 明确决定了「进度展示方案」，修改此功能需先与她沟通
```

### pre-compact-reminder.py

```
输入：preCompact 标准字段（trigger, context_usage_percent 等）
逻辑：
  1. 检查 .teamwork/drafts/current.json 是否存在
  2. 如存在，获取最后修改时间
输出：
  {
    "user_message": "上下文即将折叠。决策草稿最后更新于 X 分钟前，请确认重要决策已记录。"
  }
```

### 防死循环机制

```
/tmp/cursor-hooks/
└── decision-{conversation_id}        # pre-commit flag
```

pre-commit 首次拦截时创建 flag，AI 完成决策记录后重新执行 commit 时检测到 flag 即放行并删除。pre-push 无需 flag 机制——它只做校验，不触发 Agent 自动重试。

## 三、Rules AI 引导层

`.cursor/rules/teamwork-decisions.mdc` 是系统的核心行为引导，让 AI 天然具备团队决策意识。

### 完整 Rule 内容

```markdown
---
description: 团队协作决策冲突检测与草稿维护
globs:
alwaysApply: true
---

# 团队协作决策管理

你正在一个多人协作的 vibe coding 项目中工作。

## 一、冲突检测（每次用户描述需求时）

每当用户描述一个新的需求、修改请求或功能变更时：

1. 读取 `.teamwork/decisions/` 目录下的所有 JSON 文件
2. 对每个 `decision_key`，找到最新的 active 版本
   （被更新记录 supersedes 的自动排除）
3. 将用户请求与 active 决策做业务语义比对
4. 读取 `.teamwork/config.json` 中的 `team_members` 确定当前用户身份

### `human` 类型冲突 → 必须线下沟通

当用户请求与其他成员的人工明确决策冲突时：
- 立即告知：说明哪位成员在何时做了什么决策
- 引用动机：展示 `decision.motivation`
- 要求沟通：明确要求用户先与该成员线下确认
- 等待确认：用户必须回复"已沟通，可以继续"后才执行
- 不可跳过：即使你判断当前方案更优，也不能绕过

### `human_ai` 类型冲突 → 提醒，当前用户判断为准

- 告知用户存在冲突，简要展示原决策
- 说明这是人和 AI 共同做出的决策
- 当前用户可直接决定是否覆盖

### `ai` 类型冲突 → 简要提及，直接执行

- 一句话提及存在旧的 AI 决策
- 直接按当前用户要求执行

### 无冲突

正常执行，不做任何额外提示。
不要在没有冲突时主动告知"我检查了决策清单没有发现冲突"。

## 二、实时决策草稿维护

在整个对话过程中，维护决策草稿 `.teamwork/drafts/current.json`。

### 何时更新

1. 用户做出明确的业务决策（如"我们用方案A"）
2. 用户与你讨论后达成共识
3. 你自主做出了设计决策（用户未明确指定，你选择了方案）
4. 收到上下文折叠通知时，确认草稿完整性

### 何时不更新

- 纯代码 bug 修复、类型错误修正
- AI 理解偏差后的纠正
- 纯重构、格式化

### 草稿格式

与正式决策记录格式一致，标记为 draft：
{
  "status": "draft",
  "session_id": "...",
  "entries": [ ... ]
}

### 更新方式

追加式更新：新 entry 追加到 entries 数组，不重写整个文件。

## 三、文件位置

- 决策记录：`.teamwork/decisions/*.json`
- 决策草稿：`.teamwork/drafts/current.json`
- 团队配置：`.teamwork/config.json`
```

### 为什么用 Rules 做冲突检测

| 维度 | Hook（`beforeSubmitPrompt`） | Rule（`.cursor/rules/`） |
|------|---------------------------|------------------------|
| 理解能力 | Python 脚本做字符串匹配 | AI 做语义级业务比对 |
| 交互体验 | 只能 block/allow | 自然语言解释冲突、引导沟通 |
| 上下文 | 只拿到当前消息 | 拥有完整对话上下文 |
| 性能 | 每条消息启动进程 | 零额外进程 |
| 灵活性 | 改脚本 | 改文字描述 |

## 四、Push 工作流（端到端）

### 时序

```
对话过程中：
  Rule 驱动 AI 实时维护 .teamwork/drafts/current.json
  用户做决策 / 达成共识 → 追加 entry
  preCompact 触发 → 提醒确认草稿
         │
         ▼
用户/AI 执行 git commit -m "xxx"
         │
         ▼
  pre-commit-decision.py 拦截
  ├─ flag 存在 → 放行
  └─ flag 不存在 → deny + agent_message
         │
         ▼
  AI 执行 agent_message 指令：
  1. 读取 drafts/current.json（草稿已有大部分内容）
  2. 结合 staged diff 增补/修正
  3. 读取已有 decisions/，匹配 decision_key，标注 supersedes
  4. 判断 is_major_change
  5. 呈现给用户审阅
         │
         ▼
  用户审阅决策记录：
  - 可修改需求/方案描述、决策类型、大变更标记等
  - 全部是噪声 → 确认"无业务变更"，加 [skip decision]
         │
         ▼
  AI 保存正式记录、删除草稿、git add、重新 commit
         │
         ▼
用户/AI 执行 git push
         │
         ▼
  pre-push-validate.py 校验
  ├─ 完整 → 放行
  └─ 大变更缺背景 → deny，要求补充
```

### decision_key 匹配逻辑

AI 生成决策记录时，读取已有的 `.teamwork/decisions/` 文件，提取所有 `decision_key`：
- 已有 `csv-import-progress-display`，本次又改了导入进度 → 复用 key，加 `supersedes`
- 全新的"导出为 Excel"功能 → 创建新 key `csv-export-excel`

由 AI 语义理解完成，不是字符串匹配。

### "全是噪声"的快速通道

如果本次提交全是 bug 修复、AI 纠错、重构，用户确认后不生成决策文件，commit 消息加 `[skip decision]`。hook 检测到此标记直接放行。

## 五、Pull 工作流（端到端）

### 时序

```
用户/AI 执行 git pull
         │
         ▼
  Git 完成拉取，ORIG_HEAD 记录拉取前位置
         │
         ▼
  postToolUse hook 触发 post-pull-review.py
  1. 检查命令是否为 git pull
  2. git log ORIG_HEAD..HEAD --name-only
  3. 筛选 .teamwork/decisions/*.json
  4. 读取、排序（重要度 + 时间）
  5. 格式化为分层文本
         │
         ▼
  通过 additional_context 注入对话
  AI 直接呈现给用户
```

### 排序规则

1. `is_major_change` 优先
2. `human` 决策优先于 `human_ai` 优先于 `ai`
3. 同级别按时间倒序

## 六、冲突检测工作流（端到端）

### 时序

```
用户在 Cursor 中输入需求描述
         │
         ▼
  AI 按 Rule 自动检查：
  1. 读取 decisions/，取每个 decision_key 最新 active 版本
  2. 语义比对用户请求 vs active 决策
  3. 判定冲突类型
         │
         ├─ human 冲突 → 拦截，要求线下沟通 → 等待"已沟通"确认
         ├─ human_ai 冲突 → 提醒，用户自行判断
         ├─ ai 冲突 → 简要提及，直接执行
         └─ 无冲突 → 静默执行
         │
         ▼
  执行后，AI 更新 drafts/current.json：
  - 如有 supersede → 记录 supersedes 信息
  - 如为新决策 → 创建新 entry
```

### 冲突解决后的决策链更新

当冲突通过线下沟通解决后，新的决策记录自动通过 `supersedes` 替代旧版。后续任何人再涉及同一 `decision_key` 时，Rule 只看最新版本，不会重复触发已解决的冲突。

## 七、实时渐进式草稿

### 问题

如果仅在 commit 时生成决策记录，AI 上下文窗口可能已折叠，丢失对话早期的决策信息。

### 方案

Rule 驱动 AI 在对话过程中实时维护 `.teamwork/drafts/current.json`：

- 用户做出业务决策 → 立即追加 entry
- 用户与 AI 达成共识 → 立即追加 entry
- AI 自主做设计决策 → 立即追加 entry
- 上下文折叠前 → preCompact hook 提醒确认草稿

### 草稿生命周期

```
会话开始
  → AI 创建/读取 drafts/current.json
  → 对话中实时追加 entries
  → preCompact 时确认完整性
  → commit 时：草稿作为基础 → 增补修正 → 定稿 → 转为正式记录
  → 正式记录保存后删除草稿
```

### 与 commit 时定稿的对比

| 维度 | 仅 commit 时生成 | 实时渐进 + commit 时定稿 |
|------|-----------------|------------------------|
| 信息完整性 | 受上下文窗口限制 | 实时捕获，不受折叠影响 |
| commit 时开销 | 需要大量回忆和分析 | 草稿已就绪，增补即可 |
| 决策动机记录 | 事后重构，可能不准确 | 当场记录，鲜活准确 |
| 额外进程开销 | 无 | 无（Rule 驱动） |

## 八、文件清单

| 文件路径 | 类型 | 用途 |
|---------|------|------|
| `.cursor/hooks.json` | 配置 | Hooks 注册 |
| `.cursor/hooks/session-init.py` | 脚本 | 会话启动注入 |
| `.cursor/hooks/pre-commit-decision.py` | 脚本 | Commit 守门 |
| `.cursor/hooks/pre-push-validate.py` | 脚本 | Push 守门 |
| `.cursor/hooks/post-pull-review.py` | 脚本 | Pull 变更呈现 |
| `.cursor/hooks/pre-compact-reminder.py` | 脚本 | 折叠提醒 |
| `.cursor/rules/teamwork-decisions.mdc` | Rule | AI 行为引导 |
| `.teamwork/config.json` | 配置 | 团队设置 |
| `.teamwork/decisions/*.json` | 数据 | 正式决策记录 |
| `.teamwork/drafts/current.json` | 数据 | 草稿（gitignored） |

## 九、实现顺序

1. **数据层基础**：创建 `.teamwork/` 目录结构、config.json、.gitignore
2. **Rule 层**：`teamwork-decisions.mdc`（冲突检测 + 草稿维护）
3. **session-init.py**：会话启动注入
4. **pre-commit-decision.py**：Commit 守门（替换现有 pre-commit-changelog.py）
5. **pre-compact-reminder.py**：折叠提醒
6. **post-pull-review.py**：Pull 变更呈现
7. **pre-push-validate.py**：Push 校验
8. **hooks.json 更新**：注册所有 hooks
9. **端到端测试**：模拟多人协作场景验证
