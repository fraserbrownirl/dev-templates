# dev-templates — Lovable SEO pack

Private templates for **[@fraserbrownirl](https://github.com/fraserbrownirl)**: prerendered Lovable apps → GitHub Actions → Cloudflare Pages.

| File | Role |
| --- | --- |
| `SEO_SETUP.md` | Drop at **root of each Lovable repo**; Lovable + Cursor treat it as the implementation contract. |
| `CURSOR_RUNBOOK.md` | Cursor-only: MCP setup, pasteable prompts (DRAFT until validated in Phase B/C), failure modes. |
| `HANDOVER_TEMPLATE.md` | Copy to a project as gitignored `HANDOVER.md` after Cursor initial setup. |

**New project:** put `SEO_SETUP.md` in the Lovable repo → Lovable phase → open repo in Cursor → paste the initial setup prompt from `CURSOR_RUNBOOK.md`. Operational meta and phases: `mds/LOVABLE_SEO_TOOL_SPEC.md` in the companion **lovable-seo** repo (same machine / monorepo layout as this pack).
