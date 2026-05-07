# Pasteable prompts

Copy a block, fill bracketed placeholders, paste into Cursor agent mode with the Lovable repo open.

---

## INIT — per-project initial setup

```text
Run Lovable SEO Phase 2 (Cursor) initial setup. SEO_SETUP.md at repo root is authoritative — read it first.

Inputs:
- GitHub repo: [OWNER/REPO]
- Cloudflare Pages project name: [NAME]
- Custom domain: [example.com]
- Zone on this Cloudflare account: [yes / no / check]
- Supabase URL: [https://xxx.supabase.co]
- Supabase publishable key: [key]

Hard rules:
- Never print secret values. GitHub Actions secrets only.
- After each mutating Cloudflare op, read back to confirm state.
- Read .cursor/rules/divergences.mdc if present — do not "fix" anything listed there back to SEO_SETUP.md.

Procedure:
1. Verify Lovable phase: .github/workflows/deploy.yml (Node 22, npx wrangler@4, with literal PROJECT_NAME_HERE), prerender.mjs (with sitemap+robots emission), og-render.mjs, og-template.html, functions/_middleware.ts, src/routes.ts, public/_redirects all exist. Stop and list gaps if not.
2. Cloudflare MCP: confirm [NAME] free; create Pages project [NAME] in Direct Upload mode; read back.
3. GitHub MCP: set Actions secrets — CLOUDFLARE_API_TOKEN, CLOUDFLARE_ACCOUNT_ID, VITE_SUPABASE_URL, VITE_SUPABASE_PUBLISHABLE_KEY, VITE_SITE_URL (canonical https URL, no trailing slash). List secret names (not values) to confirm all five. Use `gh secret set NAME --body "value"` — never `--body -` (treats `-` as literal).
4. GitHub MCP: replace PROJECT_NAME_HERE in deploy.yml with [NAME]. Show diff before commit.
5. Cloudflare MCP: attach [domain] to Pages project.
6. DNS:
   - Zone on Cloudflare → create CNAME (or apex) to [NAME].pages.dev; verify with dig.
   - Zone elsewhere → print exact records (type/name/target/TTL); STOP, await "done"; verify with dig (~2 min cap, then ask).
7. GitHub MCP: workflow_dispatch on the deploy workflow; poll to terminal state; paste log excerpts on failure.
8. Verify:
   - `curl -s https://[NAME].pages.dev/` returns real H1/title in HTML, not SPA shell.
   - `curl -sI https://[NAME].pages.dev/` returns 200, content-type text/html.
   - `curl -sI https://<branch>.[NAME].pages.dev/` includes `X-Robots-Tag: noindex, nofollow, noarchive` (middleware noindexes non-canonical hosts).
   - `curl -s https://[NAME].pages.dev/sitemap.xml | head` returns valid `<urlset>` XML.
   - `curl -sI https://[NAME].pages.dev/og/default.png` returns image/png.
9. Initialise .cursor/rules/divergences.mdc with "_None yet._" if not already present (template in dev-templates).
10. If the project's primary content table is empty at first deploy, leave `STRICT_PRERENDER` unset and log it as a TEMPORARY divergence (D-something) with an explicit removal trigger (e.g. `>= 10 rows for >= 7 days`). Do not gate it forever.

Report completed steps and any remaining manual items.
```

---

## DRIFT — recovery after a failed build

```text
GitHub Actions deploy failed after a Lovable edit. Pull the latest failed run log via GitHub MCP. Identify failing step and file.

Read SEO_SETUP.md and .cursor/rules/divergences.mdc. Diff the broken files against the contract (respecting listed divergences and any inline // DIVERGENCE: comments). Propose a patch. Do NOT apply until I reply "approved".

On approval: apply via GitHub MCP, trigger workflow_dispatch, watch the run.
```

---

## REBUILD — fresh prerender, no code change

```text
Trigger workflow_dispatch on the Cloudflare deploy workflow via GitHub MCP. Stream or poll until done.
```
