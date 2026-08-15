---
name: apitube-news-api
description: Query the APITube News API — authenticate, search and filter worldwide news articles by keyword, entity, sentiment, source, country and date, and parse the JSON response. Use when an agent needs live or historical news data, media monitoring, or news for a RAG pipeline.
license: MIT
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

Keys prefixed `api_test_` run against the same data and do not consume quota, so use one while
wiring up an integration. Only `body` and `body_html` are truncated, and the cut is marked with
`[Test mode — use a live key for full content]`; title, description, entities, sentiment,
categories and source come through in full. The Free plan truncates the same two fields, with
the marker `[Upgrade subscription plan]`. Check for those strings to detect a truncated body
instead of guessing from its length.
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
| `/news/local` | GET · POST | Geo search — `lat` + `lng` + `radius` in km |
| `/news/raw` | GET · POST | Unenriched articles: `id`, `title`, `href`, `created_at`, `description`, `body`, `body_html`, `author`, `keywords`, `source` only — no sentiment, entities or taxonomy |
| `/news/event-types` | **GET only** | The 44 classified event types and their categories |
| `/people` · `/companies` · `/journalists` | **GET only** | Reference profiles, not articles: `id`, `name`, `links`, `profile` (`outlets` for journalists) |
| `/suggest/entities` · `/suggest/categories` · `/suggest/topics` · `/suggest/industries` | **GET only** | Autocomplete an ID from a name prefix |
| `/balance` | **GET only** | Remaining quota for the key |
| `/news/stream` | **GET only** | Live Server-Sent Events feed |

The article and aggregation endpoints take the same parameters either as a query string on
`GET` or as a JSON body on `POST` — use `POST` when a long filter set would blow past URL
length limits. The endpoints marked GET-only answer `404` to a `POST`, not `405`, so a wrong
method looks like a wrong path.

`/news/stream` answers `200` with `content-type: text/event-stream` and takes the same filters.
Between matches it emits SSE **comment** lines — the literal bytes `: heartbeat\n\n` — not
events. A parser that assumes every frame is `event:`/`data:` will choke on them, so drop lines
beginning with `:`. A stream whose filters match nothing sends only heartbeats and delivers no
articles. Streaming is billed per delivered article, so narrow the filters rather than opening
a second connection.

## Key parameters for `/news/everything`

Filters combine with AND. Comma-separated values inside one parameter combine with OR.

**The per-parameter value cap is enforced by silent truncation, not by an error.** Most list
filters take at most 3 comma-separated values; `event.type` and `article.id` take 5. Pass more
and the extras are dropped with no warning — the request still returns `200`:

```
entity.id=1297647                              → 126 957
entity.id=1580517,474,1223812                  →   2 110
entity.id=1580517,474,1223812,1297647          →   2 110   ← 4th ignored
entity.id=1297647,1580517,474,1223812          → 127 040   ← now the 4th is the ignored one

event.type=ipo,layoffs,bankruptcy,recall,data-breach              → 26 511
event.type=ipo,layoffs,bankruptcy,recall,data-breach,earthquake   → 26 511   ← 6th ignored
event.type=earthquake                                             →  9 536   ← it does match on its own
```

Only the first N are applied, where N is that parameter's cap, so **order decides which value
you lose**. Split larger sets across several requests and merge on article `id`.

- `title` — keyword search in article titles, 2–100 chars; outside that range it is
  `400 ER0007`, whose message reads "between 1 and 100 characters" but rejects a single
  character anyway. Quotes for an exact phrase, `"climate change"~2` for proximity. Percent-
  encode the value. **Title search is limited to a 31-day `published_at`
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
  `disease.name`, `disaster.name`, `sport.name` — filter by entity name instead of an ID.
  The name must match the dictionary exactly, and a miss is a **`400` error, not an empty
  result**: `location.name=New York` → `ER0218`, `brand.name=iPhone` → `ER0222`,
  `sport.name=football` → `ER0250`, `event.name=Olympics` → `ER0228`,
  `disaster.name=flood` → `ER0224`. `person.name=Elon Musk` and
  `organization.name=Google` do resolve. Confirm a name with `/suggest/entities` before
  relying on it, or filter by `entity.id`.
- `source.domain`, `source.id`, `source.country.code`, `source.bias` (`left`, `center`, `right`),
  `source.rank.opr.min` / `.max` — Open PageRank authority. In practice the data tops out at 8:
  over a two-day window `min=6` matched 39 431 articles, `min=7` matched 3 940, `min=8` matched
  80, and `min=9` matched none. Use 5 for "mainstream", 7 for "major outlet"; asking for 9+
  silently returns nothing.
- `sentiment.overall.polarity` — `positive` and `negative` work; **`neutral` currently matches
  nothing**, because neutral articles are stored without a polarity value even though their
  responses show `"polarity": "neutral"`. For neutral coverage use a score band instead:
  `sentiment.overall.score.min=-0.1&sentiment.overall.score.max=0.1`. The score filters
  (`sentiment.overall.score.min` / `.max`, range −1…1) are reliable in all three cases, and
  `sentiment.title.*` / `sentiment.body.*` behave the same way.
- `is_breaking`, `is_duplicate`, `is_high_quality`, `has_author`, `has_image`, `has_video`,
  `is_premium_source`, `is_verified_source`, `is_clickbait` — all verified to filter.
  **`is_paywall` is dead**: `is_paywall=1` returns zero articles across the whole index, and
  `is_paywall=0` returns everything, so it cannot be used to avoid paywalled links.
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
(`text`, `applied`, `ignored`, `cached`, `model`). Explicit parameters always win over the prompt.

It is accepted on `/news/everything`, `/news/top-headlines`, `/news/count`, `/news/trends`,
`/news/local`, `/news/raw`, `/news/stream` and the four taxonomy endpoints. On `/news/story`
and `/news/article` it is **silently ignored** — no error, just an unfiltered result. On
`/news/raw` most of what a prompt produces cannot apply either, since that endpoint only
understands dates, sorting and pagination.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?prompt=Tesla+and+Elon+Musk+news+in+English+for+the+last+10+days&per_page=10"
```

It costs 2 extra points on a cache miss (translations are cached), and it requires the Basic
plan or above — Free and Starter get `403 ER0706`. Other errors: `ER0800` (length out of range),
`ER0801` (translation service down), `ER0802` (nothing usable in the wording).

## Response shape

### Envelope

Eleven keys, always the same:

```json
{
  "status": "ok",
  "limit": 2,
  "path": "http://api.apitube.io/v1/news/everything?language.code=en&per_page=2",
  "page": 1,
  "has_next_pages": true,
  "next_page": "http://api.apitube.io/v1/news/everything?language.code=en&per_page=2&page=2",
  "has_previous_page": false,
  "previous_page": "",
  "export": { "json": "http://…&export=json", "xlsx": "…", "csv": "…", "tsv": "…",
              "xml": "…", "rss": "…", "parquet": "…", "jsonl": "…" },
  "request_id": "ad527b55-4083-4174-a0b6-0a0a1f7adc8b",
  "results": [ /* … */ ]
}
```

**`path`, `next_page` and every `export` URL come back as `http://`, not `https://`.** Rewrite
the scheme before following one, or your pagination loop downgrades to plaintext HTTP. Safer
still: keep your own base URL and just increment `page` while `has_next_pages` is `true`.
`previous_page` is `""` — an empty string, not `null` — on the first page.

On failure `status` is `"not_ok"` and the body carries an `errors` array instead of `results`:
`[{ "status": 400, "code": "ER0208", "message": "…", "links": { "about": "…" }, "timestamp": "…" }]`.
Branch on `status`, not on the HTTP code alone.

### Article object

Every article carries the same **33 fields** — the set does not vary between articles.
Abridged, with the real shapes:

```json
{
  "id": 3072206097,
  "href": "https://gulfnews.com/world/asia/india/…",
  "published_at": "2026-08-15T06:39:23.000Z",
  "title": "UAE-India: A generational partnership for a shared future",
  "description": "On the auspicious occasion of the 80th Independence Day…",
  "body": "…plain text…",
  "body_html": "<div><div><p>…</p></div></div>",
  "language": "en",
  "translations": { "en": { "title": null, "description": null } },
  "author": { "id": 20059328, "name": "A GN Focus Report" },
  "image": "https://media.assettype.com/…",
  "categories": [ { "id": 268, "name": "emerging market", "score": 0.8,
                    "taxonomy": "iptc_mediatopics",
                    "links": { "self": "…/news/category/iptc_mediatopics/medtop:20001248" } } ],
  "topics":     [ { "id": "industry.green_energy_news", "name": "Green Energy Industry News",
                    "score": 0.9, "links": { "self": "…" } } ],
  "industries": [ { "id": 444, "name": "Credit Unions", "links": { "self": "…" } } ],
  "entities":   [ { "id": 1751049, "name": "IHC (International Holding Company)",
                    "type": "organization", "frequency": 1,
                    "links": { "self": "…", "wikipedia": "…", "wikidata": "…" },
                    "sentiment": { "score": 0, "polarity": "neutral",
                                   "mentions": { "positive": 0, "neutral": 1, "negative": 0 } },
                    "title": { "pos": [] },
                    "body":  { "pos": [ { "start": 4510, "end": 4539 } ] },
                    "metadata": { "type": "business", "subtype": "company", "aliases": [],
                                  "country": { "code": "AE", "name": "United Arab Emirates" },
                                  "founded": { "year": 1998 }, "description": "…" } } ],
  "locations_mentioned": [ { "name": "Abu Dhabi", "country": "AE",
                             "lat": 24.45111, "lng": 54.39694, "type": "city" } ],
  "source": { "id": 9186, "domain": "gulfnews.com",
              "home_page_url": "https://gulfnews.com", "type": "news", "bias": "left",
              "rankings": { "opr": 6 },
              "location": { "country_name": "United Arab Emirates", "country_code": "ae" },
              "favicon": "https://www.google.com/s2/favicons?domain=https://gulfnews.com" },
  "sentiment": { "overall": { "score": 0.11, "polarity": "positive" },
                 "title":   { "score": 0.13, "polarity": "positive" },
                 "body":    { "score": 0.09, "polarity": "positive" } },
  "summary": [ { "sentence": "…", "sentiment": { "score": 0.14, "polarity": "positive" } } ],
  "keywords": [], "links": [], "media": [],
  "readability": { "flesch_kincaid_grade": 17, "flesch_reading_ease": 17.9,
                   "automated_readability_index": 17.5, "difficulty_level": "expert",
                   "target_audience": "academic", "reading_age": 22,
                   "avg_words_per_sentence": 25, "avg_syllables_per_word": 1.9 },
  "story":  { "id": 3072206097, "uri": "https://api.apitube.io/v1/news/story/3072206097" },
  "shares": { "total": 6, "facebook": 3, "twitter": 2, "reddit": 1 },
  "is_duplicate": false, "is_free": true, "is_breaking": false,
  "read_time": 8, "sentences_count": 42, "paragraphs_count": 1,
  "words_count": 1049, "characters_count": 7163
}
```

Notes that matter when you parse this:

- `entities[]` carries **per-entity sentiment** (`score`, `polarity`, and a `mentions`
  breakdown) and `frequency`, the mention count. This is the field to read when you need
  "how did this article talk about *this* company", as opposed to the article-level
  `sentiment`.
- **Do not use `entities[].body.pos` to highlight text.** The `start`/`end` offsets are
  measured against an upstream version of the article that the API does not return; they line
  up with neither `body` nor `body_html`, and the drift is not a constant you can correct for
  (4, 7, 19, 23, 53 characters across entities in a single article). Some entities have offsets
  while their name does not appear in `body` at all. Locate mentions by searching `body`
  yourself.
- `image` is a plain string URL, while `media` is a separate array — an article can have an
  `image` and an empty `media`. When there is no image the value is `""`, never `null`.
- `author` is always an object, never a string and never `null`. With no byline it is
  `{"id": null, "name": ""}` — test `author.name`, not `author`, or every anonymous article
  looks attributed.
- `translations.en.{title,description}` are `null` for articles already in English.
- `keywords`, `links` and `media` are empty arrays roughly a third of the time, and `topics` is
  empty for about half of all articles. `entities` and `categories` are reliably populated.
  Do not treat an empty array as a parsing failure.
- `story.uri` is a ready-made link to `/news/story/{articleId}` for this article. **`story.id` is not
  a cluster key** — it always equals the article's own `id`, including for articles returned
  together by `/news/story`. You cannot group articles by comparing `story.id`; call
  `/news/story/{articleId}` and use the returned set.

The article `href` points at the original publisher — link there, do not present the body as
your own.

### Do not feed response values straight back into filters

Filter parameters and response fields are not always spelled the same, and for categories they
do not even hold the same value.

| You filter on | It comes back as |
|---|---|
| `source.rank.opr.min` | `source.rankings.opr` |
| `category.id=medtop:20001248` (IPTC code) | `categories[].id` = `11` (internal integer) |
| `sentiment.overall.polarity` | `sentiment.overall.{score, polarity}` |

Passing a response category ID back as a filter fails: `category.id=11` returns
`400 ER0206 "category with ID '11' not found"`, while `category.id=medtop:20001248` returns
299 articles. The IPTC code for a category is in its `links.self`
(`/news/category/iptc_mediatopics/medtop:20001248`) — take it from there, or resolve names
with `/suggest/categories`. Entities (`entities[].id`) and industries (`industries[].id`) are
integers in both directions and round-trip cleanly; topics are dotted slugs in both.

The full field list is at https://docs.apitube.io/platform/news-api/response-structure

## Rate limits and errors

- Free: 10 requests per minute, 100 per day, and pagination stops after page 5 —
  `page=6` returns `400 ER0173`. Starter and above: 50 requests per minute. Test keys
  (`api_test_`) get their own 15 per minute.
- Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (seconds).
- `429 ER0203` — rate limit exceeded; back off until the reset. Repeated violations lead to
  a temporary ban (`ER0204`).
- `401 ER0230` — the key has expired. `403 ER0601` / `ER0602` / `ER0603` — the key is restricted
  by IP, referrer, or endpoint scope.
- Full list: https://docs.apitube.io/platform/news-api/http-response-codes

Check remaining quota at any time with `GET /v1/balance`.
