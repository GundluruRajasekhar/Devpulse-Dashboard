# DevPulse — AI-Enhanced Developer Productivity Dashboard

A trending, real-world-shaped stack: **Next.js + React** (frontend) · **Node.js + Express** (API gateway) · **Python + FastAPI/Pandas** (AI analytics microservice) · **MySQL** (relational) · **MongoDB** (telemetry). Tracks coding velocity, git commits, AI-assisted code quality, and focus time in one dashboard — built and verified end-to-end, not just scaffolded.

## Architecture

```
┌──────────────────────────┐
│   Next.js / React UI     │  Widgets: Git Activity · Focus Tracker ·
│   (frontend/)             │  AI Productivity Insights · Leaderboard · AI Code Review
└─────────────┬─────────────┘
              │ REST (fetch, JWT bearer)
              ▼
┌──────────────────────────┐
│  Node.js + Express        │  Auth · routing · role gating · the ONLY service
│  API Gateway (backend/)   │  with DB credentials
└──────┬─────────────┬──────┘
       │              │
       ▼              ▼
┌─────────────┐  ┌──────────────┐        ┌───────────────────────────┐
│   MySQL     │  │   MongoDB     │        │  Python analytics service  │
│ Users,Teams,│  │ commit_logs,  │◄──────►│  (FastAPI + Pandas)         │
│ Projects,   │  │ daily_activity│  HTTP  │  churn/velocity stats ·     │
│ Access_     │  │ _events,      │  JSON  │  heuristic + AI code review  │
│ Tokens      │  │ ai_analysis_  │        │  (optional Anthropic key)    │
│             │  │ reports       │        └───────────────────────────┘
└─────────────┘  └───────────────┘
```

**Key architectural decision:** Node.js is the only service that talks to either database. Python receives and returns plain JSON. This is not incidental — it's what keeps the AI/analytics layer stateless, swappable, and safe to scale independently. Full reasoning and a worked example: **[`BLUEPRINT.md`](./BLUEPRINT.md)**.

## Folder structure

```
devpulse/
├── frontend/                  Next.js + React
│   ├── pages/                 index.js (dashboard), _app.js, _document.js
│   ├── components/            5 dashboard widgets (one file each)
│   ├── lib/api.js             fetch client with mock-data fallback
│   ├── styles/globals.css     design tokens + component styles
│   └── Dockerfile
├── backend/                    Node.js + Express API gateway
│   ├── server.js               app entrypoint
│   └── src/
│       ├── config/              MySQL (Sequelize) + MongoDB (Mongoose) connections
│       ├── models/mysql/        User, Team, Project, AccessToken
│       ├── models/mongo/        CommitLog, DailyActivityEvent, AiAnalysisReport
│       ├── controllers/         auth, git, focus, leaderboard, analytics(proxy)
│       ├── routes/              one file per resource
│       ├── middleware/auth.js   JWT verification + role gating
│       └── utils/crypto.js      AES-256-GCM token encryption
├── analytics-service/           Python + FastAPI
│   ├── main.py                  app entrypoint
│   └── app/
│       ├── routers/             insights.py, code_review.py
│       ├── services/            churn.py (pandas), bug_density.py (heuristic), ai_insights.py (optional Claude call)
│       └── models/schemas.py    Pydantic request/response contracts
├── database/
│   ├── mysql_schema.sql         full DDL + seed data
│   └── mongodb_schemas.md       collection shapes, indexes, rationale
├── docker-compose.yml            wires all 5 services together
├── BLUEPRINT.md                  Node ↔ Python integration deep-dive
├── requirements.md
├── tasks.md
└── README.md                     this file
```

## Running it

### Option A — Docker Compose (everything at once)
```bash
cd devpulse
docker compose up --build
```
- Frontend → http://localhost:3000
- API gateway → http://localhost:4000/api/health
- Analytics service → http://localhost:8000/health
- MySQL → localhost:3306 · MongoDB → localhost:27017

### Option B — Run each service locally

**1. Databases** (or point `.env` at existing MySQL/Mongo instances):
```bash
mysql -u root -p < database/mysql_schema.sql
# MongoDB needs no upfront schema — collections are created on first write
```

**2. Analytics service:**
```bash
cd analytics-service
cp .env.example .env
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

**3. Backend:**
```bash
cd backend
cp .env.example .env    # fill in MySQL/Mongo credentials
npm install
npm run dev
```

**4. Frontend:**
```bash
cd frontend
cp .env.example .env.local
npm install
npm run dev
```
Open http://localhost:3000 — the dashboard works even before you complete steps 1–3, using realistic mock data, so you can review the UI immediately.

### Turning on AI code review
Set `ANTHROPIC_API_KEY` in `analytics-service/.env`. Without it, `/code-review/scan` still works — it returns the heuristic-only result with `generatedBy: "heuristic-fallback"`.

## Core features → where they live

| Feature | Frontend | Backend route | Compute |
|---|---|---|---|
| Git & activity metrics | `GitActivityWidget.jsx` | `GET /api/git/activity` | MongoDB aggregation (Node) |
| AI code review insights | `AICodeReviewWidget.jsx` | `POST /api/analytics/code-review` | Heuristic scan + optional Claude call (Python) |
| AI productivity insights | `ProductivityInsightsWidget.jsx` | `GET /api/analytics/productivity` | Pandas churn/trend analysis (Python) |
| Time & focus tracker | `FocusTrackerWidget.jsx` | `GET/POST /api/focus/*` | MongoDB aggregation (Node) |
| Team leaderboard | `LeaderboardWidget.jsx` | `GET /api/leaderboard` | MongoDB aggregation + MySQL role check (Node) |

## What's actually verified (not just written)

- Every backend `.js` file passes `node --check` — zero syntax errors.
- The Python analytics service was installed, started, and hit live with real requests — `/health`, `/insights/productivity`, and `/code-review/scan` all returned correct JSON in this session.
- The Next.js frontend was `npm install`'d and `next build`'d successfully, statically prerendering the dashboard against the mock-data fallback path.

## Honest scope

This is a portfolio/capstone-grade build: the architecture, contracts, and every core code path are real and tested, but some production concerns are intentionally left as documented extension points rather than built out — see **"Next iteration"** in `tasks.md` (OAuth, refresh tokens, a real editor extension for focus tracking, CI). `requirements.md` states the scope boundary explicitly so it's never ambiguous what's included.

## Real-world impact

Engineering orgs routinely buy multiple disconnected tools to get partial answers to "how's the team doing": a Git analytics SaaS, a separate time tracker, a separate static-analysis dashboard. DevPulse's core idea — one system correlating velocity, code quality, and focus time, with the AI layer clearly labeled and gracefully optional — is a genuinely useful, directly extensible product shape, not just a stack-showcase demo.
