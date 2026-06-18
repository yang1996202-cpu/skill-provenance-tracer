# 示例：追溯 workflow-review skill 的形成链路

## 触发

用户问："找一下 workflow-review skill 的真相源"

## Step 1 执行（确定性优先）

```
List ~/.claude/skills/workflow-review/
→ 发现 SKILL.md, README.md, scripts/, examples/, .git/

Read SKILL.md
→ 拿到 description、origin story 提到"从 workflow-auditor 改名"

Read README.md
→ 拿到面向人的项目故事，提到起源和迭代

git log --oneline
→ 10 次提交，从 142dc43 (5/30) 到 8b43070 (6/17)

List references/
→ (此 skill 无 references 目录)

判断：Step 1-4 已经讲清楚起源和迭代，但"最近讨论的决策细节"需要会话记录
→ 进入 Step 2
```

## Step 2 执行（会话搜索）

判断会话在 Claude Code 里（用户说"在 Claude Code 内最近讨论"）：

```
Grep pattern="workflow-review|workflow.*audit" path="~/.claude/projects" output_mode="count"
→ 命中多个文件，按时间戳和命中数排序

挑出主会话 de71a917.jsonl（170 次命中，6/17 22:52）

Python 一次提取 user + assistant 消息
→ 过滤掉 tool_result / command / task-notification
→ 拿到 29 条真实用户消息
```

## Step 3 交叉验证

| 节点 | 会话原文 | git log 验证 |
|------|---------|-------------|
| "10 次提交" | assistant 消息提到 | ✅ `git rev-list --count HEAD` = 10 |
| "5/30 起源" | assistant 消息提到 | ✅ `git log` 最早 commit 142dc43 |
| "6/17 定型" | assistant 消息提到 | ✅ `git log` 最新 commit 8b43070 |
| "改名 workflow-review" | README 提到 | ✅ git log 有 commit 1c3de61 |

## Step 4 输出

### 形成链路图

```
阶段一 · 起源（5/29）[会话原文]
  ├─ home dir 首次提及 [会话原文]
  ├─ code dir 65 次命中 [会话原文]
  └─ 3 场密集讨论 [会话原文]

阶段二 · 创建（5/30–6/2）[git log]
  ├─ 5/30 142dc43 workflow-auditor v2 [git log]
  ├─ 5/30 1c3de61 改名 workflow-review [git log]
  ├─ 5/30 4480a8d Add README [git log]
  └─ 6/2 e623770 可靠性改进 [git log]

阶段三 · 定型（6/17）[git log + 会话原文]
  ├─ 6/17 5350443 内置 trace 引擎 [git log]
  └─ 6/17 8b43070 聚类阈值修复 [git log]
```

### 三层真相源

1. **会话记录**：`~/.claude/projects/-Users-yang-code/de71a917-*.jsonl`
2. **git 历史**：`~/.claude/skills/workflow-review/.git`（10 次提交）
3. **GitHub 远端**：`github.com/yang1996202-cpu/workflow-review`

## 常见失败模式

| 失败 | 原因 | 修正 |
|------|------|------|
| conversation_search 三连空 | 在 WorkBuddy 搜 Claude Code 会话，索引不互通 | 第一次空就换 Grep 扫 .jsonl |
| Python 提取把命令输出当用户发言 | user 类型混着真实发言和 tool_result | 必须按 type 字段过滤 |
| 把"10 次提交"当会话原文标 | 这数字不在会话里，是从消息推断的 | 单独跑 git log 验证，标 [git log] |
| line 号引用失效 | /compact 操作改变了 line 号 | 用时间戳 + 消息前 50 字做锚点 |
