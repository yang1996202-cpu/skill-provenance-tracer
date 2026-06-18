# Skill 真相源链路追踪器

> 追溯一个 Skill 的形成链路与真相源。适用于跨 Agent 环境（Claude Code、WorkBuddy、Alice、其他）。

## 解决什么问题

你做了一个 Skill，过了一段时间想知道"这东西当初是怎么来的"——哪些对话讨论的、哪些文件引用的、哪些 commit 迭代的、哪些决策点关键的。

让 AI 去找，它通常上来就搜索会话历史，结果绕一大圈，34 步才出结果，还可能把推断当事实画进链路图里。

这个 Skill 解决两个问题：
1. **效率问题**：让 AI 先走确定性资源（List + Read + git log），只在不够时才搜索会话，把 34 步压到 4 步
2. **可信度问题**：强制标注每个节点的证据等级，禁止把推断画得像实证——因为跨 Agent 追溯最大的风险不是慢，是抓错

## 适合 / 不适合

**适合**：
- 想还原一个 Skill 的形成链路（起源 → 创建 → 迭代 → 定型）
- 想找 Skill 背后的真相源（论文、报告、引用资料、设计决策）
- 在多个 AI 工具间穿梭工作，需要追溯某个产物的形成过程
- 想对比"为什么这次 AI 检索快，那次慢"

**不适合**：
- 审查 Skill 质量好不好 → 用 [skill-auditor](https://github.com/yang1996202-cpu/skill-auditor)
- 精简 Skill 的 token → 用 skill-slimmer
- 从零创建新 Skill → 用 skill-creator
- 从经验提取 Skill 约束 → 用 skill-extractor

## 核心方法：先走确定性，再走搜索

"找形成链路"有两种完全不同的任务形态，工具选择差一个数量级：

| 形态 | 数据在哪 | 主力工具 | 速度 |
|------|---------|---------|------|
| **文件引用链** | SKILL.md / README / references / 脚本互相引用 | List + Read | 快（直线） |
| **会话形成链** | 散落在 Agent 会话历史里，无索引 | 会话搜索 / JSONL 解析 | 慢（网状试错） |

大部分"找真相源"任务是第一种。SKILL.md 写了 origin story，README 写了设计动机，git log 写了迭代历史，references/ 存了引用的原始资料。顺着这些文件读一遍，链路就出来了。

只有当文件里没有、必须还原"当时对话是怎么决策的"时，才进入第二种。

### Before / After

**慢路（AI 新手本能，真实踩坑记录）**：
```
conversation_search → 空 → conversation_search → 空 → conversation_search → 空
→ Bash ls 找不存在的目录 → Grep 试 pattern ×3 → Python 探 JSONL 结构
→ Python 提 user → Python 提 assistant → 34 步才出结果
```

**快路（确定性优先）**：
```
List skill 目录 → Read SKILL.md → Read README → git log
→ 顺着引用读 references/ → 28 步出结果，零失败
```

差别不在数据，在**先走确定性还是先走搜索**。

## 跨 Agent 支持

| Agent | 会话存哪 | 用什么搜 | 何时可用 |
|-------|---------|---------|---------|
| WorkBuddy | 服务器索引 | `conversation_search` | 始终可用，但只搜 WorkBuddy 内对话 |
| Claude Code | `~/.claude/projects/<encoded-cwd>/*.jsonl` | `Grep` + Python 解析 | 文件存在就能扫 |
| Alice | Alice 项目内文件 | List + Read（通常不需要会话搜索） | 文件里就有 |
| 其他 Agent | 不确定 | 先问用户会话存在哪 | —— |

## 证据等级标注

跨 Agent 追溯最大的风险是**抓错了你不知道**。AI 从会话推断出"10 次提交"画进图，这数字根本不在会话里——它是推断的，不是事实。

每个形成节点必须标证据等级：

| 等级 | 标记 | 含义 |
|------|------|------|
| A | `[文件原文]` | 直接读到的文件内容 |
| A | `[git log]` | git 提交记录，时间 + hash 可验证 |
| B | `[会话原文]` | 从 JSONL 或 conversation_search 提取的对话 |
| C | `[推断]` | 从上下文推断的，未直接验证 |
| D | `[补的]` | 为了图完整补的，没有证据 |

**禁止把 C/D 级画得像 A 级。**

## 如何安装

```bash
# 方式 1：直接 clone 到 WorkBuddy skills 目录
git clone https://github.com/yang1996202-cpu/skill-provenance-tracer.git ~/.workbuddy/skills/skill-provenance-tracer

# 方式 2：复制到 Claude Code skills 目录
git clone https://github.com/yang1996202-cpu/skill-provenance-tracer.git ~/.claude/skills/skill-provenance-tracer
```

装好后对 AI 说"找一下 X skill 的真相源"或"还原这个 skill 的形成过程"即可触发。

## 典型提示词

- "找一下 workflow-review skill 的真相源"
- "这个 skill 是怎么来的，还原一下形成过程"
- "对比一下两次检索为什么速度不一样"
- "这个 skill 引用了哪些资料"

## 7 条踩坑规则（都在 SKILL.md 里）

1. conversation_search 不是万能搜索——第一次空就停
2. 上来就搜索是最贵的路——先 List + Read
3. Python 解析的元数据不是事实——必须 git log 验证
4. 多 Agent 形成链路——每个 Agent 用对应工具
5. 截断会丢上下文——要么读全文要么标注
6. line 号不可引用——用时间戳 + 消息前 50 字
7. 会话内容含敏感信息——输出前 REDACT

详见 [SKILL.md](./SKILL.md)。

## License

MIT
