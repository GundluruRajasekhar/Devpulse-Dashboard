# Requirements — DevPulse: AI-Enhanced Developer Productivity Dashboard

## 1. Problem Statement

Engineering managers and developers both struggle with the same question in different directions: "Is the team actually productive, and where's the friction?" Existing tools tend to answer only one slice — GitHub's own insights show commit graphs with zero context on code quality or focus time; time trackers show hours with zero connection to what was actually shipped; static analyzers flag issues with no sense of a developer's velocity trend. None of them combine git activity, AI-assisted code quality signals, and focus/deep-work time into one picture — and the few that try are typically closed SaaS products with no visibility into how the "AI insight" was actually derived.

**DevPulse** combines all three signals — git velocity, AI-assisted code review, and focus time — behind one dashboard, with a deliberately polyglot, real-world architecture: a Next.js/React frontend, a Node.js API gateway that owns all data access, and a Python microservice that does the numerical and AI-assisted analysis. This mirrors how this kind of system is actually built in industry (JS for product surface + API orchestration, Python for the data/AI-heavy compute), not a toy single-language demo.

## 2. Target Users

- **Individual developers** who want visibility into their own velocity, focus quality, and code-health trends without being surveilled by a punitive tool.
- **Engineering managers / team leads** who need team-level signal (sprint velocity, anonymized leaderboard) without exposing every individual's raw activity by default.
- **Engineering/product reviewers** evaluating the architecture of a genuinely multi-service, polyglot AI application.

## 3. Functional Requirements

### 3.1 Git & activity metrics
- FR-1: Ingest commit events (sha, additions, deletions, files changed, timestamp, author, project) into MongoDB (`commit_logs`).
- FR-2: Expose a daily-aggregated activity series (commit count, churn) over a configurable rolling window.
- FR-3: Track pull-request/merge frequency as part of the same event model (extensible field on ingestion).

### 3.2 AI code review insights
- FR-4: Accept one or more code snippets and run a deterministic, dependency-free heuristic scan (long functions, TODO/FIXME density, nested-loop complexity, bare exception handling) producing a bug-density score and complexity flags.
- FR-5: Optionally enrich the heuristic result with an AI-generated narrative and refactor suggestions via the Anthropic API, grounded in the specific heuristic flags and code content.
- FR-6: If no API key is configured, or the AI call fails for any reason, fall back to a deterministic, heuristic-derived summary — the endpoint must never fail because the AI layer is unavailable.
- FR-7: Persist every scan as an `ai_analysis_reports` document in MongoDB, tagged with which method produced it (`generatedBy`).

### 3.3 Time & focus tracker
- FR-8: Log discrete activity events of type `focus_session`, `distraction`, or `break`, each with start/end time and duration.
- FR-9: Summarize a user's tracked time over a rolling window into total minutes per type and a derived 0–100 focus score.

### 3.4 Team leaderboard
- FR-10: Rank developers within a window by commit count and churn.
- FR-11: Anonymize developer identity (`Developer A/B/C…`) for any requester who is not a `manager` or `admin`, using role information from MySQL.

### 3.5 Cross-cutting
- FR-12: Authenticate users via email/password against MySQL `Users`, issuing a JWT for subsequent API calls.
- FR-13: Store third-party provider tokens (e.g. GitHub) encrypted at rest (`Access_Tokens.token_encrypted`), never in plaintext.
- FR-14: All dashboard widgets must render usable content even when the backend is unreachable, using realistic mock data as a fallback (frontend-only development / demo mode).

## 4. Non-Functional Requirements

- NFR-1 (Separation of concerns): Only the Node.js API gateway connects to MySQL or MongoDB. The Python analytics service is a stateless compute layer reachable only via HTTP from Node (see `BLUEPRINT.md`).
- NFR-2 (Graceful degradation): Analytics-service downtime must degrade the dashboard (a "degraded" flag / empty insights), never crash the API gateway or the frontend.
- NFR-3 (AI optionality): The AI code-review layer must be fully optional; the product must be complete and demonstrable with zero external API key configured.
- NFR-4 (Security posture): Passwords hashed with bcrypt; provider tokens encrypted with AES-256-GCM; JWT-protected routes; CORS restricted to the configured frontend origin.
- NFR-5 (Portability): The whole system must run locally via `docker-compose up`, or each service independently for local development.
- NFR-6 (Explainability): Every AI or heuristic output must be traceable to specific input data (a specific commit range, a specific code snippet, specific flagged lines) — no unexplained scores.
- NFR-7 (Privacy-conscious defaults): Leaderboard identity is hidden by default; only elevated roles see identified rankings.

## 5. Out of Scope (for this build)

- Real GitHub/GitLab OAuth flow and webhook signature verification (the `Access_Tokens` table and ingestion endpoint are designed for this; the actual OAuth handshake and webhook receiver are left as an integration exercise).
- Production-grade auth (refresh tokens, session revocation, MFA).
- A real browser/editor extension for automatic focus-time tracking (the `/api/focus/sessions` endpoint is ready to receive events from one).
- Horizontal scaling / load balancing configuration beyond the provided `docker-compose.yml`.

## 6. Success Criteria

- `docker-compose up` brings up all five services (MySQL, MongoDB, Python analytics, Node API, Next.js frontend) with no manual steps beyond copying `.env.example` files.
- The dashboard is fully browsable with zero configuration via mock-data fallback.
- Posting a code snippet with an obvious issue (bare `except:`, nested loop, long function) produces a non-zero bug-density score and at least one complexity flag from the heuristic engine alone.
- With `ANTHROPIC_API_KEY` set on the analytics service, the code-review summary and suggestions are snippet-specific, not generic boilerplate.
- Leaderboard responses are anonymized for a `developer`-role JWT and identified for a `manager`-role JWT.
