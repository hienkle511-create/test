# Our Budget — shared household budgeting app

A Monarch Money–style budgeting app for a married couple, with a three-way
expense split (Joint / Baby / Personal-per-spouse) layered on top of normal
spending categories. Dashboard with a donut chart, income-vs-expense trend,
a cash-flow Sankey diagram, budget-vs-actual bars, a filterable transaction
ledger with itemized-receipt splits, and a budgets and categories view.

## Honest note on the stack

This was specced as **Next.js 14 App Router + Prisma (Postgres in
production, SQLite locally) + Tailwind + shadcn/ui + Recharts + d3-sankey**.
That's still the *right* stack for this app. It isn't what's running here,
for one specific reason: **the sandbox this was originally built in had no
route to the npm registry at all** (`registry.npmjs.org`, `jsr.io`,
`unpkg.com`, `cdn.jsdelivr.net`, GitHub codeload, apt, and PyPI all returned
`403 host_not_allowed` — verified directly, not assumed), so `next`,
`prisma`, `tailwindcss`, `shadcn/ui`, `recharts`, and `d3-sankey` could not
be installed there.

**A normal dev machine, CI runner, or host does not have that problem** —
`npm install` will work fine anywhere sane. So rather than ship a mockup,
this repo is a real, running app built from what *was* available in that
sandbox: a hand-rolled architecture that mirrors what Next's App Router
would give you (file-per-route pages in `src/pages/`, a shared layout in
`src/layout/`, server-rendered React, small bundled client scripts for
progressive-enhancement interactivity), a data layer using Node 22's
built-in `node:sqlite` against the exact schema documented in
`prisma/schema.prisma`, and hand-written CSS in `public/styles.css` that
reproduces the same soft-shadow / rounded-card / light-palette look
Tailwind + shadcn would have produced. Charts (`src/charts/`) are plain SVG
instead of Recharts/d3-sankey.

**If you want the "real" stack** (which is genuinely recommended for
anything beyond a demo — a proper migration system, battle-tested chart
components, etc.), this codebase is the reference implementation to port
from: the data model, routes, queries, and UI are all already fully
specified here. The concrete porting steps:

1. `npx create-next-app@latest` and move `src/pages/*.tsx` content into
   `app/*/page.tsx` files using the existing `Shell`/`Sidebar` layout as
   `app/layout.tsx`.
2. `npm install prisma @prisma/client` and run `npx prisma migrate dev`
   against `prisma/schema.prisma` (already written and ready to use as-is)
   instead of `src/schema.sql` / `src/db.ts`.
3. `npm install -D tailwindcss` and `npx shadcn@latest init`, then swap
   `public/styles.css`'s classes for Tailwind utilities / shadcn
   primitives — the class names in `src/components/` and `src/pages/` map
   almost 1:1 conceptually (Card, Badge, progress bar, table).
4. `npm install recharts d3-sankey` and swap `src/charts/Donut.tsx` /
   `TrendLine.tsx` / `Sankey.tsx` for their Recharts/d3-sankey equivalents;
   `src/lib/queries.ts` already returns the exact data shapes needed.

## Running it locally

Requires **Node.js 22.5+** (for the built-in `node:sqlite` module).

```bash
npm install
npm run seed     # creates dev.db and populates ~6 months of sample data
npm run dev       # starts the server on http://localhost:3000
```

Then visit `http://localhost:3000`. Other useful scripts:

```bash
npm run build     # tsc --noEmit typecheck + bundles the small client-side
                   # interactivity scripts (src/client/*.ts) with esbuild
npm start          # same as `npm run dev` — runs the server directly via tsx
npm run screenshot # Playwright screenshots of all 4 pages (requires
                   # `npx playwright install chromium` first on a normal
                   # machine — the original sandbox had browsers preinstalled)
```

There's no build step that emits compiled JS for the server itself — both
`dev` and `start` run `src/server.tsx` directly through `tsx`, which strips
TypeScript/JSX at runtime via esbuild. That's why `tsx` and `esbuild` are
regular `dependencies` (not `devDependencies`) in `package.json`: they're
needed at runtime / deploy-build time, not just for local development.

## Deploying

This is a single Node process serving both the API-less SSR pages and a
SQLite file on local disk — that shape fits a **simple always-on container
host with a persistent volume** much better than a split
frontend-platform + managed-Postgres setup. Recommended: **Railway**.

### Railway (recommended)

1. Push this repo to GitHub.
2. In Railway, "New Project" → "Deploy from GitHub repo" → select the repo.
3. Railway auto-detects Node via Nixpacks. Set the build command to
   `npm run build` and the start command to `npm start` (Settings →
   Deploy) if it doesn't infer them from `package.json` already.
4. Add a **Volume** (Settings → Volumes) and mount it at, e.g., `/data`.
   Then set the `DB_PATH=/data/dev.db` env var (Settings → Variables) —
   `src/db.ts` already reads `process.env.DB_PATH` and falls back to the
   in-repo path when it's unset, so this is the only change needed to
   persist the database across redeploys. A 2-person household's write
   volume is trivially light for a single SQLite file on a persistent
   volume — no need for a managed database.
5. Railway sets `PORT` automatically and injects it as an env var; the
   server already reads `process.env.PORT` (see `src/server.tsx`), so
   nothing to change there.
6. After the first deploy, run `npm run seed` once against the deployed
   instance (Railway → your service → "Shell" gives you a one-off command
   runner) so the mounted volume has data. Re-running `seed` is safe — it
   clears and repopulates tables idempotently.
7. Deploy. Every subsequent push redeploys the container; the volume (and
   its `dev.db`) survives redeploys.

### Alternatives

- **Render**: same shape as Railway — a Web Service with a persistent Disk
  mounted for the SQLite file; slightly more manual env/volume wiring.
- **Fly.io**: works well for this too via a Fly Volume attached to the app,
  with `fly.toml` pointing the mount at the SQLite file's directory; better
  if you want the app to live close to a specific region.

(Vercel isn't a good fit here specifically *because* of the local SQLite
file — Vercel's filesystem is ephemeral/read-only-ish per request and has
no persistent volumes, which is the whole reason Railway/Render/Fly are
recommended instead.)

## Known rough edges

- **Category add/edit/delete** on `/categories` is an intentional
  non-persisting stub (DOM-only, via `prompt()`/`confirm()`, with a toast
  saying so) — nothing is written to the database.
- **`@types/node` and `node:sqlite`**: `node:sqlite` is still an
  experimental Node API. If `npm run build`'s typecheck complains it can't
  find types for `node:sqlite`, your resolved `@types/node` predates
  DefinitelyTyped adding it — bump `@types/node`, or add back a small
  ambient shim (`declare module "node:sqlite" { export class DatabaseSync
  { ... } }`) the way this project briefly did during development in the
  registry-less sandbox.
- **Seed volume**: ~63 transactions across 6 months (including 3 itemized
  receipts) — a bit above the "30–50" target for extra realism across all
  three split types and every category.
- **`prisma/schema.prisma`** is documentation of the intended production
  schema (see the porting steps above), not something `prisma generate`
  is run against in this build — the actual runtime schema is
  `src/schema.sql`, applied directly via `node:sqlite` in `src/db.ts`.
