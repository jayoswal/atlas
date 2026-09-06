# Repo Spec — `hrms-web` (Web UI)

> **Owner:** Senior Frontend Engineer · **Stack:** React 18.3 + TypeScript 5.4 + Vite 5.4 · **Depends on:** API Gateway only. Read [`../ARCHITECTURE.md`](../ARCHITECTURE.md) (auth, error envelope, pagination, versioning) and [`../DEVELOPMENT.md`](../DEVELOPMENT.md) (git/PR/CI conventions) first — not repeated here.

- **Upstream (consumes contracts of):** `svc-identity`, `svc-time`, `svc-expense`, `svc-workflow` — **exclusively via the gateway** at `/api/v1`.
- **Downstream:** none (outermost client).
- **Contracts owned:** none. The UI is a **pure consumer** that generates a typed client from the OpenAPI specs in `platform-outerloop/contracts/openapi/`.

## 1. Responsibilities

- Render all employee/manager/finance workflows for Time & Expense as a **cohesive enterprise application**, not a set of demo pages.
- Own client-side session (token storage, refresh, route guards) and RBAC-driven navigation.
- Never contain business rules that belong to a service (cap math, accrual math). It may *display* server-computed flags and *pre-validate* for UX; the server is authoritative.

## 2. Tech stack (pinned)

| Concern | Choice | Version |
|---|---|---|
| Runtime | Node | 20 LTS |
| Framework | React | 18.3 |
| Language | TypeScript | 5.4 (strict, `noUncheckedIndexedAccess`) |
| Bundler/dev server | Vite | 5.4 |
| State + data fetching | Redux Toolkit + RTK Query | 2.x |
| UI kit | MUI (Material UI) with a **fully custom theme** | 5.16 |
| Data grid | MUI X Data Grid (Pro-parity features via community where possible) | 7.x |
| Forms | React Hook Form + Zod resolver | latest |
| Routing | React Router | 7.18.3 |
| Charts | Recharts (loaded from CDN in prod build) | 2.x |
| i18n / dates | `react-i18next` + `date-fns` + `Intl` | latest |
| API types | `openapi-typescript` → generated client | 7.13.0 |
| Testing | Vitest + React Testing Library; Playwright (E2E); axe-core (a11y) | latest |
| Lint/format | ESLint (typescript-eslint, jsx-a11y), Prettier | latest |

---

## 3. Enterprise design system (this is a product, not a demo)

The single most common "tell" of a learning-grade UI is **default Material components on white cards with the stock blue/purple palette**. `hrms-web` ships a bespoke design system implemented as an **MUI theme override** (`src/theme/`) so no screen looks like unstyled MUI. All values below are design **tokens** — components read tokens, never literals.

### 3.1 Design principles

1. **Density first.** HRMS users process many rows. Default to the *compact* density; generous whitespace is for marketing sites, not operational tools.
2. **State is visible.** Every record shows status as a **pill + color + label** (never color alone — a11y). Approvals, overtime flags, and policy violations are scannable at a glance.
3. **Structured, not decorative.** One elevation level for surfaces, one for overlays. Borders and dividers carry structure; drop shadows are reserved for overlays (menus, dialogs, drawers).
4. **Consistency over novelty.** Same table, same filter bar, same detail-drawer pattern across Time and Expense. Learn one screen, know them all.
5. **Accessible by construction.** WCAG 2.1 AA: 4.5:1 text contrast, visible focus rings, full keyboard operation, semantic landmarks.

### 3.2 Color tokens (`theme/palette.ts`)

Brand is a **restrained corporate indigo-slate**, deliberately not MUI's default blue/purple. Semantic colors are separate from brand.

| Token | Light | Dark | Use |
|---|---|---|---|
| `brand.primary` | `#2B4C7E` | `#6E97D0` | primary actions, active nav, links |
| `brand.primaryContrast` | `#FFFFFF` | `#0B1017` | text on primary |
| `surface.canvas` | `#F5F7FA` | `#0E141B` | app background |
| `surface.paper` | `#FFFFFF` | `#151D26` | cards, tables, panels |
| `surface.sunken` | `#EDF1F6` | `#1B2530` | table header, inset areas |
| `border.default` | `#DCE1E8` | `#26313D` | dividers, cell borders |
| `text.primary` | `#16202E` | `#E7ECF2` | body |
| `text.secondary` | `#5A6675` | `#9AA7B6` | labels, meta |
| `status.success` | `#1F7A4D` | `#4FB483` | approved, reimbursed |
| `status.warning` | `#B4692B` | `#D6924E` | pending, policy-flagged |
| `status.danger` | `#B3261E` | `#E5766E` | rejected, over-cap, errors |
| `status.info` | `#2B6C8F` | `#5FB0CF` | informational |

Contrast for every text/background pair is verified ≥ 4.5:1 (≥ 3:1 for large text and UI borders).

### 3.3 Typography (`theme/typography.ts`)

- **Type family:** IBM Plex Sans (UI) + IBM Plex Mono (IDs, amounts, codes). Loaded via Google Fonts. This reads as "systems/enterprise," not the Inter/Roboto default.
- **Scale (rem, 16px base):** display 2.0 / h1 1.5 / h2 1.25 / h3 1.125 / body 0.875 (**14px is the enterprise body size**) / caption 0.75. Line-height 1.4–1.5.
- **Tabular numbers** (`font-variant-numeric: tabular-nums`) on every column of amounts, hours, and dates.
- Uppercase labels use `letter-spacing: 0.06em`.

### 3.4 Spacing, shape, elevation

- **8pt grid** (`theme.spacing(1)=8px`); dense tables use 4px vertical rhythm.
- **Radius:** `4px` for controls, `8px` for panels/dialogs. No pill-radius cards.
- **Elevation:** `0` surfaces use a 1px border (not shadow); `overlay` (menus/dialogs/drawers) uses a single soft shadow token. Two elevation levels total.

### 3.5 Application shell (`features/shell/AppShell.tsx`)

Every authenticated route renders inside a persistent shell — the hallmark of enterprise software:

```
┌───────────────────────────────────────────────────────────────┐
│ TopBar: logo · global search · env badge · notifications · user│
├──────────┬────────────────────────────────────────────────────┤
│ SideNav  │ Breadcrumb  ·  Page title  ·  primary action        │
│ (collaps)│ ─────────────────────────────────────────────────── │
│  Time    │  Filter bar (chips, search, saved views)            │
│  Expense │ ─────────────────────────────────────────────────── │
│  Approv. │  Data region (table / detail / dashboard)           │
│  Admin   │                                                     │
│  Report  │                                                     │
└──────────┴────────────────────────────────────────────────────┘
```

- **SideNav:** grouped, RBAC-filtered, collapsible to icons; active item uses `brand.primary`. Sections: My Work (Time, Expenses), Manage (Approvals), Admin (Employees, Policies, Categories), Insights (Reports).
- **TopBar:** global command search (`⌘K` palette), environment badge (`LOCAL`/`DEV`/`PROD`), notification bell (approval events), user menu (role, cost center, sign out).
- **Content region:** breadcrumb + H1 + a single right-aligned primary action; then a **filter bar**; then the data region.

### 3.6 Component inventory (shared, in `components/`)

| Component | Behavior / spec |
|---|---|
| `DataTable` | Wraps MUI X Data Grid: server-side **pagination** (`limit`/`offset`), column sort, multi-filter, column show/hide, density toggle, sticky header, CSV export, saved views. Compact by default. |
| `StatusPill` | icon + label + semantic color; maps a status/`error.code` enum to token + text. Never color-only. |
| `FilterBar` | search input + removable filter chips + "Saved views" menu; state serialized to the URL query string (shareable, back-button safe). |
| `DetailDrawer` | right-side drawer for record detail/edit without losing table context; focus-trapped, `Esc` to close, deep-linkable. |
| `FormDialog` | modal form (RHF + Zod); inline field errors mapped from server `error.details`; disabled submit while pending; single primary action. |
| `MoneyCell` / `MoneyInput` | minor-units ↔ locale string via `lib/money.ts`; currency-aware (Scenario 4); right-aligned, tabular. |
| `StatCard` | KPI tile for dashboards — used sparingly, only where a number is the point. |
| `Toast` | non-blocking success/error notifications; action verbs ("Timesheet submitted"). |
| `EmptyState` / `Skeleton` / `ErrorState` | the three mandatory non-success states (see §3.7). |
| `PageHeader` | breadcrumb + title + primary action, consistent across all pages. |

### 3.7 Mandatory interaction states

Every data view implements **four** states — never a blank screen:
- **Loading:** skeleton rows matching the final layout (no spinners-on-white).
- **Empty:** illustration-free, a one-line explanation + the primary action ("No expense reports yet — Create report").
- **Error:** render `error.code` mapped through `lib/errorCode.ts` to a plain-language message + retry; include the `correlation_id` in a copyable "details" affordance for support.
- **Success:** the data.

Forms additionally show **pending** (disabled, spinner-in-button), **field-level validation** (from Zod and from server `error.details`), and **conflict** handling (`409`/idempotency).

### 3.8 Accessibility & responsiveness

- WCAG 2.1 AA. `jsx-a11y` lint + `axe-core` assertions in component tests + a Playwright axe pass per key screen (CI gate).
- Full keyboard operability; visible focus ring token; focus management on route change, drawer, and dialog; ARIA landmarks (`banner`, `navigation`, `main`).
- Responsive: three breakpoints — desktop (primary, ≥1200), laptop (≥900, nav auto-collapses), tablet (≥600, table → stacked cards for read views). Mobile is read-optimized, not the primary target (enterprise HRMS is desktop-first).
- Light/dark theme via a user toggle persisted to `localStorage`, defaulting to system preference.

---

## 4. Folder structure

```
hrms-web/
  src/
    app/                # store.ts, rootReducer, router.tsx, providers
    theme/              # palette.ts, typography.ts, components.ts (MUI overrides), index.ts
    api/
      generated/        # types generated from OpenAPI (machine-owned)
      baseApi.ts        # RTK Query base: baseUrl=/api/v1, auth header, error normalize
      identity.ts time.ts expense.ts workflow.ts
    features/
      shell/            # AppShell, SideNav, TopBar, CommandPalette
      auth/             # login, token refresh, useAuth, RequireRole guard
      timesheets/       # list, editor, PTO balance widget (Scenario 3)
      expenses/         # report editor, category dropdown (S1), currency (S4)
      approvals/        # manager queue + decision (S2)
      onboarding/       # create-employee wizard + provisioning status (S5)
      admin/            # employees, policies, categories
      directory/        # org / people
    components/         # DataTable, StatusPill, FilterBar, DetailDrawer, FormDialog, ...
    lib/                # money.ts, currency.ts, correlationId.ts, errorCode.ts, rbac.ts
    types/
  tests/e2e/            # Playwright specs mirroring SCENARIOS.md
  .env.example  Dockerfile  vite.config.ts
```

## 5. API client generation (contract-first)

```bash
npm run gen:api      # openapi-typescript contracts/openapi/*.yaml -> src/api/generated
```
- `src/api/generated/` is committed but machine-owned; CI fails if it drifts from the pinned contract versions.
- RTK Query endpoints wrap generated types; components consume typed hooks (`useGetPtoBalanceQuery`).
- A contract-changing scenario PR = bump pinned contract version → `gen:api` → adjust features → tests.

## 6. Auth flow (client side) — kept simple

- Login → one `token` held in memory (Redux) + mirrored to `localStorage` so a refresh survives (fine for a local teaching app; no refresh-token dance).
- `baseApi` injects `Authorization: Bearer <token>` and a generated `X-Correlation-Id`.
- On `401` → clear token, redirect to `/login` (re-login *is* the refresh story).
- Route guards use decoded `roles` claims; the server still enforces RBAC (the
  guard is UX only).

## 7. Screen specifications

| Screen | Route | Primary calls | Scenario |
|---|---|---|---|
| Login | `/login` | `POST /auth/login`, `GET /identity/me` | — |
| My Timesheets | `/time` | `GET /time/timesheets`, `GET /time/pto/balance` | 3 |
| Timesheet editor | `/time/:id` | `GET/PUT /time/timesheets/:id`, `POST …/submit` | 2, 3 |
| Expense reports | `/expenses` | `GET /expense/reports` | — |
| Expense editor | `/expenses/:id` | `GET /expense/categories`, line CRUD, `POST …/submit` | 1, 4 |
| Approvals queue | `/approvals` | `GET /approvals`, `POST /approvals/:id/decision` | 2 |
| Create employee | `/admin/employees/new` | `POST /identity/employees`, poll `GET /…/profiles/:id` | 5 |
| Directory / Org | `/directory` | `GET /identity/employees` | — |

Each list uses `DataTable`; each detail uses `DetailDrawer` or a full detail route; each create/edit uses `FormDialog`. All four §3.7 states are required.

### P2 delivered surface

`hrms-web@11dcc11` delivers authenticated `/time` and `/expenses` workspaces
using generated Time and Expense OpenAPI types plus RTK Query. Employees can
create, edit, and submit timesheets and expense reports, add up to seven
timesheet entries, view PTO balance, add categorized multi-currency expense
lines with receipt links, and see loading/error/empty states. The richer shared
table/drawer design system and separate detail routes remain incremental UI
work rather than prerequisites for the P2 domain exit criterion.

---

## 8. Development guide (how to build it)

**Bootstrap:**
```bash
npm create vite@5 hrms-web -- --template react-ts
# add deps
npm i @reduxjs/toolkit react-redux @mui/material @mui/x-data-grid @emotion/react @emotion/styled \
      react-router-dom react-hook-form @hookform/resolvers zod date-fns i18next react-i18next
npm i -D vitest @testing-library/react @testing-library/jest-dom jsdom \
      @playwright/test axe-core eslint prettier openapi-typescript msw
```

**Coding standards:** TS `strict` + `noUncheckedIndexedAccess`; no `any` (lint error); components are function components with typed props; data access **only** through RTK Query hooks (no `fetch` in components); business logic in `lib/` and feature `*.selectors.ts`, never in JSX; every user-facing string via i18n keys. Prettier + ESLint enforced pre-commit (husky + lint-staged).

**Build order (first two sprints):**
1. Theme + `AppShell` + routing + auth guard skeleton (renders with mocked user).
2. `baseApi` + `gen:api` wired to the real gateway; login flow end-to-end.
3. `DataTable` + `StatusPill` + `FilterBar` + the three non-success states (the reusable spine).
4. Timesheets list/editor → Expenses list/editor → Approvals queue.
5. Scenario features layered on (S1–S5) as their contracts land.

**Definition of done (per screen):** typechecks (strict); unit + a11y (axe) tests green; all four states implemented; Playwright happy-path passing; generated client in sync with pinned contract; keyboard-operable; both themes verified.

## 9. Testing

- **Unit/component:** Vitest + RTL; mock the API with **MSW** using response shapes from generated types; assert `StatusPill`/error mapping and the four states.
- **A11y:** `axe-core` in component tests + Playwright axe pass per key screen (CI gate; violations fail the build).
- **E2E:** Playwright specs in `tests/e2e/` map 1:1 to [`../SCENARIOS.md`](../SCENARIOS.md), run against a running estate (`docker compose up --build -d`).
- **Contract-drift gate:** CI runs `gen:api` and fails if `src/api/generated` changes — forcing regeneration in the contract-bump PR.
- **Coverage gate:** ≥ 80% statements on `lib/` and feature reducers/selectors.

## 10. CI (GitHub Actions, in this repo)

`lint (eslint + jsx-a11y) → typecheck → gen:api drift check → unit + a11y tests → build → docker build/publish`. E2E + full axe sweep run in `platform-outerloop`'s integration pipeline against a composed environment. **DoD:** strict typecheck clean, tests green, generated client synced to pinned contract versions.

## 11. Implementation notes (as-built, P4)

Added `/employees` (HR_ADMIN-gated, replacing the disabled P3 nav
placeholder): `features/admin/CreateEmployeePage.tsx` is a form that calls a
new `createEmployee` mutation (`api/identity.ts`), then renders a
provisioning-status panel that polls the new `getTimeProfile`/
`getExpenseProfile` queries (`api/time.ts`/`api/expense.ts`) every 1.5s,
stopping each once its profile resolves — Workflow exposes no profile-read
endpoint, so only Time and Expense are polled, matching the Scenario 5
sequence diagram. Evidence: `hrms-web@b81a195`, ESLint/`tsc -b`/Vitest
(8/8)/Vite build green.
