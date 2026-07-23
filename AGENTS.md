# Workspace: workspace-one

## Projects in this workspace
- `platform-docs/` — shared documentation (this project)
- `cozycorner/` — public catalog site for cozy home goods (Next.js 16 App Router + Supabase)
- `avocado.kiss/` — culinary recipe magazine (Next.js 16 App Router + Supabase)
- `web.admin/` — multi-site admin panel for managing site content (Vite + React SPA + Supabase)

## Documentation index
Read these files only when the task requires it:

| Topic | File | Read when |
|---|---|---|
| Database schema | platform-docs/database/schema.md | Any DB query, migration, or Supabase task |
| Admin panel API | platform-docs/admin-panel/api.md | Building features that call admin endpoints |
| Admin components | platform-docs/admin-panel/components.md | Reusing or extending admin UI components |
| Admin Media section | platform-docs/admin-panel/media.md | Editing Media section or folders (any section) |
| Admin Products section | platform-docs/admin-panel/products.md | Editing Products; before any new CRUD section |
| Admin Blog/SEO Posts | platform-docs/admin-panel/blog.md | Editing Blog or SEO Posts sections |
| Admin Pages section | platform-docs/admin-panel/pages.md | Editing Pages section |
| Admin status & roadmap | platform-docs/admin-panel/status.md | Planning admin work; checking pending tasks |
| Sites overview | platform-docs/sites/overview.md | Starting work on any public-facing site |
| CozyCorner architecture | platform-docs/sites/cozycorner.md | Any work in the cozycorner repo |
| Avocado Kiss architecture | platform-docs/sites/avocado-kiss.md | Any work in the avocado.kiss repo |
| Coding standards | platform-docs/methodology/coding-standards.md | New files, refactoring, code review |
| Testing strategy | platform-docs/methodology/testing.md | Writing/running tests; choosing a test tool; CI |

## Workflow rules
1. Before starting a task — check if a relevant doc file exists in the index above
2. After completing a task that changes architecture, API, or DB — update the relevant doc file
3. Never write feature details into AGENTS.md — put them in the appropriate doc file above
4. When creating a new site project — add it to "Projects" section, create sites/[site-name].md, AND wire testing: register it in the root `../AGENTS.md` project registry and follow methodology/testing.md → «Добавление нового проекта» (so any AI agent auto-picks it up for test routing)

## Context discipline
- AGENTS.md = index only (you are reading it now)
- Doc files = loaded on demand, only when needed
- Never summarize doc file contents back into AGENTS.md
