Here’s a concrete, end-to-end build plan. It’s structured as phases with crisp deliverables, acceptance criteria, and a minimal but scalable tech stack: Next.js + TypeScript + Supabase (Postgres, Auth, RLS) + Tailwind + d3. Replace any part you dislike, but keep the sequencing.

# Phase 0 — Scope, constraints, and success criteria

**Decide:**

* Single-user vs multi-user from day 1.
* Web-only PWA vs web + native later.
* Data retention and export needs (CSV/JSON).
* Privacy posture (no 3rd-party analytics initially, self-hosted if needed).

**Acceptance criteria:**

* A one-page “Product Spec” with: core flows, non-goals, color scale definition (0..1), and a cut-line for MVP.

# Phase 1 — Data model and access rules

**Schema (Supabase Postgres):**

* `habits(id, user_id, name, color, active, created_at)`
* `habit_logs(id, user_id, habit_id, date, completed)`
* `daily_completion` view: per user, per date, `completion_ratio` and `total_habits`.

**Constraints & indexes:**

* Unique `(habit_id, date)` on `habit_logs`.
* Index on `(user_id, date)`.

**RLS policies (owner-only):**

* `habits`: `auth.uid() = user_id` for all ops.
* `habit_logs`: same policy.

**Acceptance criteria:**

* Queries for a date range return correct ratios including edge cases:

  * Day with no assigned habits → null ratio.
  * Day with 0/N → 0.0;  N/N → 1.0.

# Phase 2 — Repository and project layout

```
habit-tracker/
├─ apps/
│  └─ web/
│     ├─ app/
│     │  ├─ api/
│     │  │  ├─ days/route.ts
│     │  │  ├─ habits/route.ts
│     │  │  └─ logs/route.ts
│     │  ├─ dashboard/page.tsx
│     │  ├─ settings/page.tsx
│     │  └─ layout.tsx
│     ├─ components/
│     │  ├─ HabitGrid.tsx
│     │  ├─ HabitList.tsx
│     │  └─ DayDialog.tsx
│     ├─ lib/
│     │  ├─ supabase.ts
│     │  ├─ schema.ts
│     │  └─ color.ts
│     ├─ tests/
│     └─ styles/
├─ supabase/
│  ├─ migrations/
│  └─ seed.sql
└─ tooling/
   ├─ playwright/
   └─ k6/
```

**Acceptance criteria:**

* `pnpm dev` boots the web app locally; Supabase project configured; ENV set with typed loader.

# Phase 3 — Authentication, session, and RBAC

**Tasks:**

* Supabase Auth (email or OAuth).
* Server-side session retrieval in Next.js Route Handlers.
* Protect API routes with session checks; enforce RLS at DB.

**Acceptance criteria:**

* Unauthenticated users can’t read/write any data.
* Authenticated user sees only their data (verified through tests).

# Phase 4 — API surface (thin)

**Routes:**

* `GET /api/days?from=&to=` → `[ { date, completion_ratio, total_habits } ]`
* `GET /api/habits` → list habits.
* `POST /api/habits` → create habit.
* `PATCH /api/habits/:id` → rename/toggle `active`.
* `POST /api/logs` → upsert `{ habit_id, date, completed }`.

**Validation:**

* Zod schemas for inputs and outputs.
* Return typed errors and consistent status codes.

**Acceptance criteria:**

* Contract tests pass (Vitest) for all routes including auth failures.

# Phase 5 — UI: dashboard heatmap and daily editing

**Components:**

* `HabitGrid`: 7×53 calendar grid, d3 color scale mapping `0 → red`, `1 → green`, null → neutral gray.
* `DayDialog`: shows the day’s habits with y/n toggles, updates `habit_logs`.
* `HabitList`: CRUD for habits; color picker; active toggle.

**UX rules:**

* Clicking a cell opens `DayDialog`.
* Editing is optimistic with rollback on failure.
* TanStack Query caches days + habits; invalidates appropriately.

**Acceptance criteria:**

* Grid renders a full year with correct colors; tooltips show “X of N completed”.
* Editing a habit toggles the ratio and recolors the cell within 150 ms locally.

# Phase 6 — PWA, offline, and sync

**Tasks:**

* `next-pwa` with service worker.
* Cache static assets and last N months of `/api/days`.
* Queue offline writes; sync when online.

**Acceptance criteria:**

* App “installs” on mobile.
* Offline: user can open day dialog, toggle, and see the UI update. On reconnect, server state matches.

# Phase 7 — Testing and quality gates

**Unit tests (Vitest):**

* Color scaling at boundaries: null, 0, 1, floating values.
* Zod schemas reject malformed payloads.

**E2E tests (Playwright):**

* Sign in → create habit → mark complete → grid updates.
* RLS violation attempts fail.
* Offline flow: toggle during offline, resync on online.

**Static analysis:**

* ESLint + TypeScript strict.
* Pre-commit hooks (lint + test).
* Sentry wire-up for error reporting.

**Acceptance criteria:**

* CI pipeline runs lint, unit, E2E headless, and typecheck; all green.

# Phase 8 — Performance, accessibility, and polish

**Perf:**

* Avoid over-rendering cells; memoize cell components.
* Virtualize if needed for multiple years.
* Add SWR cache keys scoped by `user_id` + date range.

**A11y:**

* Keyboard navigation across grid.
* ARIA roles for grid and dialogs.
* Color-blind friendly theme option (e.g., blue scale plus symbol overlay).

**Acceptance criteria:**

* Lighthouse: Performance ≥ 90, A11y ≥ 95 on dashboard.
* Keyboard-only operation for core flows.

# Phase 9 — Deployment, backups, and observability

**Deploy:**

* Vercel for Next.js; Supabase managed.
* Environment split: dev, staging, prod.
* Database migrations via Supabase CLI.

**Backups & data safety:**

* Supabase automated backups enabled; retention policy documented.
* Disaster recovery drill: snapshot restore tested in a staging project.

**Observability:**

* Sentry DSN set; basic dashboards for error rates.
* Simple k6 script for smoke load on `/api/days` and `/api/logs`.

**Acceptance criteria:**

* Production URL live behind custom domain with HTTPS.
* Backup restore checklist executed once.

# Phase 10 — Analytics, notifications, and roadmap

**Optional features:**

* Streaks: consecutive `ratio == 1`.
* Reminders: Supabase Edge Functions + cron for daily email or push (if you add FCM).
* Data export: CSV/JSON for habit logs and daily ratios.
* Teams or shared views (later): org_id and share tokens.

**Acceptance criteria:**

* Streaks visible, correct across month/year boundaries.
* Export produces a valid CSV tested on a real dataset.

---

## MVP cut-line

* Authenticated user
* Create habits, toggle completion on any calendar day
* Heatmap with red→green scale, null gray for no habits
* PWA offline toggle with eventual sync
* Owner-only RLS enforced

Ship this before adding streaks, reminders, multi-year views, or teams.

---

## Risk register (brief)

* **Habit set drift:** when users add/remove habits mid-day, ratios change. Mitigate with clear UX copy (“Today’s goal is dynamic as you add habits”).
* **Color ambiguity:** red/green is not accessible for some users. Provide at least one alternate palette plus an optional icon overlay.
* **Offline conflicts:** last-write-wins is acceptable for single-user. Log conflicts for review.
* **RLS mistakes:** keep all writes using authenticated server-side clients and integration tests that attempt cross-user access.

---

## Concrete next tasks (sequence)

1. Initialize repo, Next.js app, Tailwind, TypeScript strict, ESLint, Playwright.
2. Create Supabase project; apply schema + RLS; seed one demo user and habits.
3. Implement `GET /api/days` and `GET/POST /api/habits`; add Zod contracts.
4. Build `HabitGrid` that renders a single month from mock data; wire to API after.
5. Build `DayDialog` with toggles; wire to `POST /api/logs` with optimistic updates.

Once those five are done, you’ll have a working vertical slice you can iterate on quickly.

