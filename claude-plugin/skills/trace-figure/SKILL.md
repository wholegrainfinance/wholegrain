---
name: trace-figure
description: Trace a number back to the document it came from — find the source file, page and extracted text behind a valuation, transaction or KPI. Use when the user asks where a figure came from, to verify or source a number, to find the document behind a record, or doubts a value.
user-invocable: true
---

# /wholegrain:trace-figure — from a number back to its page

Every extracted record should be traceable to a document. This skill closes that
loop, which is what makes a figure defensible rather than merely present. It
only reads.

## Method

1. **Find the record.** `query_measurements`, `query_transactions` or
   `query_events`, filtered to the entity and period in question. Get the
   record's own `id`, not just its value — two records can carry the same
   number for different reasons.

2. **Find the file.** Records cite their source documents as `references`:
   `[{file_id, ref_name}]`, where `ref_name` says where in the document the
   fact is ("Capital call no. 7, page 2"). The `query_*` tools do not return
   them; read them with either

   - `call_api` `GET /measurements/<id>` (or `/transactions/<id>`,
     `/events/<id>`, `/entities/<id>`) with query `workspace_id=<id from
     get_workspace_info>`, or
   - `run_sql`: `select file_id, ref_name from measurement_files where
     measurement_id = '<id>'` — likewise `transaction_files`, `event_files`,
     `entity_files`, `cashflow_files`.

   Where there is no reference, fall back to `search_documents` with the entity
   name and the figure itself — extracted numbers usually appear verbatim.
   Older records may still carry unresolved citations in
   `add_data.references_unresolved`.

3. **Read the passage.** `get_file_chunks` on that file returns the extracted
   text, chunk by chunk with page numbers. If the file was never chunked it
   returns the whole extraction instead (`source: "file_extraction"`, no page
   number, possibly `truncated`). Quote the passage, and give the page when
   there is one — otherwise the `ref_name`.

## What to report

State the chain: **value → record → file → page or ref_name → quoted text.** A
trace that stops at the file name has not verified anything; the passage is the
evidence.

Mind the unit conventions when comparing: a valuation's `value` is the price of
**one** unit, so a document stating a position total matches `value × units`,
not `value`. A transaction's `outflow_amount` is signed (negative = received).

## Being honest about failure

- **No reference on the record** means it was entered or computed rather than
  linked to a document. Say that. It is a real finding, not a dead end.
- **`search_documents` returns nothing** when the deployment has no embeddings
  provider or the documents are not chunked — it is vector search only. Search
  the extracted text directly instead, e.g. `run_sql`: `select file_id from
  file_extractions where md_format ilike '%49,828%' group by file_id` — try the
  figure in the document's own number format, and the entity name, before
  concluding the document is absent.
- **The passage disagrees with the record.** Report both, quote the source, and
  do not adjust the record — an extraction error and a stale value need
  different fixes, and only a human can tell which this is.

Never present a figure as sourced when only the file was found and the passage
was not read.
