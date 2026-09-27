<h1 align="center">CLIENTORA</h1>
<h3 align="center">A CRM built the way a sales team would actually want to use one</h3>

<p align="center">
<img src="https://img.shields.io/badge/status-live-brightgreen" />
<img src="https://img.shields.io/badge/license-MIT-blue" />
<img src="https://img.shields.io/badge/node-%E2%89%A518-339933?logo=node.js&logoColor=white" />
<img src="https://img.shields.io/badge/TypeScript-strict-3178C6?logo=typescript&logoColor=white" />
<img src="https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white" />
</p>

---

### Why this exists

Most CRM side-projects stop at a login page and a pretty dashboard. Clientora doesn't — it's a full monorepo with real authentication, real role-based access control enforced on the server (not just hidden buttons on the frontend), a drag-and-drop pipeline that doesn't jank, and an AI lead-scoring feature that fails gracefully instead of crashing when someone forgets to set an API key.

It started as a scoped 10-phase build and grew into something I'd genuinely ship. Below is what's actually in it.

## Stack

Frontend lives in `apps/web`: Vite, React, TypeScript, Tailwind, React Query, Zustand, React Router with lazy-loaded routes, React Hook Form + Zod, Recharts for the charts.

Backend lives in `apps/api`: Express + TypeScript on top of Prisma, split cleanly into routes → controllers → services → repositories so business logic never leaks into a route handler.

Database is Postgres, fully normalized — Users, Customers, Leads, Deals, Tasks, Meetings, Notes, Activities, Notifications, Audit Logs, RefreshTokens.

## What's actually built

**Auth** — register, login, logout, bcrypt hashing. JWT access tokens kept in memory, refresh tokens as rotated opaque tokens in an httpOnly cookie with reuse detection. Session restores silently on reload instead of bouncing you to a login screen every time you refresh. Email verification and password reset both work (console mailer in dev, since nobody wants to configure SMTP for a demo). RBAC middleware is wired and actually gates routes, not just menu items.

**Dashboard** — one aggregate endpoint (`/api/v1/dashboard/overview`) that's role-scoped: a rep sees their own numbers, a manager or admin sees the whole org. Revenue, customers, new leads, deals won — each with a month-over-month delta. Three Recharts visualizations (revenue trend, pipeline funnel, customer growth) plus recent deals, today's tasks, and a live activity feed so the dashboard never looks empty on a fresh clone (there's a seed script for that).

**Customers** — full CRUD, server-side pagination, search, filtering, sorting. The important part is at the repository layer: a sales rep trying to open someone else's customer by guessing an ID gets a 404, not a 403 — a 403 would've confirmed the record exists, which is its own kind of leak. Soft deletes only, so nothing disappears from audit history.

**Leads** — CRUD plus dedicated assign and score-recalculate endpoints. Scoring is a small, deterministic, pure function (source × pipeline status × recency) — deliberately boring so it's explainable, and so the AI scorer later has something solid to build on top of instead of replacing it. Assignment rules are enforced server-side: reps can self-claim an unassigned lead, full stop, and any attempt to grab someone else's gets rejected regardless of what the request body says.

**Pipeline** — one call (`/api/v1/deals/board`) returns the whole Kanban board, pre-grouped and pre-sorted by stage. Drag-and-drop uses `@dnd-kit` with optimistic updates — cards move the instant you drop them and only snap back if the server actually rejects the change. Moving a card re-sequences both the column it left and the column it landed in, so ordering doesn't slowly drift over time the way a lot of Kanban implementations do.

**Reports** — revenue, leads, customers, and performance reports, each filterable by date range and, for managers/admins, by owner. Exports actually work: CSV hand-built with proper quote escaping, XLSX through `exceljs` with a styled header and frozen panes, PDF through `pdfkit` with page numbers. Locked to Admin/Sales Manager, matching what's visible in the sidebar.

**Notifications** — real Server-Sent Events, not a `setInterval` pretending to be realtime. Since `EventSource` can't send an Authorization header, that one route authenticates through a query param token instead, verified against the same JWT secret as everything else. Notifications fire off real events already happening in the app — a lead getting assigned, a deal getting dragged to Won, someone leaving a note on a customer you own. There's a 60-second polling fallback in case an SSE connection quietly dies, because it will, eventually, for someone.

**AI lead scoring** — a separate endpoint calls the Anthropic API directly (plain fetch, no SDK) for a qualitative read on a lead: score, confidence, a couple sentences of reasoning, one concrete next action. It's stored in its own fields so it never overwrites the rule-based score. The prompt hands the model the deterministic score as context and explicitly tells it not to just repeat it back. If `ANTHROPIC_API_KEY` isn't set, you get a clean 503 telling you exactly what to configure — not a stack trace. Model output gets validated against a Zod schema before it ever touches the database.

**Deployment** — caught two bugs before they became a 2am production incident: a hardcoded relative API path that only works when frontend and backend share a domain, and a `SameSite=Lax` cookie that browsers silently drop on cross-site requests. Both fixed (`VITE_API_URL` + `SameSite=None; Secure` in prod, still `Lax` locally). Multi-stage Dockerfile so the final image doesn't ship dev dependencies or source maps. Ready-to-go `railway.json`, `render.yaml`, and `vercel.json`. ESLint's flat config actually has rules behind it now (it didn't for the first several phases, if I'm honest) and CI runs lint + typecheck + build on every push.

## Getting it running

You'll need Node ≥ 18 and Postgres (or Docker).

```bash
npm install
```

Copy the env file and fill in the two things it actually needs:

```bash
cp apps/api/.env.example apps/api/.env
# set DATABASE_URL and JWT_ACCESS_SECRET
```

`ANTHROPIC_API_KEY` is optional — leave it blank and everything works fine, the AI lead-scoring button just tells you it's not configured instead of doing anything weird.

No Postgres locally?

```bash
docker run --name crm-postgres -e POSTGRES_USER=crm_user \
  -e POSTGRES_PASSWORD=crm_password -e POSTGRES_DB=crm_dev \
  -p 5432:5432 -d postgres:16
```

Then run the migration and seed some data so the app isn't a blank void the first time you open it:

```bash
cd apps/api
npm run prisma:generate
npm run prisma:migrate -- --name init
npm run prisma:seed
```

And from the repo root:

```bash
npm run dev
```

- Web: `http://localhost:5173`
- API: `http://localhost:4000/api/v1/health`
- Status page: `http://localhost:5173/status`

Log in with any seeded account, password is `Password123` for all of them:

| Email | Role |
|---|---|
| `admin@clientora.dev` | Admin — sees the whole org |
| `manager@clientora.dev` | Sales Manager — org-wide + Reports access |
| `exec@clientora.dev` | Sales Executive — scoped to their own book of business |

## Deploying

**API → Railway or Render.** Railway picks up `railway.json` and builds off the API's Dockerfile automatically — add a Postgres plugin and set `DATABASE_URL`, `JWT_ACCESS_SECRET`, `WEB_ORIGIN` (once the frontend's up), and `COOKIE_SECURE=true`. Render reads `render.yaml` and stands up both the API and a managed Postgres instance in one go — just fill in `WEB_ORIGIN` and `ANTHROPIC_API_KEY` after the first deploy.

Either way, run the migration once against prod:

```bash
DATABASE_URL="<production-url>" npx prisma migrate deploy --schema apps/api/prisma/schema.prisma
```

**Web → Vercel.** Set the project's root directory to `apps/web`, add `VITE_API_URL` pointing at your deployed API plus `/api/v1`. `vercel.json` handles the SPA rewrites so refreshing on `/pipeline` or `/reports` won't 404.

Last step either way: go back to the API and set `WEB_ORIGIN` to the real Vercel URL — CORS needs an exact match — then redeploy.

## What's not built yet

Task and meeting management didn't make the cut — Notifications already has "task due" and "meeting reminder" types wired up in the code, just waiting for something to trigger them. Team/territory management for lead assignment is another obvious next step. And the notification bus is single-process by design right now; swapping the in-process pub/sub for Redis is the documented path if this ever needs to run on more than one instance.

## License

MIT
=======
# Clientora CRM Systems

> A full-stack, production-ready CRM built from the ground up — auth, role-based access control, a real-time dashboard, pipeline management, AI-assisted lead scoring, and one-click deploy configs. No boilerplate left half-finished.

![Status](https://img.shields.io/badge/status-production--ready-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Node](https://img.shields.io/badge/node-%E2%89%A518-339933?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?logo=typescript&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)

---

## 📖 Overview

Meridian CRM is a monorepo CRM application covering the full lifecycle a sales team actually needs: customer records, lead capture & scoring, a drag-and-drop deal pipeline, exportable reports, live notifications, and an optional AI co-pilot for lead qualification — all gated by real role-based permissions, not just hidden UI.

It's built as a reference-quality implementation: every module has server-side authorization, optimistic UI where it matters, and a seed script so it's never a blank screen on first run.

---

## ✨ Features

### 🔐 Authentication & Access Control
- Register / login / logout with bcrypt-hashed passwords
- JWT access tokens (in-memory) + rotating opaque refresh tokens (httpOnly cookie, with reuse detection)
- Silent session restore on page load
- Email verification & forgot/reset password flows (console mailer stub in dev)
- Full RBAC middleware — **Admin**, **Sales Manager**, **Sales Executive**

### 📊 Executive Dashboard
- Role-scoped `GET /api/v1/dashboard/overview` — reps see their own book of business, managers/admins see org-wide numbers
- Revenue, Customers, New Leads, and Deals Won cards with month-over-month deltas
- Recharts visualizations: revenue trend, pipeline funnel, cumulative customer growth
- Recent deals, today's tasks, and a live activity feed

### 👥 Customer Management
- Full CRUD with pagination, search, status filtering, and sorting — all server-side
- Repository-level scoping: reps can't see, edit, or even discover other reps' customers (returns a clean `404`, not a leaky `403`)
- Soft deletes to preserve audit history
- Detail drawer with related leads/deals and an inline notes thread

### 🎯 Lead Management & Scoring
- Full CRUD plus dedicated `assign` and `score/recalculate` actions
- Deterministic, explainable rule-based scoring engine (source quality × status × recency)
- Server-enforced assignment rules — reps can only self-claim unassigned leads
- Searchable customer picker, live score bar, inline status editing

### 🗂️ Sales Pipeline
- Kanban board powered by a single `GET /api/v1/deals/board` call, pre-sorted and pre-aggregated by stage
- Drag-and-drop (`@dnd-kit`) with optimistic updates and automatic rollback on rejection
- Correct re-sequencing on both the source and destination columns — ordering never drifts
- `closedAt` managed automatically on Won/Lost transitions

### 📈 Reports & Exports
- Revenue, Leads, Customers, and Performance reports with date-range and owner filtering
- One-click export to **CSV**, **XLSX** (styled via `exceljs`), and **PDF** (via `pdfkit`)
- Locked down to Admin / Sales Manager roles, matching the sidebar

### 🔔 Real-Time Notifications
- True push delivery via **Server-Sent Events** — no polling required
- Triggered by real domain events (lead assignment, deal won/lost, new notes) — no fake demo data
- In-process pub/sub hub with a documented upgrade path to Redis for multi-instance deployments
- Bell icon, unread badge, mark-one/mark-all-read, and a 60s polling fallback safety net

### 🤖 AI-Assisted Lead Scoring
- `POST /api/v1/leads/:id/score/ai` calls the Anthropic API directly (no SDK) for a qualitative read
- The rule-based score is fed *into* the AI prompt as context, not replaced by it
- Fails safe: rate-limited, missing API key returns a clean `503` instead of a crash, all model output validated with Zod before it touches the database
- Full UI states: empty, loading, error (with a specific "not configured" message), and populated

### 📦 Deployment-Ready
- Two real cross-domain bugs fixed *before* they'd bite in production (API base URL, `SameSite` cookie policy)
- Multi-stage Docker build for a lean production image
- Ready-made `railway.json`, `render.yaml`, and `vercel.json`
- Flat-config ESLint (TypeScript + React Hooks rules) — actually wired up, not just installed
- GitHub Actions CI: lint, typecheck, and build on every push/PR

---

## 🧱 Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | Vite · React · TypeScript · Tailwind CSS · React Query · Zustand · React Router (lazy/code-split) · React Hook Form · Zod · Recharts |
| **Backend** | Express · TypeScript · Prisma (routes → controllers → services → repositories) |
| **Database** | PostgreSQL |
| **Realtime** | Server-Sent Events |
| **AI** | Anthropic API (Claude) |
| **Infra** | Docker · Railway / Render (API) · Vercel (Web) · GitHub Actions |

---

## 🗺️ Project Structure

```
meridian-crm/
├── apps/
│   ├── web/                # Vite + React frontend
│   │   ├── src/
│   │   │   ├── components/shared/   # DataTable, Drawer, ConfirmDialog, CustomerPicker...
│   │   │   ├── pages/
│   │   │   └── ...
│   │   └── vercel.json
│   └── api/                 # Express + Prisma backend
│       ├── src/
│       │   ├── routes/
│       │   ├── controllers/
│       │   ├── services/
│       │   ├── repositories/
│       │   └── lib/         # lead-scoring.ts, notification-bus.ts, ...
│       ├── prisma/
│       │   ├── schema.prisma
│       │   └── seed.ts
│       ├── Dockerfile
│       └── railway.json / render.yaml
└── .github/workflows/ci.yml
```

---

## ⚡ Getting Started

### Prerequisites
- Node.js ≥ 18
- PostgreSQL 16 (or Docker)

### 1. Install dependencies

```bash
npm install
```

### 2. Configure the database

```bash
cp apps/api/.env.example apps/api/.env
# edit DATABASE_URL, and set JWT_ACCESS_SECRET to a long random string
```

> 💡 **AI lead scoring is optional.** Leave `ANTHROPIC_API_KEY` blank and everything else works — the "Analyze with AI" button just surfaces a clear "not configured" message. Set the key to enable real analysis.

No local Postgres? Spin one up:

```bash
docker run --name crm-postgres -e POSTGRES_USER=crm_user \
  -e POSTGRES_PASSWORD=crm_password -e POSTGRES_DB=crm_dev \
  -p 5432:5432 -d postgres:16
```

### 3. Run migrations & seed demo data

```bash
cd apps/api
npm run prisma:generate
npm run prisma:migrate -- --name init
npm run prisma:seed
```

### 4. Start everything

```bash
npm run dev
```

| Service | URL |
|---|---|
| 🌐 Web | http://localhost:5173 |
| 🔌 API | http://localhost:4000/api/v1/health |
| 🩺 Diagnostics | http://localhost:5173/status |

### 🔑 Demo accounts

Password for all: `Password123`

| Email | Role | Scope |
|---|---|---|
| `admin@meridiancrm.dev` | Admin | Org-wide dashboard |
| `manager@meridiancrm.dev` | Sales Manager | Org-wide dashboard + Reports |
| `exec@meridiancrm.dev` | Sales Executive | Own deals/customers/leads only |

---

## ☁️ Deployment

### API → Railway or Render

**Railway** — create a project from this repo (it auto-detects `railway.json` and builds via `apps/api/Dockerfile`), add a Postgres plugin, then set:

| Variable | Value |
|---|---|
| `DATABASE_URL` | from the Railway Postgres plugin |
| `JWT_ACCESS_SECRET` | a long random string |
| `WEB_ORIGIN` | your deployed Vercel URL |
| `COOKIE_SECURE` | `true` |
| `ANTHROPIC_API_KEY` | optional — enables AI lead scoring |

**Render** — push the repo; Render reads `render.yaml` and provisions the API **and** a managed Postgres instance together. Set `WEB_ORIGIN` and (optionally) `ANTHROPIC_API_KEY` after the first deploy.

Run the migration once against production either way:

```bash
DATABASE_URL="<production-url>" npx prisma migrate deploy --schema apps/api/prisma/schema.prisma
```

### Web → Vercel

Import the repo, set **Root Directory** to `apps/web`, and add:

| Variable | Value |
|---|---|
| `VITE_API_URL` | `https://<your-api-domain>/api/v1` |

`vercel.json` handles the SPA rewrites, so deep links like `/pipeline` won't 404 on refresh.

### 🔁 Final step

Set `WEB_ORIGIN` on the API to the live Vercel URL (CORS requires an exact match) and redeploy. Optionally run `npm run prisma:seed` against production for demo accounts.

---

## 🧭 Roadmap

- [ ] Task & meeting management (Notifications' "due" and "reminder" types are already wired to hook into this)
- [ ] Team / territory management for the leads assignment picker
- [ ] Redis-backed pub/sub for multi-instance notification delivery

---

## 🤝 Contributing

Issues and PRs are welcome. Please run `npm run lint` and `npm run typecheck` before opening a pull request — CI will block on both.

---

## 📄 License

MIT © Meridian CRM Contributors
