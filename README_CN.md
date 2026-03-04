# Notion Memory Store

基于 Notion 的跨平台持久化记忆层。在 Claude.ai、Claude Code 和 OpenClaw 之间存储、召回和管理 AI 对话记忆。

## 特性

- **零配置**: 通过数据库名称约定自动发现，无需配置任何 ID
- **写入记忆**: 将对话分解为原子化、可搜索的记忆条目
- **召回记忆**: 双路召回（结构化查询 + 语义搜索），带相关性排序
- **跨平台**: 支持 Claude.ai、Claude Code 和 OpenClaw
- **去重**: 自动检测重复和冲突的记忆
- **自动建库**: 首次使用时自动创建数据库

## 工作原理

采用 **约定优于配置** 的设计：Notion 数据库固定命名为 **"Memory Store"**。每次调用时，skill 会在你的 Notion 工作区中搜索该名称的数据库并直接使用。无需配置文件、环境变量或复制粘贴 ID。

- **首次使用**: 如果 "Memory Store" 不存在，写入 skill 会询问在哪个页面下创建
- **后续使用**: skill 按名称找到已有数据库，直接使用
- **跨平台共享**: 同一个数据库在 Claude.ai、Claude Code 和 OpenClaw 之间共享

> **注意**: 不要在 Notion 中重命名 "Memory Store" 数据库。skill 依赖这个确切名称来发现它。
> 如果重命名，skill 会认为数据库不存在并提议创建新的。

## 前置条件

- 一个 Notion 工作区
- 将 Notion 集成连接到你的 AI 平台:
  - **Claude Code**: Notion MCP 服务器 (通过 `.mcp.json` 或 Claude Code 设置)
  - **Claude.ai**: Notion Connector (设置 -> Connectors)
  - **OpenClaw**: 安装 [notion skill](https://clawhub.ai/steipete/notion) (提供 Notion API 访问能力)

## 安装

### Claude Code (插件)

```bash
git clone https://github.com/smilelight/notion-memory-store.git
claude plugin add ./notion-memory-store
```

### Claude.ai (上传 Skill)

1. 从本仓库下载两个 SKILL.md 文件:
   - `skills/memory-to-notion/SKILL.md`
   - `skills/recall-from-notion/SKILL.md`
2. 打开 https://claude.ai/customize/skills -> 点击 **+** -> **Upload a skill**
3. 上传每个 SKILL.md
4. 确保 Notion 工作区已通过 **Connectors** 连接

### OpenClaw / 其他平台

1. 安装 [notion skill](https://clawhub.ai/steipete/notion) (提供 Notion API 访问能力)
2. 下载 SKILL.md 文件，按照你所在平台的 skill 导入方式导入即可

## 使用方法

### 写入记忆

说以下任意一句触发记忆归档:
- "总结记忆" / "summarize memory"
- "归档对话" / "同步记忆到Notion"
- 或直接调用: `/memory-to-notion`

### 召回记忆

召回 skill 会在需要上下文时**主动触发**。你也可以主动说:
- "回忆一下" / "recall"
- "你还记得吗" / "之前我们讨论过"
- 或直接调用: `/recall-from-notion`

## 数据库 Schema

| 字段        | 类型        | 说明                                   |
|-------------|-------------|----------------------------------------|
| Title       | Title       | 记忆的一句话概括                         |
| Category    | Select      | Fact / Decision / Preference / Context / Pattern / Skill |
| Content     | Rich Text   | 记忆的详细内容                           |
| Source      | Select      | Claude.ai / ClaudeCode / Manual / OpenClaw / Other |
| Status      | Select      | Active / Archived / Contradicted        |
| Scope       | Select      | Global / Project                        |
| Project     | Rich Text   | 项目名 (Scope=Project 时填写)            |
| Expiry      | Select      | Never / 30d / 90d / 1y                  |
| Source Date | Date        | 原始对话发生时间                         |

## 注意事项

- **数据库名称即契约**: skill 通过搜索确切名称 "Memory Store" 来发现数据库，请勿重命名。
- **每个工作区一个数据库**: 如果存在多个同名 "Memory Store" 数据库，skill 会使用第一个匹配项，请避免重复。
- **需要 Notion 连接**: skill 需要 Notion 的搜索/读取/写入权限，请确保你的 Notion 集成已正确连接。

## 插件结构

```
notion-memory-store/
├── .claude-plugin/
│   └── plugin.json                # 插件元数据
├── skills/
│   ├── memory-to-notion/
│   │   └── SKILL.md               # 写入记忆
│   └── recall-from-notion/
│       └── SKILL.md               # 召回记忆
├── README.md
└── LICENSE
```

## 许可证

MIT
