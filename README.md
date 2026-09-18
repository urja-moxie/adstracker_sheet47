# Ad tracker

Monthly ad production dashboard, reading live from the Notion Projects database.

## What's here

```
server.js          Node server: serves the dashboard + /api/data proxy to Notion
public/index.html  the dashboard (self-contained, no build step)
package.json       start script, Node 20+
.env.example       template for local runs
```

No dependencies. Nothing to build.

## How it works

The browser can't call Notion directly — Notion's API sends no CORS headers, and a
token in front-end code would be readable by anyone who opens the page. So
`server.js` holds the token and exposes one read-only endpoint:

```
GET /api/data  ->  { ads: [...], fetchedAt, counts }
```

The dashboard renders a baked-in snapshot instantly, then swaps in live data when
`/api/data` responds. If Notion is unreachable or misconfigured, it keeps showing
the snapshot and prints the reason in the footer, so the page never breaks.

Responses are cached in memory for 5 minutes (`CACHE_SECONDS`). One refresh costs
about 12 Notion API calls.

## Environment variables

| Variable | Required | Notes |
|---|---|---|
| `NOTION_TOKEN` | yes | Internal connection secret, starts with `ntn_` |
| `NOTION_DB_ID` | yes | Projects database id |
| `CACHE_SECONDS` | no | Defaults to 300 |
| `TARGETS_DIR` | no | Where the shared Targets CSV/overrides are persisted (defaults to `./data`) |
| `PORT` | no | Railway injects this automatically |

## Notion setup

1. **notion.so/profile/integrations → New connection.** Choose **Internal**, not
   Public. Public gives you OAuth credentials instead of a token, and the type
   can't be changed after creation.
2. **Capabilities: Read content only.** The dashboard never writes.
3. **Copy the secret** (`ntn_...`) into Railway's Variables tab. Not into this
   repo, not into the HTML.
4. **Share the database with the connection.** Open the Projects database as a
   full page → `⋯` → **Connections** → add the one you made. Skipping this
   returns zero rows with no error, which looks exactly like a broken token.

## Deploy to Railway

**From GitHub (recommended)**

1. Push this folder to a repo.
2. Railway → **New Project** → **Deploy from GitHub repo**.
3. Railway detects Node and runs `npm start`. No build command needed.
4. **Variables** tab → add `NOTION_TOKEN` and `NOTION_DB_ID`.
5. **Settings → Networking → Generate Domain**.

**From the CLI**

```bash
npm i -g @railway/cli
railway login
railway init
railway up
railway variables --set NOTION_TOKEN=ntn_xxx --set NOTION_DB_ID=366d03c824e7800c98cb000b5bbccdc1
railway domain
```

## Verifying

```bash
curl https://YOUR-APP.up.railway.app/healthz        # -> ok
curl https://YOUR-APP.up.railway.app/api/data | head -c 400
```

The dashboard footer tells you which state you're in:

| Footer text | Meaning |
|---|---|
| `Live from Notion · 515 ads · updated 14:32` | Working |
| `Showing saved snapshot — NOTION_TOKEN is not set` | Variable missing |
| `Showing saved snapshot — Notion 401 ...` | Bad token |
| `Showing saved snapshot — Notion returned 0 rows...` | Step 4 not done |

## Run locally

```bash
cp .env.example .env      # fill in the real token
node --env-file=.env server.js
```

Then open http://localhost:3000

## Things worth knowing

- **Funnel stage (ToFu/MoFu/BoFu) comes from "Sheet 47 Funnel", not "Funnel".**
  That property links each ad to a messaging item, and the item's own `Funnel`
  property carries the stage — so it's a two-hop lookup. If `Sheet 47 Funnel`
  is blank on an ad, or the linked item has no stage set, that ad shows as
  Unspecified; there's no fallback to the old per-ad `Funnel` field. The
  database that `Sheet 47 Funnel` links to needs the same "share with the
  connection" treatment as `Portfolios`/`Messaging Funnels` below, or the API
  won't see it. `GET /api/debug` reports `sheet47.itemsSeen` /
  `sheet47.itemsWithStage` so you can tell a sharing problem (0 items seen)
  apart from items just not being staged yet.
- **Targets are shared across everyone who opens the dashboard.** Uploading a
  Targets CSV, or setting a per-portfolio target by hand, is saved server-side
  via `POST /api/targets/csv` / `POST /api/targets/override` and read back by
  every visitor from `GET /api/targets` — one upload updates the dashboard for
  the whole team, not just your own browser. It's written to a JSON file at
  `TARGETS_DIR/targets.json` (default: a `data/` folder next to `server.js`).
  **Railway's local disk is wiped on every redeploy** — if you want this to
  survive deploys, add a Railway **Volume** and set `TARGETS_DIR` to its mount
  path. Without a Volume, targets reset to the baked-in defaults each time you
  redeploy (Notion ad data is unaffected either way, since that's fetched live).
- **Targets are still baked in** as a starting point, sourced from the portfolio breakdown sheet — only
  five portfolios have them. Notion supplies the ads, not the plan. To make
  targets live too, create a Notion database with Portfolio / Funnel / Messaging /
  Video reqd / Static reqd and it can be pulled the same way.
- **The URL is public** to anyone who has it. If that matters, put Cloudflare
  Access or Railway's own protections in front of it.
- **Status routing lives in two places** — `server.js` (LIVE / PIPE sets) and the
  dashboard's footer text. Change both together.
- **Uses Notion API version `2025-09-03`.** Databases are containers holding one
  or more *data sources*; rows are queried per data source. The server discovers
  them from `NOTION_DB_ID` and merges all of them, de-duplicating by page id.
  Set `NOTION_DATA_SOURCE_ID` to target just one.
- **`GET /api/sources`** lists the data sources in your database with their names
  and ids — useful when deciding whether to pin one.
