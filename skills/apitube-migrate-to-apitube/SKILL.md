---
name: apitube-migrate-to-apitube
description: Port code from another news API to APITube — parameter mappings, response field translation and drop-in shims for NewsAPI.org, GNews, Mediastack, NewsData.io, Perigon, NewsCatcher, Marketaux, Aylien, Bing News, GDELT and others. Use when migrating an existing integration rather than writing a new one.
---

# Migrating to the APITube News API

APITube publishes a per-vendor migration kit for each major news API: every parameter mapped
and executed against both live APIs, a drop-in shim so your call sites stay unchanged, and an
explicit list of what has no equivalent. Start from the kit for your current provider rather
than re-deriving the mapping.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Kits: https://github.com/apitube

## Find the right kit

| Migrating from | Repository |
|---|---|
| NewsAPI.org (v2) | https://github.com/apitube/newsapi-org-apitube-migration-kit |
| NewsAPI.ai / Event Registry | https://github.com/apitube/newsapi-ai-apitube-migration-kit |
| GNews.io (v4) | https://github.com/apitube/gnews-apitube-migration-kit |
| Mediastack | https://github.com/apitube/mediastack-apitube-migration-kit |
| NewsData.io | https://github.com/apitube/newsdata-io-apitube-migration-kit |
| NewsCatcher (v3) | https://github.com/apitube/newscatcher-apitube-migration-kit |
| The News API | https://github.com/apitube/thenewsapi-apitube-migration-kit |
| World News API | https://github.com/apitube/worldnewsapi-apitube-migration-kit |
| Perigon | https://github.com/apitube/perigon-apitube-migration-kit |
| Marketaux | https://github.com/apitube/marketaux-apitube-migration-kit |
| Aylien / Quantexa News API (v6) | https://github.com/apitube/quantexa-apitube-migration-kit |
| Bing News Search (v7, retired) | https://github.com/apitube/bing-news-api-apitube-migration-kit |
| GDELT DOC 2.0 | https://github.com/apitube/gdelt-apitube-migration-kit |
| SerpApi `google_news` | https://github.com/apitube/serpapi-apitube-migration-kit |
| ScrapingBee news scraping | https://github.com/apitube/scrapingbee-apitube-migration-kit |

Each kit contains `reference/parameter-mapping.md`, `reference/response-mapping.md`,
`reference/limitations.md`, category and language mappings, runnable `examples/`, and a
`shim/` for Node and Python.

## What changes in every migration

**Authentication.** APITube takes the key three ways — `Authorization: Bearer KEY`,
`X-API-Key: KEY`, or `api_key=KEY` in the query string. Header names are case-sensitive in
the sense that matters: `X-Api-Key` becomes `X-API-Key`, `apiKey` becomes `api_key`.

**Dotted parameter names.** APITube uses dot notation — `source.country.code`,
`published_at.start`, `sentiment.overall.polarity`. Most providers use flat camelCase, so a
mechanical rename covers a large share of the work.

**Exclusion is a prefix, not a separate parameter.** Instead of `excludeDomains`, every filter
has an `ignore.` twin: `ignore.source.domain`, `ignore.language.code`, `ignore.entity.id`,
`ignore.title`.

**Comma-separated values cap at 3** for most list filters (5 for `event.type` and
`article.id`). Providers that allowed unlimited lists need the query split into several
requests.

**Article text arrives by default.** Many providers return a truncated `content` field and
expect you to scrape the rest. APITube returns `body`, so the scraping step in your pipeline
usually disappears along with its failure modes.

## Mapping example: NewsAPI.org v2

| NewsAPI.org | APITube | Notes |
|---|---|---|
| `q` | `title` | APITube searches titles; see the caveat below |
| `q` with boolean operators | `query` | Use the boolean query language instead |
| `searchIn=title` | `title` | Exact equivalent |
| `searchIn=description` / `content` | — | No equivalent |
| `sources` | `source.domain` | NewsAPI source IDs (`bbc-news`) are not domains |
| `domains` | `source.domain` | Max 3 values |
| `excludeDomains` | `ignore.source.domain` | Max 3 values |
| `from` / `to` | `published_at.start` / `published_at.end` | ISO 8601 works unchanged |
| `language` | `language.code` | ISO 639-1 |
| `country` | `source.country.code` | Lowercase |
| `category` | `category.id` | IPTC MediaTopic code — see the kit's category mapping |
| `sortBy=publishedAt` | `sort.by=published_at` | Also the APITube default |
| `sortBy=relevancy` | `sort.by=relevance` | Ranking differs |
| `sortBy=popularity` | `sort.by=source.rank.opr&sort.order=desc` | Open PageRank is the analogue |
| `pageSize` | `per_page` | Max 250, versus 100 |
| `page` | `page` | Both 1-based |

NewsAPI.org forbade combining `sources` with `country` or `category`. APITube has no such
restriction — combine `source.domain`, `source.country.code` and `category.id` freely.

## The one thing to check before you commit

**Full-text search semantics differ from most providers.** `title` searches article titles,
not the body. If your existing integration relies on matching words anywhere in the article
text, that behaviour does not carry over directly — plan for it rather than discovering it in
production. Two things to try instead:

- `query` — the boolean query language, for expressions with `AND` / `OR` / `NOT` and grouping.
  See https://docs.apitube.io/platform/news-api/query-builder
- Structured filters — `entity.id`, `topic.id`, `category.id`, `industry.id`, `event.type`.
  These usually replace keyword matching outright and match more reliably, because they run
  against extracted entities rather than string occurrences. Resolve IDs with `/suggest/*`.

Each kit's `reference/limitations.md` lists the rest of the gaps for that specific provider,
with no glossing over — read it before you plan the cutover.

## Suggested migration order

1. Read `reference/parameter-mapping.md` and `reference/limitations.md` in your kit.
2. Get a key at https://apitube.io and verify one real query against your current provider's
   output, side by side.
3. Drop in the kit's `shim/` so existing call sites keep their old signatures, and run your
   test suite against it.
4. Replace keyword filters with structured ones where the kit flags a semantic gap.
5. Remove the shim once call sites have been rewritten to native APITube parameters.

## Common questions

**My provider is not in the list.** The general mapping above plus the `apitube-news-api`
skill covers a new integration. Field-for-field parity questions go to
https://apitube.io/contact.

**Will my response parsing keep working?** No — field names differ. Each kit ships
`reference/response-mapping.md` for exactly this. The most common surprise: you filter on
`source.rank.opr.min`, but the value comes back as `source.rankings.opr`.

**How much history can I backfill?** Filters on taxonomy, source or language have no date
range limit. Anything using `title` search is capped at a 31-day window (`400 ER0110` beyond
that), so pull historical archives with structured filters.
