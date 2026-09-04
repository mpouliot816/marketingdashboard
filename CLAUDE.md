# CLAUDE.md

Working rules for this repository. Read this before changing anything.

This is an internal marketing performance dashboard for a multifamily property
operator. It ingests a weekly Excel workbook, persists it to Supabase (Lovable
Cloud), and surfaces metrics, forecasts and alerts. It replaces a manual
Excel-to-static-HTML process.

Stack: TanStack Start (TypeScript, React 19), Supabase, Tailwind, shadcn/ui.
Package manager: **bun**. Scripts: `dev`, `build`, `typecheck`, `lint`, `test`,
`ci` (`typecheck && test`).

---

## Repository status — read this first

The application source is **not currently in this repository**. As of the last
session, `mpouliot816/marketingdashboard` contained only `AUDIT.md`, this file,
and the CI workflow. The code lives in the Lovable project `carbonamarketgoalseek`
(id `1f0a5336-86ae-4986-99b5-dff6eafdf9a9`).

The Lovable README claims "every change made in Lovable is committed straight to
this repository." That has not happened. Verify where the code actually is before
assuming a file path exists.

The five documents referenced below (`docs/*.md`) are **not yet committed**.
Where this file points at them, it is describing the intended structure, not a
file you can currently open. If you add them, the state rule below applies from
that moment on.

---

## The non-negotiables

These are the product's reason for existing. The dashboard replaced a process
that quietly lied; every rule here exists to stop it lying again.

1. **A missing value is never rendered as zero.** Absence and zero are different
   facts. Use the `Corrected` envelope in `src/lib/corrections.ts` — `notCaptured`,
   `missing`, `quarantined`, `checkSource` — and render through `displayValue`
   or `CorrectedCell`. An empty query result is not a healthy zero.
2. **Never display a number that cannot carry provenance.** Every metric in
   `src/lib/metrics.ts` has a `provenance` record naming source, sheet and
   transformation. A figure with no provenance record does not go on screen.
3. **Never generate, jitter, interpolate or synthesise data to fill a gap.** If
   the data is not there, say it is not there. This is enforced by
   `src/lib/__tests__/no-invented-data.test.ts`, which bans `Math.random`,
   `jitter`, and hash-derived figures from `src/lib`. That test exists because a
   fabricated weekly series was once feeding a live alert. Do not weaken it.
4. **Append-only with supersession, never deletion.** Fact rows carry
   `valid_from` / `valid_to`. A new load sets `valid_to` on the rows it replaces
   and inserts new ones. Never `DELETE` a fact row. Never `UPDATE` a fact value
   in place.
5. **One canonical selector per metric.** `selectMetric` in `src/lib/metrics.ts`
   is the only way a metric is read. KPI cards, tables, charts, alert evidence
   and CSV export all call it with the same `Filters` shape.
6. **Corrections live in one module.** `src/lib/corrections.ts`. A correction
   applied in a component is a correction that some other surface will miss.
7. **Role titles, never personal names.** The UI attributes actions to
   "Director of Property Management", not to a person.
8. **Golden fixtures may not be edited to agree with the code.** `src/lib/golden.ts`
   holds frozen known-good figures. If a fixture fails, the default assumption is
   that the code or the data is wrong. A fixture may only move with a recorded
   entry in `FIXTURE_AMENDMENTS` giving the previous value, the date and the
   reason. `unloggedAmendments()` treats an unrecorded change as a failure of the
   harness itself. This rule was written after a fixture was quietly lowered from
   74 to 48 to match a short dataset.

---

## Security rules

1. **Roles live in a separate table.** `public.user_roles`, keyed to
   `auth.users`. Never a column on `profiles` or in user metadata — anything the
   account can write is not an authorisation source. `user_roles` has a SELECT
   policy only; the absence of INSERT/UPDATE/DELETE policies is what stops a user
   granting themselves a role.
2. **Table grants are wide open; RLS is the only gate.** Both `anon` and
   `authenticated` hold full table privileges (`arwdDxtm`) on public tables — the
   Supabase default. Do not read a `GRANT` in a migration as protection. If a
   table needs to be unwritable, it needs *no policy* for that operation.
3. **RLS on every table, with at least one policy.** RLS enabled with zero
   policies denies everything and reads as a bug. RLS disabled exposes the table.
4. **Every `SECURITY DEFINER` function pins `search_path`.** `SET search_path = public`.
   An unpinned definer function is a privilege-escalation vector.
5. **Policies call the helper functions, never query `user_roles` directly.**
   Use `has_role`, `is_portfolio_wide`, `can_act_on_property`. A policy that
   selects from `user_roles` inside a `user_roles` policy recurses.
6. **Server actions must resolve the caller's role themselves.** Server functions
   write through the service-role client, which **bypasses RLS entirely**. The
   middleware `requireSupabaseAuth` verifies the JWT and gives you `context.userId`
   and a token-scoped `context.supabase` — read `user_roles` through that before
   doing anything. `ingestWorkbook` and `ratifyThresholdChange` both do this;
   copy that shape.
7. **Attribution columns come from `auth.uid()`, never from client input.**
   Who acted and under which role are stamped by database triggers
   (`stamp_alert_attribution`, `stamp_proposal_author`), which also reject a role
   the account does not hold. Do not add an attribution column that the client
   sets directly.
8. **The service-role key never reaches the browser.** It is read from
   `process.env`, never `import.meta.env`, and never `VITE_`-prefixed. In a
   `*.functions.ts` file — which does ship to the client bundle — import it
   *dynamically inside the handler*:
   `const { supabaseAdmin } = await import("@/integrations/supabase/client.server");`
9. **No real PII in the deployed database.** Person-level fields are replaced with
   synthetic identities on the way in (`src/lib/synthetic-identity.ts`). Emails use
   the reserved `.invalid` TLD, phones the reserved `555-01xx` range. The source
   workbook has ~11,700 real resident records; none of it may reach Cloud or git.

---

## The state-update rule

> `docs/KNOWN-ISSUES.md` is the current state of this project. It must be updated
> in the same commit as any change that alters what it describes. If you fix
> something listed there, mark it resolved with the date. If you discover
> something new, add it. If you change behaviour the other docs describe, update
> those too. **A commit that changes behaviour without updating the docs is
> incomplete.**

This is not a style preference. The documentation in this repository was written
from conversation rather than from the code, and a external audit found it
materially wrong in several places. The only thing that keeps it true is updating
it in the same commit as the change.

---

## The other documents

| File | Read it when |
|---|---|
| `docs/KNOWN-ISSUES.md` | **Always, first.** Current defects, their status and dates. Update it in the same commit as any behaviour change. |
| `docs/ENGINEERING.md` | Before changing the data path — selectors, corrections, the append-only model, the invariant harness. |
| `docs/GOVERNANCE.md` | Before touching roles, RLS policies, the proposal/ratification flow or the two-person rule. |
| `docs/DATA-SOURCES.md` | Before changing the parser, the property alias table, or anything about the weekly workbook's sheets. |
| `docs/OPERATIONS.md` | Before changing a screen, or when you need to know what the UI is supposed to do. |
| `README.md` | Orientation only — roles, routes, how to run it. |
| `AUDIT.md` | The external audit that produced most of the above rules. Historical, but the reasoning is worth reading once. |

---

## How this project is built

**Two agents, one repository.** Features are built by the **Lovable agent** in
the Lovable editor. Review, audit and hardening happen in separate Claude Code
sessions. Both write to the same repository.

**The sync is bidirectional on `main`.** Lovable commits its changes to the
connected branch, and commits pushed to `main` sync back into the Lovable editor.
Keep `main` in a working state.

**Never rewrite published history.** Per `AGENTS.md`, force-pushing, rebasing,
amending or squashing commits that are *already pushed* rewrites history on
Lovable's side and can destroy the project's history there. Rebasing your own
unpushed work onto someone else's commits is fine. Force-pushing a shared branch
is not.

**Agent self-reports have been unreliable.** Summaries have described fixes that
were not made, and documentation has asserted behaviour that the code did not
have. Some specific things that turned out to be false when checked:

- A doc comment stating property names are matched exactly, over a function that
  fuzzy-matches on prefixes.
- A harness invariant written so that it compares a value to itself and can never
  fail.
- A "single-selector rule" guard whose regex cannot match the violations that
  actually exist in the codebase.
- A source header describing a weekly series as "deterministic bucketing of the
  event pull" when it was a constant plus hash jitter.

**So: verify against the code and the database, not against a summary.** Read the
migration and the policy, not the comment above it. Query the live database for
row counts and totals rather than trusting a number in a document. When a
docstring and the code disagree, the code is what ships — fix the docstring and
report it.

---

## Known invariant violations

These are documented rules the code does **not** currently honour. They are
findings, not permissions — do not copy these patterns.

- **The single-selector rule is violated on the Overview page.**
  `src/components/dashboard/KpiCards.tsx` computes `totals.spend / totals.leases`,
  `pct(totals.toursBooked, totals.leads)` and `pct(totals.leases, totals.leads)`
  directly instead of calling `selectMetric`. `src/routes/_authenticated/index.tsx`
  does the same in its "Marketing efficiency" panel — cost per lease, spend per
  leasable unit and tour-to-lease are all computed inline. `selector-guard.test.ts`
  passes anyway: its regex only matches arithmetic over `ds.perf`-style raw rows,
  not arithmetic over an already-computed `Totals` object.
- **"Applications started" is a hardcoded string** on the Overview page
  (`value="Not captured"`), not derived from `detectApplicationsBreak`. It will
  still read "Not captured" after the upstream feed is repaired.
- **`selectMetric`'s applications branch uses `scope.every(...)`**, so a
  multi-week scope containing one good week returns a number that silently
  includes the dead weeks as zeros. It should be `scope.some(...)`.
- **Ingest has no transaction boundary.** Supersession commits before the inserts,
  so a mid-run failure leaves the current view partially empty with a
  `load_batches` row claiming success.
- **`fact_lead` supersession is unscoped** — it retires every current lead row
  regardless of which weeks the uploaded workbook covers.

---

## Before you commit

- `bun run typecheck` and `bun run test` pass (`bun run ci` runs both).
- `bun run lint` is clean for files you touched.
- `docs/KNOWN-ISSUES.md` reflects what you changed.
- No fixture was edited to make a test pass.
- No secret, no real workbook, no `.xlsx`, and no real resident data is staged.
