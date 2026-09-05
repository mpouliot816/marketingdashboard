# Data sources and definitions

Where every number comes from, why sources disagree, and which upstream feeds are broken.

> **Verified against the parser and the live database on 4 September 2026.**
> Claims about Elise's internals, Atlas coverage and Yardi GL could not be
> checked — none of those systems is reachable from this repository, and they
> are marked *unverified*. Three handling claims in the defects table below were
> wrong, and the "Corrected earlier finding" section contradicts the code.

---

## Sources

| System | Provides | Status |
|---|---|---|
| **EliseAI** | Leads, tours, applications, leases, lost reasons, response time, tour show/no-show | Via weekly workbook. Not connected live. |
| **Google Ads** | Campaign spend, conversions, impression share, optimization score | Via weekly workbook |
| **Meta Ads** | Spend | Via weekly workbook |
| **Zillow / Apartments.com / Yelp** | Spend | Invoice only. No practical API. |
| **Yardi Voyager** | Unit status, supply, occupancy, marketing GL actual vs budget, concession costs | Reference figures seeded; not connected live |
| **Atlas** | Parallel marketing dataset | Reconciliation only. Covers 10 of 14 properties. |

**The workbook is the source of record**, not Atlas. Atlas covers ten properties; the workbook covers fourteen. For the week of 17–23 August, Atlas held 505 leads against the workbook's 867. Hills at Hoover, Residences at Overlook, Santa Fe, Villa Siena and RATO have no Atlas marketing coverage. Never present a portfolio total that mixes the two.

*The workbook side is confirmed: 867 leads for that week, 14 properties. The Atlas side is **unverified** — Atlas is not reachable from this repository. The five uncovered properties are encoded in the seed script as `NO_ATLAS = {hoover, rato, santafe, siena, lakeside}` — note that it lists **Lakeside, not Residences at Overlook**, so either this paragraph or the seed script has the wrong fifth property. `properties.atlas_covered` in the database follows the seed script.*

---

## Why sources disagree

Four systems will never agree, and reconciling them to zero would destroy real information. Each pair carries a documented expected variance. The test is whether the gap **matches its explanation**, not whether it is zero.

### Elise dedupes on first-touch date

This is the largest single cause of variance and it is a real methodology, not an error. Elise attributes a lead to the date it first saw that person. A lead arriving from Zillow this week gets reported under an earlier week if Elise already has that email or phone. The marketing team cross-checks ILS lead exports against Elise and manually adds confirmed-missing leads to the workbook.

So the workbook is not an Elise export. It is an Elise export plus hand-added ILS leads, on first-touch dating.

### Two correct tour show rates that differ by 14 points

The workbook divides attended by all tours. Elise's calendar divides by tours with a recorded outcome. For June to August: 37.6% versus 51.3%. **Both are correct.** This is a definition variance. Label both, never average them, never let one silently replace the other.

### Elise counts more tours than the workbook

For 24–30 August, Elise's calendar holds 235 active tours; the workbook reports 191. Elise's raw calendar also contains blackout entries (staff availability blocks) and cancelled tours which are not tours at all. After excluding those, a roughly 20% gap remains, consistently, because the workbook draws from the pre-aggregated Metrics by Community report rather than raw events. **The counting rules inside that report are undocumented.** This is the largest open definition question.

### Workbook spend vs Yardi GL

Platform-reported spend versus invoiced spend, plus timing lag and a partly different property set. Roughly $255K over 33 weeks versus $251K January–July. Decompose into coverage, timing and definition; flag only unexplained residual.

*The $255K anchor is the golden fixture, with a ±$9,000 tolerance. The live database currently totals **$263,380.62** across 33 weeks — inside the band by about $620. It moved from $271,530.75 when the ingest fix removed duplicate spend rows, not because spend was restated. Treat the margin as thin: any further drift fails the fixture. The $251K Yardi GL figure is unverified — Yardi is not reachable from here.*

---

## Known upstream defects

Do not "fix" these by hiding them. They are real and they are handled deliberately.

| Defect | Effect | Handling |
|---|---|---|
| **`Application Started` = 0** for every property, every week since Feb 2026 | Two funnel stages unmeasurable | Renders "Not captured", never 0. Derived rates suppressed. |
| **`engagedToLead` inverted** at source (divides leads by engaged) | Every raw value exceeds 1.0; one property reads 12.5 | Inverted in the correction layer; values still above 1.0 flagged "Check source" |
| **Tour ratios cross week boundaries** | Weekly booked/attended exceeds 100% | Cohort computation is the honest one; weekly values above 100% flagged as timing artifacts |
| **Channel names fragment** — "Google Ads" / "google_ads" / "GAds" | One channel split three ways; ~$10,300 at Landings | Canonical channel map, with the fold shown to users |
| **~11% of dispositioned leads are junk** (spam, test, duplicate, wrong contact, vendor, resident, fraudulent) | Cost-per-lead understated | Net-of-junk is the headline; gross is the footnote. *Measured: 513 junk of 4,673 lost records = 11.0%, not ~9%.* |
| **~65% of leads have no recorded outcome** | Neither leased nor lost | Surfaced per property on Data Integrity, ranked worst first. *Panel confirmed built. The 65% figure is unverified — 428 of 11,011 current leads carry a lease date and 4,673 lost records exist, but the join between them is the thing this table calls unreliable, so the exact share depends on the method.* |
| **One property logs more lost records than leads** (302 vs 266), 57% "Unknown" | Disposition data unreliable there | Flagged; not corrected. *The over-logging flag is built and renders an "over-logged" badge. The specific 302/266 figures are unverified — the worst property now carries 1,183 lost records.* |
| **Property names differ across three sheets** in one workbook | Three properties silently failed to join to their goals | ⚠ **Partly wrong — see "Alias resolution is fuzzy" below.** |
| **Week labels inconsistent and date-coercible** | `"August 10 - 16"` parses as 10 August **2016** | ⚠ **Half right.** The parser never coerces a label to a date type — it extracts month and day with a regex and rebuilds the key, so the 2016 failure genuinely cannot happen. But there is **no canonical week list** and nothing is matched against one. A label with no year falls back to a hardcoded `2026`. |
| **Expense `TOTAL` column null on every row** | Totals absent | ⚠ **Not "always".** The parser reads `num(r["TOTAL"]) \|\| sum(five channels)`. Because `TOTAL` is null today it falls through to the channel sum and the result is correct — but any future workbook with a populated `TOTAL` silently overrides the channels, including a wrong one. |
| **Negative durations** in Elise events | A tour attended before it was booked | Quarantined, excluded from averages, counted on Data Integrity |
| **`first_contact` is a channel field**, not a boolean; ~32% "Unknown" | Attribution gap | Shown as a channel split. *Built. Seeded table; portfolio Unknown share renders on Data Integrity.* |

### ⚠ Alias resolution is fuzzy, and three sheets drop rows silently

There **is** a canonical alias table — `PROPERTIES` in `workbook.ts`, 14 properties with their known spellings — and it does the job the defects table credits it with. Two things in that row are wrong.

**It fuzzy-matches.** `matchProperty` falls through to a bidirectional prefix test: `key.startsWith(alias) || alias.startsWith(key)`. So a short or truncated cell value matches the first alias that begins with it, in map insertion order. `"the"` resolves to The Benton. `"l"` resolves to Lamar Lofts. A truncated export cell is silently attributed to the wrong property rather than raised. The `data.unmapped_property` rule describes itself as "Never fuzzy-matched", and `parseWorkbook`'s own docstring says "never a fuzzy match" — both contradict the function.

**Three sheets drop unmapped rows without recording anything.** Raw Data, Expense and Lead Data call `note()` before skipping, so an unmapped label lands in `load_batches.exceptions`. **Occupancy Data, Lost Leads and Campaigns - Gads do not** — those rows vanish with no exception, no alert and no trace.

**And an exception does not stop the load.** Where `note()` does run, the row is still dropped and ingest returns `ok: true` with `status: "complete_with_exceptions"`. The stated intent is that an unmapped name raises an exception *rather than* dropping the row. What happens is both.

### ⚠ "Corrected earlier finding" — this section contradicts the running code

The original text read:

> An earlier analysis reported that roughly half of all tours portfolio-wide had no recorded show/no-show status. **That was wrong.** It counted blackout entries and cancelled tours, neither of which can carry a show status. Excluding those, completeness is about 96%. Three properties are genuinely poor: Lamar (38% unmarked), Santa Fe (73%), Villa Siena (37%). Everywhere else is at or above 95%.

**The deployed app says the opposite, and none of the per-property figures match.** `elise.ts` holds the seeded completeness table, and the Data Integrity page renders it verbatim under the heading "Roughly half of all tours carry no recorded outcome":

| | This document | `elise.ts` / the live app |
|---|---|---|
| Portfolio unmarked | ~4% (96% complete) | **49.7%** — 1,735 unmarked of 3,491 tours |
| Lamar Lofts | 38% unmarked | **79.2%** — 206 of 260 |
| Santa Fe | 73% unmarked | **80.0%** — 12 of 15 |
| Villa Siena | 37% unmarked | **51.1%** — 23 of 45 |
| "Everywhere else at or above 95%" | — | **Nine of fourteen properties are below 70% complete** |

The `observationQualifier` printed inline next to every tour show rate on the Overview reads "computed on 50.3% of tours — 1,735 unrecorded". The `data.tour_status_completeness` rule is set to fire below 50%.

**I cannot tell which is right.** Elise is not reachable from this repository, so I cannot re-run the pull that would settle it. What is certain is that the document and the product currently tell a user two different things about the same metric, and the product's version is the one on screen.

**Resolve this before the next weekly meeting.** If the ~96% correction is right, `elise.ts` is seeded from the superseded pull and every completeness figure in the app is wrong, along with the alert that fires on it. If the seeded table is right, this paragraph is wrong and should go. One of the two is misleading somebody.

---

## Supply is the denominator

The funnel measured demand against nothing for its entire history. Five unit statuses are distinct and must never be collapsed into "vacant":

- **Vacant Unrented Ready** — leasable today. This is the marketing denominator.
- **Vacant Unrented Not Ready** — in turn. Future supply, gated by make-ready.
- **Vacant Rented Ready** — leased, awaiting move-in. Not marketable.
- **Vacant Rented Not Ready** — leased, waiting on turn. Move-in delay risk.

This reclassifies alerts rather than adding charts. "Spend with zero leases" branches: with no leasable units it is a make-ready problem owned by the Property Manager; with inventory available it is a marketing problem. Every conversion alert shows leasable count in its evidence so nobody reads a funnel number without its denominator.

As of the 25 August snapshot, the nine core properties held **five** units leasable that day. Lakeside had the portfolio's largest marketing spend, seven vacant units, and all seven already leased.

*Unverified. The supply snapshot is a seeded fixture in `supply-store.ts` / `supply.ts`, not a database table, so these figures were not checked against anything. The four-status distinction above is genuinely implemented and the alerts do branch on leasable count as described.*

---

## Provenance

Every displayed figure resolves to a provenance record: source system, file, sheet, column, extraction date, load date, transformation applied, correction applied, and whether a human has verified it.

**If a figure cannot carry provenance, it does not get displayed.** No exceptions. A defect was found where a weekly trend was generated synthetically rather than sourced; that is the failure mode this rule exists to prevent.

*The synthetic trend is deleted and a test now stops it returning. Provenance records exist for all twelve metrics in `METRICS` and are written to the `provenance` table on every ingest.*

***One class of figure still has no provenance record.*** *The seeded constant tables in `elise.ts` — tour completeness, portfolio completeness, speed to lead, portfolio speed, first-contact splits, negative durations — render on the Overview and Data Integrity pages as portfolio figures. They carry a source string in a file comment, not a provenance record, and they do not move when a workbook is ingested. A reader cannot tell them apart on screen from figures that came from the load. Either give them provenance records or label them as seeded in the UI.*

*Related: the "Verification status" panel reports that **no row in the pipeline has ever been human-verified** — the workbook carries no sign-off column, so `human_verified` is false on every provenance row. That is honest, and it means the last field in the list above is currently always "no".*
