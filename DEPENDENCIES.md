# Dependencies

本 Skill 不依赖任何外部包或脚本。

核心工具均为 AI Agent 内置：
- `List` / `Read` / `Grep` / `Bash`（文件系统操作）
- `conversation_search`（WorkBuddy 会话搜索，仅 WorkBuddy 环境需要）
- `git`（读取提交历史，系统自带）

如果使用 Claude Code 会话搜索功能，需要 Python 3 运行时来解析 .jsonl 文件。
