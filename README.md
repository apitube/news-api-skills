# APITube News API — Agent Skills

Official [Agent Skills](https://agentskills.io/) for the [APITube](https://apitube.io) News
API. They teach a coding agent how to authenticate, search worldwide news, read sentiment and
entities, stream new articles, aggregate coverage, and migrate off another news API — without
the agent guessing parameter names.

Agent Skills are an open standard, so these work in Claude Code, Cursor, Codex, GitHub
Copilot, VS Code, Gemini CLI, Amp, Goose, opencode, Roo Code, Windsurf, JetBrains Junie and
every other skills-compatible agent.

You need an API key: **[get one at apitube.io](https://apitube.io)** — the free tier is 100
requests/day and does not require a card.

## Skills

| Skill | What it covers |
|---|---|
| [`apitube-news-api`](skills/apitube-news-api/SKILL.md) | Core: authentication, endpoints, the `/news/everything` filters, response shape, rate limits |
| [`apitube-mcp`](skills/apitube-mcp/SKILL.md) | The hosted MCP server at `mcp.apitube.io` — tools, prompts, client config |
| [`apitube-media-monitoring`](skills/apitube-media-monitoring/SKILL.md) | Brand, company and person tracking: resolve entity IDs, measure share of voice, de-duplicate |
| [`apitube-news-sentiment`](skills/apitube-news-sentiment/SKILL.md) | Overall, title, body and per-entity sentiment; clickbait and headline/body gap |
| [`apitube-breaking-news`](skills/apitube-breaking-news/SKILL.md) | Top headlines, the breaking flag, 44 classified event types, story clustering |
| [`apitube-news-stream`](skills/apitube-news-stream/SKILL.md) | Push delivery: SSE with resume and freshness windows, signed webhooks with HMAC verification |
| [`apitube-news-trends`](skills/apitube-news-trends/SKILL.md) | Aggregation: trending entities and topics, growth rate, period comparison, time buckets |
| [`apitube-news-export`](skills/apitube-news-export/SKILL.md) | CSV, XLSX, Parquet, JSONL, XML and RSS exports; field selection; safe bulk pagination |
| [`apitube-migrate-to-apitube`](skills/apitube-migrate-to-apitube/SKILL.md) | Porting from NewsAPI.org, GNews, Mediastack, NewsData.io, Perigon, Aylien and 9 more |

## Install

### Any agent — one skill

```bash
npx skills add apitube/news-api-skills --skill apitube-news-api --agent claude-code
```

`--agent` takes your agent's slug — the [skills.sh](https://skills.sh) registry covers Claude
Code, Cursor, Codex, GitHub Copilot, VS Code, Gemini, Windsurf, opencode, Amp, Goose, Roo,
Cline, Kiro, Trae, Zed and others; run `npx skills add --help` for the current list. Drop
`--skill` to install the whole set.

### Claude Code — as a plugin marketplace

```
/plugin marketplace add apitube/news-api-skills
```

### Manual — copy the folders

Skills are plain folders; copy the ones you want into your agent's skills directory.

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

Cursor, Codex, VS Code and opencode also read the Claude directories for compatibility. Check
your agent's documentation if it is not listed — the format is identical everywhere, only the
directory differs.

## Provide the API key

None of these skills embed a key. Export it in the environment your agent runs in:

```bash
export APITUBE_API_KEY="your_key"
```

Then pass it as `Authorization: Bearer $APITUBE_API_KEY`, `X-API-Key: $APITUBE_API_KEY`, or
`api_key=` in the query string. Test keys prefixed `api_test_` return masked bodies and do not
consume quota — use one while wiring things up.

## Skills without installing anything

APITube also serves its skills over HTTP under the
[Agent Skills discovery spec](https://agentskills.io/), so an agent can find them on its own:

- Index: https://docs.apitube.io/.well-known/agent-skills/index.json
- MCP server card: https://docs.apitube.io/.well-known/mcp/server-card.json

And there is a hosted MCP server at `https://mcp.apitube.io/` if you would rather give the
agent tools than instructions — see the `apitube-mcp` skill.

## Links

- Website — https://apitube.io
- Documentation — https://docs.apitube.io
- Pricing — https://apitube.io/pricing
- API reference — https://api-reference.apitube.io
- SDKs and migration kits — https://github.com/apitube
- Contact — https://apitube.io/contact

## Contributing

Found a parameter that behaves differently from what a skill says? Open an issue with the
request you ran and the response you got. Corrections to documented behaviour are the most
useful contribution here — these files are read by agents that will act on them.

## License

MIT — see [LICENSE](LICENSE).
