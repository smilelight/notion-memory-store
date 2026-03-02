---
name: memory-to-notion
description: >
  Summarize and archive conversation memories to Notion. Trigger when the user says
  "总结一下记忆", "总结记忆", "summarize memory", "归档对话", "整理记忆", "同步记忆到Notion",
  "把记忆写入Notion", "回顾一下最近聊了什么并记录下来", or any variation asking to review,
  summarize, archive, or export conversation history / chat memories to Notion.
disable-model-invocation: true
user-invocable: true
---

# Memory to Notion

This skill retrieves the user's past conversation history, analyzes it for valuable and meaningful
content, decomposes conversations into **atomic memory entries**, and writes them as rows into
the **Memory Store** Notion Database.

## Configuration

Before executing any steps, read the configuration file to obtain Notion IDs:

**Config path**: `~/.config/memory-store/config.json`

Use the Read tool to read `~/.config/memory-store/config.json`. The file contains:

```json
{
  "notion": {
    "data_source_id": "<data_source_id>",
    "database_url": "<database_url>"
  }
}
```

Extract `data_source_id` and `database_url` from the config. Use these values wherever
this document references `<data_source_id>` or `<database_url>`.

**If the config file does not exist or is unreadable**, tell the user:
"Memory Store is not configured yet. Please run `/setup-memory-store` first to create the
database and generate the config."
Then stop execution.

## Target: Memory Store Database

All memories are written into the **Memory Store** database using the IDs from config:
- **Data Source ID**: `<data_source_id from config>`
- **Database URL**: `<database_url from config>`

### Schema

| Property     | Type         | Description                              |
|-------------|-------------|------------------------------------------|
| Title       | Title       | 记忆的一句话概括                            |
| Category    | Select      | Fact / Decision / Preference / Context / Pattern / Skill |
| Tags        | Multi-select| Python, Architecture, Notion, ClaudeCode, Git, Integration, Workflow, Preference, DevTools (可扩展) |
| Content     | Rich Text   | 记忆的详细内容                              |
| Source      | Select      | Claude.ai / ClaudeCode / Manual / OpenClaw / Other |
| Confidence  | Select      | High / Medium / Low                       |
| Status      | Select      | Active / Archived / Contradicted          |
| Scope       | Select      | Global / Project                          |
| Project     | Rich Text   | 项目名（Scope=Project 时填写，Global 时留空） |
| Expiry      | Select      | Never / 30d / 90d / 1y                    |
| Source Date | Date        | 原始对话发生时间                            |
| Last Used   | Date        | 最后被召回使用的时间                         |
| Memory ID   | Unique ID   | MEM-0001 格式，跨平台引用标识                |

### Category 定义

- **Fact**: 客观事实 -- 用户的技术栈、工具、环境、组织等
- **Decision**: 架构决策、技术选型、方案选择
- **Preference**: 用户偏好 -- 编码风格、工具配置、交互习惯
- **Context**: 背景信息 -- 项目上下文、行业知识、见解观察
- **Pattern**: 行为模式 -- 工作流程、重复出现的需求
- **Skill**: 技能知识 -- 学到的命令、API、技巧

## Workflow

### Step 1: Read Configuration

Use the Read tool to read `~/.config/memory-store/config.json`.
Parse the JSON and extract `notion.data_source_id` and `notion.database_url`.
If the file is missing or invalid, instruct the user to run `/setup-memory-store` and stop.

### Step 2: Gather Conversation Content

根据当前平台选择不同策略：

**Claude.ai**（有历史对话 API）：
- 使用 `recent_chats(n=20)` 获取最近对话
- 可用 `after`/`before` 参数筛选时间范围
- 可用 `conversation_search` 按关键词检索
- 综合归档时可多轮分页（最多 ~5 轮）

**Claude Code**（仅当前会话）：
- 从当前对话上下文中提取有价值的信息
- 用户在对话中说"记下来"/"总结记忆"时触发
- 回顾本次会话中产生的事实、决策、偏好等
- 无法访问历史会话，只处理当前对话内容

### Step 3: Check Existing Memories (Dedup & Conflict Detection)

Before writing new memories, search the Memory Store to avoid duplicates and detect conflicts.

Use `Notion:notion-search` with `data_source_url: "collection://<data_source_id>"` to check for:
1. **Duplicates**: Same fact already stored -> skip
2. **Updates**: Same topic but information changed -> update existing entry's Content, set old info as Contradicted if needed
3. **Conflicts**: New info contradicts existing memory -> create new entry as Active, mark old one as Contradicted

### Step 4: Decompose into Atomic Memories

Each conversation may yield 0-N memory entries. The key principle is **one fact per row**.

**Decomposition rules:**
- Each memory should be self-contained and independently meaningful
- Don't store entire conversation summaries -- extract individual facts, decisions, preferences
- Title should be a single declarative sentence (可以搜索的)
- Content provides enough detail to be useful without the original conversation

**Examples of good decomposition:**

A conversation about "setting up a new Python project" might yield:
```
MEM: "用户使用 uv 而非 pip 管理 Python 依赖"        -> Category: Preference
MEM: "项目 OpenClaw 使用 FastAPI + PostgreSQL 架构"  -> Category: Decision
MEM: "用户偏好用 Ruff 做代码格式化和 lint"            -> Category: Preference
```

**What NOT to store:**
- Transient Q&A that can be easily re-searched ("Python 的 GIL 是什么")
- Pleasantries and small talk
- Failed attempts with no useful outcome
- Information the user explicitly asked to forget

### Step 5: Write to Memory Store

Use `Notion:notion-create-pages` with `parent.data_source_id = "<data_source_id>"`.

**For each memory entry, set properties:**

```json
{
  "Title": "一句话概括",
  "Category": "Fact|Decision|Preference|Context|Pattern|Skill",
  "Tags": "[\"Tag1\", \"Tag2\"]",
  "Content": "详细的记忆内容，足够让其他 AI 平台理解和使用",
  "Source": "Claude.ai|ClaudeCode|OpenClaw|Manual|Other",
  "Confidence": "High|Medium|Low",
  "Status": "Active",
  "Scope": "Global|Project",
  "Project": "项目名（Scope=Project 时填写）",
  "Expiry": "Never|30d|90d|1y",
  "date:Source Date:start": "YYYY-MM-DD",
  "date:Source Date:is_datetime": 0
}
```

**Scope guidelines:**
- **Global**: 跨项目通用的记忆 -- 用户偏好、通用工具链、个人习惯、全局决策
- **Project**: 特定项目的记忆 -- 项目架构、项目专属配置、项目内的技术决策
- 当无法判断时，默认设为 Global（大多数偏好和事实是全局的）
- Project 字段填项目名（如 "OpenClaw"、"skills"），与用户的项目目录名或仓库名一致

**Confidence guidelines:**
- **High**: User explicitly stated this, or it's a clear fact/decision
- **Medium**: Inferred from conversation context, likely accurate
- **Low**: Speculative or based on limited evidence

**Expiry guidelines:**
- **Never**: Stable facts and preferences (name, tools, architecture)
- **1y**: Knowledge that may become outdated (tool versions, project status)
- **90d**: Contextual info with limited shelf life (current tasks, temporary decisions)
- **30d**: Very transient info (this week's todos, temporary workarounds)

**Tags:**
- Use existing tags when possible (see schema above)
- New tags are auto-created by Notion; use sparingly and consistently
- Use English tag names for cross-platform compatibility

### Step 6: Handle Conflicts

If Step 3 found conflicting memories:

1. Update the **old** memory's Status to "Contradicted" using `Notion:notion-update-page`:
   ```json
   {"command": "update_properties", "properties": {"Status": "Contradicted"}}
   ```
2. Create the **new** memory with Status "Active" (default)
3. In the new memory's Content, optionally note what it supersedes: "(更新: 之前记录为 XX)"

### Step 7: Report Results

After writing to Notion, provide the user with a summary:
- How many conversations were processed
- How many memories were created / updated / skipped
- List of new entries with their MEM IDs and Titles
- Any conflicts detected and how they were resolved
- Any conversations that might need manual review

**Example report format:**
```
Memory archival complete

Processed 8 conversations, generated 12 memories:
- New: 10
- Updated: 1 (MEM-0003 user location updated from Beijing to Shenzhen)
- Skipped: 3 low-value conversations

New memories:
| MEM# | Title | Category |
|------|-------|----------|
| MEM-0007 | ... | Fact |
| MEM-0008 | ... | Decision |
...
```

## Important Notes

- **Atomic entries**: One fact per row. Never dump a whole conversation summary into one entry.
- **Language**: Write memory Title and Content in the same language as the original conversation (typically Chinese). Tags use English.
- **Idempotent**: Always check for existing memories before writing. Running the skill twice on the same conversations should not create duplicates.
- **Source accuracy**: 根据当前平台自动设置 Source（Claude.ai -> "Claude.ai"，Claude Code -> "ClaudeCode"，OpenClaw -> "OpenClaw"）。
- **Preserve details**: Keep important code snippets, commands, config values, URLs verbatim in Content.
- **User control**: This database is the user's open memory -- don't store anything the user wouldn't want to see. When in doubt, ask.
- **Cross-platform ready**: Write Content so that any AI platform reading this memory can understand and use it without additional context.

## Example Interaction

**User**: 总结一下记忆

**Claude**:
1. Reads `~/.config/memory-store/config.json` to get `data_source_id`
2. Calls `recent_chats(n=20)` to get recent conversations (or reviews current session in Claude Code)
3. Searches Memory Store for existing entries to avoid duplicates
4. Analyzes each conversation, decomposes into atomic memories
5. Writes entries via `Notion:notion-create-pages` to the configured `data_source_id`
6. Reports: "Processed 5 conversations, generated 8 new memories, updated 1, skipped 2 low-value conversations."
