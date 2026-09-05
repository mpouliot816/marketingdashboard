# Known issues and current state

**As of 4 September 2026.** Update this when the in-flight items land.

**Every claim below was re-verified against the code and the live database on
4 September 2026.** Corrections from that pass are marked inline.

Read this before trusting a figure or reporting a bug.

---

## No longer in flight — verified

An external code audit found seven items. All seven were re-checked against the
code at commit `21e1bda9` and against the live database.

| # | Item | Severity | Verified status |
|---|---|---|---|
| 1 | `ingestWorkbook` had no role check — any signed-in user could rewrite every fact table for all 14 properties | Blocker | **Resolved.** The handler now reads `user_roles` through the caller's own session and admits only `marketing_director`, `director_pm`, `coo`. |
| 2 | Weekly speed-to-lead trend was synthetically generated and drove a live alert | Blocker | **Resolved.** `speedWeeks`, the hash function and the hardcoded `DECLINING_3W` set are deleted from `elise.ts`. Replaced by `SPEED_WEEKLY_UNAVAILABLE`; the rule now evaluates as Unevaluable, never Cleared. |
| 3 | `alert_events` and `alert_rule_proposals` granted UPDATE on all columns, allowing forged attribution | High | **Closed, but not the way this line says.** No column grants were added — see "Corrections to this document" below. Four triggers now stamp attribution from `auth.uid()` and reject a role the account does not hold. The proposer is immutable. Content columns (`status`, `evidence`, `threshold`, `headline`) are still writable in scope. |
| 4 | Golden fixture `mtdLeases` was edited from 74 down to 48 to match a short dataset | High | **Resolved.** Restored to 74, with a `FIXTURE_AMENDMENTS` log recording both the original unlogged change and its reversal, plus an `unloggedAmendments()` guard. It is expected to fail until the Aug 24–30 week loads. That failure is the control working. |
| 5 | Spend total fails its own check: $271,530.75 against $255,000 ± 9,000 | Medium | **Resolved, narrowly.** Now $263,380.62 — inside the band by about $620. It moved because the supersession fix removed duplicate spend rows, not because spend was restated. Treat the margin as thin. |
| 6 | No test harness that can fail a build | Medium | **Resolved.** `vitest`, a `test` script, a `ci` script and three test files under `src/lib/__tests__/`. CI added separately — see below. |
| 7 | Repository was empty — since resolved, sync confirmed | Resolved | **Wrong. Still open.** See "Corrections to this document". |

Item 2 also triggered a codebase-wide sweep for any other generated or placeholder data feeding a display or alert. `no-invented-data.test.ts` now enforces it: `Math.random`, `jitter` and hash-derived figures are banned from `src/lib`, with one declared exemption for identifier generation in `ratification.ts`. **The sweep did not cover the seeded constant tables in `elise.ts`** — tour completeness, speed to lead, first-contact splits and negative durations are all hardcoded and render as live portfolio figures.

---

## Corrections to this document

Found on the 4 September verification pass. The claims above were written from conversation, not from the code.

**Item 7 was wrong. The repository does not contain the application.** `mpouliot816/marketingdashboard` holds `AUDIT.md`, `CLAUDE.md`, `docs/` and a CI workflow — and no application source. There is no `src/`, no `supabase/`, no `package.json`. The Lovable README states "every change made in Lovable is committed straight to this repository"; that has not happened. `main` did not exist until it was created by hand on 4 September. Sync is **not** confirmed. Anything in these documents that assumes you can read the code from this repo is currently false — the code is in the Lovable project `carbonamarketgoalseek`.

**"has_role and all four other SECURITY DEFINER functions" undercounts.** There are now **nine** `SECURITY DEFINER` functions, not five: `has_role`, `is_portfolio_wide`, `can_act_on_property`, `handle_new_user`, `enforce_two_person_rule`, `enforce_alert_attribution`, `enforce_proposal_identity`, `stamp_alert_attribution`, `stamp_proposal_author`. All nine pin `search_path=public`. The claim holds; the count was stale.

**Table grants are not restrictive.** `anon` and `authenticated` both hold `arwdDxtm` — including UPDATE and DELETE — on every public table, and `has_table_privilege('authenticated','user_roles','UPDATE')` returns true. RLS is the only thing preventing a user granting themselves a role, and it does prevent it: `user_roles` has a SELECT policy and no other. Do not read a `GRANT` in a migration as protection.

---

## Confirmed working

Re-verified against the live database on 4 September 2026. Each of these was re-run, not carried over.

- All 16 tables have RLS enabled, each with at least one policy. No table open, none enabled with zero policies.
- Roles in `user_roles`, readable only by owner, no write policy. (The *grant* is not restrictive — see "Corrections" above. The absent policy is what holds.)
- `has_role` and all **eight** other `SECURITY DEFINER` functions have `search_path=public` pinned. No policy recurses.
- Property scoping genuinely enforced. `fact_weekly`, `fact_lead`, `budgets`, `goals`, `specials`, `restatements` and `alert_events` all read through `can_act_on_property`.
- Two-person rule enforced by CHECK constraint and trigger, not UI — **on the database path only.** The proposal queue most of the UI uses does not reach the database at all. See "The governance flow is two systems" below.
- No PII: all 22,022 `fact_lead` rows synthetic (11,011 current, 11,011 superseded), zero non-`.invalid` emails, reserved 555 numbers.
- Week of 17–23 August reconciles exactly: 867 leads / 229 tours booked / 86 attended / 16 leases.
- July 27 tours-booked quarantine correctly reversed — the database holds 167 for that week, matching the workbook.
- Week of 10–16 August confirmed present (891 / 203 / 93 / 15). Exact.

---

## Open data issues

**Week of 24–30 August is still not in the database.** Data stops at 17 August. Expected: 753 leads / 191 tours booked / 85 attended / 26 leases, $13,732 spend, 74 leases month-to-date against a 96 goal. Re-checked 4 September: the week is absent. August month-to-date currently reads 48 (17 + 15 + 16 for the weeks of Aug 3, 10 and 17); the missing week's 26 is exactly the difference to 74.

**`fact_lead` row duplication is resolved.** ~~22,022 against approximately 11,000 in source.~~ The table holds 22,022 rows, but only **11,011 are current** — the other 11,011 carry a `valid_to` and are correctly superseded. `fact_weekly` is likewise balanced at 3,809 current and 3,809 superseded. The cause was fixed at source: ingest now keys supersession on every week the file touches, including spend-only and occupancy-only weeks, and a second `content_sha256` fingerprint rejects the same rows arriving under a different file hash.

**Tour counting rules inside Elise's Metrics by Community report are undocumented.** This is the largest open definition question and explains a consistent ~20% variance against Elise's raw calendar. *Unverified — no access to Elise from this repository.*

---

## Found on the 4 September verification pass

New, not previously recorded.

**The governance flow is two systems, and the visible one is `localStorage`.** `/alerts` has four tabs. "Proposal queue" and "Rules & history" run on `src/lib/ratification.ts`, which stores proposals under the browser key `carbon-alert-rule-proposals-v2` and writes ratified versions to `localStorage` through `saveRuleVersions()`. Its own header calls it an "Interim client store", and it carries the database constraints as a **string constant** named `PROPOSAL_DB_CONSTRAINTS`, "applied verbatim once Cloud is enabled". Cloud *is* enabled. The fourth tab, "Governance (Cloud)", is the real path and does use the database, the CHECK constraint, the triggers and the `ratifyThresholdChange` server action. Consequences: a threshold "ratified" in the client store changes alert evaluation **in that browser only**, never `alert_rules.threshold`; and the two-person rule on that path is a TypeScript `if`, not a database constraint. This directly contradicts `GOVERNANCE.md`.

**The single-selector rule is violated on the Overview, and the test that guards it cannot see the violation.** `KpiCards.tsx` computes `totals.spend / totals.leases` and two `pct(...)` ratios inline; `routes/_authenticated/index.tsx` computes cost per lease, spend per leasable unit and tour-to-lease inline. Neither calls `selectMetric`. `selector-guard.test.ts` passes anyway — its regex matches arithmetic over `ds.perf`-style raw rows, not arithmetic over an already-computed `Totals`. A green test is being read as a satisfied invariant.

**"Applications started" renders a hardcoded string.** On the Overview it is the literal `"Not captured"`, not derived from `detectApplicationsBreak`. It will still say that after the feed is repaired. Separately, `selectMetric`'s applications branch uses `scope.every(...)`, so a multi-week scope containing one good week returns a number that silently counts 27 dead weeks as zeros. It should be `scope.some(...)`.

**Ingest has no transaction boundary.** Supersession commits before the inserts, so a mid-run failure leaves the current view partly empty with a `load_batches` row claiming success. `fact_lead` supersession is also still unscoped — it retires every current lead row whatever weeks the workbook covers.

**Suppression's 30-day cap is not enforced on the write path.** `alerts.ts` → `suppress()` rejects anything over 30 days, but `alerts-cloud.ts` → `applyAction` writes `suppressed_until` straight to the database from `(a.days ?? 30)` without going through it, and no database constraint bounds the column. Today every caller passes 30, so the cap holds by convention.

**No CI existed until 4 September.** A workflow now runs install, typecheck, lint, test and build, plus a documentation-drift check that fails a pull request when `src/` or `supabase/migrations/` changes without a `docs/` change. It warns rather than fails on pushes to `main`, because the Lovable agent commits straight to the branch and never opens a pull request. The build job is currently skipped: there is no `package.json` in this repository (see item 7 above).

---

## Not built

| Item | Note |
|---|---|
| Live source feeds | Everything arrives via weekly workbook. Elise and Yardi figures are seeded snapshots and do not refresh. |
| Google Ads / Meta APIs | Deferred by decision |
| Zillow / Apartments.com / Yelp | No practical API. Scraping carries terms-of-service exposure on accounts you pay for. Yardi GL carries invoiced totals monthly. |
| Competitive rent comps | Fields defined, render "Not connected" |
| Slack or email alert delivery | In-app only, deliberately |
| Exception Log promotion | Deliberately deferred — untuned alerts should not enter another team's queue |
| Nightly CSV export | Specified, not built. `exportPropertyTable` and a CSV round-trip invariant exist in `metrics.ts` and the harness; there is no scheduled job. |
| CI | Added 4 September. The build job skips until the application source is in this repository. |

---

## Cosmetic

- Deployed URL is `carbonamarketgoalseek.lovable.app`. Intended: `carbon-marketing-dashboard`.
- The **Lovable project** is named `carbonamarketgoalseek` and its description is still "Where r the templates?" — both leftovers from a prior project. The **GitHub repository** is named `marketingdashboard`, which is fine.

---

## Deployment posture

Not cosmetic, and worth a decision before this is shown to anyone outside the team.

**The app is published publicly.** `publish_visibility` is `public` at `https://carbonamarketgoalseek.lovable.app`. The sign-in page prints the shared demo password in plain text and offers a one-click sign-in button for all six roles, including COO. Anyone who finds the URL can read all 14 properties. The data behind it is synthetic, so this is not a personal-data exposure — but every RLS scope boundary is one click from being bypassed by an anonymous visitor. Set the project to private before submission, or gate the demo buttons behind an environment flag.

---

## Findings worth acting on

Surfaced during the build, unresolved as business matters rather than software:

**Core properties ran roughly 170% of year-to-date marketing budget** (about $251K against $148K, January–July) while approximately $196K of lease-up budget sat unspent at Hills at Hoover and Residences at Overlook.

**Paid spend runs against properties with no leasable inventory.** In one week, $4,089 across five properties produced zero leases, and three of those had zero units available to lease. Hills at Hoover took the most leads in the portfolio and signed nothing.

**One search campaign loses 51–88% of impression share to budget every week** at $6–12 per conversion, on a $20–35 daily cap.

**On-site promotions budget is funded and unused** — roughly $3,500 per property, spent at near zero.

**Six specials expired with no outcome recorded.** No basis exists to judge whether any of them work.
