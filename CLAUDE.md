# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: MANDATORY for any `.vue` file creation or significant modification
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase structure questions or multi-file pattern searches
- **general-purpose**: Use for complex multi-step tasks

### Skills
- **backend-api-test**: Use when writing or modifying tests in `tests/backend/`

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only — use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

## Commands

```bash
# Start backend
cd server && uv run python main.py

# Start frontend
cd client && npm install && npm run dev

# Run all backend tests
cd tests && uv run pytest -v

# Run a single test file
cd tests && uv run pytest backend/test_inventory.py -v

# Run a single test
cd tests && uv run pytest backend/test_inventory.py::TestInventoryEndpoints::test_get_all_inventory -v

# Run with coverage
cd tests && uv run pytest --cov=../server --cov-report=html

# Build frontend for production
cd client && npm run build
```

No linting tools are configured (no ESLint, no Ruff/Black).

## Architecture

**Stack**: Vue 3 + Composition API + Vite (port 3000) → FastAPI + Uvicorn (port 8001) → JSON files in `server/data/`

**Frontend structure:**
- `client/src/views/` — page-level components (Dashboard, Inventory, Orders, Spending, Demand, Reports)
- `client/src/components/` — reusable UI (FilterBar, modals, ProfileMenu, LanguageSwitcher)
- `client/src/composables/` — shared reactive state: `useFilters.js`, `useAuth.js`, `useI18n.js`
- `client/src/api.js` — Axios client; strips `'all'` values before building query params
- `client/src/locales/` — English and Japanese translation files

**Backend structure:**
- `server/main.py` — all FastAPI routes and Pydantic models
- `server/mock_data.py` — loads JSON files at startup into module-level variables; restart to reload data
- `server/data/*.json` — source of truth for all mock data

**Test structure:**
- `tests/backend/conftest.py` — `client` fixture wrapping FastAPI `TestClient`
- `tests/backend/test_inventory.py`, `test_dashboard.py`, `test_misc_endpoints.py` — 55 tests total

## Key Patterns

**Filter system**: `useFilters.js` holds 4 reactive refs (`selectedPeriod`, `selectedLocation`, `selectedCategory`, `selectedStatus`) as a singleton composable shared across all views. Each view calls `getCurrentFilters()` to build the query object passed to `api.js`.

**Filter propagation**: Vue filter change → `useFilters` ref update → view `watch` triggers API refetch → `api.js` builds `URLSearchParams` (omitting `'all'` values) → backend `apply_filters()` / `filter_by_month()` → filtered JSON response → `computed` properties re-derive display data.

**Quarter filtering**: The backend `QUARTER_MAP` maps `Q1-2025` etc. to a list of months. Pass `month=Q2-2025` and the backend expands it — no frontend logic needed.

**Reactivity convention**: Raw API data lives in `ref()` (e.g., `allOrders`, `inventoryItems`). Everything derived from it (totals, filtered subsets, chart data) lives in `computed()`. Never mutate props; never store derived values in refs.

**i18n**: `useI18n.js` is a singleton composable. Use `t('key')` for all UI strings. Currency auto-selects: `USD` for `en`, `JPY` for `ja`. Locale persists in `localStorage` under `app-locale`.

**Adding a new endpoint**: (1) Add data to a JSON file in `server/data/`, (2) load it in `mock_data.py`, (3) add a Pydantic model in `main.py`, (4) add the route with `apply_filters()`, (5) add the API method to `client/src/api.js`.

## Common Issues

1. `v-for` keys must be unique domain values (`sku`, `order_number`, `month`) — never `index`
2. Validate dates before calling `.getMonth()` — orders without `actual_delivery` return `null`
3. Changing JSON field names requires updating the Pydantic model in `main.py` to match
4. Inventory endpoints do not support `month` filtering (inventory has no time dimension)
5. Revenue targets: $800K/month for a single month, $9.6M YTD when all months selected
6. `mock_data.py` loads at import time — code changes to data loading require a server restart

## Design System

- Colors: slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`)
- Status colors: green (success), blue (info), yellow (warning), red (danger)
- Charts: custom SVG — no charting library
- Layout: CSS Grid for page structure, Flexbox for component internals
- No emojis anywhere in the UI
- Styles: scoped per component; global resets and variables in `client/src/App.vue`
