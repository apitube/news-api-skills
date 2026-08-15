---
name: apitube-mcp
description: Connect an AI assistant to APITube News via the hosted MCP server (no install). Use to search and filter news in natural language from Cursor, Claude Desktop, or any MCP client.
---

# APITube MCP Server

A hosted Model Context Protocol (MCP) server that exposes the APITube News API to AI
assistants — no installation or maintenance required. Ask for news in natural language and
the assistant turns it into an API call.

- Endpoint: `https://mcp.apitube.io/` (JSON-RPC over HTTP `POST`)
- Protocol version: latest `2025-11-25`; the server negotiates down to the version the client requests (older
  versions such as `2024-11-05` are supported).
- Server version: `1.0.0`
- Get an API key: https://apitube.io/
- Docs: https://docs.apitube.io/platform/news-api/ai/mcp-server

## Available tools

- `search_news` — search news articles with filtering by title/keywords, language,
  sentiment, entities, media content, source quality, breaking-news flag, and date ranges.
- `suggest` — resolve a name or prefix (e.g. "Tesla", "spo") to APITube
  entity/category/topic/industry IDs. Use it first when you need an exact id for the
  `search_news` filters (`entity.id`, `category.id`, `topic.id`, `industry.id`).

## Prompts

Ready-made query templates (surfaced as slash commands in MCP clients), built on top of the tools above:

- `monitor_company` (args: `company`, optional `days`) — track recent news and sentiment about a company.
- `topic_sentiment` (args: `topic`, optional `language`) — break down sentiment of coverage about a topic.
- `breaking_news` (args: optional `subject`, `country`) — surface the latest breaking news.
- `compare_coverage` (args: `subject_a`, `subject_b`) — compare volume and sentiment for two subjects.

## Configure an MCP client

The hosted server speaks HTTP JSON-RPC; authenticate with `Authorization: Bearer`.

For clients with native remote-server support (Cursor at `~/.cursor/mcp.json`, VS Code,
Windsurf):

```json
{
  "mcpServers": {
    "apitube-news": {
      "url": "https://mcp.apitube.io/",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

For stdio-only clients (Claude Desktop at
`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS), bridge with
`mcp-remote`:

```json
{
  "mcpServers": {
    "apitube-news": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.apitube.io/",
               "--header", "Authorization: Bearer YOUR_API_KEY"]
    }
  }
}
```

Restart the assistant after editing the config. Then ask things like
"Find positive breaking news about Tesla in English from the last week."

## Direct HTTP access

The server also works as a plain JSON-RPC API. Authenticate with `Authorization: Bearer`.

```bash
curl -X POST https://mcp.apitube.io/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "search_news",
      "arguments": { "title": "AI technology", "language": { "code": "en" }, "is_breaking": 1 }
    },
    "id": 1
  }'
```

To verify the server is reachable, call `initialize` with the same protocol version
(`2024-11-05`); a healthy server replies with `serverInfo.name` = "APITube News MCP-Server".
