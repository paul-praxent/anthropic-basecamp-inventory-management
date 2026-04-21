---
description: Redesign Vue 3 app UI into a modern SaaS-style interface with a vertical navigation sidebar
---

Redesign this Vue 3 application's UI into a modern SaaS-style interface. Replace the horizontal top navigation bar with a fixed vertical sidebar on the left. The result should look polished, professional, and consistent — like Linear, Notion, or Vercel's dashboard.

## Step 1 — Audit the current layout

Read `client/src/App.vue` to understand:
- What navigation links exist and their routes
- What components are in the top nav (ProfileMenu, LanguageSwitcher, FilterBar)
- What global CSS classes are defined (`.top-nav`, `.nav-container`, `.main-content`, etc.)

Also read `client/src/components/FilterBar.vue` and `client/src/components/ProfileMenu.vue` to understand their structure and dimensions.

## Step 2 — Redesign App.vue using the vue-expert agent

Spawn the **vue-expert** agent to rewrite `client/src/App.vue`. The agent must:

**Structural changes:**
- Change `.app` from `flex-direction: column` to `flex-direction: row`
- Remove the `<header class="top-nav">` and replace with `<aside class="sidebar">`
- Move `<FilterBar />` from below the header into the top of `.main-content` as a page-level sub-header strip
- Keep `<router-view />` inside `.main-content`
- Keep the modals (`ProfileDetailsModal`, `TasksModal`) at the bottom of the template — they are not layout

**Sidebar structure (top to bottom):**
1. `.sidebar-brand` — company name (`t('nav.companyName')`) as wordmark, subtitle below it
2. `.sidebar-nav` — vertical list of `<router-link>` elements, one per route, with a small SVG icon beside each label
3. `.sidebar-footer` — `LanguageSwitcher` and `ProfileMenu` stacked vertically

**SVG icons for nav items** — inline SVG, 16×16 viewBox, stroke-based (no fill), stroke-width 1.5. Use these paths:
- Overview/Dashboard: `M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6` (house icon, 24px viewBox)
- Inventory: `M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10M4 7v10l8 4` (cube icon, 24px viewBox)
- Orders: `M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2` (clipboard icon, 24px viewBox)
- Finance/Spending: `M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2m0-8c1.11 0 2.08.402 2.599 1M12 8V7m0 1v8m0 0v1m0-1c-1.11 0-2.08-.402-2.599-1M21 12a9 9 0 11-18 0 9 9 0 0118 0z` (currency icon, 24px viewBox)
- Demand: `M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z` (chart icon, 24px viewBox)
- Reports: `M9 17v-2m3 2v-4m3 4v-6m2 10H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z` (document chart icon, 24px viewBox)
- Backlog (if the route `/backlog` exists): `M4 6h16M4 10h16M4 14h10` (list icon, 24px viewBox)

**Sidebar CSS requirements:**
- Width: `240px`, fixed/sticky, full viewport height (`100vh`)
- Background: `#0f172a` (dark slate — contrast against white main content)
- No box-shadow; use a subtle `1px solid rgba(255,255,255,0.06)` right border
- `.sidebar-brand`: padding `1.5rem 1.25rem 1rem`, brand name in white (`#f8fafc`), subtitle in muted slate (`#64748b`), font-size 0.8rem
- `.sidebar-nav`: padding `0.5rem`, `display: flex; flex-direction: column; gap: 2px`
- Nav links: `display: flex; align-items: center; gap: 0.625rem; padding: 0.625rem 0.875rem; border-radius: 6px; color: #94a3b8; font-size: 0.875rem; font-weight: 500; text-decoration: none; transition: all 0.15s ease`
- Nav link hover: `color: #f1f5f9; background: rgba(255,255,255,0.06)`
- Nav link `.active` (use `exact-active-class` or manual `:class` binding): `color: #f8fafc; background: rgba(255,255,255,0.1); font-weight: 600`
- Nav link SVG: `flex-shrink: 0; opacity: 0.7` (1.0 on active/hover)
- `.sidebar-footer`: `margin-top: auto; padding: 1rem 1.25rem; border-top: 1px solid rgba(255,255,255,0.08); display: flex; flex-direction: column; gap: 0.5rem`

**Main content CSS requirements:**
- `.main-content`: `flex: 1; min-width: 0; display: flex; flex-direction: column; overflow: hidden`
- Remove the old `max-width: 1600px` and `margin: 0 auto` from `.main-content` — the sidebar handles horizontal space
- Add a `.content-body` wrapper inside `<main>` around `<router-view />` with `padding: 1.5rem 2rem; flex: 1; overflow-y: auto`

**FilterBar placement:**
- Keep `<FilterBar />` as a sticky sub-header strip inside `.main-content`, above `.content-body`
- Give it `position: sticky; top: 0; z-index: 50` so it sticks as the user scrolls the content area

**Global CSS cleanup in the same file:**
- Remove `.top-nav`, `.nav-container`, `.nav-tabs`, `.nav-tabs a`, and all their sub-rules
- Add the new `.sidebar`, `.sidebar-brand`, `.sidebar-nav`, `.sidebar-footer` rules
- Keep all other global rules (`.card`, `.stat-card`, `.badge`, table rules, etc.) unchanged

## Step 3 — Verify FilterBar works in its new context

Read `client/src/components/FilterBar.vue`. If its styles assume it sits below a `70px` top nav (e.g., `top: 70px`), update those to `top: 0`. Do not change FilterBar's internal layout or filter logic.

## Step 4 — Visual QA in the browser

Use Playwright MCP tools to verify the result:

1. Navigate to `http://localhost:3000`
2. Take a screenshot and confirm:
   - The sidebar is visible on the left with a dark background
   - All nav links are present vertically
   - The main content occupies the right portion of the screen
   - The FilterBar appears as a horizontal strip at the top of the content area
   - The active route link is highlighted in the sidebar
3. Click two different nav links and screenshot each to verify routing still works and active state updates
4. Check that no console errors are thrown

If the servers are not running, start them first: kill ports 3000 and 8001, then run `cd server && uv run python main.py` and `cd client && npm run dev` in the background.

## Step 5 — Report

Summarize:
- Which files were changed
- Before/after layout description
- Any edge cases handled (e.g., active route detection, FilterBar repositioning)
- What to manually test (responsive behavior, modal z-index, language switcher in footer)
