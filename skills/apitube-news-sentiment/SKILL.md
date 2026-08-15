---
name: apitube-news-sentiment
description: Filter and analyse news sentiment with the APITube News API — overall, title and body polarity, per-entity sentiment, clickbait and title/body gap detection. Use when an agent needs tone-aware news data instead of running its own sentiment model.
---

# News sentiment with APITube

Every article indexed by APITube arrives with sentiment already scored, so you filter on tone
in the query instead of downloading text and running a model over it. Scores run from
**−1 (negative) to +1 (positive)**; polarity is `positive`, `negative` or `neutral`.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Docs: https://docs.apitube.io/platform/news-api/everything

## Three levels of sentiment

APITube scores the article three ways, and they can disagree — that disagreement is itself
a signal.

| Scope | Filter | Response field |
|---|---|---|
| Whole article | `sentiment.overall.polarity`, `sentiment.overall.score.min` / `.max` | `sentiment.overall.{score,polarity}` |
| Headline only | `sentiment.title.polarity`, `sentiment.title.score.min` / `.max` | `sentiment.title.{score,polarity}` |
| Article body | `sentiment.body.polarity`, `sentiment.body.score.min` / `.max` | `sentiment.body.{score,polarity}` |

Each also has an exact-match form without `.min` / `.max`: `sentiment.overall.score=0.5`.

```bash
# Strongly negative English coverage of a topic in the last 3 days
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?topic.id=technology&sentiment.overall.score.max=-0.4&language.code=en&published_at.start=NOW-3DAY&per_page=50"
```

### Select neutral coverage by score, not by polarity

`polarity=positive` and `polarity=negative` work as expected. **`polarity=neutral` currently
matches nothing** — neutral articles are stored without a polarity value, so the filter
returns zero results even though the articles exist and their responses show
`"polarity": "neutral"`. Use a score band instead:

```bash
# Neutral-toned coverage: score close to zero
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?language.code=en&sentiment.overall.score.min=-0.1&sentiment.overall.score.max=0.1&per_page=50"
```

The same applies to `sentiment.title.polarity` and `sentiment.body.polarity`. Filtering by
score is the reliable path in all three cases, and it lets you set your own neutral band.

## Sentiment toward a specific entity

Article-level sentiment answers "is this article negative". It does **not** answer "is this
article negative **about my company**" — a piece can be upbeat overall while criticising one
named organisation. For that, use per-entity sentiment:

- `entity.sentiment.polarity` — `positive` / `negative` / `neutral`
- `entity.sentiment.score.min` / `.max` — −1…1

Combine it with `entity.id` or a `*.name` filter to bind the sentiment to that entity.
Used on its own it matches an article where **any** extracted entity carries that sentiment.

```bash
# Articles that speak negatively about entity 1580517 specifically
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?entity.id=1580517&entity.sentiment.polarity=negative&published_at.start=NOW-7DAY"
```

In the response, each element of `entities[]` carries its own `sentiment` object, so you can
read the per-entity score directly:

```json
"entities": [
  { "id": 1580517, "name": "Tesla, Inc.", "type": "organization",
    "sentiment": { "score": -0.62, "polarity": "negative" } }
]
```

Related error codes when the values are malformed: `ER0290`, `ER0291`, `ER0292`.

## Headline vs body: clickbait and spin

Two derived filters compare the headline against the text:

- `sentiment.mixed=1` — title polarity differs from body polarity
- `sentiment.consistent=1` — title and body agree
- `sentiment_gap.min` / `sentiment_gap.max` — size of the gap between title and body score,
  range 0…2
- `is_clickbait=1` / `is_clickbait=0` — clickbait classification

```bash
# Alarmist headlines over calm text — a classic outrage-bait pattern
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?sentiment.mixed=1&sentiment_gap.min=0.8&language.code=en&per_page=25"
```

Use `sentiment.consistent=1` when you want a clean training or evaluation set and need the
headline to actually represent the article.

## Ranking and aggregating by tone

- `sort.by=sentiment.overall.score` with `sort.order=asc` puts the harshest coverage first;
  `desc` puts the most favourable first. `sentiment.title.score` and `sentiment.body.score`
  work the same way.
- `sort.by=controversy` surfaces polarising coverage rather than one-sided coverage.
- Faceting gives the distribution without downloading articles:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/everything?entity.id=1580517&published_at.start=NOW-30DAY&facet=1&facet.field=sentiment.overall.polarity,source.bias&per_page=1&fl=id"
```

**Facet values come back as raw codes, not labels.** Translate them yourself:

```json
"facets": {
  "sentiment.overall.polarity": [ { "value": null, "count": 37490 },
                                  { "value": 1, "count": 21926 },
                                  { "value": -1, "count": 13028 } ],
  "source.bias": [ { "value": 3, "count": 50263 }, { "value": 2, "count": 11265 },
                   { "value": 0, "count": 10067 }, { "value": 1, "count": 849 } ]
}
```

Polarity: `1` positive, `-1` negative, `null` neutral (no value stored). Bias: `0` left,
`1` center, `2` right, `3` unclassified — and `3` is usually the largest bucket, so exclude it
before computing a left/center/right split or the numbers will look wrong.

For a sentiment time series, use range faceting over `published_at`:

```
&facet.range=1&facet.range.field=published_at&facet.range.start=NOW-30DAY&facet.range.end=NOW&facet.range.gap=%2B1DAY
```

`facet.range.field` also accepts `sentiment.overall.score`, `sentiment.title.score` and
`sentiment.body.score` — that gives you a histogram of scores rather than a timeline.

## Reading the numbers honestly

- **Neutral is the majority class.** Most news writing is neutral in tone; a feed that looks
  85 % neutral is normal, not a bug. Compare against a baseline period, not against zero.
- **Score ≠ importance.** A −0.9 score on an obscure blog and on a national outlet are the
  same number. Weight by `source.rankings.opr` or filter with `source.rank.opr.min` before
  averaging.
- **Language matters.** Scoring quality varies by language; when you compare markets, compare
  each language against its own baseline rather than pooling them.
- **Sentiment is not stance.** A negative article about a competitor's recall is not positive
  coverage of you. Bind sentiment to an entity when the question is about a subject.

## Common questions

**Can I get sentiment without the article body?** Yes — add
`fl=id,published_at,source.domain,sentiment.overall.score,sentiment.overall.polarity`
to return only those fields. Much smaller responses when you aggregate over many articles.

**Is there a neutral band I can tune?** Polarity is assigned server-side. If you need a
different cut-off, filter on the raw score instead: `sentiment.overall.score.min=0.15`
treats everything below 0.15 as not-positive for your purposes.

**Which endpoints accept these filters?** All the article endpoints —
`/news/everything`, `/news/top-headlines`, `/news/count`, the taxonomy endpoints, and the
live `/news/stream` feed.
