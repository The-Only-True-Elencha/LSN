# House Bill Calendar

A shared bill calendar for the House account. It shows bills on their due dates, paydays, and paid checkmarks. It also has a paycheck-by-paycheck view and a running record of what each pay period leaves in House, with a suggested steady auto-transfer.

The live copy is a claude.ai artifact. `index.html` is its source.

## How it runs

The page is written for the claude.ai artifact runtime. It reaches four runtime capabilities through `window.claude.use(name)`:

| Capability | Used for |
|---|---|
| `db` | Shared storage for bills, paychecks, paid marks and settings |
| `user` | Recording who checked a bill paid, and showing their name |
| `mcp` (Gmail: `search_threads`, `get_thread`) | The "Check my email" button reads recent bill emails from the viewer's own Gmail |
| `sample` | Claude pulls bill amounts and due dates out of those emails |

Opened outside claude.ai, every capability resolves to `null` and the page shows a "waiting for the shared calendar" message. To run it on your own host, replace `use("db")` with your own store and drop or replace the email check.

## Data model

All documents live in the artifact's shared database.

- `bills/{id}`
  - `name`, `amount`
  - `freq`: `monthly`, `quarterly` or `annual`
  - `day`: day of the month it's due, or `null` to set the amount aside evenly each paycheck (fuel, groceries)
  - `months`: month numbers (1 to 12) for quarterly and annual bills
  - `autopay`, `paused`, `account`, `note`, `kind`
- `income/{id}`
  - `name`, `amount`
  - `anchor`: any past deposit date, `YYYY-MM-DD`
  - `everyDays`: usually 14
  - `toHouse`: whether it counts toward House bills
  - For a bonus instead of a regular paycheck: `months` (month numbers) and `day`. It lands on the first payday on or after that day in each listed month, and `anchor`/`everyDays` are ignored.
- `paid/{billId}__{YYYY-MM-DD}`: one document per bill occurrence marked paid, with `billId`, `date`, `amount`, `by` (user id) and `at`
- `settings/main`
  - `balance`, `balanceDate`: House balance at the end of that day
  - `cushion`: the minimum you want House to keep
  - `transfer`: the current auto-transfer per paycheck
  - `payAnchor`: a payday that pay periods start from

## The math

- **Pay period:** 14 days starting on a payday. A bill belongs to the paycheck that lands on or before its due date.
- **Left over:** House deposits in the period, minus dated bills due in it, minus spending set aside per paycheck.
- **Projected balance:** starts from the entered House balance and steps day by day for a year. Deposits and set-aside spending apply on paydays, and unpaid bills apply on their due dates. Bills already checked paid are treated as already out.
- **Suggested transfer:** the lower of two numbers, rounded down to $5.
  - The average left over across the next 26 paychecks.
  - The largest steady transfer that keeps the projected balance above the cushion every day of the next year.

## Daily email check

A scheduled Claude task, "Bill email check", runs every morning at 7:43 Eastern. It reads the last three days of bill and payment emails and applies what it finds:

- checks off payments
- updates changed amounts
- adds new recurring bills

Each change is written to `emaillog/{messageId}-{action}` with the before and after values, and the page's Email updates panel can undo it. Each run also writes `meta/emailcheck`, with `lastRun` and a short `summary`.
