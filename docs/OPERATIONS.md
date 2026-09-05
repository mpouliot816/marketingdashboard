# Operations

The weekly run, and what to do when something looks wrong.

> **Verified against the UI code on 4 September 2026.** Three things this
> document describes are **not built**: the restatement log, the unmapped-names
> panel, and the upload summary. They are marked ⚠ NOT BUILT where they appear.
> Everything else on the Data Integrity page and the alert cards is there as
> described.

---

## The weekly update

Runs once a week, after the reporting week closes on Sunday. Owner: Marketing Analyst prepares, Marketing Director uploads.

### 1. Build the workbook

Source paths, per sheet. These come from the existing marketing SOP and have not changed.

| Sheet | Source |
|---|---|
| Raw Data | EliseAI → Reports → Leasing → **Metrics by Community**. Secondary grouping: Marketing Source. Show Full Metrics: True |
| Lead Data | EliseAI → Reports → Leasing → **Portfolio Summary** → Leads Funnel |
| Lost Leads | EliseAI, lost/cancelled prospects for the period |
| Campaigns - Gads | Google Ads, campaign view, matched to the reporting week |
| Expense | Google Ads, Meta Ads, Zillow, Apartments.com — same date range across all four |
| Specials | Manual. Property, offer, start date, end date |
| Lead Goals | Monthly lease goal × trailing 4-week close rate |
| Traffic | Website/ILS traffic for the period |

**Before you upload, check these four things.** Each has caused a real defect:

1. **Property names match across sheets.** Raw Data, Expense and Lead Goals have historically used three different spellings. "Claxton Pecan" vs "Claxton". "Residence at the Overlook" vs "Residences at Overlook". Mismatched names do not merge, they fail — the ingest raises an unmapped-name exception rather than silently dropping the row.
2. **Week labels are identical across sheets.** Raw Data has used "August 24 to 30" while Expense used "August 24 - 30". Pick one form and use it everywhere.
3. **Every property with leads has a spend row, and vice versa.** Residences at Overlook once had 83 leads and no spend row; RATO had $1,228 of spend and no leads row. Neither can be evaluated on cost per lead.
4. **No placeholder or malformed dates.** Three Specials rows were dated "August 1. 2026" with a period instead of a comma.

### 2. Upload

Sign in as Marketing Director. Upload the workbook. The server parses it, writes a batch record, and returns a summary: rows parsed, weeks affected, restatements detected, alerts fired, exceptions raised.

*Upload is now restricted server-side to the Marketing Director, the Director of Property Management and the COO. Any other role gets an explicit refusal naming the roles it holds.*

~~**Read that summary.** It is the only place restatements surface at upload time.~~

**⚠ NOT BUILT.** The server action does return that summary, and `IngestSummary` carries every field named above. `UploadCard.tsx` discards it — on success it renders only `Loaded {filename}`. Rows parsed, weeks affected, restatements detected, alerts fired and exceptions raised are computed, returned, and never shown to anyone.

Combined with the missing restatement panel below, **restatements currently surface nowhere in the UI.** The `restatements` table holds 8 rows that no screen displays. Until this is fixed, read them with a query.

### 3. Check the Data Integrity page

Before looking at any performance number:

- **Dashboard Confidence** score in the header. If it dropped, the page explains why. *Built — with the full component/weight/contribution breakdown.*
- **Coverage** — any missing weeks or properties. *Built, as a property × week matrix with a dataset switcher.*
- **Freshness** — load lag over 3 days flags. *Built, and the 3-day threshold is correct.*
- **Restatements** — any figure that changed versus what was previously reported. A change above 10% on a closed week is an alert, not a footnote. **⚠ NOT BUILT.** There is no restatement panel on this page. The `data.restatement_closed_week` rule exists and is locked at 10%, so the alert fires; the log it refers to does not exist.
- **Quarantined values** — anything the correction layer suppressed this week. *Built, with reported value, expected value, deviation and reason.*
- **Unmapped names** — must be resolved, not ignored. **⚠ NOT BUILT** as a panel. Unmapped labels are captured into `load_batches.exceptions` at ingest and the `data.unmapped_property` rule fires, but nothing on this page lists them.

*Also on the page and not listed above: disposition completeness, verification status, tour status completeness, first-contact channel split, negative durations, channel normalization and goal calibration.*

### 4. Review alerts

Sorted by state, then severity, then property. Work Escalated first. Every card shows the evidence, the age, the owning role, and what will actually clear it.

---

## What to do when something looks wrong

### A number changed from last week and nobody changed anything

~~Check the restatement log on the Data Integrity page.~~ **⚠ There is no restatement log in the UI.** The data exists — every figure that moves between batches writes a row to `restatements` with old value, new value and delta, and ingest is doing this correctly. Nothing renders it. Until a panel exists, query the table directly.

Restatements are normal — Elise attributes leads on first-touch date, so a lead arriving from an ILS this week can be reported under an earlier week. What matters is the size and whether it was expected.

### Two screens show different numbers

They should not. Every metric has one canonical selector and every surface calls it. If you see a divergence, that is a defect, not a display quirk. Report it with both screens and the filter you had applied.

**⚠ Two known divergences, so check these before reporting a new one.**

*The Overview KPI cards and the "Marketing efficiency" panel do not go through the canonical selector — they compute cost per lease, tour-to-lease and two ratios inline. See `ENGINEERING.md` rule 4.*

*Alert thresholds have two sources that have already drifted. `perf.tour_show_rate` reads **40%** from the code's `DEFAULT_RULES` and **45%** from `alert_rules` in the database. Server-side evaluation at ingest uses the database value; the alert card and the rules table display the code value. Every other rule currently agrees.*

### An alert won't go away

Check its clearing class on the card. Four kinds:

- **Self-clearing** — needs the metric inside threshold for two consecutive weeks. One good week is not enough, deliberately.
- **Evidence required** — will not clear because the condition stopped appearing. Missing data must actually load.
- **Decision required** — needs an attributed human decision with a reason.
- **Manual only** — budget overruns. The money is spent; acknowledge it.

### An alert says "Unevaluable"

The metric driving it can no longer be computed, usually because its source data is missing or quarantined. This is not the same as passing. It stays until the input returns.

### A tour show rate looks impossible

Rates above 100% happen because a tour booked on Friday is attended the following Monday, and the weekly ratio crosses that boundary. The cohort figure is the honest one.

*The behaviour is real and the app labels it correctly as a timing artifact. The two examples in the original text are not in the loaded data — neither The Benton nor Lakeside has a week where attended reaches booked. Ten property-weeks currently exceed 100%: Capri Palms 2 of 1 (13 Jul) and Claxton 4 of 2 (27 Apr) are the worst at 200%, and Westminster Club accounts for five of the ten. Quote none of them.*

### Something needs to stop firing right now

Suppress it: 30 days maximum, requires a reason, attributed to you, visible in the Suppressed group the whole time, and it auto-resumes. Suppression is single-person on purpose because it is temporary and loud. Changing a threshold is permanent and quiet, which is why that needs two people.

*The reason is required, attribution is stamped by a database trigger from your session, and expiry auto-resumes. The 30-day maximum is not enforced on the write path — the "Suppress 30d" button is the only caller and it always sends 30. See `GOVERNANCE.md`.*

---

## Escalation

Red alerts unacknowledged for 3 business days and Amber for 14 days move to the Escalated group at the top of the alerts view.

*Verified. `DEFAULT_ESCALATION = { redBusinessDays: 3, amberDays: 14 }`, and business days convert to calendar days rather than being counted loosely. Escalated is first in `GROUP_ORDER`, and each card states its own escalation clock.*

Alerts route to the role that can fix the condition, not to whoever noticed. A tour show-rate collapse goes to the Property Manager. A budget overrun goes to Marketing. A missing data load goes to Technology. If an alert is routed to you and the fix is not yours, say so on the card rather than acknowledging it.

---

## What is not automated yet

- **Live source feeds.** Elise, Yardi, Google Ads and Meta are not connected directly. Everything arrives through the weekly workbook.
- **Google Ads and Meta APIs.** Deferred.
- **Zillow, Apartments.com, Yelp.** No practical API. Spend arrives by invoice; Yardi's GL carries the invoiced totals monthly.
- **Competitive rent comps.** Fields exist and render "Not connected."
