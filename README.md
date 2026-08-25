# Hitech Analytics Dashboard

Standalone analytics dashboard for **Hitech Construction Ltd** — survey reports,
site photos, workforce records and geospatial progress along a road/bridge
alignment, aggregated out of Supabase and rendered as a single operational view.

It is a sibling to the main **hitech-portal**: both read the same Supabase
project, but this app queries the database directly with the service role key
rather than going through the portal's API.

---

## Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16.2.6, App Router, Turbopack |
| Data | Supabase (`@supabase/supabase-js`, `@supabase/ssr`) |
| Auth | `iron-session` encrypted cookie, Django `pbkdf2_sha256` hash verification |
| Maps | Mapbox GL JS |
| ETL | Standalone Python scripts (pandas → Supabase upserts) |
| Language | TypeScript |

> **Next.js version note:** this pins Next.js 16.2.6. Some APIs differ from
> older releases — consult `node_modules/next/dist/docs/` rather than assuming.

## Architecture

```
src/
  app/
    layout.tsx              Root layout — mounts DashHeader, loads fonts
    page.tsx                Redirects / → /dashboard
    globals.css             CSS reset + keyframes
    login/page.tsx          Login (amber theme, dark card)
    dashboard/page.tsx      Main dashboard — skeuomorphic, self-contained
    progress/page.tsx       Chainage / progress view
    api/
      auth/login/route.ts   POST — verify against Supabase auth_user
      auth/logout/route.ts  POST — destroy session cookie
      auth/me/route.ts      GET  — current user or 401
      dashboard/route.ts    GET  — aggregated analytics payload
      map/route.ts          GET  — geospatial features for Mapbox
      progress/route.ts     GET  — chainage progress data
  components/
    DashHeader.tsx          Sticky 52px header — logo, user, logout
    SideNav.tsx             64px icon rail; hidden on /login and <640px
    HitechMap.tsx           Mapbox GL map
  lib/
    session.ts              iron-session config (cookie: hitech-dashboard-session)

scripts/                    One-off Node maintenance + verification scripts
docs/platform-api.md        Reference for the sibling hitech-portal API
sync_to_supabase.py         ETL: Google Drive Excel → Power BI M-code transforms → Supabase
sync_progress.py            ETL: progress/chainage data
sync_ogun.py                ETL: Ogun-specific dataset
```

### Data flow

Source Excel files live in Google Drive. `sync_to_supabase.py` pulls them,
reapplies the transformations originally written as Power BI M-code, and upserts
into these Supabase tables:

| Table | Source sheet |
|---|---|
| `hitech_report_hitechreport` | `Main_Survey_Data` (new + old) |
| `hitech_report_hitechphoto` | `PowerBI_Photo_Links`, joined on `globalid` |
| `hitech_report_hitechemployee` | `name` |
| `hitech_report_hitechsupervisor` | `site_supervisor_gr` |
| `hitech_report_hitechengineer` | `site_engineers` |
| `hitech_report_hitechmachine` | `machine_1` |

The Next.js app only ever reads those tables — it does not write to them.

## Getting started

```bash
npm install
npm run dev          # http://localhost:3000
```

For the Python ETL scripts:

```bash
pip install pandas openpyxl supabase google-api-python-client
python sync_to_supabase.py
```

## Configuration

Create `.env.local`:

| Variable | Required | Purpose |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Yes | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Yes | Server-side Supabase access. **Never expose to the browser** — it bypasses RLS. |
| `SESSION_SECRET` | Yes | iron-session encryption key (32+ chars) |
| `NEXT_PUBLIC_MAPBOX_TOKEN` | Yes | Mapbox GL access token |

## API reference

All routes require a valid session cookie unless noted. `401` = unauthenticated.

### `POST /api/auth/login`
Verifies against Django-style `pbkdf2_sha256` hashes in the Supabase `auth_user`
table and sets the `hitech-dashboard-session` cookie (httpOnly, encrypted).

```
Body: { "identifier": "user@example.com", "password": "plaintext" }
200:  { "ok": true }
400:  { "error": "Email and password are required." }
401:  { "error": "Invalid credentials." }
```

### `POST /api/auth/logout`
```
200: { "ok": true }
```

### `GET /api/auth/me`
```
200: { "user": { "id", "first_name", "last_name", "email", "is_staff", "is_superuser", "role" } }
401: { "user": null }
```

### `GET /api/dashboard`
Returns the full aggregated analytics payload — summary counters
(`totalReports`, `reportsThisMonth`, `activeProjects`, `totalPhotos`,
`uniqueReporters`), plus the breakdowns the dashboard renders.

### `GET /api/map`
Geospatial features for the Mapbox layer.

### `GET /api/progress`
Chainage-based progress data for `/progress`.

## Maintenance scripts

Run with `node scripts/<name>.mjs`.

| Script | Purpose |
|---|---|
| `mint-session.mjs` | Generate a valid session cookie for scripted testing |
| `backfill-chainage.mjs` | Backfill chainage values on existing rows |
| `check-ranges.mjs` | Validate chainage ranges |
| `verify-hr-filters.mjs` | Verify workforce filter behaviour |
| `click-filter-check.mjs` | UI filter regression check |
| `visual-check.mjs` | Visual smoke check |

## Deployment

Deploys to Vercel as a standard Next.js app. Set all four environment variables
in the Vercel project. The Python ETL scripts run separately — on a schedule or
by hand — and are not part of the Vercel build.

## Related

- **hitech-portal** — the main platform. Its full API surface is documented in
  [`docs/platform-api.md`](docs/platform-api.md).
