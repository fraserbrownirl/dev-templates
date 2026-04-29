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

**Build:** GitHub Actions on `ubuntu-latest`. Puppeteer runs against the locally-built SPA, served via `serve-handler` on a random port. Captures and writes static HTML per route. ~5 minutes for ~400 routes at concurrency 4.

**Host:** Cloudflare Pages in **Direct Upload** mode. No Git integration on Cloudflare. GitHub Actions pushes built `dist/` via `wrangler pages deploy`.

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

Puppeteer waits on `window.__PRERENDER_READY__ === true` with a 10-second timeout per route. The flag flips only when the route is mounted **and** all tracked promises have settled.

### Per-route meta: `react-helmet-async`

Each page component owns its `<title>`, `<meta name="description">`, `<meta property="og:*">`, and canonical URL via `<Helmet>`. Helmet writes to the live DOM. Puppeteer captures the post-Helmet DOM. The static HTML matches what the SPA serves at runtime.

Wrap the app root in `<HelmetProvider>` in `main.tsx`.

### Prerender script: `prerender.mjs` (project root)

Behavior:

1. Top-of-file guard: `if (process.env.PRERENDER !== '1') process.exit(0);`. Local `npm run build` stays fast unless explicitly opted into prerendering.
2. Import `routes` from `src/routes.ts` via `tsx`.
3. Resolve all `getStaticPaths()` async — abort with non-zero exit if any returns zero rows (sanity assertion: catches misconfigured Supabase env vars).
4. Boot `serve-handler` on a random free port over `dist/`.
5. Launch Puppeteer headless Chromium. Create a page pool with size from `process.env.PRERENDER_CONCURRENCY` (default 4). Workers pull URLs off a queue.
6. Per URL: navigate → `page.waitForFunction('window.__PRERENDER_READY__ === true', { timeout: process.env.PRERENDER_READY_TIMEOUT_MS ?? 20000 })` → capture `document.documentElement.outerHTML` → write to `dist/{path}/index.html` (root path → `dist/index.html`, overwriting the SPA shell). On failure, retry once with the same page (configurable via `PRERENDER_RETRIES`, default 2 attempts total).
7. Post-emit secret-leak grep over everything written under `dist/`. Patterns: `service_role`, `\beyJ[A-Za-z0-9_-]{20,}` (broad JWT-prefix; catches bare tokens), `postgres://`. Build fails on match.
8. Log each route as `✓ /path` or `✗ /path: <error>`. Exit non-zero on any route failure after all retries are exhausted.

Tunables (all environment variables, no code edits):

| Var | Default | Notes |
|---|---|---|
| `PRERENDER_CONCURRENCY` | `4` | Use `1` on GitHub Actions free runners — their network throughput to Supabase REST flakes under parallel load (validated 2026-04-29). Local 8-core Macs handle 4 reliably. |
| `PRERENDER_READY_TIMEOUT_MS` | `20000` | Bump to `45000` for projects with heavy "load all rows" queries on the homepage. Values <5000 are clamped to 5000. |
| `PRERENDER_RETRIES` | `2` | Total attempts per route. Each failed attempt logs a `retry` line; only after all attempts fail does the route count as failed. |

Chromium uses ~200–300 MB per page; even 4 pages fit comfortably on a 7 GB GitHub Actions runner — concurrency tuning is about network contention to Supabase, not memory.

### SPA fallback: `public/_redirects`

Single line:

```
/*  /index.html  200
```

Vite copies `public/` into `dist/` automatically. Cloudflare Pages reads `dist/_redirects` for any route the prerenderer didn't emit. Real `dist/{route}/index.html` files take precedence; the SPA shell is only served for genuinely unknown routes.

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
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Build with prerender
        env:
          VITE_SUPABASE_URL: ${{ secrets.VITE_SUPABASE_URL }}
          VITE_SUPABASE_PUBLISHABLE_KEY: ${{ secrets.VITE_SUPABASE_PUBLISHABLE_KEY }}
          PRERENDER: '1'
          PRERENDER_CONCURRENCY: '1'        # GH Actions runners flake at 2+; raise only on self-hosted/beefy runners
          PRERENDER_READY_TIMEOUT_MS: '45000'  # raise above default 20000 for heavy homepages
        run: npm run build
      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy ./dist --project-name=PROJECT_NAME_HERE --branch=main  # TODO(human/cursor): replace with actual Cloudflare Pages project name
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
```

`timeout-minutes: 15` gives ~3× the expected ~5-minute run time. Anything beyond is a Puppeteer hang and should fail fast.

`workflow_dispatch` enables manual re-runs from the GitHub UI without a code push. Use this when content changes (e.g. new rows in a Supabase table that drives prerendered pages) require a fresh build without code changes.

### `package.json` scripts

```json
"build": "vite build && node prerender.mjs",
"build:client": "vite build"
```

`PRERENDER=1` lives in the workflow env block, not the script. Local `npm run build` exits early at the top of `prerender.mjs`. Local `PRERENDER=1 npm run build` runs the full pipeline for verification.

**Required devDependencies:** `puppeteer`, `serve-handler`, `tsx`.

---

## Files touched

| File | Change |
|---|---|
| `.github/workflows/deploy.yml` (new) | GitHub Actions: install, build with PRERENDER=1, wrangler pages deploy. `timeout-minutes: 15`. |
| `public/_redirects` (new) | `/*  /index.html  200` |
| `src/routes.ts` (new) | Static routes + async `getStaticPaths` for dynamic routes |
| `src/App.tsx` | Map `<Routes>` from `src/routes.ts` |
| `src/main.tsx` | Install `window.__trackPending` + pending-Set + `__PRERENDER_READY__` signal after mount; wrap app in `<HelmetProvider>` |
| `prerender.mjs` (new, root) | Puppeteer crawler — see "Prerender script" above |
| `src/pages/*.tsx` | Wrap Supabase calls in `__trackPending`; use `<Helmet>` for per-route title/description/canonical/OG |
| `package.json` | Add `puppeteer`, `serve-handler`, `tsx` to devDependencies; `react-helmet-async` to dependencies; update `build` and `build:client` scripts |
| `.gitignore` | Verify `dist/` and `node_modules/` are listed; add `HANDOVER.md` |

**Explicitly not created:** `netlify.toml`, `_headers`, `.nvmrc`, `cloudflare.toml`, any edge function, any SSR server.

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
| `CLOUDFLARE_API_TOKEN` | Cloudflare dashboard → My Profile → API Tokens → "Edit Cloudflare Workers" template (covers Pages). One-time per Cloudflare account. |
| `CLOUDFLARE_ACCOUNT_ID` | Visible in any Cloudflare zone's dashboard sidebar. |
| `VITE_SUPABASE_URL` | Same value as in `.env`. |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Same value as in `.env`. **Use this exact env var name. Do NOT use `VITE_SUPABASE_ANON_KEY`.** |

---

## Security posture

- Build-time DB access uses the **publishable key** (the anon key already shipped to clients). No service-role key at build time, ever.
- Tables read at build time must have public-select RLS enabled. Verify this for any new dynamic-route table.
- Sanity assertion in `prerender.mjs`: abort with non-zero exit if any `getStaticPaths` returns zero rows. Catches misconfigured env vars before they ship a build with missing routes.
- Post-emit secret-leak grep over everything in `dist/`. Patterns: `service_role`, `\beyJ[A-Za-z0-9_-]{20,}` (long JWT), `postgres://`. Build fails on match.

---

## What is and isn't in scope

**In scope:**
- Build-time prerendering of all routes declared in `src/routes.ts`.
- Per-route meta tags via Helmet.
- Static deployment to Cloudflare Pages via GitHub Actions.
- SPA fallback for unknown routes.
- Manual rebuild trigger via `workflow_dispatch`.

**Out of scope:**
- User-agent-based cloaking or differential rendering.
- Edge SSR or any runtime server-rendering.
- Service-role key access at build time.
- `netlify.toml`, `_headers`, `.nvmrc`, `cloudflare.toml`.
- DNS changes performed by Lovable. (Cursor handles DNS automation via Cloudflare MCP when the zone is on Cloudflare. Otherwise the human adds DNS records manually at their registrar.)
- PR/branch preview deployments. (`wrangler pages deploy --branch=<name>` can be added later as a separate workflow.)
- Supabase-trigger → `workflow_dispatch` webhook for auto-rebuild on content edits. (Manual re-run via GitHub UI for now.)
- Caching of `~/.cache/puppeteer` across runs. (Chromium re-downloads ~170 MB per fresh runner; not worth complexity at this scale.)

---

## Verification checklist

After implementation, the following must be true:

1. Locally: `PRERENDER=1 npm run build` produces `dist/index.html`, `dist/<route>/index.html` for every static route, and `dist/<dynamic-path>/index.html` for every path resolved from `getStaticPaths`.
2. View source on a sample of those files → unique `<title>`, unique `<h1>`, real content visible without JavaScript, internal `<a href>` links present.
3. Sanity assertion fires when Supabase env vars are blanked. (Verify by temporarily clearing `VITE_SUPABASE_URL` and re-running; build must abort.)
4. Concurrency tunable: `PRERENDER_CONCURRENCY=8 PRERENDER=1 npm run build` succeeds. Default stays at 4 for production.
5. Secret-leak grep passes on the build log.
6. After first GitHub Actions deploy: `curl -s https://<project>.pages.dev/<some-route>` returns HTML with the route's actual content in the response body — not the SPA shell.
7. Client-side navigation between routes still feels instant (no full page reloads). React hydration succeeds without console errors.
8. After DNS cutover: GSC "Inspect URL" on 5–10 sample URLs → "Request indexing" returns success.

---

## Sequencing for implementation

When Lovable (or any agent) implements this spec from scratch:

1. **Migrations first**, if the project uses Supabase tables for dynamic routes. The tables and RLS policies must exist before components query them.
2. **Routing plumbing**: `src/routes.ts`, `src/main.tsx` signal + tracker, `src/App.tsx` route mapping, `<HelmetProvider>` wrapping.
3. **Page edits**: wrap Supabase calls in `__trackPending`, add Helmet to each page.
4. **Prerender script**: `prerender.mjs` at project root with all behaviors listed under "Prerender script."
5. **Build plumbing**: `package.json` scripts and devDependencies, `public/_redirects`.
6. **Workflow file**: `.github/workflows/deploy.yml` with `PROJECT_NAME_HERE` placeholder and `TODO(human/cursor)` comment.
7. **Local verification**: `PRERENDER=1 npm run build`, inspect `dist/`, confirm each item in the verification checklist.
8. **Handover**: leave a clear note that the workflow file's `PROJECT_NAME_HERE` placeholder needs filling in, secrets need setting, and the Cloudflare Pages project needs creating — all of which happen in the Cursor phase, not the Lovable phase.

---

## Operational state: `HANDOVER.md`

Each project has a `HANDOVER.md` at its root, **gitignored** (never committed). This is Cursor's working memory across sessions. The template lives in the templates repo (`~/dev/templates/HANDOVER_TEMPLATE.md`); Cursor copies it into a new Lovable repo during initial setup, fills it in, and adds `HANDOVER.md` to `.gitignore`.

Why gitignored: Lovable can rewrite committed files on prompt-edits, and the contents (timestamps, current state, known divergences) drift from reality if other people work on the repo or if Cursor regenerates state from live data. State files don't belong in version control. If the file is lost (new machine, accidental delete), Cursor reconstructs it in seconds by reading current Cloudflare project state and the latest GitHub Actions run.

Lovable does not read `HANDOVER.md`. Only Cursor does.

---

## Drift recovery

Lovable will sometimes rewrite `package.json`, `src/main.tsx`, `prerender.mjs`, or other files this spec describes. When the GitHub Actions build fails because of this:

1. Cursor reads the failed workflow run log via GitHub MCP.
2. Cursor diffs the current state of relevant files against this spec.
3. Cursor proposes a fix and waits for human approval before writing.
4. On approval, Cursor writes the fix via GitHub MCP and retriggers `workflow_dispatch`.

Cursor's runbook contains the exact prompt for this. Drift recovery is the recurring operational mode of the system, not a rare event.

---

## Hard rules

- Do not add SSR, edge functions, or any runtime server-rendering.
- Do not use the Supabase service-role key at build time.
- Do not duplicate the route list. `src/routes.ts` is the only place routes are declared.
- Do not skip the `__PRERENDER_READY__` signal in favor of `setTimeout` or `networkidle`.
- Do not commit `HANDOVER.md`. It is gitignored, always.
- Do not create `netlify.toml`, `_headers`, `.nvmrc`, or `cloudflare.toml`. None apply here.
- Do not modify `vite.config.ts` beyond what's strictly necessary for prerender support.
- Do not bake secrets into any committed file.

If Lovable encounters an instruction during a prompt-edit that conflicts with anything in this spec, it must surface the conflict rather than silently choose one.
