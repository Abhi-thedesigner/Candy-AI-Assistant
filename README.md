# Candy Cab — Cab Business Management

A full-stack app for running a cab business: bookings, live trip tracking (simulated), fleet, drivers, staff, finance and reports.

- **Frontend:** React 18 + Vite, Tailwind CSS, Lucide icons, Recharts (light mode only)
- **Backend:** Node.js + Express REST API, zod validation, helmet, rate limiting
- **Database:** PostgreSQL 16, parameterized SQL through `pg`

> All records in the seed are **demo data** (fictional names, phones and registrations). The app shows a "Demo data" badge in the header.

---

## Run everything in Docker (recommended)

Requirements: **Docker Desktop**.

```bash
docker compose up -d --build
```

Open **http://localhost:4000**.

| Container | What it runs | Exposed |
| --- | --- | --- |
| `candycab-web` | nginx serving the built React app; proxies `/api` to the API | `localhost:4000` |
| `candycab-api` | Express API (Node 20) | internal only (port 5055) |
| `candycab-db` | PostgreSQL 16; data kept in the `candycab_pgdata` volume | `127.0.0.1:5432` (for local dev only) |

- All three use `restart: unless-stopped`, so they come back after a reboot or a Docker restart.
- On first start the API creates the schema and loads demo data **only if the database is empty**. Existing data is never overwritten.
- To change the database password or timezone, copy `.env.example` to `.env` before the first start.
- Logs: `docker compose logs -f api`. Stop: `docker compose down` (data stays in the volume).
- Reset to fresh demo data: `docker compose exec api sh -c "node src/db/init.js && node src/db/seed.js"`.
- `docker compose down -v` deletes the volume and **all data**.

## Local development (without Docker for the app)

Requirements: **Node.js 18+**, and Docker Desktop for the database (or your own PostgreSQL 13+).

```bash
npm run setup
```

```bash
docker compose up -d db
```

```bash
npm run db:reset
```

Then run the API and the web app in two terminals:

```bash
npm run dev:api
```

```bash
npm run dev:web
```

Open **http://localhost:5173**. The API runs on **http://localhost:5055** (health check: `/api/health`).

### Configuration

| File | Purpose |
| --- | --- |
| `server/.env` (copy of `server/.env.example`) | `PORT`, `CORS_ORIGIN`, `APP_TIMEZONE`, `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`, optional `PGSSL` |
| `client/.env` (copy of `client/.env.example`) | `VITE_API_PROXY` (dev proxy target), optional `VITE_API_BASE_URL` for production builds |
| `docker-compose.yml` | Local PostgreSQL on port 5432 (user `candycab`, db `candycab`) |

`APP_TIMEZONE` (default `Asia/Kolkata`) is applied to both Node and the PostgreSQL session, so "today" means the same thing everywhere.

**Using your own PostgreSQL instead of Docker:** create a database and user, put the credentials in `server/.env`, then run `npm run db:reset`.

### Database scripts (in `server/`)

| Command | What it does |
| --- | --- |
| `npm run db:init` | Drops and recreates all tables from `server/db/schema.sql` |
| `npm run db:seed` | Clears tables and loads demo data (dates are relative to today) |
| `npm run db:reset` | `db:init` + `db:seed` |
| `npm run check` | Syntax check + verifies the DB is reachable and lists row counts |

### Checks

```bash
npm run check
```

Runs the server/DB check, ESLint on the client (zero warnings allowed) and a production build.

---

## Pages

| Route | Page | Highlights |
| --- | --- | --- |
| `/` | Dashboard | Total cabs, active trips, today's revenue, available drivers; month-to-date revenue / expenses / outstanding / profit; recent bookings; alerts (insurance, permits, service, licences, unassigned trips, overdue payments); upcoming trips; driver availability; quick actions. No map. |
| `/live-tracking` | Live Tracking | Trips in progress with simulated progress, speed, ETA and signal; trip timeline; driver and cab details; upcoming pickups. Auto-refreshes every 10 s. |
| `/bookings` | Bookings & Trips | Search, filter by status / trip type / date; create, edit, delete; booking detail with payments; status actions (confirm → start → complete, cancel, reopen). Fare auto-calculates from the rate card. |
| `/fleet` | Fleet Management | Cab CRUD, registration, vehicle type, fuel, availability, maintenance status, insurance / permit / service dates with expiry highlighting. |
| `/drivers` | Drivers | Profile cards, contact details, assigned vehicle (one cab per driver), availability and account status, recent trips. |
| `/staff` | Managers & Staff | Staff CRUD, roles, departments, activate / deactivate accounts. |
| `/finance` | Charges & Finance | Tabs: overview (date range), payments, expenses, outstanding balances (record payment in one click), trip charges (rate cards + fare calculator). |
| `/reports` | Reports & Analytics | Date presets and custom range; revenue vs expenses, trips per day, bookings by status, revenue by vehicle type, trip-type and expense breakdowns, top drivers; CSV export. |

**Logout** sits at the bottom of the sidebar. Authentication isn't built yet, so it shows a signed-out screen.

---

## Project structure

```
docker-compose.yml        PostgreSQL for local development
server/
  db/schema.sql           Tables, constraints, indexes
  src/
    config.js             Environment configuration
    index.js              Express app (helmet, CORS, rate limit, JSON limit)
    validators.js         zod schemas for every resource
    routes/index.js       REST routes
    controllers/          cabs, drivers, staff, bookings, tracking, finance, dashboard, reports
    middleware/           validation + error handling (maps PG errors to 400/409)
    utils/                HttpError, asyncHandler, parameterized INSERT/UPDATE/WHERE builders
    db/                   pool, init, seed, check scripts
client/
  tailwind.config.js      Design tokens (brand palette, shadows, font)
  src/
    components/ui/        Button, Card, Badge, DataTable, Modal/ConfirmDialog, Field inputs,
                          Toolbar filters, StatCard, Toast, loading/empty/error states
    components/forms/     FormModal — config-driven create/edit forms with validation
    components/layout/    Collapsible sidebar, top bar, navigation config
    hooks/                useApi (loading/error/polling), useEntityActions (CRUD flow)
    lib/                  API client, formatters (INR, dates), shared option lists
    pages/                One file per page
```

## REST API

All endpoints are under `/api`. List endpoints accept `search`, `status`, `type`, `from`, `to`.

| Resource | Endpoints |
| --- | --- |
| Dashboard | `GET /dashboard` |
| Tracking | `GET /tracking` |
| Cabs | `GET/POST /cabs`, `GET/PUT/DELETE /cabs/:id` |
| Drivers | `GET/POST /drivers`, `GET/PUT/DELETE /drivers/:id` |
| Staff | `GET/POST /staff`, `GET/PUT/DELETE /staff/:id` |
| Bookings | `GET/POST /bookings`, `GET /bookings/counts`, `GET/PUT/DELETE /bookings/:id`, `PATCH /bookings/:id/status` |
| Finance | `GET /finance/summary`, `GET /finance/outstanding` |
| Payments | `GET/POST /payments`, `PUT/DELETE /payments/:id` |
| Expenses | `GET/POST /expenses`, `PUT/DELETE /expenses/:id` |
| Rate cards | `GET /rate-cards`, `PUT /rate-cards/:id` |
| Reports | `GET /reports?from=YYYY-MM-DD&to=YYYY-MM-DD` |

Business rules enforced on the server: allowed status transitions; a trip can only start once a driver and cab are assigned and free; starting or finishing a trip updates driver availability, cab status and the odometer; a cab can be assigned to only one driver; cabs and drivers on an active trip can't be deleted; payments can't be recorded against cancelled bookings.

## Security notes

- Every query is parameterized. Dynamic column lists come only from whitelists in code.
- Input is validated with zod on the server and mirrored on the client.
- `helmet`, a CORS allow-list from `CORS_ORIGIN`, a 100 kB JSON body limit and rate limiting are on.
- Secrets live in `.env` files, which are git-ignored.
- **Not yet implemented:** authentication and role-based access. Add these before exposing the API outside a trusted network.

## Next phases (not in scope yet)

Real GPS / maps integration, authentication & roles, dark mode, notifications (SMS/email), invoices/PDF export.
