---
name: apitube-news-trends
description: Aggregate news coverage with the APITube News API — trending entities, topics, categories and sources, growth rates, period comparison and time buckets. Use when an agent needs counts and rankings rather than a list of articles.
---

# News trends and aggregation

`/news/trends` answers "what is being written about most" without downloading a single
article body. It aggregates matching articles by one or more fields and can add growth
analysis, period comparison and a time-series breakdown.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Docs: https://docs.apitube.io/platform/news-api/trends

## The basic call

`field` is required and takes up to 5 comma-separated values from a fixed set:
`source.id`, `category.id`, `topic.id`, `industry.id`, `entity.id`.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/trends?field=entity.id&published_at.start=NOW-7DAY&language.code=en&per_page=10"
```

```json
{
  "status": "ok",
  "field": "entity.id",
  "per_page": 10,
  "offset": 0,
  "total_count": 45866,
  "total_articles": 834079,
  "sort": "count",
  "order": "desc",
  "mincount": 1,
  "trends": [
    { "value": { "id": 1297647, "name": "United States", "type_id": 2, "wikidata_id": 30 },
      "count": 9035, "percentage": 1.08, "growth_rate": 12.55 }
  ],
  "request_id": "781c41a4-a940-4e52-a4ed-28ad6bff4c94"
}
```

`value` is an enriched object when APITube can resolve the ID, and a plain string otherwise.
`percentage` is the share of `total_articles`; `growth_rate` is articles per hour over the
value's lifetime.

Entity results carry a numeric `type_id`: 1 person, 2 location, 3 organization, 4 brand,
5 product, 6 natural-disaster, 7 disease, 8 event, 14 sport. Note that `/suggest/entities`
returns the type as a **string** (`"organization"`) while trends returns `type_id` — do not
assume one shape across both endpoints.

Request several fields at once and `trends` becomes an object keyed by field name, each
holding its own `trends`, `total_count` and `total_articles`:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/trends?field=entity.id,topic.id,source.id&published_at.start=NOW-1DAY&per_page=5"
```

## Narrowing what gets aggregated

Trends accepts a subset of the article filters — everything you need to scope the population
before counting:

`published_at.*`, `language.*`, `source.*`, `category.*`, `topic.*`, `industry.*`,
`entity.*`, `is_breaking`, `is_duplicate`, plus the matching `ignore.*` exclusions and
`prompt`.

```bash
# Which companies dominate negative-free technology coverage in German media this week
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/trends?field=entity.id&topic.id=industry.technology_news&source.country.code=de&published_at.start=NOW-1WEEK&per_page=20"
```

**Maximum date range is 30 days.** For longer horizons, run consecutive windows and stitch
them together yourself.

## Sorting and thresholds

| Parameter | Effect |
|---|---|
| `sort` | `count` (default), `value`, `growth_rate`, `change`, `trending_score` |
| `order` | `asc` or `desc` (default) |
| `per_page` | results per field, 1–100, default 10 |
| `offset` | pagination offset |
| `mincount` | drop values with fewer than N articles, 1–10000, default 1 |
| `percentile` | keep only values above the given percentile, 1–100 |

`mincount` is the fastest way to remove long-tail noise: `mincount=5` hides everything
mentioned fewer than five times in the window.

## Growth: what is rising, not just what is big

Raw counts favour perennially popular subjects. Two mechanisms surface movement instead.

**`trending=1`** computes a growth rate and a `trending_score` over a window set by
`trending_days` (7–30, default 14). Each item gains `trending_score` and `trending_history`:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/trends?field=entity.id&published_at.start=NOW-7DAY&trending=1&trending_days=7&sort=growth_rate&per_page=15"
```

Two things to know before you rely on it.

**Trending queries are heavy.** Always scope them with `published_at.start`, and expect the
occasional `502` on a wide window — a `502` here is a timeout, not a bad request, so retry it
or narrow the range rather than changing the parameters.

**`sort=trending_score` is not applied.** The response comes back sorted by `count` and echoes
`"sort": "count"`. Sort by `growth_rate` instead, or pull the results and rank by
`trending_score` yourself. Always read the `sort` field in the response to see what the server
actually used — `growth_rate`, `value` and `change` are honoured.

**`compare=1`** contrasts the current window against an earlier one. It requires
`compare_window`, which accepts values like `24HOURS`, `7DAYS`, `1WEEK`, `1w`, `2m`. Each
item then carries `previous_count`, `change_absolute` and `change_percent`:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/trends?field=topic.id&published_at.start=NOW-1DAY&compare=1&compare_window=7DAYS&sort=change&per_page=20"
```

`sort=change` requires `compare=1`; verified working. `sort=trending_score` is documented but
does not take effect — see the note above.

## Time series

`time_bucket` takes `hour`, `day`, `week` or `month` and adds a `buckets` object plus a
`total` to every item:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/trends?field=entity.id&entity.id=1580517&published_at.start=NOW-14DAY&time_bucket=day"
```

Pick the bucket to match the window: `hour` over 14 days produces 336 points per item and is
rarely what you want; `day` over a fortnight is readable.

## When to use trends, count or facets

Three different aggregation tools — do not reach for the wrong one:

- **`/news/trends`** — ranked values of a taxonomy field, with growth and comparison.
  This is the analytics endpoint.
- **`/news/count`** — one number for one filter set. Cheapest by far; use it for share of
  voice, sample sizing and progress bars.
- **`facet=1` on `/news/everything`** — a distribution returned *alongside* the articles you
  were already fetching. Use it when you need both in one round trip; facets also cover
  fields trends does not, such as `sentiment.overall.polarity`, `source.bias` and
  `published.day_of_week`.

## Common questions

**Why does `growth_rate` look small for a huge entity?** It is articles per hour across the
value's lifetime, so a long-established subject dilutes. For "what is hot right now", sort by
`trending_score` with `trending=1`, or compare windows with `compare=1`.

**Can I aggregate by author or country?** Not through `field` — it is limited to the five
taxonomy fields. Faceting on `/news/everything` covers `author.id`, `source.country.id`,
`language.id` and more.

**Is the result cached?** The response includes a `cached` boolean so you can tell a fresh
computation from a served one.
