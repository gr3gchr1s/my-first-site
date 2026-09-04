# Pinch

Local grocery price radar. Real stores and real current prices come from
**Kroger's official public API** — no scraping, no simulated pricing.

Pinch checks a fixed, curated basket of common grocery search terms (see
`ITEMS` in `api/deals.js`) against nearby stores and surfaces the ones with
an active promo right now. It is **not** a full store-inventory scan and
**not** tied to any weekly ad or circular — Kroger's public API doesn't
expose that data (see "Known limits" below).

Coverage is limited to Kroger-family banners: Kroger, Ralphs, Fred Meyer,
King Soopers, Smith's, QFC, Fry's, Harris Teeter, Mariano's, Pick 'n Save,
and a few others. If none of those are near a ZIP code, you'll get an
empty result — that's expected, not a bug.

## How it works

- `index.html` — static frontend with two tabs. Browser geolocation
  (optional) is reverse-geocoded to a ZIP via Nominatim (OpenStreetMap,
  free/keyless) — manual ZIP entry always works on its own.
  - **Deals** — nearby stores + active promos, from `/api/deals`.
  - **Basket Compare** — pick 3–6 items (default limit — see
    "Configuration" below), get an estimated total per nearby store,
    from `/api/basket`.
- `api/_kroger-client.js` — shared Kroger API client: OAuth2 token
  caching, store lookup, and product-candidate scoring/matching used by
  both endpoints below.
- `api/deals.js` — Vercel serverless function for the Deals tab. Looks up
  nearby Kroger-family stores by ZIP and pulls current regular/promo
  prices for a small tracked item list.
- `api/basket.js` — Vercel serverless function for Basket Compare. Prices
  a user-supplied item list at each nearby store and returns a per-store
  estimated total, built only from items it's confident it matched
  correctly (see "Basket matching rule" below).

Credentials never reach the browser — the frontend only ever calls your
own `/api/deals` and `/api/basket` endpoints.

### Basket matching rule

Each item you add is searched against every nearby store; up to 5
candidate products are fetched and scored against your query by token
overlap with Kroger's own product description — the **highest-scoring**
candidate is used, not just the first one with a price. A result only
counts toward a store's total when:

- a single-word query scores ≥ 0.75, or
- a multi-word query has **every** meaningful word appear in Kroger's
  description (e.g. "sourdough bread" won't match a "Sourdough Dinner
  Roll" — that's missing "bread").

Anything else is `ambiguous` and shown but excluded from the total — the
actual Kroger product name is always displayed, so nothing is ever
silently substituted. Items with no priced candidate at all are
`unavailable`; a failed Kroger request is `error` (and doesn't hide the
rest of the basket's results, since lookups run with bounded concurrency
and per-request timeouts).

A store's total is only ever called an "estimated total" when every
requested item was matched — otherwise it's labeled an "estimated
subtotal for matched items only," and stores are sorted by
(items matched, then total) so a store that priced fewer items can't
out-rank one that priced them all just by having a lower partial sum.

## 1. Get free Kroger API credentials

1. Go to https://developer.kroger.com and create an account.
2. Register a new application.
   - You don't need a redirect URI for this project (we only use the
     `client_credentials` grant, not the user-login flow).
   - Request the `product.compact` scope.
3. Copy your **Client ID** and **Client Secret**.

## 2. Deploy to Vercel (recommended — free tier works)

1. Push this folder to a GitHub repo.
2. Go to https://vercel.com, click **New Project**, and import the repo.
   No build settings needed — it's a static `index.html` plus one
   serverless function in `api/`.
3. Before the first deploy (or after, then redeploy), add environment
   variables under **Project → Settings → Environment Variables**:
   - `KROGER_CLIENT_ID`
   - `KROGER_CLIENT_SECRET`
4. Deploy. Vercel gives you a URL like `pinch.vercel.app` — open it and
   hit "Scan Local Deals".

Pushing to GitHub again auto-redeploys, so your normal `git commit` /
`git push` workflow still works from here.

## 3. Local development (optional)

```bash
npm install -g vercel
cp .env.example .env.local   # fill in your real credentials
vercel dev
```

This serves `index.html` and runs `api/deals.js` locally so you can test
before pushing.

### Testing without Kroger credentials

Set `USE_MOCK_DATA=true` in `.env.local` and both `api/deals.js` and
`api/basket.js` will serve fixtures from `api/_mock-data.js` instead of
calling Kroger. This lets you exercise the UI (including empty and error
states) with no API keys:

- ZIP `00000` → simulates zero nearby stores (both endpoints)
- ZIP `99999` → simulates a provider failure (both endpoints)
- any other valid ZIP → normal fixture data
- in Basket Compare, the item **"trigger error"** simulates one failed
  per-item lookup without a real network failure; unlisted items (e.g.
  "kombucha") simulate `unavailable`; "sourdough bread" is deliberately
  matched to a fixture description missing the word "bread," to exercise
  the `ambiguous` status

Every response includes a `meta` object (`dataSource`, `coverageNote`,
`searchedTerms`, `partial`) so the frontend — and anyone inspecting the
response — can always tell whether they're looking at live Kroger data or
mock data, and whether any per-item lookups failed.

## Configuration

Both endpoints cap their store/item counts and Kroger-call concurrency
through server-side constants, overridable via Vercel environment
variables (no code change needed). None of these are set by default —
the code falls back to the conservative defaults below.

| Env var | Default | Controls |
|---|---|---|
| `DEALS_MAX_STORES` | 5 | Max stores `/api/deals` checks per scan |
| `DEALS_CONCURRENCY` | 6 | Max concurrent Kroger calls in `/api/deals` |
| `DEALS_DEADLINE_MS` | 7000 | How long `/api/deals` keeps scheduling new lookups before returning a partial result |
| `BASKET_MAX_STORES` | 3 | Max stores `/api/basket` checks per basket |
| `BASKET_MAX_ITEMS` | 6 | Max items allowed in one basket |
| `BASKET_CONCURRENCY` | 6 | Max concurrent Kroger calls in `/api/basket` |
| `BASKET_DEADLINE_MS` | 7000 | How long `/api/basket` keeps scheduling new lookups before returning a partial result |

**These defaults are deliberately conservative and have not been
timed against a live Vercel deployment.** `BASKET_MAX_ITEMS` defaults to
6 (not the 10 the feature was originally designed for) specifically
because 4 stores × 10 items = 40 Kroger calls was never measured against
Vercel's function timeout — only assumed safe. Raise these only after
running the smoke test in the Deployment Checklist below against a real
deployment and confirming `durationMs` (see "Observability") stays
comfortably under your plan's function timeout with margin to spare. If
`index.html`'s `BASKET_MAX_ITEMS` constant is raised to match, keep the
two in sync — the frontend cap is a UX nicety, not a security boundary
(the server enforces its own limit regardless of what the client sends).

### The deadline mechanism

Both endpoints record a request-start timestamp and compute an absolute
deadline (`start + *_DEADLINE_MS`). Kroger calls run through a bounded
worker pool (`mapWithConcurrency` in `api/_kroger-client.js`); once the
deadline passes, any lookup that hasn't started yet is marked `error`
(for basket items) or treated as a failed price lookup (for deals)
**without making a new network call**, and an in-flight call's own
timeout shrinks to whatever time is left. This means:

- The function always returns a response — including whichever prices
  did resolve in time — rather than running long enough to hit Vercel's
  own timeout and produce a bare 504 with no partial data.
- `meta.partial: true` (and, for basket, an added note in
  `coverageNote`) tells the frontend when this happened.
- Total worst-case runtime is bounded by `*_DEADLINE_MS` plus one
  in-flight call's remaining timeout — not by (stores × items) growing
  unchecked.

## Observability

Both `api/deals.js` and `api/basket.js` emit one JSON line to
`console.log` per request (visible in Vercel's function logs, or your
terminal under `vercel dev`), with:

- `durationMs` — total handler time
- `storeCount` — stores actually checked
- `lookupCount` — Kroger product calls attempted (deals: `stores × 6`;
  basket: `stores × items`, both before any deadline skips)
- `matchedCount` (basket) / `dealCount` (deals) — successful results
- `partial` — true if any lookup errored or was skipped past the deadline
- `ranOutOfTime` — true if the request actually hit its deadline
  (distinct from `partial`, which can also be true from an ordinary
  per-call failure with no deadline involved)
- `providerStatus` — `"ok"` or `"error"`, plus `errorMessage` on failure

There's no dashboard or alerting wired up — this is deliberately just
structured `console.log` output, since Vercel's own log viewer (or `vercel
logs`) is enough to eyeball these during and after the smoke test below.

## Known limits (worth knowing for a first live version)

- **Rate limit**: Kroger's Products API allows 10,000 calls/day per app.
  Each scan uses roughly (stores × items) calls — comfortably inside
  that limit for personal use at the current defaults.
- **Function timeout is mitigated, not verified.** The deadline mechanism
  above stops the function from running past its own budget, but the
  actual wall-clock time of a real request against Kroger's live API
  has not been measured. Treat the current defaults (3 stores / 6 items
  for basket, 5 stores / 6 items for deals) as the starting point for the
  live smoke test, not as a proven-safe ceiling.
- **"Deals" = active promos only.** The backend only returns an item as
  a "deal" when Kroger's own API reports a live promo price lower than
  regular price — so a quiet week can mean a thin or empty list, which
  is correct behavior, not a broken scan.
- **Product matching is fuzzy.** Each tracked item is a text search term
  (e.g. "sourdough bread"); the highest-scoring candidate wins, but scoring
  is plain token overlap against Kroger's description — no synonym or
  brand awareness. Basket Compare uses a stricter bar and labels weak
  matches `ambiguous` rather than guessing; the Deals tab's curated
  6-item list has always matched well in practice, so it doesn't need the
  same gating.
- **Basket totals are estimates, not receipts.** No tax, coupons, loyalty
  pricing, digital-coupon clipping, or bag fees are reflected — only the
  per-item listed price Kroger's API returns for that store right now.
- **Basket Compare checks fewer stores and fewer items than Deals could
  fit** (3 stores / 6 items vs. Deals' 5 stores / 6 fixed items) — a
  conservative starting point, not a measured ceiling; see "Configuration"
  above.

## Deployment Checklist

Run through this once, in order, before treating a deployment as
production-ready. **Never paste your Client Secret (or any credential)
into chat, a commit, or anywhere other than Vercel's Environment
Variables UI or your own local `.env.local`.**

### 1. Create a Kroger developer application

1. Go to https://developer.kroger.com and sign in or create a free
   account.
2. Go to **My Apps** (or **Add App**) and register a new application.
   - Redirect URI: not required — this project only uses the
     `client_credentials` grant (server-to-server), never the
     user-login/Authorization Code flow.
   - Scope: request **`product.compact`**. That's the only scope both
     `/api/deals` and `/api/basket` use (`getToken()` in
     `api/_kroger-client.js` requests it explicitly).
3. Once approved, open the app's detail page and copy the **Client ID**
   and **Client Secret** into your own password manager or directly into
   Vercel (next step) — not into a text file, a chat message, or a
   commit.
4. Confirm the app is approved for the **Public** Products/Locations API
   (not just Partner-tier docs) — the free public tier is what this
   project's endpoints (`https://api.kroger.com/v1/...`) call.

### 2. Set environment variables

**Vercel (production)** — Project → Settings → Environment Variables,
scoped to Production (and Preview, if you want PR previews to hit live
Kroger data too):

| Variable | Value |
|---|---|
| `KROGER_CLIENT_ID` | your app's Client ID |
| `KROGER_CLIENT_SECRET` | your app's Client Secret |
| `USE_MOCK_DATA` | unset, or `false` |

Leave the `DEALS_*` / `BASKET_*` limit variables from "Configuration"
unset for the first deploy — the conservative code defaults apply
automatically. Only add them once the smoke test below gives you real
timing data to justify a change.

**Local mock mode** (`.env.local`, via `cp .env.example .env.local`) —
for UI development with no real credentials:

| Variable | Value |
|---|---|
| `USE_MOCK_DATA` | `true` |
| `KROGER_CLIENT_ID` / `KROGER_CLIENT_SECRET` | can stay as placeholders — unused in mock mode |

Never set `USE_MOCK_DATA=true` in the Vercel Production environment —
that would serve fixture data to real users.

### 3. Deploy

1. Push this repo to GitHub (if not already).
2. In Vercel: **New Project** → import the repo. No build settings
   needed — it's a static `index.html` plus serverless functions in
   `api/`.
3. Set the environment variables above **before** the first deploy (or
   set them after, then trigger a redeploy — env var changes don't apply
   to already-built deployments).
4. Deploy. Vercel gives you a URL like `https://<project>.vercel.app`.
5. Every subsequent `git push` to the connected branch auto-redeploys.

### 4. Live smoke test (one real ZIP, one 3-item basket)

Run this against the deployed URL, not `localhost`, with real credentials
in place and `USE_MOCK_DATA` unset. Pick a ZIP you know has Kroger-family
coverage (e.g. a Kroger, Ralphs, or Fred Meyer you can verify on Google
Maps) — using an arbitrary ZIP risks a false "no stores" result that
looks like a bug but isn't.

```bash
# 1. Deals — confirms token/store/product lookups and live pricing
curl -s "https://<project>.vercel.app/api/deals?zip=<REAL_ZIP>" | head -c 2000

# 2. Basket Compare — confirms the POST path, matching rule, and totals
curl -s -X POST "https://<project>.vercel.app/api/basket" \
  -H "Content-Type: application/json" \
  -d '{"zip":"<REAL_ZIP>","items":["whole milk gallon","large eggs","bananas"]}'
```

For each call, note the wall-clock time (`curl -w "\n%{time_total}s\n"`
appended, or just time it manually) and cross-check it against the
`durationMs` value in that request's Vercel function log — this is the
actual timing data the "Configuration" section says to gather before
raising any limit.

Then, in a real browser against the deployed URL: load the Deals tab
(confirm it auto-scans), switch to Basket Compare, add 3 items, and
confirm results render — this catches anything the `curl` calls alone
wouldn't (CORS, static asset paths, tab-switching JS).

### 5. What each response should look like

- **Successful** — HTTP 200. `stores` non-empty, `meta.dataSource:
  "kroger"`, `meta.partial: false`. Basket: at least one store has
  `isComplete: true` (all 3 test items matched) if the ZIP has good
  coverage; `deals`/basket `items[].status` values match real Kroger
  data you can spot-check in the Kroger app or website for that store.
- **Partial** — HTTP 200 still (not an error status). `meta.partial:
  true`. For basket, look at `meta.coverageNote` for the "did not finish
  in time" sentence, and check individual `items[].status === "error"`
  entries — those are the ones that failed or got deadline-skipped.
  Check the matching Vercel log line for `ranOutOfTime: true` to confirm
  it was the deadline (vs. an ordinary per-call failure).
- **Unauthorized** — if `KROGER_CLIENT_ID`/`SECRET` are wrong, missing,
  or the app isn't approved: HTTP 502 with `{"error": "Kroger auth
  failed (401)"}` (the message includes Kroger's real status code,
  surfaced from `getToken()` in `api/_kroger-client.js`). This should
  only ever happen misconfigured — a correctly configured deploy
  shouldn't see this in normal operation.
- **Rate-limited**: HTTP 502 with `{"error": "Kroger location lookup
  failed (429)"}` or `"Kroger product lookup failed (429)"` — the
  specific failing call's status code is in the message. At the
  conservative default call volumes this should be very unlikely to
  occur outside of intentionally hammering the endpoint.
- **Timed out**: with the deadline mechanism in place, a full function
  timeout (a bare Vercel 504 with no JSON body at all) should no longer
  happen under normal conditions — that's the failure mode this session's
  hardening work targets. If you see a 504 instead of a 200-with-partial
  result, that's a signal `*_DEADLINE_MS` is set too close to (or above)
  the actual Vercel function timeout and needs to be lowered, or that
  something outside this code (cold start, DNS, Vercel platform latency)
  is eating the time budget before the deadline logic even starts
  counting.

### 6. Metrics/logs to capture

Already emitted as one JSON line per request (see "Observability"
above) — no extra instrumentation needed, just watch Vercel's function
logs (`vercel logs <deployment-url>` or the dashboard) during the smoke
test:

- `durationMs` — total request duration
- `storeCount` — stores checked
- `lookupCount` — product lookups attempted
- `matchedCount` (basket) / `dealCount` (deals) — successful results
- `partial` — whether any lookup failed or was deadline-skipped
- `providerStatus` — `"ok"` or `"error"` (with `errorMessage` on failure)

Record these numbers from the smoke test in whatever tracker you use for
this project — they're the evidence needed to justify raising
`BASKET_MAX_ITEMS`/`BASKET_MAX_STORES` beyond the conservative defaults
later, per "Configuration" above.
