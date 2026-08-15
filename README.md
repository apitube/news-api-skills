# APITube News API — skill for AI coding agents

The official `SKILL.md` for the [APITube](https://apitube.io) News API. It teaches a coding
agent how to authenticate, search worldwide news, filter by entity, sentiment, source and
date, and read the JSON response — without guessing parameter names.

`SKILL.md` is an open format, so this works in Claude Code, Cursor, Codex, GitHub Copilot,
VS Code, Gemini CLI, Amp, Goose, opencode, Roo Code, Windsurf, JetBrains Junie and every other
skills-compatible agent.

You need an API key: **[get one at apitube.io](https://apitube.io)** — the free tier is 100
requests/day and does not require a card.

## The skill

[`apitube-news-api`](skills/apitube-news-api/SKILL.md) — authentication, endpoints, the
`/news/everything` filters, plain-language `prompt` queries, the response shape, pagination,
rate limits and error codes.

Every parameter, field name, limit and error code in it was executed against the live API at
`api.apitube.io` before being written down.

## Install

### Any agent

```bash
npx skills add apitube/news-api-skills --skill apitube-news-api --agent claude-code
```

This installs into the **current project**, not your home directory: the file lands in
`./.claude/skills/apitube-news-api/SKILL.md` and a `skills-lock.json` is written alongside it,
pinning the source repo and a content hash. Run it from your project root, and commit the lock
file if you want the whole team on the same revision.

`--agent` takes your agent's slug — the [skills.sh](https://skills.sh) registry covers Claude
Code, Cursor, Codex, GitHub Copilot, VS Code, Gemini, Windsurf, opencode, Amp, Goose, Roo,
Cline, Kiro, Trae, Zed and others; run `npx skills add --help` for the current list.

### Claude Code — as a plugin marketplace

```
/plugin marketplace add apitube/news-api-skills
/plugin install news-api-skills@apitube-news-api
```

The first command registers the marketplace, the second installs the plugin at user scope.
Remove them with `/plugin uninstall news-api-skills@apitube-news-api` and
`/plugin marketplace remove apitube-news-api`.

### Manual — copy the folder

```bash
git clone https://github.com/apitube/news-api-skills.git

# Works for Cursor, Codex, VS Code / Copilot and opencode alike
mkdir -p ~/.agents/skills && cp -r news-api-skills/skills/* ~/.agents/skills/

# Claude Code
mkdir -p ~/.claude/skills && cp -r news-api-skills/skills/* ~/.claude/skills/
```

`~/.agents/skills/` (and `.agents/skills/` inside a repo) is the shared location Cursor,
Codex, VS Code, GitHub Copilot and opencode all read, so one copy covers them at once.

| Agent | Personal | Project |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.agents/skills/`, `~/.cursor/skills/` | `.agents/skills/`, `.cursor/skills/` |
| Codex | `~/.agents/skills/` | `.agents/skills/` |
| VS Code / GitHub Copilot | `~/.agents/skills/`, `~/.copilot/skills/` | `.agents/skills/`, `.github/skills/` |
| opencode | `~/.agents/skills/`, `~/.config/opencode/skills/` | `.agents/skills/`, `.opencode/skills/` |

Cursor, Codex, VS Code and opencode also read the Claude directories for compatibility.

## Provide the API key

The skill embeds no key. Export it in the environment your agent runs in:

```bash
export APITUBE_API_KEY="your_key"
```

Then pass it as `Authorization: Bearer $APITUBE_API_KEY`, `X-API-Key: $APITUBE_API_KEY`, or
`api_key=` in the query string. Test keys prefixed `api_test_` return masked bodies and do not
consume quota — use one while wiring things up.

## Without installing anything

APITube also serves its skills over HTTP, so an agent can discover them on its own:

- Index: https://docs.apitube.io/.well-known/agent-skills/index.json
- MCP server card: https://docs.apitube.io/.well-known/mcp/server-card.json

There is also a hosted MCP server at `https://mcp.apitube.io/` if you would rather give the
agent tools than instructions — see
https://docs.apitube.io/platform/news-api/ai/mcp-server.

## Links

- Website — https://apitube.io
- Documentation — https://docs.apitube.io
- Pricing — https://apitube.io/pricing
- API reference — https://api-reference.apitube.io
- SDKs and migration kits — https://github.com/apitube
- Contact — https://apitube.io/contact

## Contributing

Found a parameter that behaves differently from what the skill says? Open an issue with the
request you ran and the response you got. Corrections to documented behaviour are the most
useful contribution here — this file is read by agents that will act on it.

## License

MIT — see [LICENSE](LICENSE).
