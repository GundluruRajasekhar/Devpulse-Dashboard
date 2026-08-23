# Tasks — DevPulse Build Plan

Status legend: `[x]` done and verified in this delivered build · `[ ]` planned next iteration

## Phase 0 — Architecture & contracts
- [x] Decide the service boundary: Node.js owns all DB access; Python is a stateless compute microservice (see `BLUEPRINT.md`)
- [x] Design MySQL schema for relational entities (`Teams`, `Users`, `Projects`, `Access_Tokens`, `Project_Members`)
- [x] Design MongoDB collections for telemetry (`commit_logs`, `daily_activity_events`, `ai_analysis_reports`)
- [x] Define the Node ↔ Python JSON contract with Pydantic schemas on the Python side

## Phase 1 — Database layer
- [x] `database/mysql_schema.sql` — full DDL + safe demo seed data
- [x] `database/mongodb_schemas.md` — collection shapes, indexes, and the MySQL-vs-MongoDB rationale

## Phase 2 — Node.js API gateway
- [x] Sequelize models: `User`, `Team`, `Project`, `AccessToken`
- [x] Mongoose models: `CommitLog`, `DailyActivityEvent`, `AiAnalysisReport`
- [x] AES-256-GCM token encryption utility (`src/utils/crypto.js`)
- [x] JWT auth middleware + role-gate middleware
- [x] `authController` — login, JWT issuance
- [x] `gitController` — commit ingestion + daily activity aggregation (MongoDB aggregation pipeline)
- [x] `focusController` — activity event logging + rolling focus-score summary
- [x] `leaderboardController` — commit-velocity ranking with role-based anonymization
- [x] `analyticsController` — proxies to the Python service for productivity insights and code review, with graceful-degradation fallback on failure
- [x] Route wiring + Express app (`server.js`) with CORS, JSON body limit, health check
- [x] Syntax-verified every backend `.js` file with `node --check` — all pass

## Phase 3 — Python analytics microservice
- [x] FastAPI app (`main.py`) with CORS and a `/health` endpoint
- [x] `services/churn.py` — pandas-based daily churn aggregation, trend classification, consistency scoring
- [x] `services/bug_density.py` — dependency-free heuristic static analysis (long functions, TODO density, nested loops, bare excepts)
- [x] `services/ai_insights.py` — optional Claude-powered refactor suggestions, with deterministic fallback when no API key is set or the call fails
- [x] `routers/insights.py` (`POST /insights/productivity`) and `routers/code_review.py` (`POST /code-review/scan`)
- [x] **Live smoke-tested**: installed dependencies, ran the service, and confirmed `/health`, `/insights/productivity`, and `/code-review/scan` all return correct JSON

## Phase 4 — Next.js / React frontend
- [x] `lib/api.js` — fetch wrapper for every endpoint, each with a realistic mock-data fallback so the dashboard works with zero backend running
- [x] `GitActivityWidget` — self-built SVG bar chart of daily commit/churn activity
- [x] `FocusTrackerWidget` — self-built SVG donut of focus/distraction/break time + focus score
- [x] `ProductivityInsightsWidget` — renders the Python engine's velocity trend, consistency score, and generated insights
- [x] `LeaderboardWidget` — anonymized/identified team ranking
- [x] `AICodeReviewWidget` — live textarea → scan → bug-density score, complexity flags, refactor suggestions
- [x] Dashboard page (`pages/index.js`) assembling all five widgets in a responsive grid
- [x] Design tokens: Sora/Inter/JetBrains Mono type system, dark "developer tool" palette with semantic accent colors (lime = velocity/positive, blue = focus, violet = AI, amber = warning, red = risk)
- [x] **Live build-verified**: `npm install` + `next build` completes with zero errors and statically prerenders the dashboard

## Phase 5 — Orchestration & packaging
- [x] `docker-compose.yml` wiring MySQL, MongoDB, analytics-service, backend, frontend
- [x] Per-service `Dockerfile` (backend, analytics-service, frontend)
- [x] `.env.example` for all three application services
- [x] `BLUEPRINT.md` — the Node↔Python integration contract, written for a reviewer to understand the architecture in under two minutes

## Phase 6 — Documentation
- [x] `requirements.md` — problem statement, functional/non-functional requirements, explicit scope boundaries
- [x] `tasks.md` — this build log
- [x] `README.md` — architecture diagram, folder structure, setup instructions, demo script

## Next iteration (not built in this pass)
- [ ] Real GitHub OAuth + webhook signature verification wired into the existing `Access_Tokens` / commit-ingestion endpoints
- [ ] Editor/browser extension pushing real focus-session telemetry to `/api/focus/sessions`
- [ ] Persisted refresh-token auth flow and session revocation
- [ ] Replace the heuristic bug-density scan with an actual AST-based linter integration (e.g. `ruff`/`eslint` output ingestion) alongside the current LLM layer
- [ ] Websocket/SSE push for live dashboard updates instead of on-load fetch
- [ ] CI pipeline (lint + `next build` + `node --check` + `pytest`) to keep this build's verified-clean state enforced going forward
