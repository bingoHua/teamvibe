# Cursor / Claude Code 双平台 Hooks

## 设计原理

本项目的 hooks 系统支持 Cursor 和 Claude Code 双平台，采用以下设计：

### 1. 统一的脚本位置

所有 Python hook 脚本统一放在 `.cursor/hooks/` 目录下，两个平台共享同一套脚本。

### 2. 平台兼容层

`compat.py` 提供了统一的输入/输出接口：
- 自动检测当前运行平台（Cursor 或 Claude Code）
- 统一的命令提取、项目路径获取等方法
- 平台特定的输出格式化（JSON for Cursor, stdout for Claude Code）

### 3. 配置文件路径处理

**Cursor** (`.cursor/settings.json`):
```json
{
  "command": "python3 .cursor/hooks/session-init.py"
}
```

**Claude Code** (`.claude/settings.json`):
```json
{
  "command": "cd \"$CLAUDE_PROJECT_DIR\" && python3 .cursor/hooks/session-init.py"
}
```

关键差异：
- Cursor 会自动在项目根目录执行 hooks
- Claude Code 需要显式 `cd` 到项目根目录，确保相对路径正确

## Hook 列表

| Hook | 触发时机 | 功能 |
|------|---------|------|
| `session-init.py` | SessionStart | 注入团队决策上下文 |
| `pre-commit-decision.py` | PreToolUse:Bash | 检测决策冲突，生成决策记录 |
| `pre-push-validate.py` | PreToolUse:Bash | 验证 push 前的决策完整性 |
| `post-pull-review.py` | PostToolUse:Bash | pull 后提示检查决策变更 |
| `pre-compact-reminder.py` | PreCompact | 上下文压缩前提醒保存决策草稿 |

## 迁移到新项目

1. 复制 `.cursor/hooks/` 目录到新项目
2. 复制 `.cursor/settings.json` 和 `.claude/settings.json`
3. 复制 `.teamwork/` 目录结构
4. 根据团队成员更新 `.teamwork/config.json`

## 故障排查

### Claude Code 报错找不到脚本

**症状**：`can't open file '/path/to/.cursor/hooks/xxx.py': No such file or directory`

**原因**：`.claude/settings.json` 中的命令没有 `cd` 到项目根目录

**解决**：确保所有 hook 命令都以 `cd "$CLAUDE_PROJECT_DIR" &&` 开头

### Hook 输出格式错误

**症状**：Cursor 显示 "Invalid hook response"

**原因**：脚本没有使用 `compat.py` 的输出方法

**解决**：确保所有输出都通过 `HookIO` 的方法（`context()`, `deny()`, `allow()` 等）
