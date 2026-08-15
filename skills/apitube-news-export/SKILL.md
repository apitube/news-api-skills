---
name: apitube-news-export
description: Pull news datasets out of the APITube News API — CSV, XLSX, Parquet, JSONL, XML and RSS exports, field selection with fl, and safe pagination over large result sets. Use when an agent builds a dataset, a report or a backfill rather than a single query.
---

# Exporting news data from APITube

The same query that returns JSON can return a file. Add `export=` to any article endpoint and
the response comes back in that format instead.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Docs: https://docs.apitube.io/platform/news-api/everything

## Formats

`export` accepts: `json` (default), `jsonl`, `ndjson`, `csv`, `tsv`, `xml`, `rss`, `xlsx`, `parquet`.

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?topic.id=technology&published_at.start=NOW-7DAY&per_page=250&export=csv" \
  -o tech-week.csv
```

Which to pick:

| Format | Use it for |
|---|---|
| `jsonl` / `ndjson` | streaming into a pipeline line by line; append-friendly, no closing bracket to worry about |
| `parquet` | columnar analytics, DuckDB/pandas/Spark, smallest on disk for large pulls |
| `csv` / `tsv` | spreadsheets, quick loading into a database |
| `xlsx` | a report a human will open |
| `xml` | legacy ingestion pipelines |
| `rss` | a feed reader or a CMS that already consumes RSS |
| `json` | anything programmatic where you need nested fields intact |

Flat formats (`csv`, `tsv`, `xlsx`) cannot represent nested structures the way JSON does —
if you need `entities[]` with their per-entity sentiment, export `jsonl` or `parquet`.

Every JSON response also carries a ready-made `export` object with the same query pre-built in
each format, so you can hand a user a download link without assembling the URL yourself:

```json
"export": {
  "json": "https://api.apitube.io/v1/news/everything?...&export=json",
  "csv":  "https://api.apitube.io/v1/news/everything?...&export=csv",
  "xlsx": "…", "tsv": "…", "xml": "…", "rss": "…", "parquet": "…", "jsonl": "…"
}
```

## Take only the fields you need

`fl` is a comma-separated list of fields to include. It shrinks the response and speeds up
the transfer:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?entity.id=1580517&published_at.start=NOW-30DAY&fl=id,href,title,published_at,language,source.domain,sentiment.overall.score,sentiment.overall.polarity&export=jsonl&per_page=250" \
  -o tesla-30d.jsonl
```

Article bodies dominate the payload. Omitting `body` from `fl` typically cuts the download by
an order of magnitude — do it whenever you are aggregating rather than reading.

**`fl` does not narrow CSV columns.** The CSV and TSV exports always emit their full fixed
header (`"ID",Href,PublishedAT,Title,Description,Body,Language,…`); `fl` only decides which of
those columns get values, so the rest come out empty. When you want a narrow tabular file,
export `jsonl` and project the columns yourself, or post-process the CSV. `fl` behaves as
expected in `json`, `jsonl` and `parquet`.

## Pagination over a large pull

`per_page` maxes out at 250, and is capped at 10 on Free and 50 on Starter. Walk pages with
`page`, following `next_page` from the response, and stop when `has_next_pages` is `false`.

```bash
page=1
while : ; do
  curl -sS -H "Authorization: Bearer YOUR_API_KEY" \
    "https://api.apitube.io/v1/news/everything?topic.id=technology&published_at.start=2026-07-01&published_at.end=2026-07-31&per_page=250&page=${page}&export=jsonl" \
    >> july-tech.jsonl || break
  page=$((page+1))
  [ "$page" -gt 40 ] && break
done
```

Two rules keep a long pull honest:

1. **Slice by time, not by depth.** Deep pagination over one enormous query is slower and more
   fragile than many narrow queries. Split the range into days or weeks with
   `published_at.start` / `published_at.end` and pull each slice to its end.
2. **Fix the window before you start.** Use absolute dates rather than `NOW-30DAY` for a
   backfill: new articles arriving mid-run shift a relative window and can push results
   across page boundaries, so you get duplicates and gaps in the same run.

De-duplicate on article `id` after merging slices.

## Size the job before you run it

`/news/count` returns just the number of matches, so you can decide whether a pull is 400
articles or 400,000 before spending quota on it:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/count?topic.id=technology&published_at.start=2026-07-01&published_at.end=2026-07-31"
```

```json
{ "status": "ok", "count": 12345, "request_id": "req_abc123def456" }
```

Divide by `per_page` to get the number of requests, and check it against your plan's rate
limit (10 requests/minute on Free, 50 on paid plans) and daily quota before starting.

## Limits worth knowing before a bulk run

- **Free plan:** 10 requests/minute, 100 requests/day, 10 results per request, and results are
  limited to the first 5 pages — not a bulk-export tier.
- **Title search is windowed:** any query using `title` is limited to a 31-day
  `published_at` range. A wider explicit range returns `400 ER0110`; with no dates at all the
  last 31 days are searched and the response carries an `ER0366` warning in `meta.warnings`.
  Filters on taxonomy IDs, sources or languages have **no** range limit — prefer them for
  historical pulls.
- **Rate limiting:** `429 ER0203` means slow down; read `X-RateLimit-Reset` for the number of
  seconds to wait. Persistent violations lead to a temporary ban (`ER0204`).
- Check remaining quota mid-run with `GET /v1/balance`.

## Reuse and attribution

Exported articles carry `href` pointing at the original publisher. Keep it in your dataset:
it is your provenance record, and republishing article text without linking to the source is
a licensing problem, not a technical one. Check the terms at https://apitube.io before
redistributing content.

## Common questions

**Does the export count differently against quota?** No — the export format changes the
serialisation, not the query. A page of 250 articles costs the same in `parquet` as in `json`.

**Can I export from other endpoints?** Yes, `export` works on the article endpoints —
`/news/everything`, `/news/top-headlines` and the taxonomy endpoints.

**Why is my CSV missing entity data?** Nested arrays do not survive flattening. Use `jsonl`
or `parquet` when you need `entities[]`, `categories[]` or `topics[]`.
