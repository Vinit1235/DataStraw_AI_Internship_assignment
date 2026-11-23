# DataStraw — Smart Roster Management System

A browser-based monthly roster manager for a customer-experience and operations team that
runs several D2C brands across multiple processes and operational pods.

**Live app:** https://datastraw-roster-eight.vercel.app

It generates a monthly roster from real scheduling rules (per-brand operating calendars,
minimum staffing, weekly offs, public holidays, planned leave and comp-offs), validates
it independently, and explains every conflict it cannot resolve.

---

## Table of contents

1. [What it does](#1-what-it-does)
2. [Quick start](#2-quick-start)
3. [Deployment](#3-deployment)
4. [Architecture](#4-architecture)
5. [Data model](#5-data-model)
6. [How the roster is generated](#6-how-the-roster-is-generated)
7. [Conflicts, severities and mandatory vs. preference](#7-conflicts-severities-and-mandatory-vs-preference)
8. [Assumptions I made](#8-assumptions-i-made)
9. [Sample dataset](#9-sample-dataset)
9b. [The Indian calendar](#9b-the-indian-calendar)
10. [Testing](#10-testing)
11. [Tell us how you think — the eight questions](#11-tell-us-how-you-think--the-eight-questions)
12. [Demo video script](#12-demo-video-script)
13. [Project layout](#13-project-layout)

---

## 1. What it does

| Area | What you can do |
| --- | --- |
| **Data** | Employees, pods and pod leads, brands, processes, brand↔process working rules, assignments, availability, holidays, date-specific coverage overrides — all editable from the API without touching code. |
| **Generate** | Pick a month, generate a full roster for the team in ~100 ms. The month picker runs the **Indian financial year** (April–March). |
| **Validate** | An independent validator re-checks every hard constraint against the finished roster, so a bug in the solver cannot hide a bad roster. |
| **View** | Company, pod, brand, process and employee views, plus a "by brand" coverage grid that shows who covers every brand-process on every day. |
| **Resolve** | Edit any cell, book or cancel leave, add or remove a brand assignment, change a brand's calendar or minimum staffing, or simulate a leave and see exactly what moves. |
| **Explain** | Every conflict has a message and a concrete fix hint. Configuration errors are separated from scheduling gaps. |
| **Publish** | Versioned rosters with draft/published states and CSV export. |

### The demo dataset

42 employees · 6 pods · 18 brands · 4 processes · 532 assignments per month.

* **14 of 18 brands (78%)** are run by a single dedicated employee who owns *all* of that
  brand's processes — the "single-resource brand" case.
* **4 brands** have three people of their own, usually on *different* processes
  (Customer Support vs. NDR), and two of them put two people on the same process on
  purpose.
* Every brand also draws three **float-pool backups**. A backup can cover the brand but
  does not belong to it — that is what keeps a single-owner brand alive when its owner
  takes leave, and it is why `assignments.is_backup` exists.
* 18 people cover more than one brand.

---

## 2. Quick start

### Prerequisites

* Node 22 (or 20+)
* Nothing else. A PostgreSQL database is optional — see below.

### Run it locally

```bash
npm install

# terminal 1 — API on :4000
npm run dev:api

# terminal 2 — Vite dev server on :5173 (proxies /api to :4000)
npm run dev
```

Open <http://localhost:5173>. The app generates a draft roster for the current month on
first load, so there is nothing to click before you see a grid.

**With no `DATABASE_URL` set**, the API boots PostgreSQL in-process (PGlite — Postgres
compiled to WebAssembly), applies `db/schema.sql` and seeds the sample dataset itself.
That is exactly how the public demo runs, and it means a clone works with zero setup.

**To use your own PostgreSQL** (local, Supabase, Neon, RDS — anything):

```bash
cp .env.example .env          # then fill in DATABASE_URL and DATABASE_ADMIN_URL

npm run db:migrate            # apply db/schema.sql
npm run db:seed               # load the sample dataset
```

`DATABASE_URL` is what the running app needs; `DATABASE_ADMIN_URL` is used only by
those two scripts, which run from your machine.

```bash
npm test          # 21 unit tests for the scheduling engine
npm run typecheck # tsc --noEmit across server + web
npm run build     # production build into dist/
```

### Connecting to a database

| | Host | Port | Mode | Used by |
| --- | --- | --- | --- | --- |
| `DATABASE_URL` | Supabase pooler | 6543 | transaction | the app at runtime |
| `DATABASE_ADMIN_URL` | Supabase pooler | 5432 | session | migrations + seed scripts |

Two notes that save real debugging time:

* **Use the pooler, not the direct connection.** Supabase's direct connection is IPv6-only;
  Vercel and Render are IPv4. The pooler hostname is IPv4.
* **The browser never talks to Supabase.** Every read and write goes through the Express
  function. The schema revokes `anon` and `authenticated` on all tables
  (`db/schema.sql`), so a leaked anon key cannot read rosters.

---

## 3. Deployment

> **Live:** https://datastraw-roster-eight.vercel.app — it runs with no external
> setup, because the database layer falls back to an in-process Postgres when
> `DATABASE_URL` is not set. Set that one variable and the same code uses a
> managed Postgres instead.

The app is **one Vercel project**: a Vite static build for the UI and a single serverless
function for the API, behind one URL.

```
browser ──► vercel.com/your-app
             ├── /api/*  →  api/index.ts  (Express, pg pool)
             └── /*      →  dist/        (Vite static build)
```

`vercel.json` does the routing:

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "rewrites": [
    { "source": "/api/:path*", "destination": "/api/index" },
    { "source": "/((?!api/).*)", "destination": "/index.html" }
  ]
}
```

Vercel passes the **original** request path into the function, so `/api/rosters/1` arrives
as `/api/rosters/1`. The Express router is therefore mounted twice — once at `/api` and
once at the root — so the same handler works whether the runtime strips the prefix or
not.

**Deploy steps**

```bash
npm i -g vercel
vercel link
vercel env add DATABASE_URL production          # pooler, port 6543
vercel env add DATABASE_ADMIN_URL production     # only needed for seeding
vercel deploy --prod
```

Then seed the live database once from your laptop:

```bash
DATABASE_ADMIN_URL="postgresql://…" npm run db:migrate
DATABASE_ADMIN_URL="postgresql://…" npm run db:seed
```

### Changing the configuration

Every rule in the app is a row, and every row is editable from the UI:

| What | Where |
| --- | --- |
| A brand's calendar and minimum staffing | **Configure → Working rules** — change either inline and the month is re-solved |
| New brand, process or pod | **Configure → Add** |
| A person's brand assignments | **Employee view → Brand assignments** (add or remove) |
| Leave and comp-offs | **Employee view → Time away** (book or remove) |
| A single roster cell | **Click the cell** in the grid and pick a brand/process, or clear it |
| A date-specific staffing bump | `coverage_overrides` (shown in the dataset as Onyx on 21 Nov) |

Each change re-solves the month and re-validates it, so the conflict counts in the
header always describe the roster you are looking at.

### Two backends, one interface

The app talks to Postgres through one small module (`server/db/pool.ts`) that exposes
`query`, `one` and `withTransaction`. Behind it there are two implementations:

| | When | How |
| --- | --- | --- |
| **PostgreSQL** | `DATABASE_URL` is set | `pg` connection pool — Supabase, Neon, RDS, a local server |
| **PGlite** | `DATABASE_URL` is absent | Postgres compiled to WebAssembly, running in-process |

The PGlite path exists so the deployed demo works with **zero external setup**: on a cold
start the function applies `db/schema.sql`, seeds the full sample dataset and serves it.
The SQL, the schema and the queries are identical in both modes — only the connection
differs — so switching to a real database is one environment variable, with no code
change.

The trade-off is honest and worth stating: in PGlite mode the data lives in the
function's own `/tmp`, so it is per-instance and resets when the instance is recycled.
That is fine for a demo (the roster is regenerated deterministically) but it is not a
substitute for a real database. Set `DATABASE_URL` and the same app becomes persistent.

**Connection pooling.** The function keeps `max: 1` connection per warm instance and a 5 s
idle timeout. On Vercel, `attachDatabasePool` from `@vercel/functions` closes sockets
before an instance suspends, which avoids the "database has been terminated" errors that
show up on serverless Postgres.

---

## 4. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Browser (React + Vite, TypeScript)                          │
│  Roster grid · Coverage grid · Conflict drawer · Simulator   │
└───────────────────────────┬──────────────────────────────────┘
                            │ JSON over fetch
┌───────────────────────────▼──────────────────────────────────┐
│  api/index.ts — one Vercel Function (Express)                │
│  ├── /bootstrap      config + pods + roster versions          │
│  ├── /rosters/*      generate · read · publish · delete       │
│  ├── /rosters/:id/simulate   re-solve with locked entries     │
│  ├── /rosters/:id/entries   manual edit + re-validate         │
│  ├── /views/:level   company · pod · brand · process · person │
│  └── /rosters/:id/export.csv                                 │
└───────────────────────────┬──────────────────────────────────┘
                            │ pg (node-postgres)
┌───────────────────────────▼──────────────────────────────────┐
│  PostgreSQL — 11 tables, real FKs, unique constraints          │
└──────────────────────────────────────────────────────────────┘
```

### Why the engine is a pure library

`server/engine/` has **no database and no HTTP in it**. It takes a `RosterConfig` and a
month string and returns entries, conflicts and an off plan. That is what makes the
schedule testable (20 unit tests, no database), deterministic, and reusable from a
script, a CLI or a future worker.

```
server/engine/
  types.ts     — the whole domain model, no dependencies
  dates.ts     — month/date maths, ISO weeks
  planner.ts   — demand build, operating calendars, weekly-off stage, pre-flight checks
  solver.ts    — min-cost max-flow (Dijkstra + potentials)
  engine.ts    — the 8-stage pipeline and the independent validator
```

### The eight-stage pipeline

```
 1  Pre-flight checks      configuration errors that no roster can satisfy
 2  Build demand           (brand, process, date) cells and who is eligible
 3  Assign weekly offs     per employee, per week, avoiding coverage-critical days
 4  Build supply           available people per date
 5  Solve                  min-cost max-flow, one date at a time
 6  Repair                 break and re-home weekly offs until the day is staffable
 7  Validate               independent re-check of every hard constraint
 8  Publish                version, status, CSV export
```

---


### The interface

The screen is a roster, so it is treated as a **data UI**: a calm canvas, one accent
colour, and colour reserved for data that actually carries meaning.

* **One accent, colour for data.** The chrome is neutral — greys and whites — and a
  single teal (`#0f766e`) marks the primary action and the selected state. The only
  other colours are the four process hues (teal / blue / amber / rose) on pale tints,
  plus a red for blockers and an amber for warnings. A cell's colour tells you which
  process covers it; nothing is coloured for decoration.
* **Hierarchy before chrome.** Rows are grouped by pod with a pod header and its lead,
  the person column is sticky, and the day header carries the day, the weekday, a
  festival dot, and the number of gaps that land on that day — so the eye finds the
  problems without opening anything.
* **Everything is one click away, nothing is always on.** The old layout carried a
  permanent sidebar and a permanent conflict panel, which squeezed the grid into a
  narrow column. Both are gone: filters became a horizontal bar, and issues, a person
  and configuration live in a drawer that opens on demand. The grid gets the full
  width of the window.
* **Typography.** Outfit for the interface, JetBrains Mono for dates and figures. No
  system-font stack, no icon font — every icon is inline SVG.
* **Honest states.** An assigned cell shows its brand code on the process tint, a
  weekly off says "Off", time away is a red or amber tag, and anything else is
  genuinely empty. A cell legend under the grid explains the four states so an empty
  cell is never ambiguous.
* **Every holiday says why.** A tinted column with an unexplained dot is not
  information. Each holiday column in the grid carries the festival name under the date
  — *Diwali / Lakshmi Puja*, *Guru Nanak Jayanti*, *Republic Day* — and the **Holidays**
  drawer lists every holiday in the month with its weekday, its region, and what it
  actually does to the roster: how many brand × process rules it closes. Clicking a
  holiday highlights its column in the grid.

## 5. Data model

```
pods ──────────────┐
  │ lead_id        │
  ▼                │
employees ◄───────┘
  │  ▲
  │  └── availability (leave, comp-off)      holidays (region, date, name)
  │
  ├──► assignments ──► brands ──► brand_processes ──► processes
  │        (employee × brand × process)   (calendar, min_staff)
  │
  └──► roster_entries (roster, employee, brand, process, date)

rosters (month, version, status)      conflicts (roster, date, type, severity, message)
roster_off_days (roster, employee, date)
coverage_overrides (brand_process, date, min_staff)
```

Twelve tables, real foreign keys, and unique constraints on
`(employee, brand, process)` and `(roster, employee, date, brand, process)` so the same
person cannot be assigned twice to the same slot and a roster cannot contain duplicates.

**Why `roster_off_days` exists:** the weekly-off allocation is part of the roster, not a
side effect of solving it. A published version has to say which day each person is off —
otherwise reloading the roster loses the offs, and nobody can tell "off" from "brand was
closed" on a Sunday. The allocation is decided by the engine's weekly-off stage, stored
with the roster, and read back by the grid.

**Why `brand_processes` and not a column on `brands`:** the assessment explicitly requires
that one brand's processes can follow *different* calendars ("a brand may have Customer
Support operating seven days a week while its other process follows a different
schedule"). The calendar therefore lives on the brand-process row, not the brand.

---

## 6. How the roster is generated

### Step 1 — demand, as a membership test

For every brand-process and every date in the month, ask *"does this slot operate?"*:

| Calendar | Operates on |
| --- | --- |
| `all_365` | every day, including Sundays and public holidays |
| `daily_except_ph` | every day except a public holiday in the brand's region |
| `sun_ph_off` | every day except Sundays and public holidays |
| `sat_sun_off` | Monday–Friday (the usual Monday-to-Friday operations week) |

This is a **membership test, never arithmetic**. Subtracting Sundays and then subtracting
holidays double-counts a holiday that lands on a Sunday and can erase a day that genuinely
needs staffing. Holidays are **scoped by region**: the Indian public holiday closes the
Indian brands and leaves the US brands open.

### Step 2 — supply

An employee is available on a date when they are not on leave, not on a comp-off and not
on their weekly off. Eligibility is evaluated *per date*, so a person on leave stops
counting as a candidate for that day's slots and the scarcity of a slot is judged against
who can actually turn up.

### Step 3 — the solver

Each date is solved independently as a **min-cost max-flow** problem:

```
source ──1──► employee ──1──► (brand, process) slot ──required──► sink
```

* Every available employee is a supply node with capacity 1 — one person, one slot per day.
* Every slot demands `required` people.
* **Scarcity dominates.** The slot→sink edge costs `(eligible − 1) × 1000`, so a slot only
  three people can fill is far cheaper than a slot twenty people can fill. The solver
  therefore claims scarce staff *first* and never wastes the only person who can run a
  brand on a flexible slot. This is the single most important detail in the whole
  algorithm: get it wrong and every brand is fine except the one that cannot be.
* **Preferences are tie-breakers only.** Continuity (did you run this brand yesterday?) and
  workload balancing are single-digit costs; the brand's dedicated owner gets a small
  preference so a brand stays with the person who actually runs it.

The flow is solved with Dijkstra plus potentials, which keeps every cost non-negative and
the result deterministic — the same config and month always produce the same roster.

### Step 4 — weekly offs

A weekly off is a *decision*, not a stored fact, because the choice depends on who else
can cover. The stage starts from each employee's configured preference and moves the day
only when coverage demands it. The cost of a candidate off day combines:

* how much coverage breaks if that person is off (graded by how replaceable they are),
* whether it is one of their preferred days,
* how crowded that weekday already is across the whole team.

Two details that matter more than they look:

* A **partial week** — the first or last week of a month — gets a *proportional* share of
  the usual off days. Without this, every employee ends up off on the same stub day and the
  whole roster collapses on the 1st and 30th.
* When coverage genuinely cannot be met, the stage **breaks** an off and re-homes it to
  another day in the same week. The break is reported as a warning with the new date, so
  a manager can record the comp-off. An employee's off is never silently ignored, and it
  is never silently overridden either.

### Step 5 — validation

The solver is not trusted. `validate()` re-checks the finished roster against the rules
from scratch and reports:

* coverage below the required minimum,
* one person booked twice on the same day,
* somebody scheduled while on leave or comp-off,
* somebody scheduled for a brand-process they are not assigned to,
* somebody scheduled on a day that brand-process does not operate.

---

## 7. Conflicts, severities and mandatory vs. preference

| Severity | Meaning | Examples |
| --- | --- | --- |
| **Blocker** | A published roster would be invalid, or no roster can satisfy the configuration. | Coverage gap; double booking; scheduled while on leave; `min_staff` above the number of assigned employees; a 365-day brand owned by one person who takes days off. |
| **Warning** | Legal, but a preference was overridden and a human needs to record it. | A weekly off was broken to keep coverage (with the new date suggested). |
| **Info** | A suggested next step. | "Break this off and move it to another day"; "Sonal Kapoor already handles Retention for another brand and could be borrowed." |

**Pre-flight vs. scheduling.** Configuration errors that no roster can satisfy are
detected *before* solving and reported separately. Otherwise one impossible brand buries
twenty real gaps under 30 days of identical failures.

**Mandatory vs. preference, concretely:**

| Constraint | Kind |
| --- | --- |
| Brand-process operating calendar, minimum staffing, leave, comp-off | **Mandatory** — never violated; the roster is invalid otherwise. |
| Weekly off | **Preference** — coverage wins; the break is reported as a warning. |
| Preferred weekly-off day, brand continuity, workload balance | **Preference** — used only to break ties. |

---

## 8. Assumptions I made

The assessment says not all the information will be provided. These are the calls I made,
and each one is configurable in the database rather than hard-coded:

1. **`all_365` ignores public holidays.** The name says "all 365 days". A brand that wants
   to close on holidays but still work Sundays is `daily_except_ph`. If a client needs
   "365 days except holidays", add a fifth calendar — it is one branch in
   `isOperatingDay()`.
2. **A public holiday is scoped to a region.** An Indian holiday closes Indian brands and
   leaves US brands open. The seed has two US brands to prove it.
3. **Minimum staffing is per brand-process.** `brand_processes.min_staff` plus
   `coverage_overrides` for specific dates (the seed bumps one brand-process to 2 staff on
   one day for a sale).
4. **The weekly off is a preference, not a hard constraint.** Coverage wins, and the break
   is reported. This is the explicit instruction in the assessment: "a required client
   coverage day should take priority over a preferred weekly-off day".
5. **One person, one slot per day.** No split or night shifts were modelled — the
   assessment describes daily coverage, not shift patterns. The schema supports multiple
   employees per brand-process per day, so shift times could be added without a data
   migration.
6. **Two-person brands split their processes.** Where a brand has several people, they
   usually own *different* processes, and some brands have two people on the same process
   to make the overlap visible.
7. **Floats are shared.** Single-owner brands draw backups from a shared pool rather than
   each having their own, which is what makes a four-day leave survivable.
8. **The Hindu/Saka month is indicative only.** It is a label, never an input to
   scheduling. The Gregorian date and the holiday table are the only things the engine
   reads.
9. **Festival dates follow the Drik Panchang reckoning for 2026-27.** Lunar dates can
   move by a day between regions, and the gazetted holiday list differs by state; a
   production deployment would load the state government's notification for the region
   the brand serves.

---

## 9. Sample dataset

| | |
| --- | --- |
| Employees | 42, across 6 pods (Alpha … Foxtrot), each pod with a named lead — everyone is assigned to something |
| Brands | 18, split IN/US region, one owner each |
| Processes | Customer Support, Abandoned Cart Conversion, Retention, Non-Delivery Report |
| Calendars | `all_365`, `daily_except_ph`, `sun_ph_off`, `sat_sun_off` |
| Single-owner brands | 14 of 18 (78%) |
| Deliberate 4-day leave | Lumen's owner, 8–11 Nov — a single-owner brand loses its only dedicated person |
| Deliberate structural problem | Marrow: 365-day coverage, one owner, no backup — impossible, and the system says so |
| Deliberate comp-off | Diya Sharma works a comp-off on 17 Nov |
| Float on leave | A float away on one date, affecting several brands at once |
| Coverage override | Onyx Customer Support needs 2 staff on 21 Nov instead of 1 |
| Public holidays | The **real Indian calendar for FY 2026-27** (1 Apr 2026 – 31 Mar 2027) — Independence Day, Gandhi Jayanti, Republic Day, Dussehra, **Diwali (8 Nov 2026)**, Bhai Dooj, Chhath Puja, Guru Nanak Jayanti, Christmas, Makar Sankranti, Holi, Maha Shivaratri, Good Friday and more. US-region brands get the US federal holidays instead. |
| Float pool | 16 shared employees backing 17 brands, three backups each |
| Cross-brand people | Four employees cover a second brand as well as their own |

---

## 9b. The Indian calendar

The brief describes a D2C operations company, so the calendar is an Indian one.

**Working week.** The four operating rules are named the way an Indian operations
team would describe them:

| Rule in the app | Means | Typical use |
| --- | --- | --- |
| `sun_ph_off` | **Mon–Sat** — Sundays and public holidays off | the default Indian ops week (Saturday is a working day) |
| `sat_sun_off` | Mon–Fri | brands that do not work weekends at all |
| `daily_except_ph` | every day except public holidays | seven-day brands |
| `all_365` | every day, including Sundays and holidays | critical accounts that never close |

**Financial year.** The month picker runs the **Indian financial year, 1 April to
31 March**. November 2026 is `FY 2026-27, month 8 of 12` — not month 11. The sidebar
has the twelve months of the year so a manager can walk the whole FY.

**Holidays.** The seed carries the real Indian public holidays for FY 2026-27, with
Diwali on Sunday 8 November 2026, Dussehra on 20 October, Gandhi Jayanti on 2 October,
Independence Day on 15 August and Republic Day on 26 January 2027. Holidays are scoped
by region, so the two US brands stay open on Diwali and the Indian brands stay open on
Thanksgiving.

**Lunar months.** The header also shows the Hindu month and the Saka year
(`~Kartika 1948` for November 2026 — 1948 + 78 = 2026 CE).

> **Assumption:** the lunar month name is an **approximation** shown for orientation
> only. A lunar month is defined by the panchang, so its boundaries shift by about
> eleven days a year and differ slightly from region to region. The roster itself never
> depends on it — scheduling only ever uses the Gregorian date and the holiday table.
> Building it properly would mean embedding a panchang library and storing the lunar
> month on each date; that was out of scope for a roster tool.

### Seeing the reasons, not just the dates

A holiday is only interesting because of its effect on the roster, so the app states
the effect next to the name:

| Holiday in the grid | What the Holidays drawer says |
|---|---|
| **Diwali / Lakshmi Puja** — Sun 8 Nov 2026 | *Closes 20 of 24 brand × process rules* |
| **Guru Nanak Jayanti** — Tue 24 Nov 2026 | *Closes 20 of 24 brand × process rules* |
| **Thanksgiving** — Thu 26 Nov 2026 (US) | *Closes 2 of 24 brand × process rules* |
| **Republic Day** — Tue 26 Jan 2027 | *Closes 20 of 24 brand × process rules* |

A holiday only closes a brand if that brand's calendar excludes public holidays and the
brand sits in the same region. Marrow, on 365-day coverage, runs through every one of
them — which is exactly why it is the structurally impossible brand in the demo data.

### What Diwali does to the November roster

Diwali falls on a **Sunday** in 2026, which is a nice piece of luck for a roster: the
Mon–Sat brands are closed anyway, and only the `all_365` and `daily_except_ph` brands
have to cope. Dhanteras (Fri 6 Nov), Bhai Dooj (Wed 11 Nov) and Chhath Puja (Sun 15
Nov) do close working days, and the grid marks every one of them with a festival dot.

The behaviour falls out of the rules rather than being hard-coded:

| Date | Festival | What the roster does |
| --- | --- | --- |
| Fri 6 Nov | Dhanteras | Mon–Sat brands close; Onyx (`all_365`) is staffed by a different person |
| **Sun 8 Nov** | **Diwali** | Mon–Sat brands are closed anyway; Onyx is still staffed |
| Wed 11 Nov | Bhai Dooj | A working day lost for the Mon–Sat brands |
| Sun 15 Nov | Chhath Puja | Sunday — only the `all_365` and `daily_except_ph` brands are affected |

Every month of the financial year has been checked end to end: all twelve generate
between 506 and 598 assignments with 8–9 blockers (the deliberately impossible Marrow
configuration, plus its own uncovered days) and no broken weekly offs.

---

## 10. Testing

```bash
npm test
```

21 unit tests run against the engine with no database and no HTTP. They cover the cases
that are easy to get wrong:

* a public holiday that lands on a Sunday is counted once, and the brand still closes,
* holidays are scoped by region — an Indian holiday does not close a US brand,
* `all_365` keeps operating through a holiday,
* leave on the first and last day of the month stays inside the month,
* pre-flight catches `min_staff` above headcount, a brand with no employees, an impossible
  365-day brand, and a duplicate assignment,
* nobody is ever booked twice on the same day,
* a scarce slot is claimed before a flexible one,
* the same inputs always produce the same roster (determinism),
* a date-specific coverage override raises the requirement,
* a partial first or last week does not put the whole roster off — and a two-day stub
  week does not put *half* of it off,
* weekly offs are spread across the week rather than stacked,
* locked entries are preserved exactly, and locking the whole roster changes nothing.

---

## 11. Tell us how you think — the eight questions

### 1. How did you model employees, brands, processes, pods, and their relationships?

The centre of the model is the **assignment**, not the employee. An assignment row is
`(employee, brand, process)` and is the only thing that links a person to work. Everything
else hangs off it.

* **Employee → pod** is many-to-one; the pod carries a `lead_id`, so a pod lead is data,
  not a special kind of employee.
* **Brand ↔ process** is many-to-many *through* `brand_processes`, and that junction row
  carries the two things that actually differ per pair: the **operating calendar** and
  **`min_staff`**. This is what lets Customer Support run seven days a week for Cedarly
  while its Non-Delivery Report process runs Monday-to-Friday.
* **An assignment is either *dedicated* or a *backup*.** `assignments.is_backup` separates
  a brand's own team from the shared float pool. Without it, "this brand has one
  dedicated employee" and "this brand has one employee" are indistinguishable, and the
  brief's staffing shape — 80% single-resource brands — cannot be read off the data.
* **Employee → brand** is many-to-many, and an employee can hold several assignments
  across different brands *and* different processes. The seed includes people who cover
  two brands.
* **Availability** (leave, comp-off) is a row per person per date rather than a flag on
  the employee, because availability is a property of a *day*, not of a person.
* **Public holidays** are `(region, date, name)`. Scoping by region is the difference
  between "the Indian office is closed" and "the whole company is closed", and getting it
  wrong quietly closes a US client's support line.

A deliberate non-choice: I did **not** model "shifts". The assessment describes daily
coverage requirements, so a roster entry is "this person covers this brand-process on this
day". If shifts were needed, `roster_entries` would gain a `shift_id` and the solver would
gain one more dimension — no other table changes.

### 2. How does your system generate a roster and determine whether it satisfies the required constraints?

Generation is a pipeline, and the important part is that **demand and supply are separate
objects**.

Demand is a list of `(brand, process, date)` cells, each with a `required` headcount and
a list of eligible employees. A cell exists only if the brand-process *operates* that day,
which is a membership test against the calendar and the region's holidays. Supply is a list
of people who are not on leave, not on a comp-off and not on their weekly off that date.

The solver is a **min-cost max-flow** per date. Employees are supply nodes of capacity 1,
slots are demand nodes, and the flow assigns people to slots. The cost function is
deliberately layered:

```
scarcity   (eligible − 1) × 1000     ← dominates everything else
continuity 1                          ← prefer running the same brand as yesterday
workload   min(3, days/7)            ← share the load
ownership  3                         ← prefer the brand's dedicated owner
```

Scarcity dominating by a factor of a thousand is the load-bearing decision. A slot only
three people can fill costs 2000; a slot twenty people can fill costs 19000, so the
solver always serves the constrained slot first. Invert that and the algorithm looks fine
until the one brand that cannot be covered without a specific person.

Constraint satisfaction is then **not** taken on trust from the solver. A separate
`validate()` function re-checks the finished roster from scratch — coverage, double
booking, leave conflicts, assignment legality, operating days — and its findings are
reported alongside the solver's own. If the two disagree, the validator wins and the
roster is not publishable. A solver that is quietly wrong is the failure mode that makes
a roster tool dangerous.

### 3. How did you handle the difference between single-resource brands and brands with multiple employees?

The interesting difference is not the number of people, it is the **number of ways the
brand can be left uncovered**.

* A **single-resource brand** has exactly one person who owns each of its processes. If
  that person is unavailable on an operating day, the brand *cannot* be covered by
  reassignment — the only fix is to add a second person or break the operating rule. The
  seed has Lumen: one owner, Customer Support only, and a four-day leave in the middle of
  the month. The system does not pretend to solve it; it breaks the owner's weekly off
  where it can and puts a real backup on the other days, and it flags every exception.
* A **multi-employee brand** can absorb one absence without a gap, because the remaining
  employees can pick up the process. The solver handles this by simply treating each
  eligible person as interchangeable for that slot, with the owner's preference only as
  a tie-break.

The implementation difference is that a single-resource brand needs a **float pool** to be
viable at all. Every brand in the seed therefore draws backups from a shared pool of
"float" employees in addition to its own people. Without that, 80% of the brands would
have exactly one day of fragility each, and a single sick day would break 14 brands at
once — which is precisely the failure the assessment describes ("a weekly off assigned to
an employee could leave an entire brand without coverage").

The model supports both a brand whose processes are split across people (Cedarly: one
person runs CS, another runs NDR) and a brand where two people share one process
(Pinecrest has two people on Customer Support). Both are assignment rows; the solver does
not care which shape a brand has.

### 4. What happens when an employee manages all processes for a brand but is unavailable on a required working day?

This is the Lumen case, and the system handles it in four steps.

1. **The gap is created.** On 9–11 November Lumen's owner is on leave, so Lumen has no
   eligible person on those dates. The demand cell survives — the brand still operates —
   and the supply shrinks to zero.
2. **The coverage gap is found.** The solver cannot fill the slot, and the independent
   validator independently reports the same shortfall. The pre-flight stage separately
   notices that the brand has no structural second line and says so.
3. **A resolution is offered.** The conflict carries a fix hint naming the concrete
   people who could cover it, and the simulator shows the same thing as a diff.
4. **The closest thing to a fix is applied and reported.** Lumen has a backup in the float
   pool, so the solver puts that person on Lumen for those three days and the brand stays
   covered. The owner of a *single-resource* brand is the exception: if there were
   genuinely nobody else, the gap stays open as a blocker rather than being papered over
   by silently working someone seven days straight.

Two design rules make this honest. First, an absence is never silently absorbed: the
weekly-off breaks that were needed are reported as warnings with the new off day, so the
manager can book the comp-off. Second, the roster is not rewritten around it — the
manager simulates the change, sees exactly which assignments move, and decides.

### 5. How does your system handle conflicting requirements or insufficient staffing?

It separates three kinds of problem, because they need different responses.

**Configuration errors** are detected before the solver runs. `min_staff` above the number
of assigned employees, a brand with nobody assigned, a process with nobody assigned, a
duplicate assignment, a 365-day brand whose only owner takes days off — none of these can
be fixed by a better roster, so they are reported as blockers with a fix hint ("add a
co-owner") instead of producing 30 identical coverage errors.

**Scheduling gaps** are what the solver cannot close. These are reported as blockers with
the exact brand, process and date, plus suggestions: which eligible person is on a weekly
off that could be broken, and which employee in another pod already runs that process for
another brand and could be borrowed for a day.

**Preference conflicts** are warnings. A weekly off broken to keep coverage is exactly
what the assessment asks for — coverage wins over preference — but the manager has to
record the comp-off, so the break is surfaced with the date it was moved to.

The **precedence** is fixed and explicit: a required client coverage day beats a
preferred weekly-off day. Where both cannot hold, the mandatory constraint wins and the
preference is reported as broken.

In the sample data the system produces 9 blocking conflicts for November 2026. Eight of
those blockers are one deliberately broken configuration — Marrow, a 365-day brand with one
owner and no backup — and the ninth is the pre-flight report of the same underlying
problem. Every other brand is fully covered, including Lumen during its four-day leave.
The only warnings that ever appear are weekly offs that had to be broken, which is the
preference-versus-coverage trade-off being made visible.

### 6. Which parts of the process did you automate, and which did you leave under human control? Why?

**Automated**, because they are mechanical and the volume makes manual work error-prone:

* building the demand grid from calendars, minimum staffing and date-specific overrides,
* the weekly-off assignment and its rebalancing across the team,
* the day-by-day assignment of people to slots,
* the detection of every hard-constraint violation, through an independent validator,
* the detection of *configuration* errors that make a valid roster impossible,
* the impact analysis when availability changes.

**Left under human control**, deliberately:

* **The weekly off is a preference, not a fact.** The system proposes an off pattern and
  breaks it only when coverage demands, but it reports every break rather than hiding it.
  Who works which weekend is an HR and cultural question as much as a scheduling one.
* **Publishing.** A roster with blockers cannot be published without the manager
  overriding it. The tool refuses to quietly produce something that looks finished.
* **Accepting a structural gap.** If a brand genuinely cannot be covered, the manager
  decides between hiring a second owner, changing the client's calendar, or accepting the
  gap. The system reports the consequence of each but does not choose.
* **Manual edits.** Clicking a cell to assign someone is allowed, and the change is
  re-validated immediately — but the edit sticks even if it creates a gap, because the
  manager may be mid-change. The conflict appears next to it.
* **Cross-pod borrowing.** The system suggests that someone from another pod who already
  runs the same process could cover a gap, but a manager has to approve moving a person
  between pods.

The line I drew: **automate the computation, never the accountability.** The system does
the arithmetic and the explanation; a person decides what gets published.

### 7. If the company doubled its team and brand count, what would you change?

The algorithm would survive it unchanged — it is O(people × slots) per day and the flow
network stays sparse. Four things would change:

1. **The solver becomes weekly, not daily.** Per-date independence is the main structural
   simplification: a person can be given a different brand every day, which is unfair at
   120 employees and 36 brands. A time-expanded network over a week — each employee is a
   chain of daily nodes with a cost on switching brands — would let the solver trade a
   little coverage efficiency for real continuity. That is the first thing I would build.
2. **The weekly-off stage gets a proper optimisation.** It is currently a greedy
   per-employee heuristic that balances the load across weekdays. With double the people
   it would become a transportation problem: assign off days to employees so that every
   weekday carries a balanced share of absences, while respecting preferences and
   coverage. Same machinery, better objective.
3. **The float pool becomes a scheduling problem of its own.** Today floats are assigned
   per brand statically. At scale, floats should be allocated by the solver the same way
   owners are — a float is a scarce resource with a cost, not a fixed list.
4. **The database moves off a single instance.** The schema is already normalised and the
   queries are all parameterised, so this is mostly a deployment change: a real Postgres
   instance (or a second Supabase project) instead of the pooler, plus read replicas for
   the view endpoints. I would also add composite indexes on
   `(roster_id, date)` and `(employee_id, date)` — they exist but would need revisiting
   with real query volumes.

What would *not* change: the data model, the pre-flight stage, the validator, and the
demand/supply separation. Those are the parts that took the longest to get right, and
they scale by construction.

### 8. If you had one additional week to work on this, what would you improve first?

**The weekly-off assignment, and the fairness it encodes.** It is the weakest part of the
system and the part a reviewer would push hardest on.

Today it is a greedy per-employee heuristic: each employee's off days are chosen to avoid
days where they are the only possible cover, spread across the week, and starting from
their preference. It works, and the results are even, but it is local — it cannot see that
two employees in different pods would both be better off with the same off day, and it
does not reason about whether the week's *shape* is fair for the team as a whole.

With another week I would replace it with a small **min-cost flow of its own**: people on
one side, weekday-off capacity on the other, with the cost of an assignment being the
employee's preference distance plus the coverage damage it causes. That turns "fair
distribution of weekly offs" from a heuristic into an optimisation with a stated
objective, and it would let me replace the current "break an off and re-home it" repair
step with a plan that was never broken in the first place.

Second, I would build the **weekly time-expanded solver** described in question 7 — same
network, one extra dimension for continuity — which would let me stop rotating people
between brands on consecutive days.

Third, on the product side: an **approval workflow** (pod lead reviews their pod, the
operations manager publishes) and **notifications** when a published roster changes. Both
are listed as optional in the brief, and both are what turns a scheduling tool into
something an operations team actually runs on a Monday morning.

---

## 12. Demo video script

*3–5 minutes. Six shots, no voiceover needed over the first four.*

**0:00 — The problem, in one screen.** Company view of November 2026: 42 people, 18
brands, 4 processes, one grid. Point out that the naive answer — "everyone works
Mon–Fri, one brand each" — immediately fails when one person takes leave.

**0:30 — Generate the roster.** Click **Generate roster**. It solves 532 assignments across
30 days in about 90 ms and lands on a draft with **9 blocking conflicts and 24
suggestions**. Show the conflict drawer: eight blockers all point at Marrow, and the
pre-flight entry above them says why — *365-day coverage, one owner, no backup*.

**1:15 — Brand calendars are not the same.** Switch the grid to **By brand**. Point at
Marrow running every day, Onyx running every day, the Sat/Sun-closed brands, and the two
US brands staying open on the Indian public holiday. Switch one brand's rule and show the
demand change.

**2:00 — The leave scenario.** Open Lumen. Its dedicated owner is away 8–11 November.
Open the simulator, mark those four days as leave, and run it: the owner is replaced by a
float for the three operating days, 1 November is skipped because Lumen does not operate
on Sundays, and the diff shows exactly what moved.

**2:45 — The single-resource case.** Show that when the float pool is thin the gap is
*reported*, not hidden: open the blocker, read the message and the fix hint, and show the
suggestion to break a weekly off and move it to another day in the same week.

**3:15 — Edit and validate.** Click a cell, reassign it to somebody else, and show the
conflict count update immediately. Then **Export CSV**.

**3:45 — The views.** Click through Company → Pod → Brand → Process → Employee in about
twenty seconds to show the same roster from five angles.

---

## 13. Project layout

```
roster-app/
├── api/
│   └── index.ts              Vercel function entry — exports the Express app
├── db/
│   └── schema.sql            12 tables, indexes, FKs, constraints
├── scripts/
│   ├── migrate.ts            applies db/schema.sql
│   ├── seed.ts               the sample dataset
│   └── check.ts              runs the engine and prints a conflict report
├── server/
│   ├── app.ts                Express routes, zod validation, error handling
│   ├── local.ts              local API server
│   ├── db/
│   │   ├── pool.ts           pg pool, Vercel attachDatabasePool
│   │   └── queries.ts        all SQL, camelCase aliases
│   └── engine/               pure scheduling library — no DB, no HTTP
│       ├── types.ts          domain model
│       ├── dates.ts          month and week maths
│       ├── planner.ts        demand, calendars, weekly offs, pre-flight
│       ├── solver.ts         min-cost max-flow
│       └── engine.ts         the pipeline and the validator
├── web/
│   ├── index.html
│   └── src/
│       ├── App.tsx           state and layout
│       ├── api.ts            typed client
│       ├── dates.ts          date helpers
│       ├── icons.tsx         inline SVG icons
│       ├── processStyle.ts   one hue per process
│       ├── styles.css        the design system
│       └── components/       TopBar · FilterBar · RosterGrid · CoverageGrid
│                              Drawer · ConflictPanel · EmployeePanel
│                              SimulatePanel · ConfigurePanel
├── .env.example
├── vercel.json
└── README.md
```

### Design decisions worth naming

* **No ORM.** Plain SQL through `pg`. One dependency instead of five, the queries are
  visible to a reviewer, and the schema is readable without a layer of indirection. The
  cost is hand-written SQL and no automatic type safety for query results — mitigated by
  explicit column aliases and one `ConflictRow`/`RosterEntry` interface per query.
  Swapping `pg` for `postgres` (Postgres.js) would be a single file.
* **No Docker.** One Vercel project, one `psql` connection string, no local compose file.
* **The engine is a library, not a service.** Same reason: it is testable without a
  database, and it can be run from a script, a test or a future worker.
* **Blocked rosters are drafts.** A roster with blockers is saved as a draft. It is not
  publishable until a manager looks at it.
