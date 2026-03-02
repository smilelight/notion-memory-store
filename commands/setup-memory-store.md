---
description: Initialize Memory Store - create Notion database and generate config
argument-hint: ""
allowed-tools: [Bash, Read, Write, Glob, Grep, WebFetch]
---

# Setup Memory Store

This command initializes the Memory Store for first-time users. It creates the Notion database
and writes the local configuration file.

## Prerequisites

- A Notion MCP server must be connected (via `Notion:notion-search`, `Notion:notion-create-database`, etc.)
- The user must have a Notion workspace with API access

## Setup Flow

### Step 1: Check Notion Connection

Test that the Notion MCP is available by running a simple search:

```
Notion:notion-search with query: "Memory Store"
```

If the MCP is not available, tell the user:
"Notion MCP server is not connected. Please add it to your `.mcp.json` or Claude Code settings first."

### Step 2: Check for Existing Database

Search for an existing "Memory Store" database:

```
Notion:notion-search with query: "Memory Store"
```

Look through results for a database (not a page) named "Memory Store".

- If found: Extract its `data_source_id` and skip to Step 4.
- If not found: Proceed to Step 3 to create it.

### Step 3: Create the Memory Store Database

Ask the user which Notion page should be the parent for the database.
They can provide a page URL or page ID.

Once confirmed, create the database using `Notion:notion-create-database`:

```json
{
  "parent_page_id": "<user-provided page ID>",
  "title": "Memory Store",
  "properties": [
    {
      "name": "Title",
      "type": "title"
    },
    {
      "name": "Category",
      "type": "select",
      "options": ["Fact", "Decision", "Preference", "Context", "Pattern", "Skill"]
    },
    {
      "name": "Tags",
      "type": "multi_select",
      "options": ["Python", "Architecture", "Notion", "ClaudeCode", "Git", "Integration", "Workflow", "Preference", "DevTools"]
    },
    {
      "name": "Content",
      "type": "rich_text"
    },
    {
      "name": "Source",
      "type": "select",
      "options": ["Claude.ai", "ClaudeCode", "Manual", "OpenClaw", "Other"]
    },
    {
      "name": "Confidence",
      "type": "select",
      "options": ["High", "Medium", "Low"]
    },
    {
      "name": "Status",
      "type": "select",
      "options": ["Active", "Archived", "Contradicted"]
    },
    {
      "name": "Scope",
      "type": "select",
      "options": ["Global", "Project"]
    },
    {
      "name": "Project",
      "type": "rich_text"
    },
    {
      "name": "Expiry",
      "type": "select",
      "options": ["Never", "30d", "90d", "1y"]
    },
    {
      "name": "Source Date",
      "type": "date"
    },
    {
      "name": "Last Used",
      "type": "date"
    },
    {
      "name": "Memory ID",
      "type": "rich_text"
    }
  ]
}
```

> Note: The exact API call format may vary depending on the Notion MCP implementation.
> If `notion-create-database` is not available, guide the user to create the database manually
> in Notion with the schema above, then provide the database URL/ID.

### Step 4: Write Configuration File

After obtaining the database `data_source_id`, write the config file:

**Path**: `~/.config/memory-store/config.json`

```json
{
  "notion": {
    "data_source_id": "<the database data_source_id>",
    "database_url": "<the database URL, e.g. https://www.notion.so/xxx>"
  }
}
```

Use the Bash tool to create the directory and the Write tool to write the file:

```bash
mkdir -p ~/.config/memory-store
```

Then write the JSON config using the Write tool.

### Step 5: Verify

Run a test search against the newly configured database:

```
Notion:notion-search with query: "test" and data_source_url: "collection://<data_source_id>"
```

If it returns without error, setup is complete.

### Step 6: Report

Tell the user:

```
Memory Store setup complete!

- Database: Memory Store (<database_url>)
- Config: ~/.config/memory-store/config.json
- data_source_id: <id>

You can now use:
- /memory-to-notion (or say "总结记忆") to write memories
- /recall-from-notion (or say "回忆一下") to recall memories
```
