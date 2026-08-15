---
name: apitube-breaking-news
description: Fetch breaking and top news from high-authority sources with the APITube News API — top headlines, the is_breaking flag, story clustering, and freshness controls. Use when an agent needs what is happening right now rather than an archive search.
---

# Breaking and top news with APITube

Two different questions, two different tools: **`/news/top-headlines`** answers "what is
important right now", **`is_breaking=1`** answers "what is flagged as breaking". They overlap
but are not the same filter.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Docs: https://docs.apitube.io/platform/news-api/top-headlines

## Top headlines

`/news/top-headlines` returns important news restricted to sources with an Open PageRank of
**5 or higher** — the authority cut-off is applied for you. It accepts every parameter
`/news/everything` accepts, including faceting, highlighting and export.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/top-headlines?language.code=en&source.country.code=us&per_page=20"
```

Narrow it the usual ways:

| Goal | Parameters |
|---|---|
| One country's press | `source.country.code=gb` (max 3 codes) |
| One subject area | `category.id=…`, `topic.id=…`, `industry.id=…` |
| One company or person | `entity.id=…`, or `organization.name=` / `person.name=` |
| Last few hours only | `published_at.start=NOW-6HOUR` |
| Drop opinion-free wire copy | `has_author=1` |
| Skip paywalled links | `is_paywall=0` |

## The breaking flag

`is_breaking=1` works on any article endpoint, not just top headlines:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?is_breaking=1&language.code=en&published_at.start=NOW-2HOUR&sort.by=published_at&per_page=50"
```

Combine it with `source.rank.opr.min=7` when you want a short, high-confidence alert list
rather than everything a newsroom tagged as breaking.

## Event types instead of keywords

For "tell me when something specific happens", filter on the classified event type rather
than guessing headline wording. `event.type` takes up to 5 comma-separated values:

`merger-acquisition`, `ipo`, `layoffs`, `bankruptcy`, `product-launch`, `funding-round`,
`earnings`, `partnership`, `executive-change`, `lawsuit`, `data-breach`, `recall`,
`expansion`, `closure`, `stock-movement`, `contract-award`, `spin-off`, `regulatory-action`,
`election`, `protest`, `crime`, `terrorism`, `accident`, `policy-change`, `scandal`, `death`,
`award-ceremony`, `conflict`, `diplomacy`, `health-crisis`, `migration`, `human-rights`,
`earthquake`, `hurricane`, `flood`, `wildfire`, `tornado`, `tsunami`, `volcanic-eruption`,
`drought`, `climate-event`, `pollution`, `wildlife-event`, `avalanche`.

`event.category` groups them into `business`, `society` and `environment`.

```bash
# Data breaches and recalls reported in the last day
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?event.type=data-breach,recall&published_at.start=NOW-1DAY&language.code=en"
```

The full list of event types is also available at runtime:
`GET /v1/news/event-types`.

## One event, many articles: story clustering

A breaking event produces dozens of near-identical articles. `/news/story/{articleId}`
returns the cluster around a given article, so you can show one story with its coverage
instead of forty rows:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/story/3036291250"
```

Use it after you pick a lead article: take the first result from top headlines, then pull its
story to build a "covered by 24 outlets" panel or to compare framing across sources.

## Freshness: what "recent" really means

`published_at` is the publisher's timestamp, not the moment APITube indexed the article.
Discovery, fetch and parsing add lag, so an article that appears in your results now may
carry a timestamp from a few minutes ago. Two practical consequences:

- For a rolling alert window use `published_at.start=NOW-1HOUR` and accept that some articles
  arrive slightly after their timestamp — do not treat a missing article as permanently absent
  from the feed. Date math understands `HOUR`, `DAY`, `WEEK`, `MONTH`, `YEAR` (and the short
  forms `1h`, `2d`, `1w`); there is no minute unit, so an hour is the finest window here.
- When polling, track the highest article `id` you have already seen and skip anything at or
  below it. `id` is assigned in indexing order, so it is a safer cursor than a timestamp.

For push delivery instead of polling — Server-Sent Events and webhooks — see the
`apitube-news-stream` skill.

## Common questions

**Why is a story I saw on TV not in the results yet?** Broadcast usually precedes the online
article. The API indexes published web articles; there is nothing to return until a source
publishes text.

**Can I get breaking news on the Free plan?** Yes for the REST endpoints (100 requests/day,
10 results per request, first 5 pages). Live SSE streaming is limited by plan — check
https://docs.apitube.io/platform/news-api/rate-limits.

**How do I avoid duplicate alerts?** Keep a seen-set of article `id`s, and de-duplicate on
`href` across runs. `is_duplicate=0` filters on a reprint flag and helps, but it is a hint,
not a guarantee.
