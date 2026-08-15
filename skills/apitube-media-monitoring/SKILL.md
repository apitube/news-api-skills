---
name: apitube-media-monitoring
description: Track mentions of a company, brand, product or person across worldwide news with the APITube News API — resolve the entity ID, filter coverage, measure share of voice, and de-duplicate reprints. Use for brand monitoring, competitor tracking, PR reporting and executive watchlists.
---

# Media monitoring with APITube

Monitor who is being written about, where, and in what tone. The reliable pattern is
**resolve the entity ID first, then filter on it** — matching on a raw name catches
homonyms and misses articles that spell the name differently.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Docs: https://docs.apitube.io/platform/news-api/everything

## Step 1 — resolve the entity ID

`/suggest/entities` turns a name or prefix into an APITube entity ID.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/suggest/entities?prefix=Tesla,%20Inc"
```

```json
[
  { "id": 1580517, "name": "Tesla, Inc.", "type": "organization",
    "links": {
      "self": "https://api.apitube.io/v1/news/entity/1580517",
      "wikipedia": "https://en.wikipedia.org/wiki/Tesla,_Inc.",
      "wikidata": "https://www.wikidata.org/wiki/Q478214"
    } }
]
```

Entity `type` is one of `person`, `location`, `organization`, `brand`, `product`,
`natural-disaster`, `disease`, `event`, `sport`, `unknown`. Pick the right row and be
specific with the prefix: `prefix=Tesla` returns 22 entities including "Tesla Robotaxi"
(a brand), "Nvidia Tesla" and two video games. `prefix=Tesla, Inc` returns the company.
Cache the ID once you have it; it is stable.

The same endpoint family resolves the other taxonomies: `/suggest/categories`,
`/suggest/topics`, `/suggest/industries`.

## Step 2 — pull the coverage

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?entity.id=1580517&published_at.start=NOW-7DAY&language.code=en&is_duplicate=0&sort.by=published_at&per_page=100"
```

Filters that matter for a monitoring feed:

| Parameter | Why |
|---|---|
| `entity.id` | the subject; up to 3 IDs, combined with OR |
| `published_at.start=NOW-7DAY` | rolling window, resolved per request — never goes stale |
| `is_duplicate=0` | drop articles flagged as reprints — a coarse signal, keep your own seen-set too |
| `source.rank.opr.min=5` | only sources with real authority (Open PageRank) |
| `ignore.source.domain` | mute your own newsroom or a known spam farm (max 3) |
| `language.code` | up to 3 languages per request; run several requests for wider coverage |
| `has_author=1` | skip anonymous aggregator pages |
| `source.bias` | `left`, `center`, `right` — for balance reporting |

If you track a person rather than an organization, `person.name=Elon Musk` works without an
ID lookup, and `organization.name`, `brand.name`, `location.name` behave the same way.
Name filters accept up to 3 comma-separated values.

## Step 3 — how loud is it? Use `/news/count`

`/news/count` runs the same filters but returns only a number, so it is far cheaper than
paging through results to measure volume.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/count?entity.id=1580517&published_at.start=NOW-1DAY"
```

```json
{ "status": "ok", "count": 342, "request_id": "req_abc123def456" }
```

Share of voice is two counts: your entity against a competitor's over the same window.
Run one `count` per entity and divide — do not try to compute it from a single response.

## Step 4 — break the coverage down

Faceting returns the distribution without pulling articles. Add `facet=1` and up to 5
`facet.field` values:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?entity.id=1580517&published_at.start=NOW-7DAY&facet=1&facet.field=source.id,sentiment.overall.polarity,language.id&facet.limit=20&per_page=1"
```

Useful facet fields: `source.id`, `source.country.id`, `source.bias`, `language.id`,
`category.id`, `topic.id`, `entity.id`, `sentiment.overall.polarity`, `published.day_of_week`,
`published.hour`.

For "who else is in the story", `/news/trends` with `field=entity.id` ranks co-mentioned
entities — see the `apitube-news-trends` skill.

## Step 5 — keep it running

Three ways to keep a watchlist current, in order of effort:

1. **Poll** `/news/everything` on a schedule with `published_at.start` set to the last run.
   Store the highest `id` you have seen and drop anything at or below it.
2. **Webhooks** — APITube posts new matches to your endpoint. See the `apitube-news-stream` skill.
3. **SSE** `/news/stream` — an open connection that pushes matches as they are indexed.

De-duplicate on `href` (the original publisher URL) and keep a seen-set of article `id`s.
`is_duplicate=0` filters on a reprint flag, which is a hint rather than a guarantee — do not
rely on it alone. Note also that the same story rewritten by a second newsroom is a genuinely
different article, and in monitoring that is usually what you want to keep.

## Reporting fields worth keeping

From each article in `results`, a monitoring record normally needs:
`id`, `href`, `title`, `published_at`, `source.domain`, `source.rankings.opr`,
`sentiment.overall.score`, `sentiment.overall.polarity`, `entities[]`, `language`.

Trim the payload with `fl` so you do not download article bodies you will not store:

```
&fl=id,href,title,published_at,source.domain,sentiment.overall.polarity
```

## Common questions

**Why does an article mention my brand but not come back with `entity.id`?**
The entity extractor tags what it recognises in the text. A passing mention inside a quote
may not be tagged. Run a second pass with `title` or `brand.name` to catch those, and
merge the two result sets on `id`.

**How far back can I look?** There is no range limit on `published_at` by itself. The 31-day
window applies only when you also use a title search (`400 ER0110` beyond that).

**Can I monitor several brands in one request?** Up to 3 IDs per parameter
(`entity.id=1580517,474,1223812`). Beyond that, one request per group — the results carry the
matching entities, so you can attribute them afterwards.
