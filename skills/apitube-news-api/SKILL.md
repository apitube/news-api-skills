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

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `/news/everything` | GET · POST | Search all articles with full filtering |
| `/news/top-headlines` | GET · POST | Breaking and important news from sources with OPR rank >= 5 |
| `/news/count` | GET · POST | Number of matching articles only — one `count(*)`, no article payload |
| `/news/article` | GET · POST | A single article — pass `id=<articleId>`, not `article.id` (`400 ER0232` without it) |
| `/news/story/{articleId}` | GET · POST | The cluster of articles covering the same story |
| `/news/category/{taxonomy}/{categoryId}` · `/news/topic/{topicId}` · `/news/industry/{industryId}` · `/news/entity/{entityId}` | GET · POST | Articles by taxonomy |
| `/news/trends` | GET · POST | Aggregations: top values of a field, growth, time buckets |
| `/news/event-types` | **GET only** | The 44 classified event types and their categories |
| `/suggest/entities` · `/suggest/categories` · `/suggest/topics` · `/suggest/industries` | **GET only** | Autocomplete an ID from a name prefix |
| `/balance` | **GET only** | Remaining quota for the key |
| `/news/stream` | **GET only** | Live Server-Sent Events feed |

The article and aggregation endpoints take the same parameters either as a query string on
`GET` or as a JSON body on `POST` — use `POST` when a long filter set would blow past URL
length limits. The endpoints marked GET-only answer `404` to a `POST`, not `405`, so a wrong
method looks like a wrong path.

## Key parameters for `/news/everything`

Filters combine with AND. Comma-separated values inside one parameter combine with OR.

**The 3-value cap is enforced by silent truncation, not by an error.** Most list filters take
at most 3 comma-separated values (5 for `event.type` and `article.id`). Pass more and the
extras are dropped without any warning — the request still returns `200`:

```
entity.id=1297647                              → 126 957
entity.id=1580517,474,1223812                  →   2 110
entity.id=1580517,474,1223812,1297647          →   2 110   ← 4th ignored
entity.id=1297647,1580517,474,1223812          → 127 040   ← now the 4th is the ignored one
```

Only the **first three** are applied, so order decides which value you lose. Split larger sets
across several requests and merge on article `id`.

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
  **Resolve them with `/suggest/*` first — do not guess.** The ID types differ: entities and
  industries are integers (`1580517`), topics are dotted slugs (`industry.technology_news`),
  categories are IPTC MediaTopic codes (`medtop:04000000`). A guessed value returns
  `400 ER0208`, not an empty result — `topic.id=technology`, for instance, does not exist.
- `person.name`, `organization.name`, `location.name`, `brand.name`, `event.name`,
  `disease.name`, `disaster.name`, `sport.name` — filter by entity name without looking up an ID.
- `source.domain`, `source.id`, `source.country.code`, `source.bias` (`left`, `center`, `right`),
  `source.rank.opr.min` / `.max` (Open PageRank authority), `is_premium_source`, `is_verified_source`.
- `sentiment.overall.polarity` — `positive` and `negative` work; **`neutral` currently matches
  nothing**, because neutral articles are stored without a polarity value even though their
  responses show `"polarity": "neutral"`. For neutral coverage use a score band instead:
  `sentiment.overall.score.min=-0.1&sentiment.overall.score.max=0.1`. The score filters
  (`sentiment.overall.score.min` / `.max`, range −1…1) are reliable in all three cases, and
  `sentiment.title.*` / `sentiment.body.*` behave the same way.
- `is_breaking`, `is_paywall`, `is_duplicate`, `is_high_quality`, `has_author`, `has_image`, `has_video`.
- `event.type` — one of 44 classified event types (`ipo`, `layoffs`, `funding-round`,
  `data-breach`, `earthquake`, …), up to 5; `event.category` is `business`, `society` or `environment`.
- `sort.by` — `published_at` (default), `relevance`, `engagement`, `quality`, `controversy`,
  `trust`, `source.rank.opr`, `sentiment.overall.score`, `read_time`, and more.
  `sort.order` is `asc` or `desc`.
- `page`, `per_page` — pagination. The ceiling is set by your plan: 10 on Free, 50 on Starter,
  250 above that. Exceeding it is a hard `400 ER0171` ("Limit is out of range. Your plan allows
  up to N results per page"), not a silent clamp — read N from the message to learn your cap.
- `fl` — comma-separated list of fields to return, e.g. `fl=id,title,published_at,source.domain`.
  Cuts response size sharply on large pages.
- `article.id` — fetch specific articles by ID through `/news/everything`, up to 5
  comma-separated.
- `query` — boolean search language with `AND` / `OR` / `NOT` and grouping, for expressions
  `title` cannot express: https://docs.apitube.io/platform/news-api/query-builder

### Exclusions: only 21 filters have an `ignore.` twin

`ignore.<filter>` excludes instead of includes, but **only these 21 exist**:

```
ignore.title              ignore.language.code       ignore.category.id
ignore.topic.id           ignore.industry.id         ignore.entity.id
ignore.source.id          ignore.source.domain       ignore.source.country.code
ignore.source.bias        ignore.author.id           ignore.author.name
ignore.person.name        ignore.organization.name   ignore.location.name
ignore.brand.name         ignore.event.name          ignore.event.type
ignore.disaster.name      ignore.disease.name        ignore.sport.name
```

Anything else — `ignore.is_breaking`, `ignore.has_image`, `ignore.sentiment.overall.polarity`
— is **silently dropped**, with no error and an unchanged result count. There is no negation
for sentiment, media, readability, read-time, geo or the boolean flags; invert those by
selecting the complement instead (`has_image=0` rather than `ignore.has_image=1`).

A working exclusion changes the count by exactly the excluded set: with 71 857 matching
articles and 9 from `theclemsoninsider.com`, `ignore.source.domain=theclemsoninsider.com`
returns 71 848.

Example — English AI coverage from the last week, quality sources only:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  'https://api.apitube.io/v1/news/everything?title=%22artificial%20intelligence%22&language.code=en&published_at.start=NOW-7DAY&source.rank.opr.min=5&sort.by=published_at&per_page=10'
```

**URL-encode the value.** A raw quote or space in `title` makes the server return an *empty
body* rather than an error — `"artificial intelligence"` must be sent as
`%22artificial%20intelligence%22`. If a request comes back with nothing at all, an unencoded
character in the query string is the first thing to check.

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
