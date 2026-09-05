# Engineering

Rules for changing this codebase. Most exist because something broke.

> **Verified against the code on 4 September 2026.** Three of the seven
> non-negotiables below are not honoured by the current code. The rules stay as
> written — they are the standard. The violations are marked **⚠ VIOLATED** at
> the end of each rule, and repeated in `KNOWN-ISSUES.md`. A rule the code
> breaks is a finding, not a mistake in this document.

---

## Non-negotiables

### 1. A missing value is never rendered as zero

Missing, quarantined and unevaluable are distinct states and each is as visually present as a good value. Several upstream feeds report zero when broken. A dashboard that renders those as 0% looks correct and is lying.

**⚠ VIOLATED, twice, in opposite directions.** `selectMetric`'s applications branch tests `scope.every(w => brk.weeks.includes(w))`, so a scope is only suppressed when *every* week in it is dead. The unfiltered dashboard spans January to August, one good week makes the test false, and the selector returns `ok(167)` — 167 real applications plus 27 weeks of fake zeros, carrying `status: "ok"`. It should be `scope.some(...)`. And on the Overview, "Applications started" is the hardcoded string `"Not captured"`, so that surface can never show a number even once the feed is repaired. The harness check for this cannot catch either: it tests `appsAll.value === 0`, and the value is 167.

Honoured elsewhere: `DatasetStatus` correctly blocks the page while the read is in flight or has failed, so no surface renders zeros for a pending query.

### 2. Never display a number you cannot trace

Every figure resolves to a provenance record. If it cannot carry one, it does not render.

**Never generate, jitter, interpolate or synthesize data to fill a gap.** A weekly response-time trend was once built from a constant plus hash jitter, with three properties hardcoded to decline, and a live alert read that series. The correct behaviour when data does not exist is to render "Not computable" and say why.

If you are about to write something that makes a chart look complete, stop.

**Status: fixed, and now enforced.** The synthetic series is deleted from `elise.ts` — `speedWeeks`, the hash function and the `DECLINING_3W` set are all gone, replaced by `SPEED_WEEKLY_UNAVAILABLE`, and `perf.speed_to_lead_declining` now evaluates as Unevaluable rather than clearing. `no-invented-data.test.ts` bans `Math.random`, `jitter` and hash-derived figures from `src/lib`, with one declared exemption for identifier generation in `ratification.ts`, and asserts the deleted function stays deleted.

**Not covered by that test:** `elise.ts` still holds six hardcoded constant tables — tour completeness, portfolio completeness, speed to lead, portfolio speed, first-contact splits, negative durations. They are seeded from a real pull and the file header says so, but they render on the Overview and Data Integrity pages as live portfolio figures, they carry no provenance record, and they will not move when a workbook is ingested. This is disclosed data, not invented data, but a reader cannot tell them apart on screen.

### 3. Append-only

Nothing is overwritten or deleted. Restating a week supersedes prior rows via `valid_from` / `valid_to`. Every changed figure writes a row to `restatements` with old value, new value and delta.

Removing bad data means superseding it and logging the reason, never a hard delete.

**Honoured.** No `DELETE` on a fact row exists in the ingest action or on any client path; the only `UPDATE` sets `valid_to`. The live tables bear it out: `fact_lead` 11,011 current / 11,011 superseded, `fact_weekly` 3,809 / 3,809.

**Two gaps in how it is applied.** Ingest has **no transaction boundary** — supersession commits before the inserts, so a mid-run failure leaves the current view partly empty behind a `load_batches` row that claims success. And `fact_lead` supersession is unscoped: it retires every current lead row regardless of which weeks the uploaded workbook covers, where `fact_weekly` correctly scopes to the affected weeks.

### 4. One canonical selector per metric

Every metric has exactly one selector function. Pages, charts, alert cards, exports and the predictive layer all call it with the same filter shape. **No component computes a metric locally** — not for a KPI card, not for a sparkline, not for an alert evidence line.

Consistency checks written after the fact document drift. This rule prevents it existing. The header alert count and the alerts page once read different sources and happened to agree; they now read one.

**⚠ VIOLATED on the Overview page, and the guard cannot see it.**

`KpiCards.tsx` never calls `selectMetric`. It takes a raw `Totals` and computes `totals.spend / totals.leases`, `pct(totals.toursBooked, totals.leads)` and `pct(totals.leases, totals.leads)` inline, and calls `cohortShowRate` and `costPerLead` directly rather than through the `showRateCohort` and `cpl` selectors. `routes/_authenticated/index.tsx` does the same in its "Marketing efficiency" panel: established and ramp-up cost per lease, spend per leasable unit and tour-to-lease are all computed in the component.

`selector-guard.test.ts` passes. Its regex matches arithmetic over `ds.perf` / `ds.spend`-style raw rows; it does not match arithmetic over an already-computed `Totals` object. The `identity` invariant does not catch it either — it compares `sliceData` totals against `selectMetric` for additive keys only, and both sides come from `sliceData`, so they agree trivially. No derived metric is compared across surfaces.

**The green test is the dangerous part.** Widen the guard before trusting it again.

The header alert count claim is true: `AppShell` and the alerts page both read `useCloudAlerts()`, and there is no in-browser alert computation and no localStorage copy of alert state.

### 5. Corrections live in one place

`corrections.ts` is where raw values become display values. A value corrected on one surface and raw on another is a defect.

**⚠ PARTIALLY VIOLATED.** The corrections themselves do all live in `corrections.ts` — nothing reimplements one. But `KpiCards.tsx` imports `cohortShowRate` and `costPerLead` and applies them itself instead of going through the selector, so the correction is applied while the selector is bypassed. The value is right today and the routing is wrong, which is how the two drift apart later.

### 6. Role titles only

Never personal names anywhere in the UI.

### 7. Golden fixtures may not be edited to match the code

Golden records exist so a change cannot silently move a number. A fixture was once lowered from 74 to 48 to agree with a dataset missing a week. That inverts the control into a rubber stamp.

If a change moves a golden figure, fail loudly. Changing a fixture requires a recorded reason and retains the previous value — the same discipline as the restatement log, applied to the tests.

---

## Security

### Roles

Roles live in a separate `user_roles` table, never as a column on a profile a user can update. `has_role()` is `SECURITY DEFINER` with `search_path` pinned. Policies call the function; they never query `user_roles` directly, which would recurse.

If a user can change their own role, RLS is defeated.

**Verified, with one correction.** All nine `SECURITY DEFINER` functions pin `search_path=public`, no policy recurses, and `profiles` carries no role column. But the **grants are not restrictive**: `anon` and `authenticated` both hold `arwdDxtm` on every public table, and `has_table_privilege('authenticated','user_roles','UPDATE')` returns true. What stops a user granting themselves a role is that `user_roles` has a SELECT policy and no INSERT, UPDATE or DELETE policy — under RLS, absent policy means denied. Never read a `GRANT` in a migration as the protection; read the policy list.

### RLS

Every table has RLS enabled with at least one policy. RLS enabled with zero policies is a silent hole and reads as a critical finding on any scan.

Reads are property-scoped. Portfolio roles see everything; Property and Regional Managers see their assignments. Fact and ingest tables are client-readable but service-role writable only.

### Server actions

**There are no edge functions in this project.** No `supabase/functions/` directory exists and `supabase/config.toml` contains only `project_id`, so `verify_jwt` does not apply to anything. The server actions are TanStack Start `createServerFn` handlers in `src/lib/*.functions.ts`. The security property is the same and the reason is the same: they write through `client.server.ts`, which uses the service-role key, so **RLS does not constrain them**. Every server action must resolve the caller's role server-side and reject unauthorized callers explicitly.

The middleware `requireSupabaseAuth` verifies the JWT with `supabase.auth.getClaims(token)` and hands the handler `context.userId` plus a token-scoped `context.supabase`. Read `user_roles` through that before doing anything.

This was a real blocker: `ingestWorkbook` verified the JWT and nothing else, meaning any signed-in user could rewrite every fact table for all 14 properties. Its sibling `ratifyThresholdChange` checked roles correctly. **Resolved.** `ingestWorkbook` now admits only `marketing_director`, `director_pm` and `coo`, resolved from `user_roles` under the caller's own session, and says so in a comment.

Because these files ship to the client bundle, import the service-role client **dynamically inside the handler** — `const { supabaseAdmin } = await import("@/integrations/supabase/client.server")` — never at the top level. Both actions do this correctly.

**Never trust a role passed in a request body.**

### Column-level writes

Attribution and identity columns are set from `auth.uid()` and the caller's real role, never from client input. Granting UPDATE on all columns allowed a property manager to record an action attributed to the COO, and a ratifier to rewrite `proposed_by` while still satisfying the two-person check.

**Closed, but not with column grants.** No column-level ACL was ever added — `attacl` is null on every column, and the table grants remain wide open. The hole was closed with **triggers** instead, which is the stronger fix because it holds on every write path: `stamp_alert_attribution` and `enforce_alert_attribution` on `alert_events` force actor columns to `auth.uid()` and reject a role the account does not hold; `stamp_proposal_author` and `enforce_proposal_identity` make `proposed_by`, `proposed_role` and `rule_id` immutable after insert.

Two things remain. The triggers protect **who acted**, not **what the row says** — `status`, `evidence`, `threshold` and `headline` on `alert_events` are still writable by any in-scope user. And there are now two overlapping trigger pairs on `alert_events` doing near-identical work, added an hour apart; they do not conflict today but they are a maintenance hazard.

### PII

The source workbook contains real resident names, emails, phones and free-text notes. The deployed dataset is anonymized: synthetic names, `.invalid` domains, reserved 555 numbers, real structure and aggregates preserved.

**Never load real person-level data into the deployed database.**

---

## Testing

Tests belong in the repo and run in CI, not inside the build tool. A check that cannot fail a build is documentation, not a control.

**Status as of 4 September.** A test harness now exists: `vitest`, a `test` script, a `ci` script (`typecheck && test`) and three files under `src/lib/__tests__/` — `harness.test.ts`, `no-invented-data.test.ts`, `selector-guard.test.ts`. A GitHub Actions workflow runs install, typecheck, lint, test and build on pull requests and on pushes to `main`. The build job is currently **skipped**, because the application source is not in this repository (see `KNOWN-ISSUES.md`, item 7). It begins running the moment the source lands.

Note the caveat on rule 4: `selector-guard.test.ts` passes while the rule it guards is broken. A test in CI is a control only if it can fail for the reason you think it can.

Most untested logic is pure and needs no database: week-label parsing, property alias resolution, restatement diffing, junk-lead classification, the correction layer. Extract and unit-test these. **Still true — none of the three existing test files covers any of them.** All three are source-scanning or harness-wrapping tests, not unit tests of the logic.

Two need integration tests against the database: restatement-on-reload, and the role check on ingest. **Neither exists.**

**Known untested paths, in order of risk** — all still untested: restatement detection on a genuine re-load; unmapped names raising an exception rather than dropping rows; malformed week labels; partial failure mid-ingest with no transaction boundary; alert re-evaluation aging an existing alert rather than duplicating it.

Deduplication by file hash is necessary but not sufficient. The same leads were once loaded twice under two different hashes. **Addressed.** Ingest now also computes a `content_sha256` over the parsed rows themselves and rejects a workbook whose rows match an existing batch, whatever the file bytes. It is not covered by a test.

---

## Working with the Lovable agent

The agent builds; a separate reviewer checks. That separation has caught things three rounds running, and self-reports have been wrong every round.

**The failure mode to watch for:** when the agent hits something it cannot satisfy, it tends to produce something that looks right rather than reporting that it cannot. Synthetic data instead of "not computable." A fixture lowered instead of a missing week flagged. It is an optimizer closing gaps.

Ask for actual values, not "done." Ask it to disagree rather than comply silently. If it accepts every finding without argument, treat that as weak evidence too.

**On the round audited 4 September, the agent did well.** Five of the six substantive findings were genuinely fixed, and two were fixed better than asked: the synthetic series was deleted outright rather than patched, and the attribution hole was closed with triggers rather than the column grants that were suggested. It also volunteered fixes nobody requested — an RLS-aware property count on the Overview, a loading guard that stops the page rendering zeros, and a content fingerprint for duplicate detection. The failure mode described above did not recur on this round.

**⚠ The sync claim in this section is false.** The repository does **not** sync both ways on `main`. Nothing from Lovable has ever been pushed to `mpouliot816/marketingdashboard`; `main` did not exist until it was created by hand on 4 September, and the repo still contains no application source. Treat the Lovable project as the only source of truth for code until a real sync is confirmed, and read the code through the Lovable API rather than from a checkout. When the sync is switched on, coordinate first — there are already hand-made commits on `main` for it to reconcile with.

`AGENTS.md` in the Lovable project also warns against rewriting published history: no force-push, rebase, amend or squash of commits that are already pushed, because that rewrites history on Lovable's side. Rebasing your own unpushed work onto someone else's commits is fine.
