---
name: data-gaps
description: Find gaps and inconsistencies in a Wholegrain workspace — holdings without valuations, stale marks, transactions with no matching event, documents that produced nothing. Use when the user asks what data is missing, wants a data-quality check, asks if the workspace is complete, or is preparing a report and wants to know what will be wrong.
user-invocable: true
---

# /wholegrain:data-gaps — what is missing, and what that costs

Reporting fails on absence more often than on error: a holding with no valuation
disappears from a total silently, while a wrong valuation at least looks wrong.

## What to check

**Holdings with no valuation.** `query_entities`, then `query_measurements` per
entity. Any entity with investment transactions but no valuation measurement is
invisible in every portfolio total. This is the highest-value check.

**Stale marks.** Valuations exist but the most recent is old. Report the as-of
date per entity and sort by age — "the newest mark is fourteen months old" is
actionable in a way that "valuation present" is not.

**Transactions with no event.** `query_transactions` against `query_events` over
the same period. A transaction with no explanatory event is a number without a
story, which is exactly what fails review.

**Documents that produced nothing.** `list_files`, then `get_file_chunks`. A file
with no chunks was never extracted — it looks filed but contributes nothing, and
nobody discovers this until someone goes looking for the figure that was in it.

**Periods that stop.** Measurements and events that run to a date and then end.
Usually a pipeline that quietly stopped rather than a business that did.

## Reporting

Order by **consequence, not count**. One unvalued holding worth more than the
rest of the portfolio outranks forty missing KPIs. Say what each gap does to a
report someone is about to produce.

Give the entity or file identifier for each gap so it can be acted on, and
distinguish **missing** from **zero** everywhere — they are different facts and
only one of them is a problem.

## Scope

Read-only. This skill reports gaps; it never fills them. Where a gap looks
mechanical (an extraction that failed, a sync that stopped), say which system
would need attention rather than inventing the missing value.
