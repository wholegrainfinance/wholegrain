---
name: portfolio-review
description: Summarise a Wholegrain portfolio — holdings, what was invested, latest valuations, and what moved since a given date. Use when the user asks for a portfolio overview, how the portfolio is doing, what a position is worth, or a summary of holdings across the workspace.
user-invocable: true
---

# /wholegrain:portfolio-review — what is held, what it cost, what it is worth

Assembles a portfolio picture from the ontology. This skill only reads; do not
use the write tools here.

## The shape of the data

Four tables carry the answer, and they are joined by entity, not by a
foreign-key you can see in one call:

| tool | what it holds |
|---|---|
| `query_entities` | the holdings and holders themselves — companies, funds, vehicles, accounts |
| `query_transactions` | flows: investments, calls, distributions, exits, fees, commitments |
| `query_measurements` | valuations (unit prices) and company KPIs, each with a period |
| `query_events` | the timeline that explains why a number changed — never summed |

Three conventions decide every number:

- **Transactions are signed, from the subject's side.** `outflow_amount` `+` means
  the subject paid, `−` means it received (a distribution is a negative
  outflow). `inflow_amount` is a **unit count** of `inflow_type` (`series_a2`,
  `fund_interest`, …), and an exit's is negative. `outflow_type` is the currency.
- **A valuation is the price of ONE unit.** A measurement with `instrument_type`
  set prices one unit of that instrument of the entity in `subject_id`; `unit` is
  the currency. It attaches to a position by matching transaction
  `(object_id, inflow_type)` to measurement `(subject_id, instrument_type)`.
  `metric_name` is `fair_value`, `nav`, `capital_account_value` or `share_price`.
  Rows named `<metric>_total` are totals kept for information — never multiply
  them by anything.
- **A measurement with `instrument_type` null is a KPI**, not a valuation.

## Method

1. **List the holdings.** `query_entities` first. Note the entity type: an
   operating company, a fund and a holding vehicle are not comparable, and
   summing them double-counts when a vehicle holds a company that is also held
   directly.

2. **Units and cost per position** from `query_transactions`, filtered to the
   asset (`object_id`) and, when the user asks about one holder, the holder
   (`subject_id`). Only position flows count —
   `type` in `investment`, `capital_call`, `transfer`, `exit`, `distribution`.
   Units = signed sum of `inflow_amount` per `inflow_type`; cost = signed sum of
   the money leg. Leave out `commitment` (notional, not paid), fees, invoices and
   anything with `status` `planned` or `expected` — netting them in turns
   "invested" into a different number that looks equally plausible.

3. **Latest valuation** from `query_measurements` per `(subject_id,
   instrument_type)`: the newest `period_end`. Value = units × price. **State the
   as-of date in the output.** A total built from marks struck on different
   dates is not a portfolio value at any date; say so rather than presenting one
   figure as current. No matching mark → the position is shown at cost and
   labelled so; a price with no unit count → value unknown.

4. **Ownership**, if asked: units held ÷ the issuer's newest
   `shares_outstanding` measurement. Its `unit` says the scale — `shares`, or
   `percent_scale` where a stake known only as a percentage is carried as units
   out of 100.

5. **Explain movement** with `query_events` between the two dates in question,
   rather than inferring a story from the numbers.

For a large workspace, one `run_sql` aggregate is cheaper and more reliable than
paging through `query_transactions`.

## Reporting rules

- **Give the as-of date next to every valuation.** Stale marks are the normal
  case, not the exception.
- **Say what is missing.** An entity with no valuation is not worth zero — it is
  unmeasured. Listing it as zero understates the portfolio and nobody notices.
  A real `fair_value` of 0 (a write-off) is different, and is zero.
- **Commitments are separate.** Report committed capital next to paid-in, never
  inside it.
- **Do not compute IRR or MoM by hand.** Those come from the full dated cashflow
  series; a figure derived from summary rows will disagree with whatever tool
  the user trusts, and they will trust whichever they saw first.
- **Currency is per leg.** Use `*_reporting_amount` / `reporting_value` where
  present; otherwise do not add across currencies without saying what rate was
  used, and prefer not to add at all. A null reporting amount is unknown, not 0.

## Ending

Close with what would sharpen the picture — an entity whose last valuation is
old, a holding with investments but no mark, a period where events stop. Those
gaps are usually the point of the review.
