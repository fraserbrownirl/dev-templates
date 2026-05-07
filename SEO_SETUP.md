# SEO Setup — per-project contract

> **What this file is.** The locked specification for adding build-time prerendering to a Lovable Vite + React project, with deployment via GitHub Actions to Cloudflare Pages. This file lives at the root of each Lovable repo. Both Lovable (when prompted) and Cursor (during setup and drift recovery) read it as the source of truth.
>
> **What this file is not.** A discussion document. Every decision below has been made deliberately. Do not redesign — implement.

---

## The problem this solves

Lovable scaffolds Vite + React SPAs whose initial HTML response is `<div id="root"></div>` plus a script tag. Googlebot's first-pass crawl sees an empty shell. Per-route content, titles, and meta tags never reliably reach the index, producing "Discovered – not indexed" results and failing rankings on what should be ranking content.

The fix is build-time prerendering with headless Chromium (Puppeteer). At build time, Puppeteer crawls each route of the locally-built SPA, captures the post-hydration HTML (Helmet meta included), and writes one static HTML file per route. Cloudflare Pages serves those files directly. Google sees real HTML on first crawl. The site indexes.

This is build-time only. No edge SSR. No user-agent sniffing. No runtime React rendering on a server.

---

## Architecture

**Build:** GitHub Actions on `ubuntu-latest`, Node 22. Puppeteer runs against the locally-built SPA, served via `serve-handler` on a random port. Captures and writes static HTML per route, then a second Puppeteer pass renders one OG PNG per content row. ~6 minutes for ~400 routes + ~50 OG images at concurrency 4.

**Host:** Cloudflare Pages in **Direct Upload** mode. No Git integration on Cloudflare. GitHub Actions pushes built `dist/` via `npx wrangler@4 pages deploy`. A small Pages middleware (`functions/_middleware.ts`) handles canonical-host enforcement, apex → www 301, and OG fallback (per §"Permitted Pages middleware") — no SSR, no rendering.

**Why this split:** Cloudflare's free tier has unlimited bandwidth (vs. Netlify's 100 GB/mo cap). GitHub Actions Ubuntu runners are a documented, reliable environment for headless Chromium (vs. Cloudflare's Pages build environment, which is undocumented territory for Puppeteer). Splitting "where you build" from "where you serve" gets both wins.

**Lovable's Publish button:** vestigial / preview-only. Production deploys flow through GitHub Actions only. The live site Google sees is the Cloudflare deploy, not the Lovable preview.

---

## Components

### Routing source of truth: `src/routes.ts`

Every route in the app declared in one file. Both `<Routes>` in `App.tsx` and `prerender.mjs` import from it. No duplication.

```ts
export type AppRoute = {
  path: string;
  getStaticPaths?: () => Promise<string[]>;
};

export const routes: AppRoute[] = [
  { path: "/" },
  { path: "/about" },
  { path: "/categories" },
  // ... static routes
  {
    path: "/:slug",
    getStaticPaths: async () => {
      // Query Supabase at build time using the publishable key.
      // Return concrete paths like ["/foo", "/bar"].
    },
  },
];
```

Dynamic routes have a `getStaticPaths` async function that returns concrete URL paths. The prerender script expands these into the full crawl set.

### Hydration signal: `__PRERENDER_READY__`

Puppeteer cannot reliably know when React is "done" rendering a route. `networkidle0` is flaky against Supabase realtime/websockets. Arbitrary `setTimeout` is wrong. The correct signal is explicit.

In `src/main.tsx`:

```ts
const pending = new Set<Promise<unknown>>();
(window as any).__trackPending = (p: Promise<unknown>) => {
  pending.add(p);
  p.finally(() => pending.delete(p));
};
// After mount and on every route change, when pending.size === 0
// AND the route component has rendered, set:
(window as any).__PRERENDER_READY__ = true;
// Reset to false on route change.
```

Page components wrap their Supabase queries:

```ts
const promise = supabase.from('topics').select('*');
(window as any).__trackPending?.(promise);
const { data } = await promise;
```

Puppeteer waits on `window.__PRERENDER_READY__ === true` with a 45-second timeout per route (`PRERENDER_READY_MS`, override per build). The flag flips only when the route is mounted **and** all tracked promises have settled.

### Per-route meta: `react-helmet-async`

Each page component owns its `<title>`, `<meta name="description">`, `<meta property="og:*">`, and canonical URL via `<Helmet>`. Helmet writes to the live DOM. Puppeteer captures the post-Helmet DOM. The static HTML matches what the SPA serves at runtime.

Wrap the app root in `<HelmetProvider>` in `main.tsx`.

### Prerender script: `prerender.mjs` (project root)

Behavior:

1. Top-of-file guard: `if (process.env.PRERENDER !== '1') process.exit(0);`. Local `npm run build` stays fast unless explicitly opted into prerendering.
2. Import `routes` from `src/routes.ts` via `tsx`.
3. Resolve all `getStaticPaths()` async. If `STRICT_PRERENDER=1` is set, abort with non-zero exit when any `getStaticPaths` returns zero rows (sanity assertion: catches misconfigured Supabase env vars). The strict gate exists so a freshly-deployed project with an empty content table can ship its first build before the content sync runs; flip it on once the table is reliably populated.
4. Boot `serve-handler` on a random free port over `dist/` with SPA-fallback rewrite (`{ source: "**", destination: "/index.html" }`) so client-side route segments resolve.
5. Launch Puppeteer headless Chromium. Create a page pool with size from `process.env.PRERENDER_CONCURRENCY` (default 4). Workers pull URLs off a queue.
6. Per URL: navigate (`PRERENDER_GOTO_MS`, default 30000) → `page.waitForFunction('window.__PRERENDER_READY__ === true', { timeout: PRERENDER_READY_MS })` (default 45000) → capture `document.documentElement.outerHTML` → write to `dist/{path}/index.html` (root path → `dist/index.html`, overwriting the SPA shell).
7. **Retry once on timeout.** N concurrent workers hit the same Supabase project in parallel and ~1–3 % of routes race-lose per build; requeue timed-out URLs to the back of the queue with `attemptsLeft = 1`. Non-timeout errors are not retried.
8. Emit `dist/sitemap.xml` and `dist/robots.txt` — see §"Sitemap + robots".
9. Post-emit secret-leak grep over everything written under `dist/`. Patterns: `service_role`, `\beyJ[A-Za-z0-9_-]{20,}` (long JWT), `postgres://`. Build fails on match. Allowlist: strip the `VITE_SUPABASE_PUBLISHABLE_KEY` value before re-testing the JWT regex — that key is shipped to clients by design and would otherwise always trip the grep.
10. Log each route as `✓ /path`, `↻ /path: timeout, requeued`, or `✗ /path: <error>`. Exit non-zero on any unrecovered route failure.

Concurrency tunable without code edits: `PRERENDER_CONCURRENCY=8 PRERENDER=1 npm run build` for local stress tests; production stays at 4. Chromium uses ~200–300 MB per page; 4 pages on a 7 GB GitHub Actions runner is comfortably safe.

### SPA fallback: `public/_redirects`

Single line:

```
/*  /index.html  200
```

Vite copies `public/` into `dist/` automatically. Cloudflare Pages reads `dist/_redirects` for any route the prerenderer didn't emit. Real `dist/{route}/index.html` files take precedence; the SPA shell is only served for genuinely unknown routes.

### Sitemap + robots: emitted by `prerender.mjs`

`prerender.mjs` is the only thing in the build chain that has the full resolved route list, so it's the only sane place to write the sitemap. Don't duplicate the route enumeration in a sidecar script.

`dist/sitemap.xml`:

- One `<url>` per resolved route, including dynamic paths from `getStaticPaths`.
- `<loc>` is the canonical (slash) form: `${VITE_SITE_URL}/route/`. Cloudflare Pages 308-redirects no-slash → slash for any `dist/<route>/index.html`; emitting the slashed form keeps Google off the redirect path.
- `<lastmod>` for content-detail routes (e.g. `/card/:slug`) comes from the primary content table's `updated_at` (or `last_synced_at`); everything else uses the build date.
- Per-route `<priority>` and `<changefreq>` driven by a project-local `urlMetadata(url)` helper. Defaults: `/` → `1.0/daily`, listing pages → `0.9/weekly`, detail pages → `0.8/weekly`, boilerplate (`/privacy`, `/terms`) → `0.3/monthly`.

`dist/robots.txt`:

```
User-agent: *
Allow: /
Disallow: /admin
Sitemap: ${VITE_SITE_URL}/sitemap.xml
```

Add other `Disallow:` lines as needed (e.g. throwaway routes like `/compare/*` that exist for the SPA but shouldn't burn crawl budget).

`VITE_SITE_URL` is required for both files. Build fails with a clear message if unset.

### Per-route OG images: `og-render.mjs` + `og-template.html`

Build-time PNG generator for OpenGraph images. Same Puppeteer install as the prerender step. Same gate (`PRERENDER=1`).

`og-template.html` (root): static URL-param-driven HTML template that visually mirrors the site's brand tokens (font, colors, spacing). Reads `?title=`, `?subtitle=`, `?logo=`, `?badge=`, `?rating=` from `location.search`. Sets `window.__OG_READY__ = true` after `document.fonts.ready` plus a short layout-settle delay.

`og-render.mjs` (root):

1. `if (process.env.PRERENDER !== '1') process.exit(0);` — same gate as prerender.
2. Boot `serve-handler` over the **repo root** (not `dist/`) with `cleanUrls: false, trailingSlash: false`. Default cleanUrls would 301 `/og-template.html?…` → `/og-template`, dropping the query string and breaking every render.
3. Fetch the primary content table from Supabase (`VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`). On no-env or fetch failure, render only `default.png`.
4. Concurrency from `OG_CONCURRENCY` (default 4). Per item: navigate to `/og-template.html?...` → `page.waitForFunction('window.__OG_READY__ === true', { timeout: OG_READY_MS })` (default 30000) → screenshot body at 1200×630 → write `dist/og/<slug>.png`.
5. **Two-pass tolerance.** First-pass timeout failures are requeued onto a single-worker retry pass (so Google Fonts isn't being hammered by 4 parallel pages). Manifest only lists slugs whose PNG was actually written. Exit non-zero only if `failed > 5`; small handfuls of missing per-card PNGs fall back to the default OG via the Pages middleware (see below).
6. Emit `dist/og/_manifest.json` mapping `slug → relative png path` so pages know what's available without a runtime FS check.
7. Skip already-rendered files unless `OG_FORCE=1` so incremental local reruns are cheap.

Build script chains it after prerender:

```json
"build": "vite build && node prerender.mjs && node og-render.mjs"
```

### Permitted Pages middleware: `functions/_middleware.ts`

Cloudflare Pages serves the same build artifact at multiple URLs (production `*.pages.dev`, every `<branch>.pages.dev`, and any custom domain attached to the project). A small Pages middleware solves three universal SEO hazards that no static-only deploy can address. **This is the only edge code permitted by this spec.**

Permitted in middleware:

- Response header mutation (`X-Robots-Tag`, `Cache-Control`, `Strict-Transport-Security`, etc.).
- HTTP redirects (`301`/`308` apex → www, scheme upgrade, legacy-path rewrites).
- Substitution of static files served from the deploy itself (e.g. OG fallback PNG when SPA rewrite hits a missing asset and serves `index.html` HTML in an image slot).

Forbidden in middleware:

- Database queries (Supabase or otherwise).
- HTML rendering or React execution.
- Third-party API fetches.
- User-data branching (auth, sessions, A/B).

Bright line: **middleware may rewrite responses, never produce new content.** If you need new content, prerender it. If you need session logic, do it client-side.

Reference behaviors every project should implement:

1. **Apex → www 301.** CF Pages' built-in apex redirect emits 307 (temporary), which leaks link equity and confuses Google's canonical consolidation. Force 301 in middleware before anything else runs.
2. **`X-Robots-Tag: noindex, nofollow, noarchive` on every non-canonical host.** Maintain a `CANONICAL_HOSTS` set; emit the header on every other Host. JS-disabled scrapers and Google's pre-render cache both honour the header, so it protects against indexing even when a JS-injected `<meta name="robots">` would be skipped. Onboard a new canonical host by appending it to the set.
3. **OG image fallback.** When `/og/<slug>.png` would 404 and the SPA rewrite returns `index.html` (HTML in an image slot), substitute `/og/default.png` with a short `Cache-Control: public, max-age=300` so a successful re-render replaces it quickly.

### GitHub Actions workflow: `.github/workflows/deploy.yml`

```yaml
name: Build and deploy to Cloudflare Pages
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Build with prerender + OG render
        env:
          VITE_SUPABASE_URL: ${{ secrets.VITE_SUPABASE_URL }}
          VITE_SUPABASE_PUBLISHABLE_KEY: ${{ secrets.VITE_SUPABASE_PUBLISHABLE_KEY }}
          VITE_SITE_URL: ${{ secrets.VITE_SITE_URL }}
          PRERENDER: '1'
          PRERENDER_CONCURRENCY: '4'
          OG_CONCURRENCY: '4'
          # STRICT_PRERENDER: '1'  # enable once the primary content table is reliably populated
        run: npm run build
      - name: Deploy to Cloudflare Pages
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        run: npx --yes wrangler@4 pages deploy ./dist --project-name=PROJECT_NAME_HERE --branch=main  # TODO(human/cursor): replace PROJECT_NAME_HERE with actual Cloudflare Pages project name
```

**Node version: 22.** `@supabase/supabase-js` requires native `WebSocket`, available in Node ≥ 22. Don't downgrade.

**Wrangler invocation: direct `npx wrangler@4`.** `cloudflare/wrangler-action@v3` defaults to wrangler 3.x and lags behind the current `cfut_` token format; calling wrangler@4 directly with `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` in `env` is cleaner and forward-compatible.

`timeout-minutes: 20` gives ~3× the expected ~6-minute run time (prerender + OG render). Anything beyond is a hang and should fail fast.

`workflow_dispatch` enables manual re-runs from the GitHub UI without a code push. Use this when content changes (e.g. new rows in a Supabase table that drives prerendered pages) require a fresh build without code changes.

**PR/branch preview deploys (optional).** Add `pull_request: { branches: [main] }` to the trigger and replace `--branch=main` with a dynamic branch derived from `github.head_ref` / `github.ref_name`. Cloudflare Pages serves each branch at `<branch>.<project>.pages.dev`. Skip the deploy step on fork PRs (no secrets):

```yaml
if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
```

The Pages middleware (§"Permitted Pages middleware") emits `X-Robots-Tag: noindex` on `*.pages.dev`, so preview URLs are never indexed.

### `package.json` scripts

```json
"build": "vite build && node prerender.mjs && node og-render.mjs",
"build:client": "vite build"
```

`PRERENDER=1` lives in the workflow env block, not the script. Local `npm run build` exits early at the top of both `prerender.mjs` and `og-render.mjs`. Local `PRERENDER=1 npm run build` runs the full pipeline for verification.

**Required devDependencies:** `puppeteer`, `serve-handler`, `tsx`.

---

## Files touched

| File | Change |
|---|---|
| `.github/workflows/deploy.yml` (new) | GitHub Actions: install, build with PRERENDER=1, `npx wrangler@4 pages deploy`. `timeout-minutes: 20`, Node 22. |
| `public/_redirects` (new) | `/*  /index.html  200` |
| `src/routes.ts` (new) | Static routes + async `getStaticPaths` for dynamic routes |
| `src/App.tsx` | Map `<Routes>` from `src/routes.ts` |
| `src/main.tsx` | Install `window.__trackPending` + pending-Set + `__PRERENDER_READY__` signal after mount; wrap app in `<HelmetProvider>` |
| `prerender.mjs` (new, root) | Puppeteer crawler + sitemap.xml + robots.txt — see §"Prerender script" and §"Sitemap + robots" |
| `og-render.mjs` (new, root) | OG image renderer — see §"Per-route OG images" |
| `og-template.html` (new, root) | URL-param-driven OG template loaded by `og-render.mjs` |
| `functions/_middleware.ts` (new) | Pages middleware — see §"Permitted Pages middleware" |
| `src/pages/*.tsx` | Wrap Supabase calls in `__trackPending`; use `<Helmet>` for per-route title/description/canonical/OG |
| `package.json` | Add `puppeteer`, `serve-handler`, `tsx` to devDependencies; `react-helmet-async` to dependencies; update `build` and `build:client` scripts |
| `.gitignore` | Verify `dist/` and `node_modules/` are listed. |

**Explicitly not created:** `netlify.toml`, `_headers`, `.nvmrc`, `cloudflare.toml`, any SSR server. Pages middleware permitted only under §"Permitted Pages middleware".

---

## Build matrix

| Environment | Command | PRERENDER | Output |
|---|---|---|---|
| GitHub Actions production | `npm run build` | `1` (set in workflow env) | `dist/` per-route HTML, deployed to Cloudflare |
| Local prerender test | `PRERENDER=1 npm run build` | `1` | full prerender locally |
| Lovable preview | `npm run build:client` | n/a | SPA shell only |
| Local dev | `npm run dev` | n/a | unchanged |

---

## Required GitHub Actions secrets

Set in repo Settings → Secrets and variables → Actions. Cursor handles this via GitHub MCP during initial setup; manual steps below are the fallback.

| Secret | Source |
|---|---|
| `CLOUDFLARE_API_TOKEN` | Cloudflare dashboard → My Profile → API Tokens → "Edit Cloudflare Workers" template (covers Pages). Account-scoped, Pages Write only is sufficient. New tokens use the `cfut_` prefix; pair with `npx wrangler@4`, not `wrangler-action@v3`. |
| `CLOUDFLARE_ACCOUNT_ID` | Visible in any Cloudflare zone's dashboard sidebar. |
| `VITE_SUPABASE_URL` | Same value as in `.env`. |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Same value as in `.env`. **Use this exact env var name. Do NOT use `VITE_SUPABASE_ANON_KEY`.** |
| `VITE_SITE_URL` | Canonical absolute URL of the production site, no trailing slash (e.g. `https://www.example.com`). Used for sitemap `<loc>`, robots `Sitemap:`, canonical Helmet tags, and OG `og:url`. |

**`gh secret set` gotcha.** `printf x \| gh secret set NAME --body -` interprets `--body -` as the literal value `-`, not "read from stdin". Always use `gh secret set NAME --body "value"` or pipe stdin without `--body`. This bites every time it's used wrong.

---

## Security posture

- Build-time DB access uses the **publishable key** (the anon key already shipped to clients). No service-role key at build time, ever.
- Tables read at build time must have public-select RLS enabled. Verify this for any new dynamic-route table.
- Sanity assertion in `prerender.mjs`: when `STRICT_PRERENDER=1`, abort with non-zero exit if any `getStaticPaths` returns zero rows. Catches misconfigured env vars before they ship a build with missing routes. Leave the gate off only while the primary content table is provably empty (initial deploy); flip it on as soon as content is reliably present.
- Post-emit secret-leak grep over everything in `dist/`. Patterns: `service_role`, `\beyJ[A-Za-z0-9_-]{20,}` (long JWT), `postgres://`. Build fails on match. **Allowlist:** strip the `VITE_SUPABASE_PUBLISHABLE_KEY` value from each file before re-testing the JWT regex — the publishable key ships to clients by design and would otherwise always trip the grep.

---

## What is and isn't in scope

**In scope:**
- Build-time prerendering of all routes declared in `src/routes.ts`.
- Per-route meta tags via Helmet.
- Per-route OG image generation.
- `sitemap.xml` + `robots.txt` emission.
- Static deployment to Cloudflare Pages via GitHub Actions.
- SPA fallback for unknown routes.
- Manual rebuild trigger via `workflow_dispatch`.
- Pages middleware for header mutation, redirects, and static-file substitution (per §"Permitted Pages middleware").
- Optional PR/branch preview deploys (per §"GitHub Actions workflow" → "PR/branch preview deploys (optional)").

**Out of scope:**
- User-agent-based cloaking or differential rendering.
- Edge SSR or any runtime React rendering.
- Pages middleware doing DB calls, HTML rendering, third-party fetches, or user-data branching.
- Service-role key access at build time.
- `netlify.toml`, `_headers`, `.nvmrc`, `cloudflare.toml`.
- DNS changes performed by Lovable. (Cursor handles DNS automation via Cloudflare MCP when the zone is on Cloudflare. Otherwise the human adds DNS records manually at their registrar.)
- Supabase-trigger → `workflow_dispatch` webhook for auto-rebuild on content edits. (Manual re-run via GitHub UI for now.)
- Caching of `~/.cache/puppeteer` across runs. (Chromium re-downloads ~170 MB per fresh runner; not worth complexity at this scale.)

---

## Verification checklist

After implementation, the following must be true:

1. Locally: `PRERENDER=1 npm run build` produces `dist/index.html`, `dist/<route>/index.html` for every static route, `dist/<dynamic-path>/index.html` for every path resolved from `getStaticPaths`, plus `dist/sitemap.xml`, `dist/robots.txt`, and `dist/og/<slug>.png` per primary-content row.
2. View source on a sample of those files → unique `<title>`, unique `<h1>`, real content visible without JavaScript, internal `<a href>` links present.
3. With `STRICT_PRERENDER=1` set: sanity assertion fires when Supabase env vars are blanked. (Verify by temporarily clearing `VITE_SUPABASE_URL` and re-running; build must abort.)
4. Concurrency tunable: `PRERENDER_CONCURRENCY=8 PRERENDER=1 npm run build` succeeds. Default stays at 4 for production.
5. Secret-leak grep passes on the build log (with the publishable-key allowlist applied).
6. After first GitHub Actions deploy: `curl -s https://<project>.pages.dev/<some-route>` returns HTML with the route's actual content in the response body — not the SPA shell.
7. `curl -sI https://<branch>.<project>.pages.dev/` includes `X-Robots-Tag: noindex, nofollow, noarchive`. `curl -sI https://<apex>` returns `301` to `https://www.<apex>` (when both are mapped).
8. `/og/<known-slug>.png` returns `image/png`. `/og/<bogus>.png` falls back to `image/png` with `Cache-Control: max-age=300` (the default OG via middleware).
9. Client-side navigation between routes still feels instant (no full page reloads). React hydration succeeds without console errors.
10. After DNS cutover: GSC "Inspect URL" on 5–10 sample URLs → "Request indexing" returns success.

---

## Sequencing for implementation

When Lovable (or any agent) implements this spec from scratch:

1. **Migrations first**, if the project uses Supabase tables for dynamic routes. The tables and RLS policies must exist before components query them.
2. **Routing plumbing**: `src/routes.ts`, `src/main.tsx` signal + tracker, `src/App.tsx` route mapping, `<HelmetProvider>` wrapping.
3. **Page edits**: wrap Supabase calls in `__trackPending`, add Helmet to each page.
4. **Prerender script**: `prerender.mjs` at project root with all behaviors listed under §"Prerender script" and §"Sitemap + robots".
5. **OG renderer**: `og-render.mjs` + `og-template.html` at project root (§"Per-route OG images").
6. **Pages middleware**: `functions/_middleware.ts` (§"Permitted Pages middleware").
7. **Build plumbing**: `package.json` scripts and devDependencies, `public/_redirects`.
8. **Workflow file**: `.github/workflows/deploy.yml` with `PROJECT_NAME_HERE` placeholder and `TODO(human/cursor)` comment.
9. **Local verification**: `PRERENDER=1 npm run build`, inspect `dist/`, confirm each item in the verification checklist.
10. **Handoff note**: leave a clear note that the workflow file's `PROJECT_NAME_HERE` placeholder needs filling in, secrets (incl. `VITE_SITE_URL`) need setting, and the Cloudflare Pages project needs creating — all of which happen in the Cursor phase, not the Lovable phase.

---

## Drift recovery

Lovable will sometimes rewrite `package.json`, `src/main.tsx`, `prerender.mjs`, or other files this spec describes. When the GitHub Actions build fails because of this:

1. Cursor reads the failed workflow run log via GitHub MCP.
2. Cursor diffs the current state of relevant files against this spec.
3. Cursor proposes a fix and waits for human approval before writing.
4. On approval, Cursor writes the fix via GitHub MCP and retriggers `workflow_dispatch`.

The pasteable DRIFT prompt lives in `~/dev/templates/SETUP_PROMPT.md`. Drift recovery is the recurring operational mode of the system, not a rare event.

---

## Hard rules

- Do not add SSR or runtime React rendering. Pages middleware permitted only under §"Permitted Pages middleware" (header mutation, redirects, static-file substitution; no DB calls, no rendering, no third-party fetches).
- Do not use the Supabase service-role key at build time.
- Do not duplicate the route list. `src/routes.ts` is the only place routes are declared. The sitemap/OG renderer/prerender all consume it.
- Do not skip the `__PRERENDER_READY__` signal in favor of `setTimeout` or `networkidle`.
- Do not pin Node below 22. `@supabase/supabase-js` requires native `WebSocket`.
- Do not call wrangler via `cloudflare/wrangler-action@v3`. Use `npx --yes wrangler@4` directly with `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` in `env`.
- Do not create `netlify.toml`, `_headers`, `.nvmrc`, or `cloudflare.toml`. None apply here.
- Do not modify `vite.config.ts` beyond what's strictly necessary for prerender support.
- Do not bake secrets into any committed file.

If Lovable encounters an instruction during a prompt-edit that conflicts with anything in this spec, it must surface the conflict rather than silently choose one.
