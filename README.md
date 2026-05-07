# dev-templates — Lovable SEO pack

Private templates for **[@fraserbrownirl](https://github.com/fraserbrownirl)**: prerendered Lovable apps → GitHub Actions → Cloudflare Pages.

| File | Role |
| --- | --- |
| `SEO_SETUP.md` | Drop at **root of each Lovable repo**. Implementation contract for Lovable + Cursor. |
| `SETUP_PROMPT.md` | Pasteable prompts (init / drift / rebuild). Stays here; copy-paste into Cursor agent mode. |
| `divergences.mdc` | Copy to each Lovable repo as `.cursor/rules/divergences.mdc` (committed). Project-wide intentional deviations from `SEO_SETUP.md`. Single-file deviations go inline as `// DIVERGENCE: ...` comments. |

**New project:** put `SEO_SETUP.md` in the Lovable repo → Lovable phase → open repo in Cursor → paste the INIT block from `SETUP_PROMPT.md`.

**MCP setup** lives in `~/.cursor/mcp.json` — that file is its own documentation. Add a comment block at the top with PAT/OAuth notes if useful.
