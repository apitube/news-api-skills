# AGENTS.md

Guidance for AI agents working in this repository.

## What this repository is

Official [Agent Skills](https://agentskills.io/) for the APITube News API. Every skill is a
folder under `skills/` containing a single `SKILL.md`: YAML frontmatter with `name` and
`description`, then markdown instructions.

```
skills/
  apitube-news-api/SKILL.md
  apitube-mcp/SKILL.md
  ...
```

There is no build step, no dependencies and no executable code. That is deliberate: these
files are read by agents that act on them, and a skill that ships scripts is a skill users
have to audit.

## Rules for editing a skill

1. **Every parameter, endpoint, field name, limit and error code must be verifiable** against
   the live API or https://docs.apitube.io. If you cannot confirm it, leave it out. A wrong
   parameter name in a skill sends an agent into a retry loop.
2. **`name` in the frontmatter must match the folder name.** Agents install by folder and
   dispatch by name.
3. **`description` is the routing signal.** It is the only thing an agent sees before deciding
   to load the skill, so it must say what the skill does *and* when to reach for it.
4. **Do not put an API key in an example.** Use `YOUR_API_KEY`.
5. **Keep examples runnable.** A `curl` line should work after substituting the key.
6. Cross-reference sibling skills by name (`see the apitube-news-api skill`) rather than
   duplicating their content.

## Adding a skill

1. Create `skills/<name>/SKILL.md` with frontmatter and instructions.
2. Add the folder to the `skills` array in `.claude-plugin/marketplace.json`.
3. Add a row to the skills table in `README.md`.

## Source of truth

The API's real behaviour, in order of authority:

1. https://api.apitube.io — run the request
2. https://docs.apitube.io — the reference
3. https://api-reference.apitube.io/openapi.json — the OpenAPI spec

Where a skill and the docs disagree, the API wins and both should be corrected.
