# Atlas HRMS — Product Walkthrough

> A hands-on tour of everything built so far (P0 → P4). Follow it top to
> bottom the first time; after that, jump to whichever section you want to
> re-explore. It assumes the estate is already running — see
> [§0](#0-start-the-estate) if it isn't.
>
> This is a **local teaching build**. Every login below uses the fixed demo
> password `atlas`, seeded in plain text in
> `platform-outerloop/compose/.env.example` — this is intentionally not a
> secret.

---

## 0. Start the estate

```bash
cd /home/oswa/atlas-repos/platform-outerloop/compose
docker compose up --build -d
cd ../scripts
uv run --project .. python seed.py     # idempotent — safe to re-run any time
uv run --project .. python smoke.py    # optional: proves the whole product works end-to-end
```

Give it ~10–15 seconds after `up -d` for Postgres/RabbitMQ health checks to
pass before the four services come up healthy. `docker compose ps` should
show all 10 containers, with `postgres`, `rabbitmq`, `svc-identity`,
`svc-time`, `svc-expense`, and `svc-workflow` all `(healthy)`.

## 1. Where everything lives (open these in your browser)

| What | URL | Notes |
|---|---|---|
| **The product (hrms-web)** | http://localhost:8080/ | Everything in §3–§7 happens here. |
| API gateway (Traefik), same host | http://localhost:8080/api/v1/... | The SPA calls this; you can also `curl` it directly (examples below). |
| Traefik dashboard | http://localhost:8082/dashboard/ | See the 4 backend routers and which container is serving each. |
| RabbitMQ management UI | http://localhost:15672/ | Login `atlas` / `atlas`. Watch exchanges/queues/messages move in real time — see §6. |
| MailHog (fake SMTP inbox) | http://localhost:8025/ | Every "approval requested" email lands here — see §5. |
| Adminer (Postgres browser) | http://localhost:8081/ | System: **PostgreSQL**, Server: `postgres`, Username: `atlas`, Password: `atlas`, Database: pick one of `identity_db`, `time_db`, `expense_db`, `workflow_db`. |

## 2. Demo logins — who you can be, and what each role unlocks

All four accounts exist from the moment you run `seed.py`. The password for
every one of them is **`atlas`**.

| Email | Password | Full name | Roles | Manager | Grade / cost center | What this login is for |
|---|---|---|---|---|---|---|
| `ada@atlas.dev` | `atlas` | Ada Lovelace | `EMPLOYEE` | Grace Hopper | IC4 / CC-100 | An ordinary employee: submit time & expenses, watch your own PTO balance. Pre-filled by default when you open the login page. |
| `grace@atlas.dev` | `atlas` | Grace Hopper | `EMPLOYEE`, `MANAGER` | (none — top of the chain) | M2 / CC-100 | Ada's manager: sees and decides Ada's pending approvals. |
| `finance@atlas.dev` | `atlas` | Frances Perkins | `EMPLOYEE`, `FINANCE` | (none) | M2 / FIN-100 | Broader read access (any employee's expense reports) and policy administration — API-only today, see §8. |
| `admin@atlas.dev` | `atlas` | Henrietta Hill | `EMPLOYEE`, `HR_ADMIN` | (none) | M3 / HR-100 | The only login that can create/deactivate employees — the wizard in §7. |

**Role → capability matrix** (what actually gates what, in the code, not just in the UI):

| Capability | EMPLOYEE | MANAGER | FINANCE | HR_ADMIN |
|---|:--:|:--:|:--:|:--:|
| Log in, see own profile (`GET /identity/me`) | ✅ | ✅ | ✅ | ✅ |
| Submit/edit own timesheets, see own PTO balance | ✅ | ✅ | ✅ | ✅ |
| Submit/edit own expense reports | ✅ | ✅ | ✅ | ✅ |
| See **any** employee's expense reports (`GET /expense/reports`) | own only | own only | **all** | own only |
| See the Approvals queue / decide approvals | ✖ | ✅ (only items where *you* are the approver) | ✅ (same) | ✖ |
| Read/write workflow policies (`/workflow/policies`) | ✖ | ✖ | read + write | read only |
| Create / update / deactivate employees | ✖ | ✖ | ✖ | ✅ |
| See downstream provisioning profiles (`/time/profiles/{id}`, `/expense/profiles/{id}`) | ✖ | ✖ | ✖ | ✅ |

A note on **why Grace sees approvals but Frances doesn't (yet)**: approval
routing is always **direct-manager-only** — the approver on every approval
row is resolved from `employee_read.manager_id`, i.e. whoever the requester's
actual manager is. `FINANCE` grants you the *role gate* to open the Approvals
screen and the policy-admin API, but you'll only ever see approvals where
you're someone's real manager. In the seed data nobody reports to Frances, so
her Approvals queue will show as empty — that's expected, not a bug.

---

## 3. Employee walkthrough — log in as Ada (`ada@atlas.dev` / `atlas`)

Open http://localhost:8080/ — the login form is pre-filled with Ada's
credentials, so you can just click **Sign in**.

1. **Overview (`/`).** You land on a welcome screen: "Welcome, Ada Lovelace",
   with her cost center and roles underneath. This comes from
   `GET /api/v1/identity/me`.
2. **Time (`/time`).**
   - You'll see your current PTO balance chip at the top — this is
     `GET /api/v1/time/pto/balance`, composed *inside* `svc-time` from Ada's
     replicated `pto_entitlement_days` (arrived via an `employee.created`
     event from Identity) plus her accrual/usage ledger. The browser never
     calls Identity directly for this — that's the "API composition, not
     browser fan-out" lesson from Scenario 3.
   - Below it, create a new timesheet: pick a Monday-starting week, fill in
     5 daily entries. Try entering **10 hours/day for 5 days (50 total)** —
     that's 10 hours over the 40-hour standard week.
   - Click **Submit**. The status badge flips to `PENDING_APPROVAL`, and
     because 10 > the seeded 5-hour overtime threshold, this timesheet will
     show up in Grace's Approvals queue flagged `OVERTIME_THRESHOLD` (§4).
3. **Expenses (`/expenses`).**
   - Click **New report**, give it a title.
   - Add a line: pick a **category** from the dropdown (`Travel`, `Meals`,
     `Lodging`, `Office supplies` — these come from `GET /expense/categories`,
     seeded reference data), an amount, a currency (try a non-home currency
     like `USD` — Ada's home currency is `GBP`, so svc-expense converts it to
     GBP at the day's seeded FX rate; use `GBP` directly for a 1:1
     no-conversion line), and a receipt URL.
   - The seeded per-receipt cap is `10000` minor units, evaluated against
     the **home-currency** total (`amount_home_minor`) — for Ada that's
     £100.00. Enter a line over that (e.g. £150) and submit the report.
   - The report's status flips to `PENDING_APPROVAL`; because the line
     exceeds the cap, it'll show up in Grace's queue flagged
     `PER_RECEIPT_CAP` (§4).
4. **Sign out** (top-right) when you're done — this clears the RTK Query
   cache too (verified by an automated test), so the next login starts clean.

---

## 4. Manager walkthrough — log in as Grace (`grace@atlas.dev` / `atlas`)

1. Sign out of Ada's session (or use a private/incognito window), sign in as
   Grace.
2. Notice the left nav now has an **Approvals** item — it's gated on
   `MANAGER`/`FINANCE` and only appears for those roles (`AppShell.tsx`).
3. Open **Approvals (`/approvals`)**. You should see the timesheet and/or
   expense report you submitted as Ada in §3, each with a policy-flag chip
   (`OVERTIME_THRESHOLD` or `PER_RECEIPT_CAP`) and a human-readable reason
   message.
4. For the timesheet: type an optional comment, click **Approve**. For the
   expense report: click **Reject** with a comment like "Over policy,
   please resubmit with a lower amount" (or approve it — your call).
5. Go back to Ada's session (or just call `GET /api/v1/time/timesheets/{id}`
   as her) — the timesheet is now `APPROVED`; the expense report you
   rejected is `REJECTED` (or `REIMBURSED` if you approved it — Expense
   collapses `APPROVED → REIMBURSED` immediately since there's no separate
   payment step in this teaching build).
6. **Check MailHog** (http://localhost:8025/) — when the approval was first
   *requested* (i.e. right after Ada submitted), an email was sent to
   `grace@atlas.dev` with a subject containing "approval requested". You
   should see it in the inbox list; open it to see the templated body.

---

## 5. Watching the machinery — RabbitMQ and email

This section doesn't require any login; it's about *seeing* the
event-driven backbone that powers §3–§4 and §7.

1. Open the **RabbitMQ management UI** (http://localhost:15672/, login
   `atlas`/`atlas`).
2. Go to **Exchanges**. You'll see one topic exchange per publishing
   service: `time.events`, `expense.events`, `workflow.events`, and
   `identity.events`.
3. Go to **Queues**. You'll see the durable consumer queues each service
   bound on startup: `workflow.timesheet-events`,
   `workflow.expense-events`, `time.workflow-decisions`,
   `expense.workflow-decisions`, and the three new P4 queues —
   `time.employee-events`, `expense.employee-events`,
   `workflow.employee-events` — all bound to the `identity.events` exchange
   for `employee.created`/`employee.updated`/`employee.deactivated`.
4. Click into any queue's **Get messages** tab immediately after doing an
   action in the UI (e.g. submitting a timesheet) to catch the message
   in flight before it's consumed — a nice way to see the exact JSON payload
   each event carries (correlation ID included).
5. **MailHog** (http://localhost:8025/) is the second observable side
   effect: every time an approval is first requested, `svc-workflow` sends
   a real SMTP email to this fake inbox — inspect headers/body there.

---

## 6. Browsing the data directly — Adminer

Open http://localhost:8081/, log in with System `PostgreSQL`, Server
`postgres`, Username `atlas`, Password `atlas`, and one Database at a time:

- **`identity_db`** — `employees`, `roles`, `employee_roles`, `credentials`.
  This is the single source of truth for who exists and what roles they have.
- **`time_db`** — `timesheets`, `time_entries`, `employee_read` (the P4
  projection kept in sync from `employee.*` events; also seeded up front for
  demo employees).
- **`expense_db`** — `expense_reports`, `expense_lines`, `categories`,
  `fx_rates`, `employee_read`.
- **`workflow_db`** — `approvals`, `policies`, `employee_read` (no
  `pto_entitlement_days`/`roles` columns here — Workflow only needs
  `email`/`full_name`/`manager_id`/`status` to route and notify).

This is a good way to *prove to yourself* that the `employee.created` event
you'll trigger in §7 really does insert a brand-new row into all three of
`time_db.employee_read`, `expense_db.employee_read`, and
`workflow_db.employee_read` — not just update the UI.

---

## 7. HR Admin walkthrough — new-hire onboarding fan-out (P4, the newest feature)

This is the flagship "event-driven fan-out" feature: creating **one**
employee in Identity automatically provisions matching records in Time,
Expense, and Workflow — with no manual step in any of those three services.

1. Sign in as `admin@atlas.dev` / `atlas`.
2. Notice the left nav now shows a real, clickable **Employees** item
   (it was a disabled placeholder before P4) — gated on `HR_ADMIN`.
3. Open **Employees (`/employees`)** and fill in the **Create employee**
   form:
   - Email: anything unused, e.g. `new.hire@atlas.dev`
   - Full name: e.g. `Nikola Tesla`
   - Grade: e.g. `IC2`
   - Cost center: e.g. `CC-100`
   - Manager ID *(optional)*: paste Grace's id
     `10000000-0000-4000-8000-000000000001` to make Grace the new hire's
     manager (so a future timesheet from them would route to Grace)
   - Home currency: `USD`
   - Roles: leave `EMPLOYEE` checked
4. Click **Create employee**. The API responds `201` immediately with
   `provisioning: "PROVISIONING"` — the employee row exists in Identity, but
   downstream systems haven't caught up *yet*.
5. Watch the **Provisioning status** panel that appears right below the
   form: two chips, "Time profile provisioning…" and "Expense profile
   provisioning…", each with a spinner. Under the hood these are polling
   `GET /api/v1/time/profiles/{id}` and `GET /api/v1/expense/profiles/{id}`
   every 1.5 seconds; each 404s with `TIME_PROFILE_NOT_FOUND` /
   `EXPENSE_PROFILE_NOT_FOUND` until its consumer has processed the event.
6. Within a second or two both chips should turn green ("Time profile
   ready" / "Expense profile ready") and a success banner appears: "Employee
   fully provisioned across Time and Expense." If you were fast enough with
   the RabbitMQ UI in §5, you'd have seen the `employee.created` message
   land in all three `*.employee-events` queues in between steps 4 and 6.
7. **Prove it in Adminer too:** open `time_db.employee_read` and
   `expense_db.employee_read` and find the new employee's row by email —
   it's there, with the derived `pto_entitlement_days` for their grade.
   Check `workflow_db.employee_read` as well — Workflow doesn't expose a
   polling endpoint (it doesn't need one; it only needs the row for approval
   routing), but the row is there too if you query the database directly.
8. **Deactivate them:** call
   `POST /api/v1/identity/employees/{id}/deactivate` (no UI button for this
   yet — see §8 for the `curl`) as `admin@atlas.dev`, then re-check
   `workflow_db.employee_read` — `status` flips from `ACTIVE` to `INACTIVE`,
   proving `employee.deactivated` fanned out too.

---

## 8. Things you can only do via the API today (no UI screen yet)

The SPA doesn't have a screen for every backend capability. These are worth
trying with `curl` to see the full product surface, not just what's wired
into `hrms-web` so far.

```bash
# Log in as Frances (FINANCE) and grab her token
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"finance@atlas.dev","password":"atlas"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')

# As FINANCE: list every employee's expense reports, not just your own
curl -s http://localhost:8080/api/v1/expense/reports \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# As FINANCE (or HR_ADMIN): read the two seeded policies
curl -s http://localhost:8080/api/v1/workflow/policies \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# As FINANCE: tighten the overtime threshold to 2 hours (only FINANCE can write)
curl -s -X POST http://localhost:8080/api/v1/workflow/policies \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"type":"OVERTIME_THRESHOLD","params":{"threshold_hours":2},"active":true}'

# Deactivate the employee you created in §7 (swap in the real id + an
# HR_ADMIN token from admin@atlas.dev)
curl -s -X POST http://localhost:8080/api/v1/identity/employees/<id>/deactivate \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

## 9. Prove the whole product still works, end to end

`platform-outerloop/scripts/smoke.py` is the automated version of §3–§7
combined: it logs in as Ada, submits a timesheet and an expense report
(both flagged), logs in as Grace to decide both, checks RabbitMQ for the
right events and MailHog for the notification, then logs in as
`admin@atlas.dev` to create a brand-new employee, waits for Time/Expense/
Workflow to all pick it up, re-reads a profile to prove the consumer is
idempotent, and finally deactivates the employee and confirms Workflow's
projection follows. Run it any time you want a single command that either
proves everything above still works, or tells you exactly which step broke:

```bash
cd /home/oswa/atlas-repos/platform-outerloop/scripts
uv run --project .. python smoke.py
```

## 10. What's *not* built yet

- No self-service "forgot password" / password change flow — the seeded
  credential is the only way in for each demo user.
- No UI screen for policy administration (§8's `curl` examples) or for
  deactivating/updating an employee — both exist as APIs only.
- Workflow's approval routing is direct-manager-only (no multi-step
  approval chains) — by design for this teaching build, see
  [`docs/repos/svc-workflow.md` §8](./repos/svc-workflow.md#8-implementation-notes-as-built-p3).
- P5 (`docs/SCENARIOS.md`) frames five cross-repo product scenarios as a
  learning exercise; most of their backend behavior (categories + caps,
  overtime approval, PTO widget, multi-currency, onboarding fan-out) is
  already implemented as part of P1–P4 above — P5 itself is about
  documenting/practicing the *change process*, not adding new runtime
  behavior, and hasn't been started yet.
