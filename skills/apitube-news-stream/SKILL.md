---
name: apitube-news-stream
description: Receive new news articles by push with the APITube News API — Server-Sent Events streaming with resume and freshness windows, or signed webhooks with HMAC verification and retries. Use when an agent needs a continuous feed instead of polling.
---

# Push delivery: SSE and webhooks

Two ways to be told about a new article instead of asking for it. **SSE** keeps a connection
open and is best inside a running process. **Webhooks** post to your URL and are best when
nothing of yours is running all the time.

- Base URL: `https://api.apitube.io/v1`
- Authentication and the full filter list: see the `apitube-news-api` skill
- Docs: https://docs.apitube.io/platform/news-api/sse-stream and
  https://docs.apitube.io/platform/news-api/webhooks

## Server-Sent Events

```bash
curl -N -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/stream?language.code=en&entity.id=1580517"
```

The stream accepts the same filters as `/news/everything`, so the feed is already narrowed
server-side. The server polls for new matches on a fixed interval and sends a heartbeat
between batches, so a quiet feed still proves the connection is alive.

```javascript
const source = new EventSource(
  'https://api.apitube.io/v1/news/stream?language.code=en&api_key=YOUR_API_KEY'
);

source.addEventListener('article', event => {
  const article = JSON.parse(event.data);
  console.log(article.id, article.title, article.href);
});

source.addEventListener('error', event => {
  // { "code": "ER0301", "message": "Balance exhausted" } and friends
  console.error(event.data);
});
```

### Resume after a disconnect

Every `article` event carries an event ID. On reconnect, send the last one you processed in
the `Last-Event-ID` header and the stream continues from there instead of replaying or
skipping. Browser `EventSource` does this automatically; a custom client must send the
header itself.

### Drop backfill with `stream.max_age`

`stream.max_age` is a rolling freshness window **in minutes**. Articles published longer ago
than that are dropped before delivery — and before billing:

```bash
curl -N -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/stream?language.code=en&stream.max_age=15"
```

An invalid value (not an integer, or below 1) is rejected on connect with `400 ER0363`.
Use it whenever your pipeline only cares about live articles; without it, a freshly opened
stream may deliver recently indexed older articles.

Delivery is ordered by publication time within each polled batch. Article age still includes
the lag between publication and indexing.

### Connection limits and stuck slots

Concurrent SSE connections are capped per plan; opening more returns `429 ER0360`. A client
killed without closing its socket can hold a slot until the connection times out, so a
restart loop can lock you out of your own quota. Clear it deliberately:

```bash
# What is currently open on this key's account
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/stream/sessions"

# Close them all (optionally ?session_id= or ?api_key_id=)
curl -X DELETE -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.apitube.io/v1/news/stream/sessions"
```

Closed clients receive an `error` event with code `ER0364`. Neither session endpoint is billed.
Call the DELETE before a scheduled redeploy and you will never see `ER0360` from your own
ghosts.

### Stream error codes

| Code | Meaning |
|---|---|
| `ER0301` | Balance exhausted mid-stream; the connection closes |
| `ER0360` | Too many concurrent connections for the plan |
| `ER0361` | API key revoked — sessions dropped |
| `ER0362` | Maximum session lifetime reached; reconnect |
| `ER0363` | Invalid `stream.max_age` |
| `ER0364` | Session terminated through the sessions endpoint |

Streaming is billed **per delivered article**, not per connection. Two parallel streams with
overlapping filters bill the same article once each — narrow the filters instead of opening
more connections.

## Webhooks

A webhook subscription posts matching articles to your URL as they are indexed. Create and
manage subscriptions in the dashboard at https://dashboard.apitube.io/webhooks — the signing
secret is shown **once** on creation, so store it immediately. Filters are built with the same
controls as the API; leave them empty to receive every new article.

Existing subscriptions can be inspected and changed over HTTP:

```bash
curl -H "X-API-Key: YOUR_API_KEY" https://api.apitube.io/v1/webhooks
curl -H "X-API-Key: YOUR_API_KEY" https://api.apitube.io/v1/webhooks/42
curl -X PATCH -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"status":"paused"}' https://api.apitube.io/v1/webhooks/42
curl -X DELETE -H "X-API-Key: YOUR_API_KEY" https://api.apitube.io/v1/webhooks/42
```

### Delivery format

```http
POST https://your-app.example.com/webhook
Content-Type: application/json
X-Webhook-Signature: sha256=<hmac>
X-Webhook-Id: 42
X-Webhook-Delivery-Id: <unique per attempt>
X-Webhook-Timestamp: 1718800000
```

```json
{
  "event": "articles.new",
  "webhook_id": 42,
  "delivered_at": "2026-06-19T12:00:00.000Z",
  "articles": [ { "id": 3036291250, "title": "…", "source": { "domain": "reuters.com" } } ]
}
```

Articles have the same shape as `/news/everything` results, up to 100 per delivery. Respond
with any `2xx` to acknowledge. A non-`2xx` response or a timeout after 30 seconds fails the
delivery and schedules a retry; after 5 failed attempts the delivery is marked failed, and
repeated failures disable the subscription until you re-enable it in the dashboard.

De-duplicate on `X-Webhook-Delivery-Id`: a retried batch carries the same delivery ID.

### Verify the signature before trusting the payload

The signature is HMAC-SHA256 over `timestamp + "." + rawBody` using the subscription's
signing secret.

```javascript
import crypto from 'node:crypto';

function verifyWebhook(secret, signature, timestamp, rawBody) {
    const expected = crypto
        .createHmac('sha256', secret)
        .update(timestamp + '.' + rawBody)
        .digest('hex');
    return signature === `sha256=${expected}`;
}
```

```python
import hmac, hashlib

def verify_webhook(secret, signature, timestamp, raw_body):
    expected = hmac.new(
        secret.encode(), (timestamp + '.' + raw_body).encode(), hashlib.sha256
    ).hexdigest()
    return signature == f'sha256={expected}'
```

Compute the HMAC over the **raw** request body, before any JSON parse-and-re-serialise —
re-serialising changes whitespace and key order and breaks the comparison.

## Choosing between them

| | SSE | Webhooks |
|---|---|---|
| You run a long-lived process | yes | not required |
| Public HTTPS endpoint needed | no | yes |
| Survives your process restarting | no, reconnect with `Last-Event-ID` | yes, retries are queued |
| Ordering | per batch, by publication time | per delivery |
| Limits | concurrent connections per plan | number of subscriptions per plan |

Test either one with an `api_test_` key first: test deliveries are masked and do not consume
quota.
