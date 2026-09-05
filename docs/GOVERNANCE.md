# Governance

Who can change what, and why it is arranged this way.

> **Verified against the code and the live database on 4 September 2026.**
> Corrections are marked in italics or flagged with ⚠. The most important one is
> in "Changing a threshold": the proposal queue most of the UI uses does not
> reach the database.

Marketing operates this tool. Property management leadership directs it. It is a marketing department tool, not a portfolio governance system. Asset management sees budget and goal alerts read-only and holds no rights here.

---

## Changing a threshold

Every alert threshold lives in the database, not in code, and changes follow a two-person flow.

```
Regional Manager / Property Manager / Marketing Analyst
        │  proposes, with a reason
        ▼
Marketing Director
        │  accepts, declines with reason, or amends the value
        ▼
Director of Property Management  or  COO
        │  ratifies or rejects with reason
        ▼
New version written. Threshold goes live.
```

**The Marketing Director cannot ratify their own change.** Enforced at the database level by a CHECK constraint and a trigger, not by hiding a button.

> **⚠ True of one path, false of the other. Read this before relying on the flow above.**
>
> `/alerts` carries four tabs, and two governance systems sit behind them.
>
> **"Governance (Cloud)"** is the real path. It writes to `alert_rule_proposals`,
> ratifies through the `ratifyThresholdChange` server action, and is covered by
> the `no_self_ratification` CHECK constraint, the `enforce_two_person_rule`
> trigger and the `stamp_proposal_author` trigger that makes the proposer
> immutable. Everything claimed above holds here.
>
> **"Proposal queue" and "Rules & history"** do not touch the database at all.
> They run on `src/lib/ratification.ts`, which stores proposals in the browser
> under `carbon-alert-rule-proposals-v2` and writes ratified versions to
> `localStorage`. Its own header calls it an "Interim client store", and it
> carries the database constraints as a **string constant** named
> `PROPOSAL_DB_CONSTRAINTS`, described as "applied verbatim once Cloud is
> enabled". Cloud is already enabled; the constant was never applied from there.
>
> On that path the two-person rule is a TypeScript `if`, checked on
> `proposedByUserId` — real logic, but not a database control, and trivially
> bypassed by clearing site data. Worse, a threshold "ratified" there goes live
> through `effectiveRule()` in **that browser only**: it changes how alerts
> evaluate for one person and never updates `alert_rules.threshold`. Two people
> can see different thresholds on the same rule.
>
> Until the two are merged, the Cloud tab is the only place a threshold change
> is real.

Ratification moves to the COO when the department head is the proposer, the value falls outside its bounds, the change disables a rule, or the rule is jointly owned. *Implemented in `ratificationAuthority()` on the client path for the disable, bounds and out-of-bounds cases. The Cloud path enforces bounds server-side — out-of-bounds ratification is refused unless the caller holds `coo` — but has no separate disable or bounds flow.*

**Bounds are set by property management leadership, not by Marketing.** Departments tune within bounds; the people who set the bounds are not the people using them. A department that can widen its own thresholds unsupervised will eventually tune itself quiet, and nobody would see it happen.

### Two behaviours worth knowing

**A pending change does not suppress anything.** The rule keeps evaluating on the current live threshold until ratification. Otherwise proposing a change becomes a way to make an alert stop firing, and the control inverts into the loophole it was built to close.

**Proposals expire after 7 days** at each stage. The live threshold simply stays. Expired proposals remain in the audit trail.

### Rule History

Every threshold change is recorded with proposer, reason, ratifier and timestamp. A rule loosened shortly after it started firing is visible on one screen.

*Built as the "Rules & history" tab on `/alerts`. The table shows proposed-at, rule, routed-to, scope, the value change, proposed by, adjusted by, reason and outcome — and for each outcome the role and date of the decline, rejection, expiry or ratification. Two corrections: it shows **role titles, not names**, which is correct under the role-titles-only rule, so read "both roles always appear together"; and there is **no plot against alert volume** — the correlation the paragraph describes has to be read off the table by eye. It also reads the `localStorage` store, so it shows only the changes made in the current browser.*

---

## Locked rules

These accept no proposals from anyone, including the COO:

- Dataset week missing entirely
- Restatement above 10% on a previously closed week
- Funnel stage reading zero while the adjacent downstream stage is non-zero
- Unmapped property name
- Spend recorded with no loaded budget

*Three more are locked in the database and were missing from this list:*

- *Self-consistency invariant failing (`data.invariant_failure`)*
- *Negative duration values in source (`data.negative_duration`)*
- *Cross-source variance beyond its explanation (`data.unexplained_residual`)*

*Eight locked rules in total, all `EVIDENCE_REQUIRED`, all confirmed `locked = true` in `alert_rules`.*

They test facts, not judgments. A threshold that can be widened until it stops firing is not a control. Technology can acknowledge and remediate them; nobody can tune them.

---

## Alert lifecycle

**States:** Open, Acknowledged, Unevaluable, Cleared, Suppressed.

One condition equals one alert. It persists across uploads with an age counter. An alert firing five weeks running looks worse than a new one. If the same condition re-fires within 4 weeks of clearing, it reopens the original and continues its age count — a flapping metric cannot reset its own clock.

*Verified. `REOPEN_WINDOW_DAYS = 28`, and a unique index on `(rule_id, coalesce(property_id,'-portfolio-'))` enforces one row per condition. Self-clearing genuinely requires `healthyWeeks >= 2`. One gap: on re-ingest, only `Acknowledged` survives — an alert deliberately `Cleared` or `Suppressed` reverts to `Open` on the next upload and its `decision_outcome` is discarded.*

### Clearing classes

| Class | Clears when | Applies to |
|---|---|---|
| **Self-clearing** | Metric inside threshold for **two consecutive weeks** | Performance conditions |
| **Evidence required** | The missing data actually loads | Data gaps, missing weeks, restatements, unmapped names |
| **Decision required** | An authorized human records a decision with a reason | Uncalibrated goals, unfundable goals, specials with no outcome |
| **Manual only** | Explicit acknowledgment with a note | Budget overruns |

**An alert never clears because its input went missing.** If the metric becomes uncomputable, the alert moves to Unevaluable with the reason stated. It is not healthy and it is not passing. This is the same discipline as never rendering a missing value as zero.

**Evidence-required alerts do not heal.** A missing week does not fix itself by not being mentioned again. The week of 10–16 August went unnoticed for twelve days in a prior system precisely because absence produced no persistent signal.

### Suppression

Time-boxed to 30 days maximum. Requires a reason and an attributed user. Auto-resumes on expiry. Stays visible in the Suppressed group throughout. No permanent mute.

*Mostly built. The reason is required, the attribution is stamped by a database trigger from `auth.uid()`, expiry auto-resumes, and the Suppressed group is one of the five in `GROUP_ORDER`. **The 30-day cap is not enforced on the write path**: `alerts.ts` → `suppress()` rejects anything over 30 days, but the Cloud action in `alerts-cloud.ts` writes `suppressed_until` straight to the database without going through it, and no database constraint bounds the column. Today the only caller passes 30, so the cap holds by convention rather than by control.*

Single-person on purpose: it is temporary and loud. Threshold changes are permanent and quiet, which is why those need two people. If something needs to stop firing at 9pm and no second person is available, suppress it rather than routing around the control.

---

## Alert routing

Alerts go to the role that can fix the condition.

| Owning role | Conditions |
|---|---|
| Property Manager | Tour show rate, tour-to-lease conversion, days-to-lease, on-site special execution, occupancy volatility, spend against no leasable inventory, aged vacancy |
| Regional Manager | Property risk score, drift detection, cross-property patterns, unknown-disposition rate |
| Marketing Director | Campaign performance, cost per lead and per lease, starved winners and bleeders, optimization score, budget pacing, unspent promotional budget |
| Director of Property Management | Goal not calibrated, unfundable goal, projected end-of-month below goal, special ended with no outcome |
| Technology | All locked data-integrity conditions — remediation only |

*Two corrections to this table.*

***"Technology" is a routing label, not an account role.*** *The `app_role` enum has exactly six values and Technology is not among them: `property_manager`, `regional_manager`, `marketing_analyst`, `marketing_director`, `director_pm`, `coo`. Alert conditions carry `ownerRole: "Technology"` for display and the alerts page offers it as a filter, but `appRoleFromTitle()` maps it to `marketing_director`, which is why all eight locked rules show `owning_role = marketing_director` in the database. Nobody can sign in as Technology, and those alerts land with the Marketing Director.*

***Several listed conditions have no rule behind them.*** *Present and correctly routed: tour show rate, occupancy volatility, spend against no leasable inventory and aged vacancy (Property Manager); drift (Regional Manager); cost per lease, budget pacing and unspent promotional budget (Marketing Director); goal not calibrated, unfundable goal and special ended with no outcome (Director of Property Management). **Not implemented as rules:** tour-to-lease conversion, days-to-lease, on-site special execution, property risk score, cross-property patterns, unknown-disposition rate, starved winners and bleeders, optimization score, and projected end-of-month below goal.*

~~Two rules are jointly owned and show both roles: special performance verdicts, and cost-per-lease deterioration where the cause is unresolved between spend efficiency and on-site conversion. Either can acknowledge; only the Director of Property Management records the closing decision.~~

***Not built.*** *There is no joint-ownership mechanism. `ownerRole` is a single value per condition and `alert_rules.owning_role` is a single column. The two named rules each carry one owner: `special.verdict_pending` → Director of Property Management, `campaign.cost_per_lease_deterioration` → Marketing Director. Implementing this needs a schema change, not a UI change.*

---

## Why proposals are open to everyone

Property and Regional Managers can propose any threshold change. This is deliberate. A 25% tour-show floor that is wrong for a 40-unit property is obvious to that Property Manager and invisible from a portfolio view. Without a proposal path, that knowledge either never arrives or arrives as complaining in a meeting.

**The queue only works if declines carry reasons.** If proposals go in and come back "no," people stop proposing within a month and the blind spot returns. That is a habit, not a feature.
