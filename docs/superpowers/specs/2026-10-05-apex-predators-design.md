# Apex Predators — Club & Tournament Site

**Design spec · 2026-10-05 · V1**

---

## 1. Purpose

Apex Predators is a badminton club. The club announces tournaments and needs
members to register teams quickly, usually from a phone, usually via a link
pasted into a WhatsApp group. Today there is no system; this site becomes the
club's public identity and the single place registration happens.

**The one sentence that defines V1:** an organiser creates a tournament in a
browser, shares one link, and teams register themselves — appearing on a public
board the moment they do.

### Success criteria

1. An organiser with no technical skill creates and opens a tournament without help.
2. A captain completes registration on a phone in under 60 seconds.
3. The registered-teams board is correct at all times and never oversells slots.
4. The site is live on a public URL at zero monthly cost.
5. A visitor who has never heard of the club understands who they are within one screen.

### Audience and scale

Internal club use. Roughly 10–40 teams per tournament, all known to the organisers.
Trust is high: no payment, no fraud controls, no identity verification in V1.

---

## 2. Decisions

Decisions already taken with the client, recorded so implementation does not relitigate them.

| # | Decision | Rationale |
|---|---|---|
| D1 | Internal club only | No public/open entry, so no payments or anti-fraud in V1 |
| D2 | Non-technical admin, so a real admin UI is required | Organiser must not touch code to run an event |
| D3 | Tournament format is configurable per event | Club runs singles, doubles and squad formats |
| D4 | Free tiers only, ₹0/month | Club has no budget |
| D5 | Neon Postgres, **not** Supabase | Supabase free pauses projects after ~1 week idle; the site is quiet between tournaments |
| D6 | Dedicated registration page, not a modal or wizard | The deliverable is a *shareable link* for WhatsApp; a modal has no URL to share, a wizard adds taps |
| D7 | Registrations appear instantly; admin can remove | High-trust internal club; no approval bottleneck |
| D8 | Dark theme, "restrained neon" | Matches the logo; gradient reserved for the CTA so the Register button always leads |
| D9 | Roster/About/gallery managed in code | Avoids building blob storage + media library for content that changes twice a year |
| D10 | No confirmation emails in V1 | Cannot send from a domain the club does not own; WhatsApp share instead |

---

## 3. Architecture

A single Next.js 15 application deployed to Vercel, talking to Neon Postgres.

```
GitHub (temciousf/apex-predators)
        │ push
        ▼
   Vercel  ── build ──▶ production deployment
        │
   ┌────┴─────────────────────┐
   │                          │
Public pages (RSC)      /admin (Auth.js)
   │                          │
   └──── Drizzle ORM ─────────┘
                │
       Neon serverless Postgres
```

**Why one app rather than a separate API:** public pages are React Server
Components that query the database directly, and the registration form submits
through a Next.js Server Action. There is no REST layer to design, authenticate,
version or document. Fewer moving parts is the dominant concern for a site that
one volunteer will maintain.

### Stack

| Concern | Choice |
|---|---|
| Framework | Next.js 15 (App Router), TypeScript strict |
| Styling | Tailwind CSS v4, design tokens as CSS custom properties |
| Database | Neon serverless Postgres |
| ORM / migrations | Drizzle ORM + drizzle-kit |
| Auth | Auth.js v5, Credentials provider, JWT session |
| Validation | Zod, schemas shared between client and server |
| Forms | react-hook-form + @hookform/resolvers/zod |
| Tests | Vitest (unit/integration), Playwright (E2E + axe a11y) |
| CI | GitHub Actions; Vercel builds on push |

### Repository layout

```
src/
  app/
    (public)/            page.tsx, tournaments/, team/, about/, gallery/, contact/
    register/[slug]/     page.tsx, success/page.tsx
    admin/               login/, tournaments/, registrations/
    api/board/[slug]/    GET — JSON for live board polling
  components/
    ui/                  button, input, card, badge, dialog
    public/              hero, tournament-card, teams-board, roster, gallery
    admin/               tournament-form, registrations-table
  server/
    db/                  schema.ts, index.ts, migrations/
    actions/             register-team.ts, admin-*.ts
    auth.ts
  lib/
    registration-schema.ts   builds a Zod schema from a tournament's config
    ref-code.ts
  content/
    squad.ts  about.ts  gallery.ts   ← D9: code-managed content
```

---

## 4. Data model

### `admins`
| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| email | citext unique not null | login identity |
| password_hash | text not null | argon2id |
| name | text not null | |
| created_at | timestamptz not null default now() | |

### `tournaments`
| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| slug | text unique not null | URL segment, e.g. `apex-open-2026` |
| name | text not null | |
| tagline | text | one line under the title |
| description | text | markdown, rendered on the detail page |
| venue_name | text not null | |
| venue_address | text | |
| map_url | text | link out to Google Maps |
| starts_at | timestamptz not null | |
| ends_at | timestamptz | |
| reg_opens_at | timestamptz not null | |
| reg_closes_at | timestamptz not null | |
| status | enum not null default 'draft' | `draft` / `published` / `closed` / `completed` |
| max_teams | integer not null | overall cap |
| min_players | integer not null | **format config** |
| max_players | integer not null | **format config** |
| require_player_phone | boolean not null default false | **format config** |
| banner_url | text | |
| created_at / updated_at | timestamptz | |

`min_players`, `max_players` and the category list are what D3 means by
"configurable format". The registration form is generated from them.

### `tournament_categories`
| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| tournament_id | uuid fk → tournaments on delete cascade | |
| name | text not null | e.g. "Men's Doubles" |
| max_teams | integer null | optional per-category cap; null = only the overall cap applies |
| sort_order | integer not null | |

Unique `(tournament_id, name)`.

### `teams`
| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| tournament_id | uuid fk → tournaments on delete cascade | |
| category_id | uuid fk → tournament_categories on delete restrict | |
| name | text not null | |
| captain_name | text not null | |
| captain_phone | text not null | E.164-normalised |
| captain_email | text null | |
| ref_code | text unique not null | e.g. `APX-7K4Q2` |
| status | enum not null default 'active' | see below |
| created_at | timestamptz not null default now() | |

Status semantics — both non-active states free the slot and hide the team from
public pages; the row is retained rather than deleted so the organiser keeps a
record:

- `active` — counts toward capacity, shown on the public board.
- `withdrawn` — the captain asked to pull out; set by the admin on their behalf
  (there is no self-service withdrawal in V1, see §13).
- `removed` — the admin removed the entry for any other reason (duplicate,
  mistake, ineligible).

Constraints:
- unique `(tournament_id, lower(name))` where `status = 'active'` — no two active teams share a name in one tournament.
- unique `(tournament_id, category_id, captain_phone)` where `status = 'active'` — a captain cannot enter the same category twice, but may enter Men's *and* Mixed.
- index on `(tournament_id, status)` — the board and the capacity count both read this.

### `players`
| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| team_id | uuid fk → teams on delete cascade | |
| name | text not null | |
| phone | text null | required only if `require_player_phone` |
| skill_level | text null | free text |
| is_captain | boolean not null default false | |
| sort_order | integer not null | |

### `registration_attempts`

Backs the per-IP rate limit in §6. Kept in Postgres rather than an in-memory
store because serverless functions do not share memory between invocations.

| Column | Type | Notes |
|---|---|---|
| id | bigserial pk | |
| ip_hash | text not null | SHA-256 of the client IP + a server-side salt; the raw IP is never stored |
| tournament_id | uuid fk → tournaments on delete cascade | |
| attempted_at | timestamptz not null default now() | |

Index on `(ip_hash, attempted_at)`. Rows older than 24 hours are deleted
opportunistically on write, so no scheduled job is needed.

Roster, About copy and gallery images are **not** database tables — per D9 they
are typed TypeScript modules under `src/content/`.

---

## 5. Site map

| Route | Contents |
|---|---|
| `/` | Hero · next-tournament card · registered-teams strip · squad preview · gallery strip · contact |
| `/tournaments` | Upcoming / Open / Past, grouped |
| `/tournaments/[slug]` | Detail, countdown, venue + map link, **registered-teams board**, Register CTA |
| `/register/[slug]` | The registration page — **this is the link shared in WhatsApp** |
| `/register/[slug]/success` | Reference code, WhatsApp share, add-to-calendar (.ics) |
| `/team` `/about` `/gallery` `/contact` | Code-managed content |
| `/admin/login` | Credentials login |
| `/admin` | Dashboard: tournaments with live registration counts |
| `/admin/tournaments/new`, `/admin/tournaments/[id]/edit` | Create/edit incl. categories and format config |
| `/admin/tournaments/[id]/registrations` | Table, search, inline edit, remove/restore, CSV export |
| `/api/board/[slug]` | Small JSON payload for board polling |

Only `published`, `closed` and `completed` tournaments are publicly reachable.
`draft` returns 404 to anonymous visitors.

---

## 6. Registration flow

1. `GET /register/[slug]` — Server Component loads the tournament, its categories
   and the current active-team count. It renders one of four states: **open**
   (the form), **not yet open**, **closed**, or **full**. Closed and full are
   page states with useful messaging, not error toasts.
2. The form is a Client Component. Player rows are generated from `min_players`
   and `max_players`; rows beyond the minimum are addable and removable.
   Validation is a Zod schema produced by `buildRegistrationSchema(tournament)`.
3. Submit calls the `registerTeam` Server Action.
4. The action **rebuilds the Zod schema from the tournament row in the database**
   and re-validates. Client-supplied constraints are never trusted — a crafted
   request must not be able to register 20 players in a doubles event.
5. Anti-abuse: a hidden honeypot field, and a per-IP rate limit of 5 submissions
   per 10 minutes held in a small Postgres table.
6. Capacity, transactionally (see below).
7. Insert `teams` + `players`, generate `ref_code`, `revalidateTag` the
   tournament's cached pages, redirect to the success page.

### Capacity under concurrency

Two captains submitting simultaneously for the last slot could both pass a naive
"is it full?" read and produce 33 teams in a 32-slot event. The insert therefore
runs inside a transaction that takes a row lock first:

```
BEGIN;
  SELECT id, max_teams FROM tournaments WHERE id = $1 FOR UPDATE;
  SELECT count(*) FROM teams WHERE tournament_id = $1 AND status = 'active';
  -- also the per-category count when the category sets max_teams
  -- if full -> ROLLBACK and return a FULL result
  INSERT INTO teams ...; INSERT INTO players ...;
COMMIT;
```

Registrations for a given tournament serialise; different tournaments do not
block each other. At 40 teams the contention cost is irrelevant.

**Driver consequence:** this path must use Neon's Pool/WebSocket driver. The
Neon HTTP driver cannot hold a multi-statement transaction. Read-only page
queries may continue to use the HTTP driver.

### Live board

Writes call `revalidateTag('tournament:<slug>')`. The board component
additionally polls `/api/board/[slug]` every 30 seconds while the tab is
visible. Websockets are not justified at this scale and do not fit the free tier.

### Withdrawal

V1 has no self-service withdrawal. The success page shows the reference code and
tells the captain to contact the organiser, who removes the team in admin
(sets `status = 'removed'`, freeing the slot).

---

## 7. Admin console

Auth.js Credentials provider with argon2id hashes and JWT sessions. Middleware
protects `/admin/*` except `/admin/login`. Sessions last 8 hours.

- **Dashboard** — tournaments with live counts and status badges.
- **Tournament form** — all fields from §4 plus a repeatable category editor and
  the format config. Status moves `draft → published → closed → completed`;
  moving to `published` requires at least one category and a `max_teams ≥ 1`.
- **Registrations table** — per tournament: search by team/captain/player, inline
  edit, remove and restore, and **CSV export** (the organiser runs the draw from
  this on match day).

The first admin is created by a seed script, not by a public signup route. There
is no admin self-registration anywhere in the application.

---

## 8. Design system

Derived by sampling the supplied logo, not chosen by eye.

| Token | Value | Use |
|---|---|---|
| `--ink` | `#05050A` | page background |
| `--surface` | `#0D0E16` | cards |
| `--surface-2` | `#141622` | raised/hover |
| `--line` | `#222536` | borders |
| `--text` | `#F4F5FA` | primary text |
| `--muted` | `#9094A8` | secondary text |
| `--cyan` | `#1B95ED` | informational accent |
| `--violet` | `#AF53E5` | gradient midpoint |
| `--pink` | `#F349B0` | urgency: countdowns, slots-remaining |
| `--grad` | `linear-gradient(100deg, var(--cyan), var(--violet), var(--pink))` | CTA, slot bar, logo mark |
| `--success` `--warn` `--danger` | `#35D39A` `#FFB020` `#FF5470` | states |

**Dosage rule (D8):** headlines are white. The gradient appears only on the
primary CTA, the slot-progress bar, the logo mark, and a soft radial glow behind
the hero. Everything else is ink, surface and text. This guarantees the Register
button is the brightest element on every page.

**Contrast, measured against `--ink`:** cyan 6.6:1, violet 5.2:1, pink 6.4:1 —
all pass WCAG AA for normal text. White on the pink end of the gradient is only
3.3:1, so **gradient-filled buttons use `--ink` labels, never white.**

Typography: a tight condensed grotesque for display, Inter for body, both from
`next/font` (self-hosted, no external request). Mobile-first; the registration
page is explicitly tested in a WhatsApp in-app browser viewport, which is where
most entries will come from.

All tokens live in one file so the palette can be changed in a single edit.

---

## 9. Error handling

| Situation | Behaviour |
|---|---|
| Tournament full | Page state, Register disabled, "Contact the organiser to join the reserve list" |
| Registration not open / closed | Page state showing the opening or closing date |
| Duplicate team name | Field-level error on the name input; all other input preserved |
| Captain already entered this category | Field-level error naming the existing team |
| Database unreachable (Neon cold start) | One automatic retry, then a "try again" card that preserves everything typed |
| Validation failure | Inline per-field errors; form state never cleared |
| Draft tournament requested anonymously | 404, not 403 — the existence of a draft is not disclosed |
| Admin destructive action | Confirmation dialog naming the team or tournament |

No error path may discard a user's typed input.

---

## 10. Security and privacy

- **Public pages show team name, category and player names only.** Phone numbers
  and emails are visible exclusively inside the authenticated admin area. The
  site publishes a list of real people and is deliberate about what it exposes.
- Secrets live only in Vercel environment variables. The repository is public, so
  `.env*` is gitignored and no connection string or hash is ever committed.
- Passwords hashed with argon2id. No password reset flow in V1; the seed script
  resets a forgotten admin password.
- Server Actions carry Next.js's built-in origin checks (CSRF protection).
- Session cookies: `httpOnly`, `secure`, `sameSite=lax`.
- Phone numbers are never written to application logs.
- All database access is parameterised through Drizzle; no string-built SQL.

---

## 11. Testing

| Level | Coverage |
|---|---|
| Unit (Vitest) | `buildRegistrationSchema` across singles/doubles/squad configs; `ref-code` uniqueness and format; phone normalisation |
| Integration (Vitest + real Postgres) | `registerTeam` happy path; duplicate name; duplicate captain-in-category; full tournament; **concurrent submission for the last slot must yield exactly one success** |
| E2E (Playwright) | Register a team → it appears on the board; admin logs in → creates → publishes → closes a tournament; CSV export downloads |
| Accessibility | axe assertions on home, tournament detail and registration pages |
| Viewport | Registration page at 360×640 and in a WhatsApp in-app browser user agent |

CI runs typecheck, lint, unit and integration tests on every push; E2E on pull
requests. Vercel promotes only after CI is green.

---

## 12. Deployment

### First-time setup

1. Scaffold the app locally; commit; push to `temciousf/apex-predators`.
2. Create a Neon account and project. Copy both the **pooled** and **direct**
   connection strings.
3. Create `.env.local` with `DATABASE_URL`, `DATABASE_URL_UNPOOLED`,
   `AUTH_SECRET`. Run `drizzle-kit migrate`, then the admin seed script.
4. Verify locally at `localhost:3000`.
5. Create a Vercel account, **Import** the GitHub repo, paste the environment
   variables, Deploy.
6. The site is live at `apex-predators.vercel.app`. **This is the URL other
   people visit.**

### Ongoing

- Every pull request gets its own preview URL automatically.
- Merging to `main` deploys production.
- Rollback is one click to any previous deployment in the Vercel dashboard.
- Custom domain, when wanted: buy `apexpredators.in`, add it in Vercel, set the
  DNS record Vercel provides; TLS is automatic and free.

### Operations

- **Monitoring:** Vercel Analytics and the Neon dashboard, both free tier.
- **Backups:** Neon's free tier retains limited history, so a scheduled GitHub
  Action runs a weekly `pg_dump` and stores it as a workflow artifact. This is
  the one genuine gap in the free-tier setup and is handled explicitly.
- **Cost:** ₹0/month. Vercel Hobby permits non-commercial use, which an internal
  club site is; selling tickets later would require Vercel Pro. Neon free
  provides 0.5 GB, far beyond this data volume.

---

## 13. Out of scope for V1

Payments and entry fees · draws, fixtures, scores and standings · player accounts
and login · email or SMS notifications · admin-uploaded images · self-service
withdrawal · multi-language · public/open registration by outside clubs.

Each is an additive change on this schema. Payments and open registration are the
natural V2; draws and results V3.

---

## 14. Content inputs still outstanding

These are content, not requirements, and do not block implementation. Until
supplied, the build uses clearly-marked placeholders that are replaced with a
single edit to `src/content/`:

| Input | Placeholder used |
|---|---|
| Home venue and city | "Velachery Indoor Stadium, Chennai" |
| Squad members | 8 placeholder cards |
| Gallery photos | 6 neutral placeholder images |
| Instagram / WhatsApp group links | Social icons hidden until URLs are provided |
| Club founding year / About copy | Short generic paragraph |

The logo supplied on 2026-10-05 is the source of truth for the palette in §8.

---

## 15. Open risks

1. **Vercel Hobby is non-commercial-use only.** Correct today. If the club ever
   charges entry fees through the site, the plan must change — this is noted in
   the payments V2 work, not deferred silently.
2. **Neon cold starts.** The free tier suspends after idle; the first request may
   take roughly half a second longer. Acceptable, and the retry in §9 covers the
   edge where it times out.
3. **No email receipt in V1.** If a captain deletes the WhatsApp message they
   have only the reference code. Mitigated by the admin's authoritative list and
   the CSV export.
