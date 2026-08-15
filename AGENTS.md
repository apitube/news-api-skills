# AGENTS.md

Guidance for AI agents working in this repository.

## What this repository is

The official `SKILL.md` for the APITube News API. The skill is a
folder under `skills/` containing a single `SKILL.md`: YAML frontmatter with `name` and
`description`, then markdown instructions.

```
skills/
  apitube-news-api/SKILL.md
```

There is no build step, no dependencies and no executable code. That is deliberate: this file
is read by agents that act on it, and a skill that ships scripts is a skill users have to
audit.

## Rules for editing

1. **Every parameter, endpoint, field name, limit and error code must be verified** by running
   it against `https://api.apitube.io`. If you cannot confirm it, leave it out. A wrong
   parameter name sends an agent into a retry loop.
2. **Do not trust the docs over the API.** Several documented behaviours do not match the
   running service; the skill records the real behaviour and says so explicitly.
3. **`name` in the frontmatter must match the folder name.** Agents install by folder and
   dispatch by name.
4. **`description` is the routing signal.** It is the only thing an agent sees before deciding
   to load the skill, so it must say what the skill does *and* when to reach for it.
5. **Do not put an API key in an example.** Use `YOUR_API_KEY`.
6. **Keep examples runnable.** A `curl` line should work after substituting the key.

## Source of truth

The API's real behaviour, in order of authority:

1. https://api.apitube.io — run the request
2. https://docs.apitube.io — the reference
3. https://api-reference.apitube.io/openapi.json — the OpenAPI spec

Where the skill and the docs disagree, the API wins and both should be corrected.
