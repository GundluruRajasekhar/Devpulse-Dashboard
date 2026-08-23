# Blueprint — Connecting Node.js to the Python Analytics Service

This is the piece that makes DevPulse a genuine polyglot microservice system rather than a single monolith wearing two languages. The rule is simple and load-bearing throughout the codebase:

> **Node.js owns every database connection. Python never touches MySQL or MongoDB directly — it only receives JSON and returns JSON.**

This keeps the Python service horizontally scalable and swappable (you could rewrite it in any language without touching data-access code), keeps a single source of truth for credentials, and keeps the "smart" computation (pandas, AI calls) cleanly separated from request handling.

## The call path

```
Browser (Next.js)
   │  GET /api/analytics/productivity?userId=2&days=14
   ▼
Node.js API Gateway (backend/src/controllers/analyticsController.js)
   │  1. Query MongoDB directly for raw commit_logs in range
   │  2. POST the raw records to the Python service
   ▼
Python Analytics Service (analytics-service/app/routers/insights.py)
   │  3. pandas aggregation → velocity trend, consistency score, insights
   ▼
Node.js API Gateway
   │  4. Return Python's JSON response straight to the browser
   ▼
Browser renders ProductivityInsightsWidget.jsx
```

## Why the boundary sits here

| If Python owned the DB connection... | With Node owning the DB connection... |
|---|---|
| Two services need MySQL/Mongo credentials, doubling the attack surface. | Only Node holds credentials; Python holds nothing sensitive. |
| Schema changes require updating data-access code in two languages. | Schema changes only touch Node's models. |
| Python service can't be safely exposed or reused by other consumers. | Python service is a pure function of its input — trivially reusable, testable, and cacheable. |

## Minimal working example

**Node → Python (backend/src/controllers/analyticsController.js):**
```js
const { data } = await axios.post(`${ANALYTICS_URL}/insights/productivity`, {
  userId: Number(userId),
  windowDays: Number(days),
  commits: commits.map(c => ({
    authoredAt: c.authoredAt,
    additions: c.additions,
    deletions: c.deletions,
    churnScore: c.churnScore,
    isMerge: c.isMerge,
  })),
}, { timeout: 8000 });
```

**Python receiving it (analytics-service/app/routers/insights.py):**
```python
@router.post("/insights/productivity", response_model=ProductivityResponse)
def productivity_insights(payload: ProductivityRequest):
    stats = analyze_commits([c.model_dump() for c in payload.commits], payload.windowDays)
    return ProductivityResponse(..., insights=build_insights(stats))
```

Pydantic (`app/models/schemas.py`) validates the payload shape on the way in, so a malformed request from Node fails fast with a 422 rather than corrupting a pandas DataFrame silently.

## Failure isolation

`analyticsController.js` wraps the Python call in try/catch and returns HTTP 200 with `{ degraded: true }` on any failure — timeout, connection refused, bad response — rather than propagating a 500 to the browser. The dashboard should never go blank because a Python worker restarted. The same pattern applies to the AI layer *inside* Python: `ai_insights.py` catches any Anthropic API failure and falls back to a deterministic heuristic summary, so a missing or rate-limited API key degrades quality, not availability.

## Extending this pattern

Adding a new Python-computed metric (e.g. "estimated PR review turnaround"):
1. Add the raw fields to a new or existing Mongoose model.
2. Add a Node controller endpoint that queries Mongo and POSTs to a new Python route.
3. Add the Python route + Pydantic schemas + a `services/*.py` module with the actual computation.
4. Add a frontend widget that calls the new Node endpoint via `lib/api.js`, with a mock-data fallback following the existing pattern.
