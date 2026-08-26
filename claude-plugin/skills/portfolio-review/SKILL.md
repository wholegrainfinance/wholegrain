---
name: portfolio-review
description: Summarise a Wholegrain portfolio — holdings, what was invested, latest valuations, and what moved since a given date. Use when the user asks for a portfolio overview, how the portfolio is doing, what a position is worth, or a summary of holdings across the workspace.
user-invocable: true
---

# /wholegrain:portfolio-review — what is held, what it cost, what it is worth

Assembles a portfolio picture from the ontology. Everything here is read-only.

## The shape of the data

Four tables carry the answer, and they are joined by entity, not by a
foreign-key you can see in one call:

| tool | what it holds |
|---|---|
| `query_entities` | the holdings themselves — companies, funds, vehicles |
| `query_transactions` | money moved: investments, distributions, fees |
| `query_measurements` | valuations and KPIs, each with a period and a scenario |
| `query_events` | the timeline that explains why a number changed |

A valuation is a **measurement**, not a field on the entity. There is usually
more than one per entity — different dates, sometimes different scenarios — so
"what is it worth" always means "the latest measurement of the right metric,
in the right scenario".

## Method

1. **List the holdings.** `query_entities` first. Note the entity type: an
   operating company, a fund and a holding vehicle are not comparable, and
   summing them double-counts when a vehicle holds a company that is also listed.

2. **Get invested capital** from `query_transactions`, filtered per entity.
   Sum the investment direction only — distributions and fees are separate
   lines, and netting them silently turns "invested" into "net cashflow",
   which is a different number that looks equally plausible.

3. **Get the latest valuation** from `query_measurements` per entity. Take the
   most recent period, and **state the as-of date in the output**. A portfolio
   total built from valuations struck on different dates is not a portfolio
   value at any date; say so rather than presenting one figure as current.

4. **Explain movement** with `query_events` between the two dates in question,
   rather than inferring a story from the numbers.

## Reporting rules

- **Give the as-of date next to every valuation.** Stale marks are the normal
  case, not the exception.
- **Say what is missing.** An entity with no valuation is not worth zero — it is
  unmeasured. Listing it as zero understates the portfolio and nobody notices.
- **Do not compute IRR or MoM by hand.** The application computes those from the
  full dated cashflow series; a figure derived from summary rows will disagree
  with the app's, and the user will trust whichever they saw first.
- **Currency is per transaction.** Do not add across currencies without saying
  what rate was used, and prefer not to add at all.

## Ending

Close with what would sharpen the picture — an entity whose last valuation is
old, a holding with investments but no mark, a period where events stop. Those
gaps are usually the point of the review.
