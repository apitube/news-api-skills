---
name: apitube-news-api
description: Query the APITube News API — authenticate, search and filter worldwide news articles, and parse the JSON response. Use when an agent needs live or historical news data.
---

# APITube News API

APITube is a worldwide News API. Use it to search and filter news articles in real time
or historically, then read structured fields (title, body, source, sentiment, entities).

- Base URL: `https://api.apitube.io/v1`
- Full docs: https://docs.apitube.io/
- OpenAPI spec: https://api-reference.apitube.io/openapi.json
- Get an API key: https://apitube.io/

## Authentication

Pass your API key in one of three ways. Headers are recommended over the query parameter.

```bash
# 1. Authorization header (most secure)
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?per_page=10"

# 2. X-API-Key header
curl -H "X-API-Key: YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?per_page=10"

# 3. Query parameter (avoid in production — leaks in logs)
curl "https://api.apitube.io/v1/news/everything?per_page=10&api_key=YOUR_API_KEY"
```

Keys prefixed `api_test_` run against the same data but return masked bodies and do not
consume quota — use them while wiring up an integration.
See https://docs.apitube.io/platform/news-api/authentication.

## Endpoints

Every endpoint accepts both `GET` (query string) and `POST` (JSON body) with the same parameters.

| Endpoint | Purpose |
|----------|---------|
| `/news/everything` | Search all articles with full filtering |
| `/news/top-headlines` | Breaking and important news from sources with OPR rank >= 5 |
| `/news/count` | Number of matching articles only — one `count(*)`, no article payload |
| `/news/article` | A single article — pass `id=<articleId>`, not `article.id` (`400 ER0232` without it) |
| `/news/story/{articleId}` | The cluster of articles covering the same story |
| `/news/category/{taxonomy}/{categoryId}` · `/news/topic/{topicId}` · `/news/industry/{industryId}` · `/news/entity/{entityId}` | Articles by taxonomy |
| `/news/trends` | Aggregations: top values of a field, growth, time buckets |
| `/news/stream` | Live Server-Sent Events feed |
| `/suggest/entities` · `/suggest/categories` · `/suggest/topics` · `/suggest/industries` | Autocomplete an ID from a name prefix |
| `/balance` | Remaining quota for the key |

## Key parameters for `/news/everything`

Filters combine with AND. Comma-separated values inside one parameter combine with OR.

- `title` — keyword search in article titles, 2–100 chars. Quotes for an exact phrase,
  `"climate change"~2` for proximity. **Title search is limited to a 31-day `published_at`
  window**: with no dates the last 31 days are searched (`200` plus warning `ER0366` in
  `meta.warnings`); an explicit wider range returns `400 ER0110`. Variants:
  `title_starts_with`, `title_ends_with`, `title_pattern`.
- `published_at.start`, `published_at.end` — `YYYY-MM-DD`, ISO 8601, or Solr-style date math:
  `NOW-7DAY`, `NOW-1WEEK`, `NOW/DAY`, `NOW-1DAY/DAY`. Relative values are resolved per request,
  so a stored query never goes stale.
- `language.code` — ISO 639-1, up to 3 comma-separated (`en`, `en,fr,de`).
- `category.id`, `topic.id`, `industry.id`, `entity.id` — taxonomy IDs, up to 3 each.
  Resolve names to IDs with `/suggest/*` first.
- `person.name`, `organization.name`, `location.name`, `brand.name`, `event.name`,
  `disease.name`, `disaster.name`, `sport.name` — filter by entity name without looking up an ID.
- `source.domain`, `source.id`, `source.country.code`, `source.bias` (`left`, `center`, `right`),
  `source.rank.opr.min` / `.max` (Open PageRank authority), `is_premium_source`, `is_verified_source`.
- `sentiment.overall.polarity` (`positive` / `negative` / `neutral`) and
  `sentiment.overall.score.min` / `.max` in the range −1…1.
- `is_breaking`, `is_paywall`, `is_duplicate`, `is_high_quality`, `has_author`, `has_image`, `has_video`.
- `event.type` — one of 44 classified event types (`ipo`, `layoffs`, `funding-round`,
  `data-breach`, `earthquake`, …), up to 5; `event.category` is `business`, `society` or `environment`.
- `sort.by` — `published_at` (default), `relevance`, `engagement`, `quality`, `controversy`,
  `trust`, `source.rank.opr`, `sentiment.overall.score`, `read_time`, and more.
  `sort.order` is `asc` or `desc`.
- `page`, `per_page` — pagination. `per_page` max is 250, capped at 10 on Free and 50 on Starter.
- `fl` — comma-separated list of fields to return, e.g. `fl=id,title,published_at,source.domain`.
  Cuts response size sharply on large pages.
- `article.id` — fetch specific articles by ID through `/news/everything`, up to 5
  comma-separated.
- `query` — boolean search language with `AND` / `OR` / `NOT` and grouping, for expressions
  `title` cannot express: https://docs.apitube.io/platform/news-api/query-builder

Every filter has an `ignore.` twin that excludes instead of includes: `ignore.source.domain`,
`ignore.language.code`, `ignore.entity.id`, `ignore.title`, and so on.

Example — English AI coverage from the last week, quality sources only:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  'https://api.apitube.io/v1/news/everything?title="artificial intelligence"&language.code=en&published_at.start=NOW-7DAY&source.rank.opr.min=5&sort.by=published_at&per_page=10'
```

## Plain-language queries with `prompt`

`prompt` (3–500 chars) describes the search in words; the API translates it into the
parameters above before running the query and returns what it used in `meta.prompt`
(`text`, `applied`, `ignored`, `cached`). Explicit parameters always win over the prompt.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?prompt=Tesla+and+Elon+Musk+news+in+English+for+the+last+10+days&per_page=10"
```

It costs 2 extra points on a cache miss (translations are cached), and it requires the Basic
plan or above — Free and Starter get `403 ER0706`. Other errors: `ER0800` (length out of range),
`ER0801` (translation service down), `ER0802` (nothing usable in the wording).

## Response shape

```json
{
  "status": "ok",
  "limit": 10,
  "page": 1,
  "has_next_pages": true,
  "next_page": "https://api.apitube.io/v1/news/everything?...&page=2",
  "results": [
    {
      "id": 3036291250,
      "href": "https://original-source.example/article",
      "published_at": "2026-05-20T10:00:00Z",
      "title": "...",
      "description": "...",
      "body": "...",
      "language": "en",
      "source": { "id": 123, "domain": "example.com", "bias": "center", "rankings": { "opr": 7 } },
      "sentiment": { "overall": { "score": 0.42, "polarity": "positive" } },
      "entities": [],
      "categories": [],
      "topics": []
    }
  ]
}
```

A successful response has `status: "ok"`. Paginate by following `next_page`, or by
incrementing `page` while `has_next_pages` is `true`. The article `href` points at the
original publisher — link there, do not present the body as your own.

Watch the names: filter parameters and response fields are not always spelled the same.
You filter on `source.rank.opr.min`, but the value comes back as `source.rankings.opr`.
Sentiment is `{ score, polarity }` under `sentiment.overall`, `sentiment.title` and
`sentiment.body`. The full field list is at
https://docs.apitube.io/platform/news-api/response-structure

## Rate limits and errors

- Free: 10 requests per minute, 100 per day, results limited to the first 5 pages.
  Starter and above: 50 requests per minute.
- Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (seconds).
- `429 ER0203` — rate limit exceeded; back off until the reset. Repeated violations lead to
  a temporary ban (`ER0204`).
- `401 ER0230` — the key has expired. `403 ER0601` / `ER0602` / `ER0603` — the key is restricted
  by IP, referrer, or endpoint scope.
- Full list: https://docs.apitube.io/platform/news-api/http-response-codes

Check remaining quota at any time with `GET /v1/balance`.
