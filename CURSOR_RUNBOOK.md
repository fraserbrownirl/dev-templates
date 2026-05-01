# Cursor runbook — Lovable SEO pipeline

Operational guide for Cursor + **GitHub MCP** + **Cloudflare MCP**. The **per-project technical contract** is always `SEO_SETUP.md` at the Lovable repo root (same content as `templates/SEO_SETUP.md` here). This runbook never redefines that contract; it tells you how to apply and maintain it.

**Prompt status convention:** Each pasteable prompt starts with `> Status: DRAFT` or `> Status: VALIDATED (YYYY-MM-DD)`. Treat **DRAFT** prompts as best-effort until a real Phase B/C run promotes them.

---

## 1. One-time MCP setup

Do this once per Cursor install (global config: `~/.cursor/mcp.json` unless you prefer project-local `.cursor/mcp.json`).

> **Secret hygiene reminder.** Never put literal tokens in `mcp.json`. Use the wrapper-script + env-file + LaunchAgent pattern documented in **§9 MCP Secret Hygiene** below. The snippets in this section show literal placeholders only as a stepping stone — when you actually wire it up, follow §9.

### GitHub MCP — self-hosted via Docker (recommended)

- **Docs:** [github/github-mcp-server](https://github.com/github/github-mcp-server) → [install-cursor.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-cursor.md).
- **Why local:** the hosted endpoint `https://api.githubcopilot.com/mcp/` is gated behind GitHub Copilot. Self-hosting the official image needs only a **PAT** plus **Docker Desktop**, with no Copilot subscription.
- **Prereq:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) running on macOS (one-time install, free for personal use).
- **Recommended `mcp.json` fragment** (literal-secret-free, references env via wrapper — see §9):

```json
{
  "mcpServers": {
    "github": {
      "command": "/Users/YOU/.config/cursor-mcp-wrappers/github-mcp.sh"
    }
  }
}
```

- **Plain fragment** (works but stores secrets in `mcp.json`; prefer the wrapper above):

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

> Status: VALIDATED (2026-04-29) on `fraserbrownirl/startupemail` → `https://startupideas.email`. See §10 for cutover lessons learned (apex CNAME flatten, Cloudflare zone-create permission gap, GH Actions network slowness, etc.).

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

> Status: VALIDATED (2026-04-29) — used to triage 3 failed deploys before run #10 went green.

```text
Using GitHub MCP, pull logs for workflow run [RUN_ID or "latest"] on this repo's deploy workflow. Summarize: outcome, slow steps, retries, Puppeteer/prerender errors, wrangler errors. Call out anything that should be fixed in the repo vs transient infrastructure.
```

---

## 6. Custom domain add/change prompt

> Status: VALIDATED (2026-04-29) — used to attach `startupideas.email` + `www.startupideas.email` to Pages project, with apex CNAME-flattening through Cloudflare-managed DNS.

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
| Custom domain stuck in `validation status: error` ("Validation is in undefined status") | Domain was added before DNS resolved to Pages | DELETE the domain via MCP, POST it again. Validation re-runs cleanly once A/CNAME points at `<project>.pages.dev`. |
| Cloudflare **MCP can mint API tokens but cannot create zones** | OAuth scope omits `zone.create` | Have human add the zone via dashboard ("Add a site"). MCP can manage records, custom domains, and API tokens after that. |
| Cloudflare API call → "user email must been verified" | New Cloudflare account never confirmed via email | User clicks verify link in their email or hits "Resend verification" in dashboard. Until done, all account-mutating API calls 403. |
| GH Actions prerender flakes at default `PRERENDER_CONCURRENCY=4` even with retry-once | GH Actions runners' network to Supabase REST throttles under parallel load | Drop to `PRERENDER_CONCURRENCY=1` and `PRERENDER_READY_TIMEOUT_MS=45000` in workflow env. Local Macs handle 4; GH free runners often need 1. |
| Cloudflare Pages 308-redirects `/foo` → `/foo/` | Pages default normalization | Functional, but `<link rel="canonical">` in HTML uses no-trailing form so search bots see one redirect hop. Optional fix via `_redirects` rules. |

---

## 8. `HANDOVER.md` usage convention

- **Session start:** If the repo is a known Lovable SEO project, read `HANDOVER.md` first (it is gitignored; create from `HANDOVER_TEMPLATE.md` if missing).
- **After operations:** Update timestamps and domain fields whenever you deploy, recover drift, or change DNS/domains.
- **Lost file:** Reconstruct by reading Cloudflare Pages project name from workflow + MCP, latest successful GitHub Actions run time, and current custom domains — takes seconds, then rewrite `HANDOVER.md` locally.

---

## 9. MCP secret hygiene

The IDE keeps `~/.cursor/mcp.json` open between sessions. Cursor auto-attaches the contents of currently-open files to chat. Any literal token in `mcp.json` will eventually leak into a chat transcript. The fix is to keep `mcp.json` literal-secret-free and route every secret through one of three patterns described below.

### Layout

```text
~/.config/cursor-secrets.env                                # mode 600 — single source of truth
~/.config/cursor-mcp-wrappers/
  github-mcp.sh                                             # wraps dockerized GitHub MCP
  load-env-launchctl.sh                                     # pushes env into launchctl for GUI Cursor
  README.md                                                 # local copy of this section
~/Library/LaunchAgents/
  com.<user>.cursor-mcp-env.plist                           # runs load-env-launchctl.sh at login
~/.zshenv                                                   # sources cursor-secrets.env for terminal-launched processes
~/.cursor/mcp.json                                          # references wrappers + ${env:VAR}; zero literals
```

### How secrets reach each MCP server

| MCP server type | Example | Secret delivery |
|---|---|---|
| stdio (docker) | GitHub MCP | Wrapper script sources env file, exports the var, exec's `docker run -e` |
| stdio (npx) | n8n-mcp | Wrapper script sources env file, exports the var, exec's `npx` |
| URL with `headers` auth | Context7 | `mcp.json` uses `${env:VAR_NAME}` interpolation; LaunchAgent populates GUI Cursor's launchctl env so the interpolation resolves |
| URL with browser OAuth | Cloudflare, Supabase, Vercel, GitHub Copilot-hosted | No local secret; tokens stored by Cursor itself in macOS Keychain or its own cache |

### `~/.config/cursor-secrets.env` template

```bash
# Mode 600. Never commit. Never paste contents into chat.
# Reload after editing:
#   launchctl unload ~/Library/LaunchAgents/com.<user>.cursor-mcp-env.plist
#   launchctl load   ~/Library/LaunchAgents/com.<user>.cursor-mcp-env.plist

export GITHUB_PERSONAL_ACCESS_TOKEN="github_pat_..."
export CONTEXT7_API_KEY="ctx7sk-..."
# add per-server env exports here as you wire up new MCPs
```

### `~/.config/cursor-mcp-wrappers/github-mcp.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
source "${HOME}/.config/cursor-secrets.env"
case "${GITHUB_PERSONAL_ACCESS_TOKEN:-}" in
  ""|PASTE_HERE_*|REPLACE_AFTER_ROTATION)
    echo "github-mcp wrapper: GITHUB_PERSONAL_ACCESS_TOKEN unset or placeholder." >&2
    exit 1
    ;;
esac
exec docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

### `~/.config/cursor-mcp-wrappers/load-env-launchctl.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
ENV_FILE="${HOME}/.config/cursor-secrets.env"
[ -r "${ENV_FILE}" ] || exit 0
while IFS= read -r line; do
  case "${line}" in
    export\ *=*)
      kv="${line#export }"
      key="${kv%%=*}"
      val="${kv#*=}"
      val="${val%\"}"; val="${val#\"}"
      case "${val}" in ""|PASTE_HERE_*|REPLACE_AFTER_ROTATION) continue ;; esac
      launchctl setenv "${key}" "${val}"
      ;;
  esac
done < "${ENV_FILE}"
```

### `~/Library/LaunchAgents/com.<user>.cursor-mcp-env.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.<user>.cursor-mcp-env</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/<user>/.config/cursor-mcp-wrappers/load-env-launchctl.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

### `~/.zshenv`

```bash
if [ -r "${HOME}/.config/cursor-secrets.env" ]; then
  set -a
  source "${HOME}/.config/cursor-secrets.env"
  set +a
fi
```

### Final `~/.cursor/mcp.json`

```json
{
  "mcpServers": {
    "cloudflare": { "url": "https://mcp.cloudflare.com/mcp" },
    "github":     { "command": "/Users/<user>/.config/cursor-mcp-wrappers/github-mcp.sh" },
    "supabase":   { "url": "https://mcp.supabase.com/mcp", "headers": {} },
    "context7":   { "url": "https://mcp.context7.com/mcp", "headers": { "CONTEXT7_API_KEY": "${env:CONTEXT7_API_KEY}" } },
    "vercel":     { "url": "https://mcp.vercel.com", "headers": {} }
  }
}
```

### Rotation procedure

1. Generate new token at the upstream provider.
2. Edit `~/.config/cursor-secrets.env`, replace the value.
3. Either `launchctl unload && launchctl load` the plist, or Cmd+Q and reopen Cursor.
4. New chat → smoke-test the affected MCP.

### Verification

```bash
stat -f '%Sp %N' ~/.config/cursor-secrets.env                    # expect: -rw-------
grep -cE '(github_pat_|eyJ[A-Za-z0-9_-]{20,}|ctx7sk-|sb_secret_)' ~/.cursor/mcp.json   # expect: 0
launchctl getenv GITHUB_PERSONAL_ACCESS_TOKEN | head -c 11       # expect: github_pat_
```

---

## 10. Cutover lessons learned (2026-04-29)

Captured from the first end-to-end run on `fraserbrownirl/startupemail` → `https://startupideas.email`. Add to the §2 procedure above for the next project; Cursor should pre-empt these failures rather than rediscover them.

1. **Cloudflare account email verification is a silent gate.** Brand-new Cloudflare accounts can OAuth into the MCP just fine, but every account-mutating API call returns `8000077: Your user email must been verified`. Step 0 of any Cloudflare cutover: confirm the user has clicked the verification email.
2. **Cloudflare OAuth scope can't create zones.** The MCP's OAuth session can manage Pages projects, mint API tokens, and edit DNS records — but not `POST /zones`. Have the human add the zone via the "Add a site" wizard in the dashboard. After that, hand back to MCP.
3. **Local repo clone is a prereq, not optional.** `git rm --cached .env` and `PRERENDER=1 npm run build` both need a local clone. If the user doesn't have one, Cursor should clone it (use the wrapper-PAT pattern: `git clone https://${PAT}@github.com/owner/repo.git` then `git remote set-url origin https://github.com/owner/repo.git` to strip the token).
4. **GitHub Actions Ubuntu runners are slower than the local Mac for Puppeteer + Supabase.** Default `PRERENDER_CONCURRENCY=4` produced 9–75 timeout failures on the runner; `PRERENDER_CONCURRENCY=1` + `PRERENDER_READY_TIMEOUT_MS=45000` got a clean 399/399. Set both in the workflow env block from day one.
5. **Apex CNAME on Cloudflare-managed zones is automatic.** Once nameservers are switched to Cloudflare, both apex and `www` records can be CNAMEs to `<project>.pages.dev` (Cloudflare flattens at the apex). No ALIAS/ANAME workarounds needed. This makes Cloudflare-managed DNS strictly preferable to keeping the zone on the registrar.
6. **Mirror existing DNS records before the NS swap.** Cloudflare's "Add a site" wizard auto-imports A/MX/TXT records from the current authoritative resolver. Verify MX/SPF/DKIM/DMARC made it across before the NS cutover so email doesn't blink. (Worked correctly on this run; document so we don't forget to verify next time.)
7. **The Cloudflare MCP is a JS-execute interface, not a per-endpoint toolset.** It exposes only `search` (against the OpenAPI spec) and `execute` (run JS that calls `cloudflare.request(...)`). Useful pattern: `accountId` is auto-injected into the JS sandbox. Always call `search` first when you don't know the path/body shape for a given operation.
8. **Custom domain validation can race the DNS swap.** Adding a domain to Pages while DNS is still on the old host puts the domain into `status: deactivated, validation: error: "Validation is in undefined status"`. Recovery: `DELETE` the domain via MCP and `POST` it again. Cleaner: wait until Cloudflare zone status is `active` AND the apex/www records resolve to `<project>.pages.dev` before adding the custom domain at all.
9. **Cancel stale workflow runs before pushing fixes.** A failed in-progress run on an outdated commit will keep consuming a runner slot and obscure status views. Always `POST /actions/runs/<id>/cancel` after pushing a follow-up fix on `main`.
10. **GitHub Actions secrets need libsodium sealed-box encryption to set programmatically.** No GitHub MCP tool exposes Actions secrets. Use a tiny Python script with PyNaCl + the GitHub REST API: GET `/actions/secrets/public-key`, encrypt with `SealedBox`, PUT `/actions/secrets/{name}`. Document this in §2 — it's the one CI step where the GitHub MCP can't help.
11. **Cloudflare zone-level redirect rules need a separate API token.** The OAuth-scoped Cloudflare MCP can read zone rulesets but cannot write to `PUT /zones/{zid}/rulesets/phases/http_request_dynamic_redirect/entrypoint` — and the deploy-time `CLOUDFLARE_API_TOKEN` (Pages-Write only) also can't. To create Single Redirect rules (e.g. `www → apex`), mint a one-off Cloudflare API token with **`Zone — Zone WAF — Edit`** + **`Zone — Zone — Read`** scoped to the **specific zone** (not the whole account), paste into `~/.config/cursor-secrets.env` as `CLOUDFLARE_API_TOKEN_RULES`, run the API call, then revoke. Account-scoped tokens (`cfat_…` with "Entire Account" resource scope) cannot touch zone-level rulesets even with rule-edit permissions — the token MUST include zone resource scope. Easiest UI path: use the "Edit zone DNS" template as a starter (forces the zone-resource picker), then swap permissions to the WAF/Read combo above.

    Working API call shape:
    ```bash
    curl -X PUT \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      "https://api.cloudflare.com/client/v4/zones/$ZID/rulesets/phases/http_request_dynamic_redirect/entrypoint" \
      -d '{"rules":[{"action":"redirect","action_parameters":{"from_value":{"status_code":301,"target_url":{"expression":"concat(\"https://APEX\", http.request.uri.path)"},"preserve_query_string":true}},"expression":"(http.host eq \"www.APEX\")","description":"www to apex 301"}]}'
    ```

12. **Trailing-slash audit cleanup is a Lovable-side fix, not infra.** Cloudflare Pages serves trailing-slash URLs natively (`/foo/` = 200) and 308-redirects no-slash (`/foo` → `/foo/`). To kill audit flags, update `<link rel="canonical">` in every page Helmet AND `sitemap.xml` to use trailing-slash form (matching what Pages serves). Lovable did this in one prompt for `startupemail` (commit 56384f0). Result: bots crawling sitemap/canonicals get clean 200s; only random external no-slash backlinks see the 308 hop.

---

## Reference — required GitHub Actions secrets

| Secret name |
| --- |
| `CLOUDFLARE_API_TOKEN` |
| `CLOUDFLARE_ACCOUNT_ID` |
| `VITE_SUPABASE_URL` |
| `VITE_SUPABASE_PUBLISHABLE_KEY` |

See `SEO_SETUP.md` for how each is obtained and for security rules (no service role at build time).
