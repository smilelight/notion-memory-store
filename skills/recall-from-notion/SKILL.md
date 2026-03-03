---
name: recall-from-notion
description: >
  Recall user memories from the Notion Memory Store. Trigger PROACTIVELY at the beginning of
  conversations where knowing the user's background, preferences, past decisions, or project
  context would improve the response. Also trigger when the user says "回忆一下", "recall",
  "你还记得吗", "之前我们讨论过", "based on what you know about me", or references past context.
user-invocable: true
---

# Recall from Notion

Read the user's memories from the **Memory Store** Notion Database and use them as context
for the current conversation.

## Database Discovery

This skill uses a **zero-config convention**: the database is always named **"Memory Store"**.

**Step 1: Locate the database**

```
POST /v1/search
{
  "query": "Memory Store"
}
```

From the results, find the item with `object: "data_source"` whose title is "Memory Store".
Extract both:
- `data_source_id` -- for querying (`POST /v1/data_sources/{id}/query`)
- `database_id` -- for reference

- **If found** -> use `data_source_id` for all subsequent queries.
- **If not found** -> silently skip recall. Do NOT prompt the user to create anything.

## Platform Adaptation

This skill describes operations using generic Notion REST API format.
Each platform's AI should translate to its available tools using the **fixed mappings** below.
Do NOT guess -- follow these mappings exactly.

### Claude Code / Claude.ai (Notion MCP Tools)

| Operation | SKILL.md Describes | Use MCP Tool | Key Parameters |
|-----------|-------------------|--------------|----------------|
| Discover database | `POST /v1/search` | `notion-search` | `query: "Memory Store"`, `content_search_mode: "workspace_search"` |
| Get data_source_id | -- | `notion-fetch` | Fetch the database, extract from `<data-source url="collection://...">` tag |
| Structured query | `POST /v1/data_sources/{id}/query` | **Not available** | Skip Path 1, use Path 2 only |
| Semantic search in DB | `POST /v1/search` + data_source_url | `notion-search` | `data_source_url: "collection://<data_source_id>"` |
| Fetch page details | `GET /v1/pages/{id}` | `notion-fetch` | `id: "<page_id>"` |

**Critical notes:**
- Discovery MUST use `content_search_mode: "workspace_search"` (default `ai_search` mode may not return databases)
- Structured database query (Path 1) is **not available** in MCP. Dual-path recall degrades to **semantic search only**. This is acceptable for <500 memories.
- Semantic search results lack full properties. Use `notion-fetch` per result to get Category, Status, Scope, etc.
- Multiple `notion-fetch` calls can run **in parallel** within one response to minimize latency.

### OpenClaw / Direct API Access

Use Notion REST API exactly as described in the workflow steps. All operations including
structured query (Path 1) are fully supported.

## When to Trigger

**Always trigger when:**
- User references past conversations or shared context ("我们之前说的", "you know my setup")
- User starts a task where personal context matters (coding, writing, planning, recommendations)
- User asks about their own preferences, decisions, or project details
- User says "recall", "回忆", "你记得吗", or similar

**Consider triggering when:**
- A new conversation starts with a domain-specific task (coding, architecture, DevOps, etc.)
- User mentions a project name, tool, or technology that might have stored context
- User asks for recommendations or opinions where past preferences would help

**Skip when:**
- Pure factual Q&A with no personal dimension ("Python 的 GIL 是什么")
- User explicitly says they want a fresh start or generic advice

## Recall Strategy

### Step 1: Discover Database

See Database Discovery above. If not found, silently skip all remaining steps.

### Step 2: Analyze Conversation Topic

From the user's message or conversation context, extract:

1. **Keywords**: Specific nouns, technologies, tools, project names
   - e.g., "Notion", "Claude Code", "orchestrator", "Python"
2. **Semantic query**: A natural language summary of the user's intent
   - e.g., "CI 配置偏好", "Python 项目架构决策"
3. **Current project** (if in Claude Code): Detect from the working directory
   - e.g., "OpenClaw", "skills", "claude_world"

### Step 3: Dual-Path Recall

Use **two parallel paths** to maximize recall coverage, then merge results.

**Path 1 -- Structured query** (precision, returns full properties):

Query the data source with keyword filters on Title, Content, and Project.

```
POST /v1/data_sources/{data_source_id}/query
{
  "filter": {
    "or": [
      { "property": "Title", "title": { "contains": "<keyword>" } },
      { "property": "Content", "rich_text": { "contains": "<keyword>" } },
      { "property": "Project", "rich_text": { "contains": "<keyword>" } }
    ]
  },
  "page_size": 50
}
```

For multiple keywords (e.g., "Notion" and "MCP"):

```json
{
  "filter": {
    "or": [
      { "property": "Title", "title": { "contains": "Notion" } },
      { "property": "Content", "rich_text": { "contains": "Notion" } },
      { "property": "Title", "title": { "contains": "MCP" } },
      { "property": "Content", "rich_text": { "contains": "MCP" } }
    ]
  }
}
```

> **MCP platforms (Claude Code / Claude.ai):** Path 1 is not available (no structured query tool).
> Skip directly to Path 2. The recall becomes single-path semantic search.

**Path 2 -- Semantic search** (coverage, catches what keywords miss):

Search within the Memory Store using the semantic query from Step 2.

```
POST /v1/search
{
  "query": "<semantic query from Step 2>",
  "data_source_url": "collection://<data_source_id>"
}
```

This catches memories that are semantically related but don't contain the exact keywords.
For example, searching "CI 配置" might find "GitHub Actions workflow 配置偏好" even though
it doesn't contain the word "CI".

> **Why dual-path?**
> Structured query is precise but only matches exact keywords -- it misses semantically
> related memories. Semantic search understands intent but returns incomplete properties
> (only id/title/highlight). Combining both gives precision + coverage.

### Step 4: Merge and Enrich

1. **Merge**: Combine results from both paths, deduplicate by page id.
2. **Enrich**: Structured query results already have full properties. For memories found
   **only** by semantic search, fetch their full properties:
   ```
   GET /v1/pages/{page_id}
   ```
   (Only fetch the delta -- skip pages already in structured query results.)

> **MCP platforms:** Since only Path 2 is available, ALL results need enrichment via `notion-fetch`.
> Fetch all results in parallel to minimize latency.

### Step 5: Filter

Apply these filters on the merged results:

**Scope filter (most important for Claude Code):**
- Always include: Scope = Global
- Include: Scope = Project where Project matches current project name
- Exclude: Scope = Project for OTHER projects
- Include: no Scope set (legacy data, treat as Global)

**Status filter:**
- **Contradicted**: Always skip
- **Archived**: Skip unless user explicitly asks

**Expiry filter:**
- 30d: Source Date + 30 days < today -> skip
- 90d: Source Date + 90 days < today -> skip
- 1y: Source Date + 1 year < today -> skip
- Never: Always include

### Step 6: Rank

**Priority scoring:**
1. **Dual-path bonus**: Found by both structured query and semantic search -> highest relevance
2. **Topic match**: Directly relates to user's current question/task
3. **Category weight**: Preference > Fact > Decision > Pattern > Skill > Context
4. **Recency**: Newer Source Date > Older

**Injection limit: 10-15 memories maximum.**

### Step 7: Inject as Context

Format recalled memories as a compact context block grouped by Category:

```
Recalled context from Memory Store:

[Preferences]
- 用户偏好用 Ruff 做代码格式化和 lint
- ...

[Facts]
- 用户是程序员，主要使用 Python
- Notion 工作区已连通，集成方式为 MCP
- ...

[Decisions]
- Memory Store 采用 Notion Database 结构
- ...
```

Rules:
- Group by Category, keep each entry 1-2 lines
- Include key details (IDs, commands, URLs) verbatim
- Only show Categories with entries
- Silently drop irrelevant memories

## Handling Edge Cases

**No results**: Proceed without memories. Don't announce unless user explicitly asked.

**Too many (>15)**: Rank strictly, inject top 10-15. Note more are available.

**Stale/wrong memories**: Flag contradictions and offer to update.

**"你怎么知道的？"**: Explain it came from Memory Store, offer to show/edit.

## Important Notes

- **Silent injection**: Don't say "我正在搜索记忆库..." unless user explicitly asked.
- **Relevance first**: When in doubt, leave it out.
- **Scope awareness**: In Claude Code, detect current project and filter accordingly.
- **Cross-platform**: Treat entries from all sources (Claude.ai, Claude Code, OpenClaw) equally.
- **Read-only**: This skill only reads from the database, never writes.
