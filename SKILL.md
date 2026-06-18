---
name: skill-provenance-tracer
description: >
  追溯一个 Skill 的形成链路与真相源。当用户问"这个 skill 是怎么来的"、
  "找一下 X skill 的真相源"、"还原这个 skill 的形成过程"、
  "对比一下两次检索为什么速度不一样"时触发。
  核心原则：先 List + Read 确定性资源（文件、git log），再补不确定资源
  （会话搜索、JSONL 解析、用户口述）。禁止上来就用搜索类工具。
  适用于跨 Agent 环境（Claude Code、WorkBuddy、Alice、其他）追溯 skill 形成链路。
  不适用于审查 skill 质量（用 skill-auditor）或精简 token（用 skill-slimmer）。
---

# Skill 真相源链路追踪器

## 你做什么

给定一个 Skill 名字或路径，还原它的形成链路：起源 → 创建 → 迭代 → 定型。每个节点标注证据等级，区分"实证"和"推断"。多 Agent 环境下，按优先级逐层扒，不要上来就走最贵的路。

## 核心洞察

**"找形成链路"有两种完全不同的任务形态，工具选择差一个数量级：**

| 形态 | 数据在哪 | 主力工具 | 速度 |
|------|---------|---------|------|
| **文件引用链** | SKILL.md / README / references / 脚本里互相引用 | List + Read | 快（直线） |
| **会话形成链** | 散落在 Agent 会话历史里，无索引 | 会话搜索 / JSONL 解析 | 慢（网状试错） |

大部分"找真相源"任务是第一种——SKILL.md 写了 origin story，README 写了设计动机，git log 写了迭代历史，references/ 存了引用的原始资料。顺着这些文件读一遍，链路就出来了。

只有当文件里没有、必须还原"当时对话是怎么决策的"时，才进入第二种。

### Before / After 对比（为什么工具选择差一个数量级）

**慢路（新手本能，本次踩坑实录）**：
```
conversation_search → 空 → conversation_search → 空 → conversation_search → 空
→ Bash ls 找不存在的目录 → Grep 试 pattern ×3 → Python 探 JSONL 结构
→ Python 提 user → Python 提 assistant → 34 步才出结果
```

**快路（Step 1 优先，Alice 那次的路径）**：
```
List skill 目录 → Read SKILL.md → Read README → git log
→ 顺着引用读 references/ → 28 步出结果，零失败
```

差别不在数据，在**先走确定性还是先走搜索**。

## 执行流程（必须按顺序）

### Step 0：判断任务形态

问自己：**用户要找的真相源，最可能写在文件里，还是只能从对话里还原？**

- 用户说"这个 skill 引用了哪些资料" → 文件引用链，走 Step 1
- 用户说"这个 skill 是怎么被设计出来的" → 先试 Step 1，不够再走 Step 2
- 用户说"还原当时讨论的决策过程" → 必然要走 Step 2，但仍要先做 Step 1 拿背景

**无论如何 Step 1 都要做。** 它便宜、确定性高、能给 Step 2 提供线索。

### Step 1：确定性资源优先（List + Read，永远从这里开始）

```
1. List skill 目录       → 看有哪些文件
2. Read SKILL.md        → 拿 description、origin story、设计原则
3. Read README.md       → 拿面向人的项目故事（通常比 SKILL.md 更详细）
4. git log --oneline    → 拿提交历史（每次 commit 是一个形成节点）
5. List references/     → 看引用了哪些外部资料
6. Read references/ 下相关文件 → 拿引用链上游
```

**关键判断**：如果 Step 1-4 已经把形成链路讲清楚了，**停**。多余的会话解析是 token 黑洞。

### Step 2：会话搜索（只在 Step 1 不够时）

#### 2a. 先判断会话在哪个 Agent 里

| Agent | 会话存哪 | 用什么工具搜 | 何时可用 |
|-------|---------|-------------|---------|
| WorkBuddy | 服务器索引 | `conversation_search` | 始终可用，但只搜 WorkBuddy 内对话 |
| Claude Code | `~/.claude/projects/<encoded-cwd>/*.jsonl` | `Grep` 扫文件 + Python 解析 | 文件存在就能扫，但要 `dangerouslyDisableSandbox` |
| Alice | Alice 项目内文件 | List + Read（同 Step 1） | 通常不需要会话搜索，文件里就有 |
| 其他 Agent | 不确定 | 先问用户会话存在哪 | —— |

#### 2b. WorkBuddy 会话搜索（conversation_search）

```
触发条件：用户在 WorkBuddy 里聊过这个 skill 的设计
查询写法：自包含——重述当前任务 + 指定要找什么 + 为什么需要
limit：先 5-10 条，不够再加
```

#### 2c. Claude Code 会话搜索（Grep + Python）

```bash
# 第 1 步：定位命中文件（一次搞定，不要多轮试 pattern）
Grep pattern="skill名|核心关键词" path="~/.claude/projects" output_mode="count"

# 第 2 步：按时间戳排序，挑命中最多的文件
# 第 3 步：Python 脚本提取 user + assistant 消息（一次提取完，不分轮）
```

**Python 脚本权威性问题**：
- JSONL 每行一个 JSON 对象，`type` 字段区分 `user`/`assistant`/`mode`/`permission-mode`/`tool_result`
- user 类型里混着真实发言、命令输出、task-notification——**必须过滤**，不能全当用户发言
- 消息可能被 `/compact` 压缩过，line 号不稳定，**不可作为引用锚点**
- 脚本提取的内容**只是会话原文，不是元数据**——commit hash、提交时间这些**不在 .jsonl 里**，必须单独跑 `git log` 验证

### Step 3：交叉验证 + 证据等级标注

每个形成节点都要标清楚证据来源：

| 证据等级 | 标记 | 含义 |
|---------|------|------|
| A | `[文件原文]` | 直接读到的文件内容（SKILL.md / README / references） |
| A | `[git log]` | git 提交记录，时间 + hash 可验证 |
| B | `[会话原文]` | 从 .jsonl 或 conversation_search 提取的对话原文 |
| C | `[推断]` | 从上下文推断的，未直接验证 |
| D | `[补的]` | 为了图完整而补的，没有证据 |

**禁止把 C/D 级证据画得像 A 级**。链路图里混了推断和实证，必须用标注区分，否则看的人会误以为全是查过的。

### Step 4：输出

- 形成链路图（SVG / 时间线），每个节点标证据等级
- 关键决策节点列表（附会话原文片段或 commit message）
- 三层真相源定位（文件 / git / 会话）
- token 消耗复盘（可选，只在用户问"为什么这么慢"或"对比速度"时附）

## Gotchas（踩过的坑，必须记住）

### 1. conversation_search 不是万能搜索

它**只搜 WorkBuddy 这个环境里的对话**。Claude Code、Alice、其他 Agent 的会话都不在索引里。

**Before**：在 WorkBuddy 里用 conversation_search 找 Claude Code 会话 → 3 次全空 → 浪费 3 步
**After**：第一次空就停，判断数据在哪个 Agent，换 Grep 扫 `~/.claude/projects/`

**判断方法**：问自己"这个讨论是在哪个 App 里发生的？"——在 WorkBuddy 聊的能搜，在 Claude Code 聊的不能。

### 2. 上来就搜索是最贵的路

**Before**：接到"找真相源"→ 立刻 conversation_search 或 Grep → 方向可能完全错 → 前 10 步全空
**After**：先 List skill 目录 → Read SKILL.md → Read README → git log → 这 4 步通常拿 70% 信息，零失败

### 3. Python 解析 JSONL 的半权威性

从 .jsonl 提取的会话原文**是事实**，但从原文里推断出的元数据（commit 数、时间、hash）**不是事实**——除非你单独跑了 git log 验证。

**Before**：从 assistant 消息文本推断"10 次提交，5/30 起源"→ 画进图 → 碰巧对了但没验证
**After**：会话只提取对话原文，元数据一律跑 `git log` 单独验证，图里标 `[git log]` 而非 `[会话原文]`

### 4. 多 Agent 形成链路

一个 skill 的形成可能分散在多个 Agent 里：
- Alice 里讨论设计原则
- Claude Code 里写代码 + 提交 git
- WorkBuddy 里审查 + 迭代

**每个 Agent 用对应工具扒**，不要指望一个工具搜全。先问用户（或从 SKILL.md 的致谢/changelog 里推断）形成过程跨了哪些 Agent。

### 5. 截断会丢上下文

Python 提取会话消息时，如果每条只取前 500 字符，长消息的关键决策上下文会被砍掉。**要么读全文，要么标注"已截断"**。

### 6. line 号不可引用

JSONL 的 line 号会因 `/compact` 操作变化。引用会话内容时用**时间戳 + 消息前 50 字**做锚点，不要用 line 号。

### 7. 会话内容含敏感信息

`~/.claude/projects/*.jsonl` 和 conversation_search 结果里可能有 API key、私人对话、密码、内部链接。

**Before**：提取会话原文直接贴进链路图 → 密钥泄露到输出里
**After**：提取后扫一遍敏感字段（key/token/password/secret），命中的替换成 `[REDACTED]` 再输出

## 反模式

| 反模式 | 为什么错 | 正确做法 |
|--------|---------|---------|
| 上来 conversation_search | 可能方向完全错（数据不在索引） | 先 List + Read 确定性资源 |
| 同一 pattern 搜 3 次 | 空结果换 query 还是空 | 第一次空就换工具 |
| Python 脚本分多轮提取 | 每轮都付解析成本 | 一次脚本提取 user + assistant 全部 |
| 只读 .jsonl 不跑 git log | 元数据不在会话里 | 会话 + git 交叉验证 |

（其余反模式已并入 Gotcha 1/2/3 的 Before/After，不重复列。）

## 跨 Agent 扩展点

未来遇到新 Agent 环境，按这个模板补充：

```
### Agent X 会话搜索
- 数据位置：
- 主力工具：
- 搜索语法：
- 权限要求：
- 何时用：
```

已知环境见 Step 2a 的表格。新环境第一次遇到时，先问用户"这个 Agent 的会话存在哪"，再决定用什么工具扒。

## 不适用

- 审查 skill 质量 → 用 skill-auditor
- 精简 skill token → 用 skill-slimmer
- 从经验提取 skill 约束 → 用 skill-extractor
- 从零创建 skill → 用 skill-creator
- 发布 skill 到 GitHub → 用 skill-publisher

这个 skill 只管"还原形成链路 + 定位真相源"，不管 skill 本身的好坏。
