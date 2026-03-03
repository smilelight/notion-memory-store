# Notion Memory Store

[中文文档](README_CN.md)

Persistent cross-platform memory layer using Notion as backend. Store, recall, and manage
AI conversation memories across Claude.ai, Claude Code, and OpenClaw.

## Features

- **Zero-config**: Database discovered by name convention, no IDs to configure
- **Write memories**: Decompose conversations into atomic, searchable memory entries
- **Recall memories**: Dual-path recall (structured query + semantic search) with relevance ranking
- **Cross-platform**: Works with Claude.ai, Claude Code, and OpenClaw
- **Deduplication**: Automatically detects duplicates and conflicting memories
- **Auto-create**: Database created automatically on first use if not found

## How It Works

The skills use a **convention-over-configuration** approach: the Notion database is always
named **"Memory Store"**. On each invocation, the skill searches your Notion workspace for
a database with this name and uses it directly. No config files, no environment variables,
no IDs to copy-paste.

- **First use**: If "Memory Store" doesn't exist, the write skill will ask where to create it
- **Subsequent uses**: The skill finds the existing database by name and uses it immediately
- **Cross-platform**: The same database is shared across Claude.ai, Claude Code, and OpenClaw

> **Important**: Do not rename the "Memory Store" database in Notion. The skills rely on
> this exact name to discover it. If you rename it, the skills will think it doesn't exist
> and offer to create a new one.

## Prerequisites

- A Notion workspace
- Notion integration connected to your AI platform:
  - **Claude Code**: Notion MCP server (via `.mcp.json` or Claude Code settings)
  - **Claude.ai**: Notion Connector (Settings -> Connectors)
  - **OpenClaw**: Notion integration per platform docs

## Installation

### Claude Code (Plugin)

```bash
git clone https://github.com/smilelight/notion-memory-store.git
claude plugin add ./notion-memory-store
```

### Claude.ai (Upload Skills)

1. Download the two SKILL.md files from this repo:
   - `skills/memory-to-notion/SKILL.md`
   - `skills/recall-from-notion/SKILL.md`
2. Go to https://claude.ai/customize/skills -> click **+** -> **Upload a skill**
3. Upload each SKILL.md
4. Make sure your Notion workspace is connected via **Connectors**

### OpenClaw / Other Platforms

Download the SKILL.md files and import according to your platform's skill mechanism.

## Usage

### Write Memories

Say any of these to trigger memory archival:
- "总结记忆" / "summarize memory"
- "归档对话" / "同步记忆到Notion"
- Or invoke: `/memory-to-notion`

### Recall Memories

The recall skill triggers **proactively** when context would help. You can also say:
- "回忆一下" / "recall"
- "你还记得吗" / "之前我们讨论过"
- Or invoke: `/recall-from-notion`

## Database Schema

| Property    | Type        | Description                            |
|-------------|-------------|----------------------------------------|
| Title       | Title       | One-line memory summary                |
| Category    | Select      | Fact / Decision / Preference / Context / Pattern / Skill |
| Content     | Rich Text   | Detailed memory content                |
| Source      | Select      | Claude.ai / ClaudeCode / Manual / OpenClaw / Other |
| Status      | Select      | Active / Archived / Contradicted       |
| Scope       | Select      | Global / Project                       |
| Project     | Rich Text   | Project name (when Scope = Project)    |
| Expiry      | Select      | Never / 30d / 90d / 1y                |
| Source Date | Date        | When the original conversation happened|

## Notes

- **Database name is the contract**: The skills find the database by searching for the
  exact name "Memory Store". Do not rename it.
- **One database per workspace**: If multiple databases named "Memory Store" exist,
  the skill will use the first match. Avoid duplicates.
- **Notion connection required**: The skills need Notion search/read/write access.
  Ensure your Notion integration is properly connected on your platform.

## Plugin Structure

```
notion-memory-store/
├── .claude-plugin/
│   └── plugin.json                # Plugin metadata
├── skills/
│   ├── memory-to-notion/
│   │   └── SKILL.md               # Write memories to Notion
│   └── recall-from-notion/
│       └── SKILL.md               # Recall memories from Notion
├── README.md
└── LICENSE
```

## License

MIT
