# Cursor runbook — Lovable SEO pipeline

Operational guide for Cursor + **GitHub MCP** + **Cloudflare MCP**. The **per-project technical contract** is always `SEO_SETUP.md` at the Lovable repo root (same content as `templates/SEO_SETUP.md` here). This runbook never redefines that contract; it tells you how to apply and maintain it.

**Prompt status convention:** Each pasteable prompt starts with `> Status: DRAFT` or `> Status: VALIDATED (YYYY-MM-DD)`. Treat **DRAFT** prompts as best-effort until a real Phase B/C run promotes them.

---

## 1. One-time MCP setup

Do this once per Cursor install (global config: `~/.cursor/mcp.json` unless you prefer project-local `.cursor/mcp.json`).

### GitHub MCP — self-hosted via Docker (recommended)

- **Docs:** [github/github-mcp-server](https://github.com/github/github-mcp-server) → [install-cursor.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-cursor.md).
- **Why local:** the hosted endpoint `https://api.githubcopilot.com/mcp/` is gated behind GitHub Copilot. Self-hosting the official image needs only a **PAT** plus **Docker Desktop**, with no Copilot subscription.
- **Prereq:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) running on macOS (one-time install, free for personal use).
- **`mcp.json` fragment** (merge under existing `"mcpServers"`):

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "ghcr.io/github/github-mcp-server"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "REPLACE_WITH_GITHUB_PAT" }
    }
  }
}
```

- **PAT scopes (minimum practical set for this workflow):**
  - **Classic PAT:** `repo` (full), `workflow`. That covers reading/writing repo files, creating repos, reading/writing **Actions** + secrets, triggering `workflow_dispatch`, reading run logs.
  - **Fine-grained PAT (preferred):** All repositories + Repository permissions: **Metadata: Read** (auto), **Administration: Read/Write** (create repos), **Contents: Read/Write** (file CRUD), **Workflows: Read/Write** (modify `.github/workflows/*.yml`), **Actions: Read/Write** (run + log + dispatch), **Secrets: Read/Write** (Actions secrets).
- **First run:** restart Cursor; the GitHub server entry will pull the image once. Verify by asking Cursor to list your repos — you should see real data and `github-*` (or `user-github-*`) tools available to the agent.

**Do not** use deprecated `@modelcontextprotocol/server-github`; use `ghcr.io/github/github-mcp-server` per GitHub's README.

### GitHub MCP — hosted alternative (requires Copilot)

If you already have GitHub Copilot:

```json
"github": {
  "url": "https://api.githubcopilot.com/mcp/",
  "headers": { "Authorization": "Bearer YOUR_GITHUB_PAT" }
}
```

If hosted returns 401 / "Copilot subscription required," switch back to the Docker block above.

### Cloudflare MCP (official)

- **URL:** `https://mcp.cloudflare.com/mcp`
- **Auth:** OAuth via Cursor's MCP UI (complete browser sign-in when prompted).
- **Docs:** [cloudflare/mcp](https://github.com/cloudflare/mcp)

**Example `mcp.json` fragment** (merge with the GitHub block above under one `"mcpServers"` object — do not overwrite the whole file):

```json
{
  "mcpServers": {
    "cloudflare": {
      "url": "https://mcp.cloudflare.com/mcp"
    }
  }
}
```

### Verify both servers

Fully quit and reopen Cursor (Cmd+Q on macOS). In a **new** agent chat:

1. "List my GitHub repositories" (or open a known private repo via MCP file tools).
2. "List my Cloudflare accounts" / "List Cloudflare zones."

You should see **real data**, not auth errors. Existing chats started before MCP changes will not see new tools — always start a fresh chat.

---

## 2. Per-project initial setup prompt

> Status: DRAFT

**When to use:** Immediately after Lovable has implemented `SEO_SETUP.md`, committed to `main`, and you have verified `PRERENDER=1 npm run build` locally in Lovable (per contract).

**Fill in** the bracketed placeholders, then paste the whole block into Cursor agent mode with the **Lovable project repo** open as the workspace root.

```text
You are running the Lovable SEO "Phase 2 (Cursor)" initial setup. The per-project contract is SEO_SETUP.md at the repo root — read it first and treat it as authoritative.

Inputs (fill by human):
- GitHub repo: [OWNER/REPO or full https URL]
- Cloudflare Pages project name (new, unused): [NAME]
- Custom domain for production: [example.com or www.example.com]
- Is the DNS zone for that domain on this Cloudflare account? [yes / no / let me check]
- Supabase project URL: [https://xxx.supabase.co]
- Supabase publishable (anon) key for Vite: [key — use same value as VITE_SUPABASE_PUBLISHABLE_KEY in GitHub secrets]
- Path to HANDOVER_TEMPLATE.md on this machine: [default ~/dev/templates/HANDOVER_TEMPLATE.md]

Hard rules:
- Never print secret values in chat or commit them to files. GitHub Actions secrets only.
- After each mutating Cloudflare MCP step, read back via API to confirm state (Direct Upload Pages project, domain attachment, etc.).
- Copy HANDOVER_TEMPLATE.md → repo root HANDOVER.md, fill placeholders, add HANDOVER.md to .gitignore if missing. Do not commit HANDOVER.md.

Procedure:
1. Confirm Lovable phase completeness — these must exist: .github/workflows/deploy.yml (contains literal PROJECT_NAME_HERE), prerender.mjs, src/routes.ts, public/_redirects. If anything is missing, STOP and list gaps vs SEO_SETUP.md.
2. Cloudflare: list Pages projects; confirm [NAME] is not already taken. Create Pages project [NAME] in Direct Upload mode. GET/read back project and confirm configuration.
3. GitHub: set repository Actions secrets (names must match SEO_SETUP.md exactly):
   - CLOUDFLARE_API_TOKEN
   - CLOUDFLARE_ACCOUNT_ID
   - VITE_SUPABASE_URL
   - VITE_SUPABASE_PUBLISHABLE_KEY
   Then list secret *names* only and confirm all four exist.
4. GitHub: read .github/workflows/deploy.yml, replace PROJECT_NAME_HERE with [NAME], commit/push or apply via MCP per your permissions — follow least surprise; show diff before push.
5. Cloudflare: attach custom domain [domain] to the Pages project.
6. DNS — branch on human answer or MCP zone lookup:
   - If zone is on this Cloudflare account: create the correct CNAME (or apex) to [NAME].pages.dev per Cloudflare Pages docs; then verify with dig/host DNS lookup.
   - If zone is NOT on Cloudflare: print exact records (type, name, target, TTL) for the user's registrar, STOP and ask them to add records and reply "done", then verify DNS. Do not spin forever — after ~2 minutes if still failing, report lookup output and ask for guidance.
7. GitHub: trigger workflow_dispatch on "Build and deploy to Cloudflare Pages" (or whatever the workflow is named). Poll until terminal state; paste concise log excerpts on failure.
8. On success: curl https://[NAME].pages.dev/ (and one deep route) and confirm HTML contains real H1/title content, not an empty SPA shell.
9. Write HANDOVER.md from template with filled fields; set DNS provider line to cloudflare or registrar as appropriate.

Report a short checklist of completed steps and anything the human must still do manually.
```

---

## 3. Drift recovery prompt

> Status: DRAFT

```text
GitHub Actions failed after a Lovable edit on this repo. Use GitHub MCP to fetch the latest failed workflow run log for the deploy workflow. Identify the failing step and file.

Read SEO_SETUP.md at repo root and HANDOVER.md if present. Diff the broken files against the contract; propose a concrete patch (diff or file-level summary). Do NOT apply changes until I reply "approved". After approval, apply the fix (edit locally or via GitHub MCP as appropriate), trigger workflow_dispatch, and watch the run.

Update HANDOVER.md: Last drift event (now), Known divergences if we intentionally deviated from SEO_SETUP.md.
```

---

## 4. Rebuild trigger prompt

> Status: DRAFT

```text
Trigger a fresh prerender build of this project: use GitHub MCP to run workflow_dispatch on the Cloudflare deploy workflow, then stream or poll logs until done. On success, update HANDOVER.md "Last successful deploy" to now (ISO timestamp).
```

---

## 5. Log diagnose prompt

> Status: DRAFT

```text
Using GitHub MCP, pull logs for workflow run [RUN_ID or "latest"] on this repo's deploy workflow. Summarize: outcome, slow steps, retries, Puppeteer/prerender errors, wrangler errors. Call out anything that should be fixed in the repo vs transient infrastructure.
```

---

## 6. Custom domain add/change prompt

> Status: DRAFT

```text
We need to [add | remove | swap] a custom domain for this Lovable project's Cloudflare Pages deployment.

Read HANDOVER.md and SEO_SETUP.md. Use Cloudflare MCP to adjust Pages custom domains on the project named [HANDOVER Cloudflare project name]. Apply DNS automation policy: if zone is on Cloudflare, create/update/delete records via MCP and verify with dig; otherwise emit exact manual DNS instructions and wait for my "done" before verifying.

Update HANDOVER.md custom domain + DNS provider fields when finished.
```

---

## 7. Failure modes reference

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| Cloudflare MCP **403** | OAuth/session scope or token cannot access resource | Re-authenticate MCP; confirm account has zone/Pages rights; retry with minimal repro API call. |
| GitHub Action "secret not found" / build missing env | Typo in secret **name** vs `SEO_SETUP.md` | Fix secret names exactly: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`. |
| Puppeteer **hangs** until timeout | `__PRERENDER_READY__` never true, or `__trackPending` leak | Inspect `src/main.tsx` and page loaders vs contract; ensure promises settle and flag flips on route change. |
| Build **green** but Google still shows thin pages | Prerendered HTML missing or wrong route | Confirm `dist/.../index.html` exists for the URL; curl live URL; Search Console URL Inspection + request indexing. |
| Custom domain **Pending** forever | DNS not pointed at Pages | `dig` CNAME/A; compare to Cloudflare Pages required targets; fix registrar or Cloudflare DNS. |
| Assumed zone on Cloudflare but lookup says **no** | Wrong account or domain under registrar DNS | Fall back to manual DNS instructions; update `HANDOVER.md` DNS provider field. |

---

## 8. `HANDOVER.md` usage convention

- **Session start:** If the repo is a known Lovable SEO project, read `HANDOVER.md` first (it is gitignored; create from `HANDOVER_TEMPLATE.md` if missing).
- **After operations:** Update timestamps and domain fields whenever you deploy, recover drift, or change DNS/domains.
- **Lost file:** Reconstruct by reading Cloudflare Pages project name from workflow + MCP, latest successful GitHub Actions run time, and current custom domains — takes seconds, then rewrite `HANDOVER.md` locally.

---

## Reference — required GitHub Actions secrets

| Secret name |
| --- |
| `CLOUDFLARE_API_TOKEN` |
| `CLOUDFLARE_ACCOUNT_ID` |
| `VITE_SUPABASE_URL` |
| `VITE_SUPABASE_PUBLISHABLE_KEY` |

See `SEO_SETUP.md` for how each is obtained and for security rules (no service role at build time).
