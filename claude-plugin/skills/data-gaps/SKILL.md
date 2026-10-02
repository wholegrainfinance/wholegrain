---
name: data-gaps
description: Find gaps and inconsistencies in a Wholegrain workspace — holdings without valuations, stale marks, transactions with no matching event, documents that produced nothing. Use when the user asks what data is missing, wants a data-quality check, asks if the workspace is complete, or is preparing a report and wants to know what will be wrong.
user-invocable: true
---

# /wholegrain:data-gaps — what is missing, and what that costs

Reporting fails on absence more often than on error: a holding with no valuation
disappears from a total silently, while a wrong valuation at least looks wrong.

Most checks below are one `run_sql` aggregate; the `query_*` tools work too but
page 100 rows at a time.

## What to check

**Holdings with no valuation.** A position is a transaction's
`(object_id, inflow_type)`; its valuation is a measurement with the same
`(subject_id, instrument_type)`. Any position with investment flows and no
matching measurement is shown at cost, never at value. A mark filed under a
different spelling of the instrument (`equity` vs `series_a`) attaches to
nothing — report those pairs as mismatches, not as missing. This is the
highest-value check.

**Totals posing as prices.** Valuation metrics (`fair_value`, `nav`,
`capital_account_value`) whose value is implausibly large next to the purchase
price per unit usually hold a position total. Rows named `<metric>_total` are
totals by design and are not valuations at all.

**Stale marks.** Valuations exist but the most recent is old. Report the
`period_end` per position and sort by age — "the newest mark is fourteen months
old" is actionable in a way that "valuation present" is not.

**Positions without units.** An investment with no `inflow_amount` has a cost
but cannot be valued, however good the mark. Likewise an issuer with stakes
reported but no `shares_outstanding` measurement has no ownership figure.

**Transactions with no event.** `event_id` is null. A transaction with no
explanatory event is a number without a story, which is exactly what fails
review.

**Records with no source.** Ontology rows with no row in their link table
(`transaction_files`, `measurement_files`, …). Not wrong, but not defensible
either.

**Documents that produced nothing.** Files with no `file_extractions` row, or
only rows in status `ERROR`, never contributed text. Do not use "no chunks" as
the test: chunks are optional, and `get_file_chunks` falls back to the
extraction when there are none. A file that looks filed but has no text is
invisible to every search and every agent, and nobody discovers this until
someone goes looking for the figure that was in it.

**Periods that stop.** Measurements and events that run to a date and then end.
Usually a pipeline that quietly stopped rather than a business that did.

## Reporting

Order by **consequence, not count**. One unvalued holding worth more than the
rest of the portfolio outranks forty missing KPIs. Say what each gap does to a
report someone is about to produce.

Give the entity, record or file identifier for each gap so it can be acted on,
and distinguish **missing** from **zero** everywhere — they are different facts
and only one of them is a problem.

## Scope

Read-only. This skill reports gaps; it never fills them, and it does not use the
write tools. Where a gap looks mechanical (an extraction that failed, a sync
that stopped), say which system would need attention rather than inventing the
missing value.
