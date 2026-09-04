# Audit — Carbon Marketing Dashboard

Date: 2026-09-04
Auditor: independent code read. No prior summary of this project was trusted.
Nothing was fixed. This is a findings-only pass.

---

## 0. What I audited against

### The git repository is empty

`/home/user/marketingdashboard` contains a `.git` directory and nothing else.

```
$ git log
fatal: your current branch 'claude/lovable-supabase-audit-sjtuux' does not have any commits yet
$ git fetch origin   # succeeds, returns nothing
```

and via the GitHub API, `list_branches` returns `[]` while reading the root path
returns:

```
409 Git Repository is empty
```

`mpouliot816/marketingdashboard` has **zero branches and zero commits**. The
application source is not in version control at all.

I therefore audited the live Lovable project **"Carbon Market"**
(`carbonamarketgoalseek`, project id `1f0a5336-86ae-4986-99b5-dff6eafdf9a9`,
tech stack `tanstack_start_ts_current`, last edited 2026-09-04) and its
attached Supabase project `xhuhpubewjcphibqwedp`, read through the Lovable and
database APIs.

**A note on citations.** The Lovable file API returns file contents without
line numbers, and there is no checkout to run `grep -n` against. Every finding
below therefore cites the **file path, the enclosing symbol, and a verbatim
quote**. Each quote is unique within its file, so `grep` locates it exactly.
Once the code is committed, line numbers follow directly.

### Repository map

| Thing | Location | Notes |
|---|---|---|
| Migrations | `supabase/migrations/` | 5 files, all dated 2026-09-04 |
| Edge functions | **none** | No `supabase/functions/` directory exists |
| `config.toml` | `supabase/config.toml` | One line: `project_id = "xhuhpubewjcphibqwedp"`. No `[functions]` block |
| Server actions | `src/lib/ingest.functions.ts`, `src/lib/governance.functions.ts` | TanStack `createServerFn`, not edge functions |
| Auth middleware | `src/integrations/supabase/auth-middleware.ts` (server), `auth-attacher.ts` (client) | Generated files |
| Routes | `src/routes/` | See below |
| Data access | `src/lib/cloud.ts` (the read), `src/lib/dataset-store.ts` (the hook) | |
| Selector layer | `src/lib/metrics.ts` | `selectMetric` is the documented canonical read |
| Correction layer | `src/lib/corrections.ts` | |
| Supabase clients | `client.ts` (publishable key), `client.server.ts` (service role) | Both generated |
| Package manager | **bun** (`bun.lock`, `bunfig.toml`) | |
| Scripts | `dev`, `build`, `build:dev`, `preview`, `lint`, `format` | **No `typecheck` script. No test runner.** |

Routes (all under `src/routes/`):

| Route | File | Guard |
|---|---|---|
| `/auth` | `auth.tsx` | public (sign-in page) |
| `/_authenticated` (layout) | `_authenticated/route.tsx` | `beforeLoad` + `ssr: false` |
| `/` | `_authenticated/index.tsx` | inherits |
| `/alerts` | `_authenticated/alerts.tsx` | inherits |
| `/budget` | `_authenticated/budget.tsx` | inherits |
| `/integrity` | `_authenticated/integrity.tsx` | inherits |
| `/predictive` | `_authenticated/predictive.tsx` | inherits |
| `/supply` | `_authenticated/supply.tsx` | inherits |
| `/system` | `_authenticated/system.tsx` | inherits |

**The Supabase anon key** comes from `import.meta.env.VITE_SUPABASE_PUBLISHABLE_KEY`
with a `process.env.SUPABASE_PUBLISHABLE_KEY` fallback, in
`src/integrations/supabase/client.ts` → `createSupabaseClient()`.

### What I could not do

- **I could not run `build`, `lint` or `typecheck`.** There is no local
  checkout, and outbound HTTPS from this container is blocked by the proxy, so
  I could neither install dependencies nor fetch the deployed bundle. Section 6
  is therefore a static read only. Weak positive evidence: the Lovable project
  reports `status: "completed"` and published successfully at 2026-09-04T22:35Z,
  so the last build did compile.
- **I could not confirm the contents of `src/data/dataset.json`.** Reasoning
  about it from the parser that produces it is recorded under PII below.

---

## BLOCKER

### B1 — `ingestWorkbook` performs no role check. Any signed-in user can overwrite the entire portfolio.

`src/lib/ingest.functions.ts` → `ingestWorkbook`

```ts
export const ingestWorkbook = createServerFn({ method: "POST" })
  .middleware([requireSupabaseAuth])
  .inputValidator((input: { fileName: string; base64: string }) => {
```

`requireSupabaseAuth` verifies the JWT and nothing else. The handler then reads
`context.userId` only to stamp `uploaded_by`. It never queries `user_roles`.

Contrast the sibling action, which does check
(`src/lib/governance.functions.ts` → `ratifyThresholdChange`):

```ts
const { data: roles } = await supabase.from("user_roles").select("role").eq("user_id", userId);
const held = new Set((roles ?? []).map((r) => r.role as string));
const isCoo = held.has("coo");
if (!isCoo && !held.has("director_pm")) {
```

Consequence: the demo `property_manager`, whose RLS scope is a single property
(`lakeside`), can call `ingestWorkbook` and rewrite `fact_weekly`, `fact_lead`,
`restatements`, `provenance` and `alert_events` for all 14 properties. The
handler runs under `supabaseAdmin` (service role), so RLS does not constrain it.

This directly defeats the graded criterion "roles whose permissions genuinely
differ": the most destructive write path in the application is not
role-differentiated. Any UI-side gating on the upload control is presentation,
not enforcement.

### B2 — The weekly speed-to-lead trend and the alert that fires on it are computed from manufactured numbers.

`src/lib/elise.ts` → `speedWeeks`

```ts
/** Deterministic weekly bucketing of the same event pull (no randomness at runtime). */
```

```ts
const DECLINING_3W = new Set(["claxton", "westminster", "benton"]);

function hash(s: string): number {
  let h = 2166136261;
  ...
}

export function speedWeeks(propertyId: string | null): SpeedWeek[] {
  const base = speedFor(propertyId)?.within15 ?? PORTFOLIO_SPEED.within15;
  const id = propertyId ?? "portfolio";
  const declining = DECLINING_3W.has(id);
  return SPEED_WEEKS.map((week, i) => {
    const jitter = (hash(`${id}|${week}`) - 0.5) * 6;
    const fromEnd = SPEED_WEEKS.length - 1 - i;
    const slide = declining && fromEnd <= 2 ? (2 - fromEnd) * -3.2 : 0;
    return { week, within15: Math.max(0, Math.min(100, base + jitter + slide)) };
  });
}
```

This is not a bucketing of any pull. It is a hardcoded base constant plus
hash-derived jitter, plus a downward slide applied to three properties named in
a literal set. `decliningThreeWeeks()` reads this synthetic series:

```ts
export function decliningThreeWeeks(propertyId: string): boolean {
  const ws = speedWeeks(propertyId).slice(-4).map((w) => w.within15);
  if (ws.length < 4) return false;
  return ws[1]! < ws[0]! && ws[2]! < ws[1]! && ws[3]! < ws[2]!;
}
```

So a "three consecutive weekly declines in the 15-minute response rate" alert
fires because Claxton, Westminster and Benton are written into a set — not
because anything was measured. The docstring asserting there is no randomness is
technically true and materially misleading: determinism is not measurement.

For an application whose entire premise is data integrity, and which will be
shown to a certification reviewer, presenting a fabricated series as an
observed one is the most serious finding in this audit.

---

## HIGH

### H1 — The certification's golden week (Aug 24–30) does not exist in the database, and the fixture was reconciled downward to hide it.

Live query against `fact_weekly` (current rows, excluding the `(spend)` and
`(occupancy)` grains) returns 33 weeks ending at **2026-08-17**. There is no
`2026-08-24` week.

Stated golden figures versus reality:

| Figure | Stated | Actual | Result |
|---|---|---|---|
| Aug 24–30 leads | 753 | — | **week absent** |
| Aug 24–30 tours booked | 191 | — | **week absent** |
| Aug 24–30 attended | 85 | — | **week absent** |
| Aug 24–30 leases | 26 | — | **week absent** |
| Aug 17–23 leads | 867 | 867 | pass |
| Aug 17–23 tours booked | 229 | 229 | pass |
| Aug 17–23 attended | 86 | 86 | pass (asserted indirectly, see H2) |
| Aug 17–23 leases | 16 | 16 | pass |
| August MTD leases | 74 | **48** | **fail** |
| August lease goal | 96 | 96 | pass |

The MTD arithmetic confirms the cause exactly: 17 + 15 + 16 = 48 for the weeks
of Aug 3, 10 and 17. Adding the missing week's 26 gives 74. The dataset is one
week short of the figures the certification will be checked against.

The fixture file was then adjusted to match the short dataset rather than the
stated golden record — `src/lib/golden.ts`:

```ts
export const GOLDEN_AUGUST_GOAL = { month: "August", leaseGoal: 96, mtdLeases: 48, frozenAt: "2026-08-25" };
```

`mtdLeases: 48`, not 74. A regression fixture rewritten to agree with the data
it is supposed to police no longer polices anything.

The seed file name confirms the vintage — `scripts/seed-cloud.ts`:

```ts
file_name: "Dashboard_Data_-_Aug_17_to_23.xlsx",
```

### H2 — A frozen golden fixture is currently failing.

`src/lib/golden.ts` → `GOLDEN`:

```ts
    id: "total.spend",
    label: "Full workbook marketing spend",
    metric: "spend",
    filters: { months: [], weeks: [], properties: [] },
    expected: 255000,
    tolerance: 9000,
```

Live total for `campaign = '(spend)'`, current rows: **$271,530.75**.
That is +$16,530.75 against a $9,000 tolerance — outside the band by nearly
double. `runGolden` would report `total.spend` as `fail` right now.

Also note the Aug 17–23 **tours attended** figure (86) is not asserted directly.
It is only implied by `anchor.showRate` (37.6% ≈ 86/229). Three of the four
anchor-week counts are frozen; attended is not.

Verified passing fixtures (live): `total.leads` 11,342 ✓, `total.leases` 462 ✓,
`shape.weeks` 33 ✓, `shape.properties` 14 ✓.

### H3 — There is no test harness. The "harness" is a runtime browser check.

`package.json` has no test runner in `dependencies` or `devDependencies` — no
vitest, no jest, no playwright — and no `test` or `typecheck` script:

```json
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "build:dev": "vite build --mode development",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier --write ."
  },
```

There are no `*.test.ts` / `*.spec.ts` files anywhere in the project.

`src/lib/harness.ts` is a genuine and unusually thorough set of twelve runtime
invariants (identity, additivity, monotonicity, commutativity, correction
consistency, alert evidence, temporal, CSV round-trip, provenance,
reconciliation, golden, single-selector). But it executes **in the browser**,
on ingest and on `/system`, and writes results to local storage. Nothing runs
in CI. Nothing fails a build. The assertions are real; the enforcement is not.

### H4 — The single-selector rule is documented, claimed, and broken on the main dashboard.

`src/lib/metrics.ts` states the rule:

```
 * Every metric has exactly ONE selector here. Overview KPIs, the property
 * table, trend charts, the predictive layer, alert evidence and CSV export
 * all read through these functions with the same Filters object. No
 * component recomputes a metric locally.
```

`src/components/dashboard/KpiCards.tsx` → `KpiCards` never calls `selectMetric`.
It takes raw `Totals` and does its own arithmetic:

```ts
  const costPerLease = totals.leases > 0 ? totals.spend / totals.leases : null;
```

```ts
      sub: `${fmtPct(pct(totals.toursBooked, totals.leads))} of leads`,
```

```ts
    { label: "Leases Signed", value: fmtInt(totals.leases), sub: `${fmtPct(pct(totals.leases, totals.leads))} lead-to-lease` },
```

(canonical equivalents exist: `costPerLease`, `leadToLease`.)

`src/routes/_authenticated/index.tsx` → `Dashboard` does the same in the
"Marketing efficiency" panel:

```tsx
              value={established.leases ? `$${(established.spend / established.leases).toFixed(2)}` : "—"}
```

```tsx
              value={totals.toursAttended ? `${((totals.leases / totals.toursAttended) * 100).toFixed(1)}%` : "—"}
```

```tsx
              value={supplyTotals.leasable ? `$${(totals.spend / supplyTotals.leasable).toFixed(0)}` : "—"}
```

The dev-mode guard that is supposed to catch this cannot — `src/lib/harness.ts`:

```ts
const RAW_ROW_ARITHMETIC =
  /(?:\b(?:ds|dataset|data)\.(?:perf|spend|occupancy|campaigns)\b\s*(?:\.reduce\(|\.filter\([^)]*\)\s*\.reduce\()|\bfor\s*\(\s*const\s+\w+\s+of\s+(?:ds|dataset|data)\.(?:perf|spend|occupancy|campaigns)\b)/;
```

The pattern only matches `ds.perf.reduce(...)` and `for (const x of ds.perf)`.
It does not match `totals.spend / totals.leases`. Worse, the scan is dev-only:

```ts
  if (!import.meta.env.DEV) return { scanned: 0, hits: [], available: false };
```

In production the invariant reports `not-evaluable` — never `fail`. So the
"single-selector rule" invariant passes while the rule is broken.

The `identity` invariant does not catch it either: it compares
`slice.totals[key]` against `selectMetric` for additive keys only, and both
sides derive from `sliceData`, so they agree trivially. No derived metric
(`costPerLease`, `cpl`, `leadToLease`) is compared across surfaces.

### H5 — `elise.ts` is a large hardcoded dataset rendered as live portfolio metrics.

`src/lib/elise.ts` contains six literal tables that never come from the
database and never change when a workbook is ingested:

```ts
const RAW_COMPLETENESS: [string, number, number][] = [
  ["claxton", 536, 308],
```

```ts
export const PORTFOLIO_COMPLETENESS = {
  tours: 3491,
  unmarked: 1735,
  marked: 3491 - 1735,
  shows: 919,
  noShows: 871,
```

```ts
const RAW_SPEED: [string, number, number, number, number, number][] = [
  ["claxton", 401, 33.7, 43.9, 66.1, 1_142_000],
```

```ts
export const PORTFOLIO_SPEED: SpeedToLead = {
  propertyId: "portfolio",
  propertyName: "Portfolio",
  events: 2876,
  within15: 43.5,
```

```ts
const RAW_FIRST_CONTACT: [string, number, number, number][] = [
  ["claxton", 178, 89, 134],
```

```ts
export const NEGATIVE_DURATIONS: NegativeDurationGroup[] = [
```

These are consumed on the live Overview page. `src/routes/_authenticated/index.tsx`
→ `SpeedToLeadSummary` renders `PORTFOLIO_SPEED` directly, and
`src/components/dashboard/KpiCards.tsx` prints
`PORTFOLIO_COMPLETENESS.markedShowRate` and `observationQualifier(null)` inside
the "Tour Show Rate (cohort)" card, mixing a database-derived cohort rate with a
hardcoded completeness qualifier in the same card.

The file is honest in its header ("Everything here is seeded from the pull
described below"), but the UI does not distinguish these from live figures.

### H6 — `alert_events` grants blanket UPDATE. Role attribution on alert actions is client-supplied and forgeable.

Migration `20260904202327_...sql`:

```sql
GRANT SELECT, UPDATE ON public.alert_events TO authenticated;
```

```sql
CREATE POLICY "alerts acknowledged within scope" ON public.alert_events FOR UPDATE TO authenticated
  USING (public.can_act_on_property(auth.uid(), property_id))
  WITH CHECK (public.can_act_on_property(auth.uid(), property_id));
```

The comment above it says "Property-scoped acknowledgement", but the policy
constrains **only the property**, not the columns and not the role. Any
in-scope user may write every column.

The client duly supplies the role and the actor —
`src/lib/alerts-cloud.ts` → `applyAction`:

```ts
  const appRole = appRoleFromTitle(a.role);
  const { data: user } = await supabase.auth.getUser();
  const uid = user.user?.id ?? null;

  const patch =
    a.kind === "acknowledge"
      ? { status: "Acknowledged", acknowledged_at: now, acknowledgement_note: a.note, acknowledged_role: appRole, acknowledged_by: uid }
```

Nothing server-side checks that the caller holds `appRole`, and nothing pins
`acknowledged_by` to `auth.uid()`. A `property_manager` acting inside their own
scope can:

- write `acknowledged_role: 'coo'` or `decision_role: 'coo'`,
- set `acknowledged_by` / `decided_by` to another user's UUID,
- set `status: 'Cleared'` with any `decision_outcome`,
- set `suppressed_until` arbitrarily far into the future,
- rewrite `headline`, `evidence`, `observed_value` and `threshold`.

The alert audit trail is therefore attacker-controlled within scope. A fix
needs either column grants (`GRANT UPDATE (status, acknowledged_at, ...)`) plus
`WITH CHECK (acknowledged_by = auth.uid() AND has_role(auth.uid(), acknowledged_role))`,
or routing the action through a server function as ratification already is.

### H7 — The proposal UPDATE policies let the governance record be rewritten, including around the two-person rule.

Migration `20260904202327_...sql`:

```sql
CREATE POLICY "proposals reviewed by marketing director" ON public.alert_rule_proposals FOR UPDATE TO authenticated
  USING (public.has_role(auth.uid(),'marketing_director'))
  WITH CHECK (public.has_role(auth.uid(),'marketing_director') AND status IN ('Accepted','Declined','Expired') AND ratified_by IS NULL);
```

```sql
CREATE POLICY "proposals ratified by pm leadership" ON public.alert_rule_proposals FOR UPDATE TO authenticated
  USING (public.has_role(auth.uid(),'director_pm') OR public.has_role(auth.uid(),'coo'))
  WITH CHECK (
    (public.has_role(auth.uid(),'director_pm') OR public.has_role(auth.uid(),'coo'))
    AND status IN ('Ratified','Rejected')
    AND (ratified_by IS NULL OR ratified_by = auth.uid())
    AND (status <> 'Ratified' OR (ratified_by IS NOT NULL AND ratified_by <> proposed_by))
```

Neither policy pins `proposed_by`, `proposed_value` or `rule_id` to their
existing values. Three consequences:

1. A `director_pm` or `coo` ratifying directly through PostgREST can set
   `proposed_by` to a third party's UUID in the same statement. The
   `no_self_ratification` CHECK and the `enforce_two_person_rule` trigger both
   compare `NEW.ratified_by <> NEW.proposed_by`, so both pass. The two-person
   rule is satisfied against a rewritten proposer.
2. A `marketing_director` can move a `Ratified` proposal back to `Accepted`
   with `ratified_by = NULL`, erasing the ratification record.
3. A `marketing_director` can change `proposed_value` on any pending proposal.

The impact is bounded — `alert_rules` and `alert_rule_versions` have no client
write policy, so the **live threshold does not move** by this route (see P7).
The damage is to the audit trail, which is precisely what the two-person rule
exists to produce.

Related, on insert — `src/components/GovernanceQueue.tsx` → `propose`:

```ts
        proposed_by: viewer.user.id,
        proposed_role: viewer.appRole as never,
```

The INSERT policy checks `proposed_by = auth.uid()` and `has_role(...)` for the
four proposing roles, but never checks that `proposed_role` is a role the caller
holds. A `property_manager` can raise a proposal stamped `proposed_role: 'coo'`,
and the queue renders it as such.

### H8 — Ingest supersedes every lead row in the table regardless of the workbook's week coverage.

`src/lib/ingest.functions.ts` → `ingestWorkbook`:

```ts
    await supabaseAdmin.from("fact_weekly").update({ valid_to: now }).is("valid_to", null).in("week", weeks);
    await supabaseAdmin.from("fact_lead").update({ valid_to: now }).is("valid_to", null);
```

`fact_weekly` is correctly scoped with `.in("week", weeks)`. `fact_lead` has no
week filter and no property filter. Uploading a workbook covering a single week
retires **every current lead row in the portfolio** and replaces it with only
that week's leads. History is not deleted (it is superseded, so `valid_to` is
set), but the current view silently loses everything outside the upload.

Current state is symmetric only by luck: both batches carried the same 11,011
leads, so `fact_lead` shows 11,011 current and 11,011 superseded.

### H9 — Ingest has no transaction boundary, and supersession happens before the insert.

Same handler. The order is: insert `load_batches` → compute restatements →
supersede → insert new rows.

```ts
    const insertChunks = async (table: string, rows: Record<string, unknown>[], size = 500) => {
      for (let i = 0; i < rows.length; i += size) {
        const { error } = await supabaseAdmin.from(table as never).insert(rows.slice(i, i + size) as never);
        if (error) throw new Error(`${table}: ${error.message}`);
      }
    };
```

Every write is a separate PostgREST call. There is no transaction and no
compensating rollback. A failure in `insertChunks` throws out of the handler
after the supersession has already committed, leaving: a `load_batches` row
claiming success, all prior rows marked superseded, and only part of the new
data inserted. The dashboard then renders a partially-empty portfolio.

`ratifyThresholdChange` has the same shape, and additionally ignores one error:

```ts
    await supabaseAdmin.from("alert_rules").update({ threshold: value }).eq("id", rule.id);
```

No `error` is captured. A failure here leaves an `alert_rule_versions` row and a
`Ratified` proposal recording a threshold change that never took effect.

### H10 — The deployed app is published publicly with six one-click sign-ins and the password printed on the page.

`src/lib/demo-accounts.ts`:

```ts
export const DEMO_PASSWORD = "CarbonDemo2026!";
```

`src/routes/auth.tsx` → `AuthPage` renders it and wires a button per role:

```tsx
            One per role, shared password <span className="num">{DEMO_PASSWORD}</span>. Permissions genuinely differ:
```

```tsx
                      onClick={() => void signIn(a.email, DEMO_PASSWORD)}
```

The Lovable project reports `is_published: true` with
`publish_visibility: "public"` at `https://carbonamarketgoalseek.lovable.app`.

For a certification demo, published credentials are defensible — a reviewer has
to get in. What is not defensible is that this is **public on the open
internet**, so anyone may sign in as COO and read all 14 properties. Every RLS
scope boundary in the system is one button click away from being bypassed by an
anonymous visitor.

The data behind it is synthetic (see P10), so this is not a personal-data
breach. Recommend switching `publish_visibility` to private before submission,
or gating the demo buttons behind an environment flag.

---

## MEDIUM

### M1 — "Applications started" renders as a number that silently includes 25 dead weeks.

The requirement is that a missing value never renders as zero.
`src/lib/corrections.ts` handles the single-week case correctly, but
`src/lib/metrics.ts` → `derive` does not handle mixed scopes:

```ts
    case "applications": {
      const brk = detectApplicationsBreak(ds);
      const scope = activeWeeks(ds, f);
      const dead = scope.every((w) => brk.weeks.includes(w));
      return dead ? notCaptured("applicationsStarted feed dead since mid-Feb 2026 — not zero") : ok(t.applications);
    }
```

`every` means the value is suppressed only when **all** weeks in scope are dead.
The default dashboard view has no week filter, so the scope spans Jan–Aug. One
good week makes `dead` false and the selector returns `ok(t.applications)`.

Live data confirms the shape: `applications` is non-zero only for the six weeks
2026-01-04 → 2026-02-08 (167 in total) and exactly 0 for the 27 weeks after.
So the unfiltered selector returns `ok(167)` — a number, with `status: "ok"`,
which is 167 real applications plus 27 weeks of fake zeros presented as data.

The harness check for this cannot catch it — `src/lib/harness.ts` →
`correctionConsistency`:

```ts
  if (appsAll.value === 0)
    failures.push({ metric: "applications", period: "all weeks", delta: "0", detail: "dead feed rendered as a zero total" });
```

It tests for exactly `0`. The actual value is 167, so the check passes.

The correct condition is `scope.some(...)`, not `scope.every(...)`.

### M2 — The Overview hardcodes the string "Not captured" instead of deriving it.

`src/routes/_authenticated/index.tsx` → `Dashboard`:

```tsx
            <Stat
              label="Applications started"
              value="Not captured"
              hint={applicationsBreakNote(detectApplicationsBreak(dataset))}
            />
```

The label is right today and right for the wrong reason. It will still read
"Not captured" after the upstream feed is repaired, because it is a literal.
This is the mirror image of M1: one surface can never show a number, the other
can never show "Not captured".

### M3 — A harness invariant is written so that it can never fail.

`src/lib/harness.ts` → `temporal`:

```ts
  const ages = readAlertAges();
  const agesAfter = readAlertAges();
  if (JSON.stringify(ages) !== JSON.stringify(agesAfter))
    failures.push({ metric: "alertAge", period: "—", delta: "—", detail: "age counters on already-fired alerts changed during re-evaluation" });
```

The same function is called twice with nothing in between. The comparison is
tautological. Additionally `readAlertAges` reads a local-storage key that the
Cloud migration abandoned:

```ts
    const raw = window.localStorage.getItem("carbon-alert-events-v1");
```

`src/lib/alerts-cloud.ts` states "there is no in-browser alert computation and
no localStorage copy", so this returns `{}` always. The invariant reports `pass`
and contributes to the confidence score while testing nothing.

### M4 — `matchProperty` fuzzy-matches on prefixes, contradicting its own contract.

`src/lib/workbook.ts` → `parseWorkbook` docstring:

```
 * Parses the weekly workbook. Unmapped property labels are pushed onto
 * `exceptions` rather than silently dropped — a name nobody has mapped is a
 * human exception, never a fuzzy match and never a discarded row.
```

`src/lib/workbook.ts` → `matchProperty`:

```ts
  for (const [alias, id] of ALIAS_MAP) {
    if (key.startsWith(alias) || alias.startsWith(key.replace(/\.+$/, ""))) return id;
  }
```

This is a fuzzy match, and it is bidirectional. `alias.startsWith(key)` means a
short or truncated cell value matches the first alias that begins with it, in
Map insertion order. `"the"` resolves to `benton` via
`"the benton apartment homes"`. `"l"` resolves to `lamar`. A truncated export
cell silently attributes rows to the wrong property. The alias list already
carries `"lakeview and lakeside a..."`, which shows the team has hit truncation
in practice.

### M5 — Unmapped names are recorded as exceptions on some sheets and dropped in silence on others.

`src/lib/workbook.ts` → `parseWorkbook`. "Raw Data", "Expense" and "Lead Data"
call `note(...)` before skipping. Three sheets do not:

```ts
  for (const r of sheet(wb, "Occupancy Data")) {
    const property = matchProperty(r["Properties"] ?? r["Property"]);
    const week = weekKeyFromLabel(r["Date"] ?? r["Week"]);
    if (!property || !week) continue;
```

```ts
  for (const r of sheet(wb, "Lost Leads")) {
    const property = matchProperty(r["building_name"]);
    if (!property) continue;
```

```ts
  for (const r of sheet(wb, "Campaigns - Gads")) {
    const week = weekKeyFromLabel(r["Date"]);
    const property = matchProperty(r["Account"]);
    if (!week || !property) continue;
```

Occupancy, Lost Leads and Campaigns rows with an unmapped property vanish with
no exception recorded anywhere.

Separately, even where `note()` runs, the row is still dropped and the ingest
still returns success — `src/lib/ingest.functions.ts`:

```ts
        status: exceptions.length ? "complete_with_exceptions" : "complete",
```

The requirement is that an unmapped name raises an exception **rather than**
dropping rows. What happens is both: the row is dropped and a counter is
attached to the batch. Nothing fails and nothing blocks.

### M6 — The Expense `TOTAL` column is trusted whenever it is non-zero.

`src/lib/workbook.ts` → `parseWorkbook`:

```ts
    const total =
      num(r["TOTAL"]) ||
      num(r["GAds Cost"]) + num(r["Meta Ads Cost"]) + num(r["Zillow"]) + num(r["Apartments"]) + num(r["YELP"]);
```

For the current workbook this behaves correctly, because `TOTAL` is null on
every row, `num(null)` returns `0`, and `0 || x` falls through to the channel
sum. But the requirement is that `TOTAL` is **not trusted**. As written, any
future workbook with a populated `TOTAL` silently overrides the five-channel
sum, including a wrong one. The channel sum should be authoritative, with
`TOTAL` used at most as a cross-check that raises an exception on mismatch.

### M7 — `resolveViewer` fails open to a portfolio-wide role and picks arbitrarily among multiple roles.

`src/lib/auth.ts` → `resolveViewer`:

```ts
  const appRole = ((roles?.[0]?.role as AppRole | undefined) ?? "marketing_analyst") as AppRole;
```

Two problems. The query has no `ORDER BY`, so a user holding more than one role
gets whichever row Postgres returns first — non-deterministic across sessions.
And a user with **no** role row silently becomes `marketing_analyst`, which
`isPortfolioWide` treats as portfolio-wide.

The database still refuses the data (`is_portfolio_wide` consults `user_roles`
and returns false, so RLS returns no rows), so this fails safe on data. It fails
open on the interface: a roleless user is shown portfolio-wide affordances over
an empty dashboard. The default should be no role and an explicit refusal.

### M8 — Version numbering in `ratifyThresholdChange` is racy and unconstrained.

`src/lib/governance.functions.ts` → `ratifyThresholdChange`:

```ts
    const { count } = await supabaseAdmin
      .from("alert_rule_versions")
      .select("id", { count: "exact", head: true })
      .eq("rule_id", rule.id);
    const version = (count ?? 0) + 1;
```

Read-then-insert with no lock. `alert_rule_versions` has no
`UNIQUE (rule_id, version)` constraint, so two concurrent ratifications of
different proposals on the same rule both write version *n+1*. There is also no
re-check that the proposal is still `Pending` at write time, so two ratifiers
can both pass the earlier status gate.

### M9 — `goals.month` has no format constraint and already holds two incompatible formats.

Live `goals` table:

| month | rows | lease_goal |
|---|---|---|
| `April` | 8 | 59 |
| `May` | 8 | 59 |
| `June` | 9 | 59 |
| `August` | 14 | 96 |
| `2026-12` | 1 | 5 |

`monthOfWeek` returns English month names (`"August"`), and
`src/lib/metrics.ts` → `behindGoalProperties` compares
`goal.month.toLowerCase() !== month.toLowerCase()`. The `2026-12` row can never
match any view — it is dead data. The column is plain `text` with no CHECK.

### M10 — `isQuarantined` recomputes every anomaly in the portfolio on every call.

`src/lib/metrics.ts` → `isQuarantined`:

```ts
export function isQuarantined(ds: Dataset, propertyId: string, key: MetricKey, week: string): boolean {
  const q = buildQuarantine(allAnomalies(ds));
  return q.has(quarantineKey(propertyId, key as never, week));
}
```

`allAnomalies` runs `detectAnomalies` for the portfolio plus all 14 properties,
each scanning 6 metrics × 33 weeks with array slicing inside the loop. Nothing
is memoised. `src/lib/harness.ts` → `monotonicity` calls it inside a
week × property loop, so a single harness run recomputes the full anomaly set
on the order of a thousand times.

---

## LOW

### L1 — `.env` is committed and `.gitignore` does not exclude it.

`.env` is present in the project file listing. `.gitignore` covers `*.local`,
`.dev.vars` and build output, but has no `.env` entry.

The contents are not secret — publishable key, URL and project ref only:

```
SUPABASE_PUBLISHABLE_KEY="sb_publishable_1SXwvF4uDKWIMl2cZu34PA_5Yz5vleI"
```

No service-role key, no third-party key. So this is hygiene, not exposure. But
the pattern invites a real secret into the file later. Add `.env` to
`.gitignore` and ship `.env.example`.

### L2 — `.gitignore` does not exclude uploaded workbooks.

No `*.xlsx`, `*.xls` or `*.csv` entry. Section 4 asks specifically that
uploaded workbooks be ignored. Nothing currently stops a real workbook — the
one with 11,700 named residents — from being committed by accident.

### L3 — Mixed local-time and UTC date handling in the parser.

`src/lib/workbook.ts` → `isoDate` uses UTC for Excel serial numbers:

```ts
  if (typeof v === "number" && v > 20000 && v < 90000) {
    const d = new Date(Date.UTC(1899, 11, 30) + v * 86400000);
    return `${d.getUTCFullYear()}-${pad(d.getUTCMonth() + 1)}-${pad(d.getUTCDate())}`;
  }
```

and local time for `Date` objects:

```ts
  if (v instanceof Date) {
    if (Number.isNaN(v.getTime())) return null;
    return `${v.getFullYear()}-${pad(v.getMonth() + 1)}-${pad(v.getDate())}`;
  }
```

With `cellDates: true`, a workbook can yield either. In a negative-offset
timezone the same instant can land on different dates by route, shifting a lead
across a week boundary.

### L4 — `weekKeyFromLabel` defaults to 2026 for any label without a year.

`src/lib/workbook.ts`:

```ts
export function weekKeyFromLabel(label: unknown, fallbackYear = 2026): string | null {
```

Correct for the current workbook and a hardcoded assumption for the next one.
A 2027 workbook whose labels omit the year parses into 2026 and merges with
existing weeks.

### L5 — Alert re-fire clobbers `Cleared` and `Suppressed` states.

`src/lib/ingest.functions.ts` → `ingestWorkbook`:

```ts
          .update({ ...payload, status: prior.status === "Acknowledged" ? "Acknowledged" : a.status, age_weeks: ageWeeks } as never)
```

Only `Acknowledged` survives an ingest. An alert a director deliberately
`Cleared` or `Suppressed` reverts to `Open` on the next upload, discarding the
`decision_outcome` the governance flow recorded.

### L6 — CSV export writes suppressed values as empty cells.

`src/lib/metrics.ts` → `exportPropertyTable`:

```ts
      return c.value === null ? null : Number(c.value.toFixed(6));
```

Correctly refuses to export a zero, but an empty cell is indistinguishable from
a genuine blank in a spreadsheet. `"Not captured"` would carry the meaning that
`displayValue` gives it on screen.

---

## Recorded passes

These were checked against the live database and the source, and they hold.

**P1 — RLS is enabled on every table, and every table has at least one policy.**
Live query over `pg_class` / `pg_policy`, all 16 tables in `public`:

| Table | RLS | Policies | Operations |
|---|---|---|---|
| alert_events | ✅ | 2 | SELECT, UPDATE |
| alert_rule_proposals | ✅ | 4 | SELECT, INSERT, UPDATE |
| alert_rule_versions | ✅ | 1 | SELECT |
| alert_rules | ✅ | 1 | SELECT |
| budgets | ✅ | 3 | SELECT, INSERT, UPDATE |
| fact_lead | ✅ | 1 | SELECT |
| fact_weekly | ✅ | 1 | SELECT |
| goals | ✅ | 3 | SELECT, INSERT, UPDATE |
| load_batches | ✅ | 1 | SELECT |
| profiles | ✅ | 2 | SELECT, UPDATE |
| properties | ✅ | 1 | SELECT |
| provenance | ✅ | 1 | SELECT |
| restatements | ✅ | 1 | SELECT |
| specials | ✅ | 3 | SELECT, INSERT, UPDATE |
| user_properties | ✅ | 1 | SELECT |
| user_roles | ✅ | 1 | SELECT |

**No table has RLS enabled with zero policies. No table has RLS disabled.**

**P2 — Roles live in a separate table and a user cannot grant themselves one.**
`user_roles` is its own table keyed to `auth.users`. It has exactly one policy,
SELECT-only, restricted to the caller's own rows (migration `...222733`):

```sql
CREATE POLICY "own role readable" ON public.user_roles
  FOR SELECT TO authenticated
  USING (user_id = auth.uid());
```

There is no INSERT, UPDATE or DELETE policy, and the grant is `SELECT` only.
`profiles` — the table users *can* update — carries `display_name` and nothing
else. There is no role column anywhere a user can write.

**P3 — `has_role()` is `SECURITY DEFINER` with a pinned `search_path`.** Verified
live against `pg_proc`; all five `SECURITY DEFINER` functions pin it:

| Function | SECURITY DEFINER | search_path |
|---|---|---|
| `has_role` | yes | `search_path=public` |
| `is_portfolio_wide` | yes | `search_path=public` |
| `can_act_on_property` | yes | `search_path=public` |
| `handle_new_user` | yes | `search_path=public` |
| `enforce_two_person_rule` | yes | `search_path=public` |
| `touch_updated_at` | no | `search_path=public` |

No unpinned `SECURITY DEFINER` function exists. Execute is revoked from `anon`
and from `PUBLIC` on all of them, and the three trigger helpers are revoked from
`authenticated` too.

**P4 — No recursive RLS.** Every policy that needs a role calls `has_role`,
`is_portfolio_wide` or `can_act_on_property`. No policy queries `user_roles`
directly. The two policies on `user_roles` and `user_properties` filter on
`user_id = auth.uid()` only, so they cannot recurse.

**P5 — Property scoping is real, not decorative.** Migration `...222733`
replaced the permissive reads with scoped ones:

```sql
CREATE POLICY "fact_weekly readable within scope" ON public.fact_weekly
  FOR SELECT TO authenticated
  USING (public.can_act_on_property(auth.uid(), property_id));
```

The same applies to `fact_lead`, `budgets`, `goals`, `specials`, and — with
correct NULL handling for portfolio-level rows — `restatements` and
`alert_events`. The demo `property_manager` scoped to `lakeside` genuinely
cannot read the other 13 properties.

**P6 — `alert_rules` cannot be UPDATEd by any client, and `alert_rule_versions`
cannot be INSERTed.** Both tables have exactly one policy, SELECT, and grants of
`SELECT` only. The threshold can move only through
`ratifyThresholdChange`, which uses the service-role client.

**P7 — Fact and ingest tables are client-readable and service-role-writable
only.** `load_batches`, `fact_weekly`, `fact_lead`, `restatements` and
`provenance` each have exactly one SELECT policy and `GRANT SELECT ... TO
authenticated` / `GRANT ALL ... TO service_role`. No client write path exists.

**P8 — Goals and budgets are writable only by `director_pm` and `coo`.**

```sql
CREATE POLICY "goals written by pm leadership" ON public.goals FOR INSERT TO authenticated
  WITH CHECK (public.has_role(auth.uid(),'director_pm') OR public.has_role(auth.uid(),'coo'));
```

Matching UPDATE policies on both tables, with the same predicate in `USING` and
`WITH CHECK`.

**P9 — proposer ≠ ratifier is enforced in the database, twice.** Not only in
TypeScript. A CHECK constraint on the table:

```sql
  CONSTRAINT no_self_ratification CHECK (ratified_by IS NULL OR ratified_by <> proposed_by)
```

and a trigger that also covers the service-role path:

```sql
CREATE OR REPLACE FUNCTION public.enforce_two_person_rule()
...
    IF NEW.ratified_by = NEW.proposed_by THEN
      RAISE EXCEPTION 'Two-person rule: the proposer cannot ratify their own proposal';
    END IF;
```

```sql
CREATE TRIGGER proposals_two_person BEFORE INSERT OR UPDATE ON public.alert_rule_proposals
  FOR EACH ROW EXECUTE FUNCTION public.enforce_two_person_rule();
```

This is exactly what the certification tests for. See H7 for the caveat that the
comparison can be satisfied against a rewritten `proposed_by`.

**P10 — No personal data reaches the database.** `src/lib/workbook.ts` never
reads a name, email or phone column: `LeadRow` is
`{ property, source, channel, created, engagedAt, tourBooked, tourAttended, applied, leased }`.
Person fields are generated on write — `src/lib/synthetic-identity.ts` via
`syntheticIdentity(i)` in both the seed script and the ingest action.

Verified live against `fact_lead` (22,022 rows across both batches):

```
sample_name    Arden Alderman
sample_email   arden.alderman.00026@example.invalid
sample_phone   (555) 0100-0000
all_synthetic  true
```

Every row has `is_synthetic_identity = true`. Emails are on the reserved
`.invalid` TLD and phones are in the reserved `555-01xx` range. No real
identifier is present.

Because `parseWorkbook` drops person columns at parse time and
`scripts/build-dataset.ts` writes only its output, `src/data/dataset.json`
should contain no identifiers either — but I did not read that file, so this
part is inference from the parser, not direct verification. Worth a one-line
`grep -c '@' src/data/dataset.json` before submission.

**P11 — The service-role key never reaches the browser.** It is read only from
`process.env`, never `import.meta.env`, and is not `VITE_`-prefixed —
`src/integrations/supabase/client.server.ts`:

```ts
  const SUPABASE_SERVICE_ROLE_KEY = process.env['SUPABASE_SERVICE_ROLE_KEY'];
```

Both server actions import it *dynamically, inside the handler*, which is the
documented-safe pattern for a `*.functions.ts` file that otherwise ships to the
client:

```ts
    const { supabaseAdmin } = await import("@/integrations/supabase/client.server");
```

The only other referent is `scripts/seed-cloud.ts`, a Node script outside the
bundle. `.env` contains no service-role key. I could not fetch the built bundle
to confirm empirically (see section 0), so this is a source-level conclusion —
a `grep -r SERVICE_ROLE .output/` after a build would close it.

**P12 — Server actions verify the JWT server-side and never trust a role from
the request body.** `src/integrations/supabase/auth-middleware.ts` →
`requireSupabaseAuth`:

```ts
    const { data, error } = await supabase.auth.getClaims(token);
    if (error || !data?.claims) {
      throw new Error('Unauthorized: Invalid token');
    }
```

`userId` comes from `data.claims.sub`, and the client it builds is scoped to the
caller's bearer token, so RLS still applies to reads inside the action.
`ratifyThresholdChange` then reads roles from `user_roles` by that `userId`. No
role is ever accepted from the payload. (B1 is that one action forgot to *use*
this, not that the mechanism is weak.)

**P13 — CSRF protection is explicitly re-registered.** `src/start.ts`:

```ts
const csrfMiddleware = createCsrfMiddleware({
  filter: (ctx) => ctx.handlerType === "serverFn",
});
```

with a comment noting that defining `start.ts` at all opts out of the automatic
install. Easy thing to get wrong; it was not.

**P14 — `ratifyThresholdChange` independently checks all four required
conditions.** Caller role, caller ≠ proposer, rule not locked, value within
bounds — plus proposal status and expiry:

```ts
    if (proposal.proposed_by === userId) {
      return { ok: false, error: "Two-person rule: you cannot ratify a proposal you raised." };
    }
    if (!["Pending", "Accepted"].includes(proposal.status as string)) {
```

```ts
      const outOfBounds = (min !== null && value < min) || (max !== null && value > max);
      if (outOfBounds && !isCoo) {
```

**P15 — Ingest does the duplicate-hash check, and restatement detection genuinely
diffs.** `src/lib/ingest.functions.ts`:

```ts
    const sha = createHash("sha256").update(bytes).digest("hex");

    const { data: dupe } = await supabaseAdmin
      .from("load_batches")
      .select("id, file_name, uploaded_at")
      .eq("file_sha256", sha)
      .maybeSingle();
```

backed by `file_sha256 text NOT NULL UNIQUE`. Restatements diff against current
rows rather than assuming a first load:

```ts
      const prev = currentMap.get(key(p.week, p.property, p.campaign)) as Record<string, number> | undefined;
      if (!prev) continue;
```

The batch writes `load_batches` + `fact_weekly` + `fact_lead` + `restatements` +
`provenance`, as claimed.

**P16 — Append-only supersession works.** No `DELETE` and no destructive
`UPDATE` on fact rows exists in the ingest action or on any client path — the
only `UPDATE` sets `valid_to`. Live state confirms both directions:

| Table | Current | Superseded | Batches |
|---|---|---|---|
| fact_lead | 11,011 | 11,011 | 2 |
| fact_weekly | 3,817 | 3,801 | 2 |

(The one destructive delete in the codebase is in `scripts/seed-cloud.ts`, a
manual operator script, not a client or server path.)

**P17 — Week labels are never coerced to a date type.** `weekKeyFromLabel`
extracts month and day with a regex and rebuilds the key by hand:

```ts
  const m = label.match(/([A-Za-z]{3,9})\.?\s+(\d{1,2})/);
  if (!m) return null;
  const mi = monthIndex(m[1] ?? "");
```

There is no `new Date(label)` anywhere on the label path. `"August 10 - 16"`
yields `2026-08-10`, not 10 August 2016. The specific failure mode named in the
brief does not occur.

**P18 — One alert count, one source.** The header badge and the alerts page both
read Cloud through the same hook. `src/components/AppShell.tsx`:

```tsx
  // Same Cloud query as the alerts page — one alert count, one source.
  const { records } = useCloudAlerts();
  const openAlerts = badgeCount(buildDisplayAlerts(records));
```

`src/lib/alerts-cloud.ts` → `useCloudAlerts` is the only reader of
`alert_events`, and there is no client-side alert evaluation and no local-storage
copy. The previously reported divergence is genuinely resolved.

**P19 — There is no second fetch path for any metric.** `src/lib/cloud.ts` →
`loadDatasetFromCloud` is the sole database read for dashboard data, consumed
through one react-query key in `src/lib/dataset-store.ts`:

```ts
export const DATASET_QUERY_KEY = ["cloud-dataset"] as const;
```

No route defines a `loader`, and the authenticated layout sets `ssr: false`, so
no server loader competes with the client fetch. The SSR divergence risk the
brief asks about does not exist here.

**P20 — No views, and no storage buckets.** `information_schema.views` for
`public` returns zero rows, so there is no view exposing `auth.users` and no
`security_invoker` question to answer. `storage.buckets` is empty — no public
bucket, nothing to lock down.

**P21 — No hardcoded secrets.** No API key, token or connection string appears
anywhere in source. The only key material is the publishable key in `.env` and
the demo password (H10), which is a deliberate fixture rather than a leak.

**P22 — The three traced metrics agree end to end.** Leads, tour show rate and
leases all resolve `fact_weekly → loadDatasetFromCloud → sliceData → selectMetric`
with the same `Filters` shape on the property table, trend chart, alert evidence
and CSV export. The divergence is at the KPI-card and Overview-panel layer only
(H4), not in the selector chain itself.

**P23 — The authenticated route tree does not render or fetch before the redirect.**
`src/routes/_authenticated/route.tsx`:

```tsx
  ssr: false,
  beforeLoad: async () => {
    const { data, error } = await supabase.auth.getUser();
    if (error || !data.user) throw redirect({ to: "/auth" });
```

The guard is client-side, which would normally be the SSR concern the brief
raises — but `ssr: false` means the subtree never renders on the server, so no
loader runs and no data is fetched ahead of the redirect. `getUser()` also
validates against the auth server rather than trusting a stored session. RLS is
the real boundary in any case.

---

## Certification readiness

**Would this pass as it stands? No.**

Taking the four graded criteria one at a time:

**1. Domain-fitting schema on Cloud — PASS.** Sixteen tables that model the
actual business: property scoping, append-only facts with `valid_from`/`valid_to`,
restatements, provenance, alert rules with versions and proposals, goals and
budgets. This is a real domain model, not a CRUD demo. It is the strongest part
of the submission.

**2. Auth with roles whose permissions genuinely differ — FAIL.** The role
model is well designed and mostly well enforced (P2, P5, P8, P9). It fails on
one specific point: **`ingestWorkbook` has no role check (B1)**. The single most
destructive action in the application — the one that rewrites every fact table
for all 14 properties — is available to any authenticated user, including a
property manager scoped to one property. A reviewer who calls that server
function as the demo `property_manager` will find the permissions do not
differ where it matters most. H6 compounds this: alert acknowledgement accepts
the acting role as a client-supplied string, so a property manager can record
an action as the COO.

**3. RLS policies that actually enforce scope — PASS, with two holes to close.**
This is the heavily weighted criterion and it largely holds up. Every table has
RLS on with at least one policy; there is no enabled-with-zero-policies table
and no disabled table; roles are in their own table that no user can write;
`has_role` is `SECURITY DEFINER` with a pinned `search_path`; no policy recurses;
and the scoping is genuine rather than `USING (true)` theatre. The two holes are
both over-broad UPDATE grants rather than data exposure: `alert_events` allows
any in-scope user to write every column including the actor and the role (H6),
and the `alert_rule_proposals` policies do not pin `proposed_by`, which lets a
ratifier satisfy the two-person constraint against a proposer they just rewrote
(H7). Neither leaks data across a property boundary. Both are fixable with
column grants and tighter `WITH CHECK` clauses.

**4. At least one server-side action — PASS.** Two of them, and
`ratifyThresholdChange` is a genuinely good example: JWT verified server-side,
roles read from the database, all four business rules re-checked independently,
and the write performed with a service-role client that no browser can reach.
Note for the submission that there are **no edge functions** — the actions are
TanStack `createServerFn` handlers, so `verify_jwt` in `config.toml` is not
applicable (the file contains only `project_id`). If the certification expects
an edge function specifically, confirm that a framework server function counts.

### What sinks it beyond the four criteria

Two findings would damage the submission independently of the rubric:

- **B2** — the speed-to-lead weekly trend, and the alert that fires on three
  consecutive declines, are generated from a hardcoded property list plus hash
  jitter. In an application sold on data integrity, a reviewer who opens
  `elise.ts` finds a fabricated series driving a live alert.
- **H1** — the golden week the certification will check (Aug 24–30: 753 / 191 /
  85 / 26) is not in the database, and `GOLDEN_AUGUST_GOAL.mtdLeases` was set to
  48 to agree with the short dataset instead of the stated 74. A frozen fixture
  edited to match the data is not a regression test. Separately, `total.spend`
  is failing right now: $271,530.75 against $255,000 ± 9,000.

### Order I would fix in

1. **B1** — add a role check to `ingestWorkbook`. One query, and it closes the
   failing graded criterion.
2. **B2** — either wire speed-to-lead to real data or label the panel as seeded
   and remove `decliningThreeWeeks` from the alert engine.
3. **H1** — load the Aug 24–30 week and restore the golden fixtures to the
   stated figures; fix or re-baseline `total.spend` with a recorded reason.
4. **H6, H7** — column-level grants and tighter `WITH CHECK` on `alert_events`
   and `alert_rule_proposals`.
5. **H10** — set the deployment to private.
6. **H8, H9** — scope the `fact_lead` supersession to the uploaded weeks, and
   move ingest into a single transaction (a Postgres function, or an RPC that
   wraps the writes).
7. **H4, M1, M2** — route the KPI cards and the Overview panel through
   `selectMetric`, and change `scope.every` to `scope.some`.

Finally, and outside the rubric: **commit the code**. The repository this audit
was requested against is empty. Whatever the certification reviews, it will not
be what is in `mpouliot816/marketingdashboard` today.
