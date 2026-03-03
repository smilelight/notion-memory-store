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

## Database Discovery

This skill uses a **zero-config convention**: the database is always named **"Memory Store"**.

**Locate the database:**

```
POST /v1/search
{
  "query": "Memory Store"
}
```

From the results, find the item with `object: "data_source"` whose title is "Memory Store".
Extract both:
- `data_source_id` -- for querying (`POST /v1/data_sources/{id}/query`)
- `database_id` -- for creating pages (`POST /v1/pages` with `parent: {"database_id": "..."}`)

- **If found** -> use `data_source_id` for queries, `database_id` for page creation.
- **If not found** -> ask the user: "No 'Memory Store' database found in your Notion workspace.
  Which page should I create it under? Please provide a Notion page URL or page ID."
  Then create the database (see Database Creation below).

### Schema

| Property    | Type        | Description                              |
|-------------|-------------|------------------------------------------|
| Title       | Title       | 记忆的一句话概括                            |
| Category    | Select      | Fact / Decision / Preference / Context / Pattern / Skill |
| Content     | Rich Text   | 记忆的详细内容                              |
| Source      | Select      | Claude.ai / ClaudeCode / Manual / OpenClaw / Other |
| Status      | Select      | Active / Archived / Contradicted          |
| Scope       | Select      | Global / Project                          |
| Project     | Rich Text   | 项目名（Scope=Project 时填写，Global 时留空） |
| Expiry      | Select      | Never / 30d / 90d / 1y                    |
| Source Date | Date        | 原始对话发生时间                            |

### Database Creation

When the database does not exist, create it under the user-specified parent page.
Use the Notion create-database API with the schema above.

### Category 定义

- **Fact**: 客观事实 -- 用户的身份背景、技术栈、工具、环境、组织等
- **Decision**: 架构决策、技术选型、方案选择
- **Preference**: 用户偏好 -- 编码风格、工具配置、交互习惯
- **Context**: 背景信息 -- 项目上下文、行业知识、见解观察
- **Pattern**: 行为模式 -- 工作流程、重复出现的需求
- **Skill**: 技能知识 -- 学到的命令、API、技巧

## Platform Adaptation

This skill describes operations using generic Notion REST API format.
Each platform's AI should translate to its available tools using the **fixed mappings** below.
Do NOT guess -- follow these mappings exactly.

### Claude Code / Claude.ai (Notion MCP Tools)

| Operation | SKILL.md Describes | Use MCP Tool | Key Parameters |
|-----------|-------------------|--------------|----------------|
| Discover database | `POST /v1/search` | `notion-search` | `query: "Memory Store"`, `content_search_mode: "workspace_search"` |
| Get IDs | -- | `notion-fetch` | Fetch the database, extract `data_source_id` from `<data-source url="collection://...">` tag |
| Dedup query | `POST /v1/data_sources/{id}/query` | **Not available** | Fall back to `notion-search` with `data_source_url` (see Step 3 note) |
| Create page | `POST /v1/pages` | `notion-create-pages` | `parent: { "data_source_id": "..." }` |
| Update page status | `PATCH /v1/pages/{id}` | `notion-update-page` | `command: "update_properties"` |
| Create database | `POST /v1/databases` | `notion-create-database` | Uses SQL DDL syntax (see Database Creation) |
| Fetch page | `GET /v1/pages/{id}` | `notion-fetch` | `id: "<page_id>"` |

**Critical notes:**
- Discovery MUST use `content_search_mode: "workspace_search"` (default `ai_search` mode may not return databases)
- Structured query for dedup is **not available** in MCP. Use semantic search as fallback:
  `notion-search` with `data_source_url: "collection://<data_source_id>"` and keywords from the candidate memory.
  Then `notion-fetch` each result to compare full properties.
- `notion-create-database` uses SQL DDL syntax, not JSON. See Database Creation section for the DDL.

### OpenClaw / Direct API Access

Use Notion REST API exactly as described in the workflow steps. All operations including
structured query for dedup are fully supported.

## Workflow

### Step 1: Discover Database

Locate the "Memory Store" database. If not found, create it (see above).

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

Before writing, query the database to check for duplicates and conflicts.
For each candidate memory, search Title and Content:

```
POST /v1/data_sources/{data_source_id}/query
{
  "filter": {
    "or": [
      { "property": "Title", "title": { "contains": "<keyword from new memory>" } },
      { "property": "Content", "rich_text": { "contains": "<keyword from new memory>" } }
    ]
  },
  "page_size": 10
}
```

> **MCP platforms (Claude Code / Claude.ai):** Structured query is not available.
> Use `notion-search` with `data_source_url: "collection://<data_source_id>"` and keywords
> from the candidate memory as query. Then `notion-fetch` each result to compare properties.

The query returns full page properties. Check for:
1. **Duplicates**: Same fact already stored -> skip
2. **Updates**: Same topic but info changed -> update existing, mark old as Contradicted if needed
3. **Conflicts**: New info contradicts existing -> create new as Active, mark old as Contradicted

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
"用户使用 uv 而非 pip 管理 Python 依赖"        -> Category: Preference
"项目 OpenClaw 使用 FastAPI + PostgreSQL 架构"  -> Category: Decision
"用户偏好用 Ruff 做代码格式化和 lint"            -> Category: Preference
"用户是一名程序员"                              -> Category: Fact
```

**What NOT to store:**
- Transient Q&A that can be easily re-searched ("Python 的 GIL 是什么")
- Pleasantries and small talk
- Failed attempts with no useful outcome
- Information the user explicitly asked to forget

### Step 5: Write to Memory Store

Create pages in the database. For each memory entry, set properties:

```json
{
  "Title": "一句话概括",
  "Category": "Fact|Decision|Preference|Context|Pattern|Skill",
  "Content": "详细的记忆内容，足够让其他 AI 平台理解和使用",
  "Source": "Claude.ai|ClaudeCode|OpenClaw|Manual|Other",
  "Status": "Active",
  "Scope": "Global|Project",
  "Project": "项目名（Scope=Project 时填写）",
  "Expiry": "Never|30d|90d|1y",
  "date:Source Date:start": "YYYY-MM-DD",
  "date:Source Date:is_datetime": 0
}
```

**Scope guidelines:**
- **Global**: 跨项目通用 -- 用户偏好、通用工具链、个人习惯、全局决策
- **Project**: 特定项目 -- 项目架构、专属配置、项目内技术决策
- 无法判断时，默认 Global
- Project 字段填项目名（如 "OpenClaw"），与用户项目目录名或仓库名一致

**Expiry guidelines:**
- **Never**: Stable facts and preferences (name, tools, architecture)
- **1y**: May become outdated (tool versions, project status)
- **90d**: Limited shelf life (current tasks, temporary decisions)
- **30d**: Very transient (this week's todos, temporary workarounds)

### Step 6: Handle Conflicts

If Step 3 found conflicting memories:

1. Update the **old** memory's Status to "Contradicted":
   ```
   PATCH /v1/pages/{old_page_id}
   { "properties": { "Status": { "select": { "name": "Contradicted" } } } }
   ```
2. Create the **new** memory with Status "Active" (default)
3. Optionally note in new Content what it supersedes: "(更新: 之前记录为 XX)"

### Step 7: Report Results

After writing, provide the user with a summary:
- Conversations processed
- Memories created / updated / skipped
- List of new entries with Titles and Categories
- Conflicts detected and how resolved

**Example:**
```
Memory archival complete

Processed 8 conversations, generated 12 memories:
- New: 10
- Updated: 1 (user location updated from Beijing to Shenzhen)
- Skipped: 3 low-value conversations

New memories:
| Title | Category |
|-------|----------|
| 用户使用 uv 管理 Python 依赖 | Preference |
| 项目 OpenClaw 使用 FastAPI 架构 | Decision |
```

## Important Notes

- **Atomic entries**: One fact per row. Never dump a whole conversation summary into one entry.
- **Language**: Title and Content in the same language as the original conversation (typically Chinese).
- **Idempotent**: Always check for existing memories before writing. Running twice should not create duplicates.
- **Source accuracy**: 根据当前平台自动设置 Source（Claude.ai -> "Claude.ai"，Claude Code -> "ClaudeCode"，OpenClaw -> "OpenClaw"）。
- **Preserve details**: Keep code snippets, commands, config values, URLs verbatim in Content.
- **User control**: Don't store anything the user wouldn't want to see. When in doubt, ask.
- **Cross-platform ready**: Write Content so any AI platform can understand and use it.

## Example Interaction

**User**: 总结一下记忆

**Claude**:
1. Searches for "Memory Store" database, obtains `data_source_id` and `database_id`
2. Reviews current session (Claude Code) or recent chats (Claude.ai)
3. Queries database for existing entries to avoid duplicates
4. Decomposes conversations into atomic memories
5. Writes entries to the database
6. Reports: "Processed 5 conversations, generated 8 new memories, updated 1, skipped 2."
