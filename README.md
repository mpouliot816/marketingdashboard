# Carbon Marketing Dashboard

Internal marketing performance and monitoring tool for Carbon Residential. Tracks the leasing funnel, paid spend, supply, alerts and forecasts across 14 properties.

**Live:** https://carbonamarketgoalseek.lovable.app
**Repo:** `mpouliot816/marketingdashboard`
**Editor:** https://lovable.dev/projects/1f0a5336-86ae-4986-99b5-dff6eafdf9a9

> **Verified against the code and the live database on 4 September 2026.**
> Corrections from that pass are marked in italics. The two that matter most:
> this repository does not currently contain the application source, and reads
> are property-scoped, so a Property Manager does not see the portfolio.

---

## Where the code actually is

**This repository holds the documentation, the audit and the CI workflow. It does not hold the application.** There is no `src/`, no `supabase/` and no `package.json` here.

The application lives in the Lovable project `carbonamarketgoalseek` (id `1f0a5336-86ae-4986-99b5-dff6eafdf9a9`). Read it through the Lovable editor or the Lovable API.

The GitHub sync described further down has never run. Do not assume a file path in these documents exists in a checkout of this repository — verify it against the Lovable project. See [docs/KNOWN-ISSUES.md](docs/KNOWN-ISSUES.md), item 7.

---

## What this replaces

A weekly Excel workbook, consolidated by hand from EliseAI, Google Ads, Meta Ads, Zillow and Apartments.com, rendered to static HTML and published to a personal GitHub Pages account. Monthly lease goals lived in browser localStorage with optional sync to a free third-party JSON bin.

Known accuracy of that process was roughly 85% against a >95% target. *Unverified — a historical claim about the previous process, with nothing in this system to check it against.*

This tool keeps the weekly upload but replaces everything downstream: the data now persists in a database with history, provenance and access control instead of being regenerated from scratch each week.

*One piece of the old process survives inside the new one. Threshold proposals and ratified rule versions are still stored in browser `localStorage`, on the "Proposal queue" and "Rules & history" tabs. Only the "Governance (Cloud)" tab writes to the database. See [docs/GOVERNANCE.md](docs/GOVERNANCE.md).*

---

## Who uses it

| Role | Reads | What they do here |
|---|---|---|
| Property Manager | **Own property only** | Acknowledges alerts for own property; proposes threshold changes |
| Regional Manager | **Own region only** | Acknowledges for own region; proposes |
| Marketing Analyst | Portfolio | Acknowledges; proposes |
| Marketing Director | Portfolio | Adjusts thresholds and advances changes for ratification; uploads the weekly workbook |
| Director of Property Management | Portfolio | Ratifies threshold changes; sets bounds; records decisions; uploads |
| COO | Portfolio | Ratifies; sets bounds; ratifies out-of-bounds changes and rule disablement; uploads |

*Two corrections to the earlier version of this table.*

***Reads are property-scoped, not portfolio-wide.** RLS restricts `fact_weekly`, `fact_lead`, `budgets`, `goals`, `specials`, `restatements` and `alert_events` through `can_act_on_property`. Confirmed against the live database: the Property Manager account is scoped to one property and the Regional Manager to four, and neither is portfolio-wide. The Overview reflects this — it retitles to "Your Properties" and warns how many properties fall outside your scope. Describing these roles as reading the portfolio understates the access control, which is the thing most likely to be graded.*

***Three roles can upload,** not one: Marketing Director, Director of Property Management and COO. The restriction is enforced server-side in `ingestWorkbook`, not in the UI.*

Marketing operates the tool. Property management leadership directs it. See [docs/GOVERNANCE.md](docs/GOVERNANCE.md).

---

## Documentation

| Doc | Read it when |
|---|---|
| [docs/KNOWN-ISSUES.md](docs/KNOWN-ISSUES.md) | **First.** Before you report a bug, or before you trust a figure |
| [docs/OPERATIONS.md](docs/OPERATIONS.md) | You run the weekly update, or something looks wrong |
| [docs/GOVERNANCE.md](docs/GOVERNANCE.md) | You want to change a threshold, or an alert won't clear |
| [docs/DATA-SOURCES.md](docs/DATA-SOURCES.md) | You need to know where a number comes from or why two numbers disagree |
| [docs/ENGINEERING.md](docs/ENGINEERING.md) | You are changing code |
| [CLAUDE.md](CLAUDE.md) | You are an agent working in this repository |
| [AUDIT.md](AUDIT.md) | You want the external audit that produced most of the rules above |

Every document was verified against the code on 4 September 2026. Corrections are marked inline with ⚠ or in italics. Several rules the documents state are **not currently honoured by the code** — those are recorded as findings rather than quietly removed. `docs/ENGINEERING.md` and `docs/KNOWN-ISSUES.md` carry the list.

---

## Stack

- TanStack Start (TypeScript, React 19), Tailwind, shadcn/ui
- Lovable Cloud (Supabase): Postgres, Auth
- Package manager: **bun**
- Built and deployed through Lovable

*There are **no Edge Functions**. No `supabase/functions/` directory exists and `supabase/config.toml` contains only `project_id`. The two server actions — `ingestWorkbook` and `ratifyThresholdChange` — are TanStack Start `createServerFn` handlers in `src/lib/*.functions.ts`. They carry the same risk as an edge function and for the same reason: they write through the service-role client, so RLS does not constrain them and each must resolve the caller's role itself.*

*The GitHub sync on `main` is **not working**. Nothing from Lovable has ever been pushed to this repository, and `main` did not exist until it was created by hand on 4 September 2026.*

## Running locally

**You cannot currently run this from a clone of this repository — the source is not here.** Work in the Lovable editor, or copy the project out of Lovable first.

Once the source is in the repository, it runs on bun:

```bash
git clone https://github.com/mpouliot816/marketingdashboard.git
cd marketingdashboard
bun install
bun run dev
```

Scripts: `dev`, `build`, `typecheck`, `lint`, `test`, `ci` (`typecheck && test`).

The app requires Cloud credentials to load data. ~~Changes pushed to `main` sync back to Lovable, and Lovable pushes its own changes to `main`.~~ *That is the intended arrangement and it is not in effect. When it is switched on, coordinate first — there are already hand-made commits on `main` for it to reconcile with, and `AGENTS.md` forbids rewriting pushed history because it destroys the project history on Lovable's side.*

## CI

A GitHub Actions workflow runs on pull requests and pushes to `main`: install, typecheck, lint, test, build, plus a check that fails a pull request when `src/` or `supabase/migrations/` changes without a corresponding `docs/` change. It warns rather than fails on pushes to `main`, because Lovable commits straight to the branch and never opens a pull request.

The build job currently **skips**, because there is no `package.json` here. It starts working the moment the source lands.

## Demo accounts

Six accounts exist for demonstration, one per role, all on `@carbondemo.app`. They read an anonymized dataset: synthetic names, `.invalid` email domains, reserved 555 phone numbers. Real aggregate figures, no real people.

*Verified: all 22,022 `fact_lead` rows carry synthetic identities and zero rows hold a non-`.invalid` email address.*

> ***The app is published publicly, and the shared demo password is printed on the sign-in page*** *next to a one-click sign-in button for every role, including COO. Anyone who finds the URL can read all 14 properties. The data is synthetic, so this is not a personal-data exposure — but every scope boundary described above is one click from being bypassed by an anonymous visitor. Set the project to private, or gate the demo buttons behind an environment flag, before showing this to anyone outside the team.*

**Do not load real resident data into the demo dataset.** The source workbook contains roughly 11,000 lead rows and 4,900 lost-lead rows with real names, emails, phone numbers and free-text notes. *Measured in the loaded dataset: 11,011 current lead rows and 4,673 lost-lead records.*

Person-level fields are replaced on the way in by `src/lib/synthetic-identity.ts`. The parser never reads a name, email or phone column from the workbook at all, so no identifier reaches the database even before anonymization.

---

## The one rule that explains most of the design

**A missing value is never rendered as zero.**

Missing, quarantined and unevaluable are distinct states, and each is shown as plainly as a good value. Most of the architecture follows from this: the correction layer, the Data Integrity page, the provenance requirement, the rule that an alert moves to Unevaluable rather than Cleared when its input disappears.

The reason is specific to this data. Several upstream feeds report zero when they are broken. `Application Started` has read zero for every property every week since February 2026 because the export is broken, not because nobody applies. A dashboard that renders that as 0% looks fine and is lying.

*Verified. The last week carrying a non-zero applications figure is 3–9 February 2026; every week from 10–16 February onward reads zero. The Unevaluable-not-Cleared rule is implemented, and `perf.speed_to_lead_declining` currently demonstrates it — its input was deleted rather than faked, so it sits Unevaluable instead of passing.*

***The rule is broken in two places, in opposite directions.*** *`selectMetric` suppresses applications only when* every *week in scope is dead, so the unfiltered dashboard returns 167 — real January figures plus 27 weeks of fake zeros — flagged as a good value. And the Overview hardcodes the string "Not captured", so that one panel can never show a number even after the feed is repaired. Both are recorded in [docs/KNOWN-ISSUES.md](docs/KNOWN-ISSUES.md).*
