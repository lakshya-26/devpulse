# DevPulse

**DevPulse** is a full-stack web app that turns your **GitHub activity** into a clear picture of how you code: commits, pull requests, repositories, languages, contribution patterns, streaks, and personal goals. Sign in with GitHub, explore a dashboard and deep stats, set daily goals, and share a **public profile** link so others can see your dev pulse—useful for students, side-project builders, and anyone who wants motivation backed by real data.

This repository is a **monorepo** with two main packages:

| Folder | Role |
|--------|------|
| `devpulse-backend` | Node.js API, GitHub integration, Postgres, Redis, auth, caching |
| `devpulse-frontend` | Vite + React SPA, charts, dashboard, goals, public profiles |

---

## Features

- **GitHub OAuth** — Sign in; the backend requests scoped access to read your repos and activity (as configured in the GitHub OAuth app).
- **Dashboard** — Summary stats, highlights, optional period comparison, links into deeper views.
- **Statistics** — Commits, contributions heatmap-style views, PR trends, repo grid, language breakdown, commit history; many responses are **cached in Redis** to stay fast and kind to GitHub’s API.
- **Repository deep dive** — Per-repo stats and charts at a dedicated route.
- **Goals & streaks** — Daily goals (e.g. coding-related targets), history, and streak logic backed by the database; scheduled jobs can keep streak data in sync.
- **Public profile** — Read-only profile by GitHub username for sharing (`/u/:username` on the frontend; public API under `/api/v1/profile`).
- **Operational extras** — Health check endpoint, optional cache introspection for operators, structured errors and validation on the API.

---

## High-level architecture

1. **Browser** loads the React app (e.g. from Vercel). The app talks to the API with **credentials / JWT** as implemented in the client.
2. **Backend** validates input, authenticates requests, calls **GitHub’s REST API** for live data, merges or derives metrics, and stores **users, goals, streaks, sessions** (and related data) in **Postgres**.
3. **Redis** holds short-lived cached payloads (stats aggregates, repo stats, compare windows, etc.) with TTLs so repeated views are fast without hammering GitHub.
4. **OAuth flow** completes on the server; the client receives a **JWT** (via redirect to `/auth/success`) and uses it for subsequent API calls.

Routes on the server are registered automatically from `*.route.js` files under `routes/`, mounted under **`/api/v1/...`** with a consistent **kebab-case** segment per module.

---

## Tech stack

**Backend:** Node.js, Express, Passport (GitHub), Sequelize (Postgres), `ioredis`, JWT, Zod validation, `node-cron` (e.g. streak updates).

**Frontend:** React, React Router, Vite, Tailwind CSS, Axios, chart components for visualizations.

**Infrastructure (typical):** Postgres + Redis (often Docker on one host), API behind HTTPS (domain + nginx + Let’s Encrypt, or a tunnel), frontend on **Vercel** with SPA rewrites so client routes like `/auth/success` resolve correctly.

---

## Prerequisites

- **Node.js** 20+ recommended  
- **PostgreSQL** 16+ (local or Docker)  
- **Redis** 7+ (local or Docker)  
- A **GitHub OAuth App** (client ID, secret, callback URL matching your API)

---

## Local development

### 1. Database & Redis

From `devpulse-backend/`, you can use Docker Compose (see `docker-compose.yml`) for Postgres and Redis, or run your own instances. Ensure host/port/user/password/database match your `.env`.

### 2. Backend

```bash
cd devpulse-backend
cp .env.example .env
# Fill all required variables (see below)
npm install
npm run db:migrate
npm run dev
```

The API listens on the port set in `PORT` (default in example is often `5000`). A simple check: `GET /health`.

### 3. Frontend

```bash
cd devpulse-frontend
cp .env.example .env
# Set VITE_API_URL to your backend origin (no trailing slash), e.g. http://localhost:5000
npm install
npm run dev
```

### 4. GitHub OAuth app (local)

- **Callback URL:** `http://localhost:<PORT>/api/v1/auth/github/callback` (same `PORT` as backend).  
- **Backend:** `CLIENT_URL` must match your Vite dev origin (e.g. `http://localhost:5173`).

---

## Environment variables

Copy **`devpulse-backend/.env.example`** and **`devpulse-frontend/.env.example`** and fill every required key. Highlights:

| Area | Purpose |
|------|--------|
| `CLIENT_URL` | Frontend origin (CORS + OAuth success redirect). |
| `GITHUB_*` | OAuth client, secret, callback URL, API base. |
| `JWT_SECRET`, `SESSION_SECRET` | Auth and session security. |
| `DATABASE_URL` + `DATABASE_SSL` | Production Postgres (see example file for dev vs prod). |
| `REDIS_URL` | Redis connection string. |
| `VITE_API_URL` | Backend origin exposed to the browser at build time. |

**Production:** Use a proper `DATABASE_URL` (including `127.0.0.1` or your DB host, correct port, and URL-encoded password if it contains special characters). `NODE_ENV=production` enables stricter CORS and secure cookies.

---

## API overview (authenticated unless noted)

Base path: **`/api/v1`**.

| Module | Examples |
|--------|----------|
| **auth** | GitHub login/callback, current user profile (`/me`). |
| **stats** | Commits, contributions, PRs, repos, compare, streak, languages, refresh cache. |
| **goals** | Create goal, today’s goal, history. |
| **repos** | Per-repo deep stats by repo name. |
| **profile** | Public read-only profile by `:username` (no auth). |
| **cache** | Operator-oriented cache stats (auth required). |

Exact paths and query parameters are defined in the route, validator, and controller files under `devpulse-backend/`.

---

## Database

- **Migrations:** `npm run db:migrate` in `devpulse-backend` (uses Sequelize CLI; `NODE_ENV` selects config in `config/database.js`).  
- **Seeds:** Optional; see `package.json` scripts.  
- On startup, the app may sync models; for production, prefer migrations as the source of truth.

---

## Deployment (summary)

- **Frontend (Vercel):** Connect the `devpulse-frontend` project; set `VITE_API_URL` to your public API HTTPS URL; include **`vercel.json`** SPA rewrites so routes like `/auth/success` work.  
- **Backend + Postgres + Redis:** Often one **EC2** (or similar) with Docker for databases and **PM2** (or systemd) for Node; **nginx** + **Let’s Encrypt** for HTTPS, or a **Cloudflare Tunnel** for quick HTTPS without a domain (ephemeral URL—fine for demos).  
- **GitHub OAuth:** Callback must exactly match your deployed API callback path; `CLIENT_URL` must match your deployed frontend.

---

## Scripts (quick reference)

**Backend:** `npm start`, `npm run dev`, `npm run lint`, `npm run db:migrate`, seed scripts as in `package.json`.  
**Frontend:** `npm run dev`, `npm run build`, `npm run preview`, `npm run lint`.

---

## Story & deeper narrative

For a **student-oriented, storytelling write-up** (motivation, design choices, no code dumps), see **`MEDIUM_ARTICLE.md`** in this repo—a draft you can paste or adapt for Medium.

---

## License

Specify your license here if the project is open source (e.g. MIT).
