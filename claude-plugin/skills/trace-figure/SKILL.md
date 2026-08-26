---
name: trace-figure
description: Trace a number back to the document it came from — find the source file, page and extracted text behind a valuation, transaction or KPI. Use when the user asks where a figure came from, to verify or source a number, to find the document behind a record, or doubts a value.
user-invocable: true
---

# /wholegrain:trace-figure — from a number back to its page

Every extracted record should be traceable to a document. This skill closes that
loop, which is what makes a figure defensible rather than merely present.

## Method

1. **Find the record.** `query_measurements`, `query_transactions` or
   `query_events`, filtered to the entity and period in question. Get the
   record's own identifier, not just its value — two records can carry the same
   number for different reasons.

2. **Find the file.** Records carry a reference to the file they were extracted
   from. Where they do not, fall back to `search_documents` with the entity name
   and the figure itself: extracted numbers usually appear verbatim in the text.

3. **Read the page.** `get_file_chunks` on that file returns the extracted text
   ordered by page. Quote the passage and give the page number.

## What to report

State the chain: **value → record → file → page → quoted text.** A trace that
stops at the file name has not verified anything; the passage is the evidence.

## Being honest about failure

- **No reference on the record** means it was entered or computed, not extracted.
  Say that. It is a real finding, not a dead end.
- **Search finds nothing** is expected if no embeddings provider is configured on
  the deployment — semantic search degrades to lexical, so paraphrases will not
  match. Try the exact figure and the exact entity name before concluding the
  document is absent.
- **The passage disagrees with the record.** Report both, quote the source, and
  do not adjust the record — an extraction error and a stale value need
  different fixes, and only a human can tell which this is.

Never present a figure as sourced when only the file was found and the passage
was not read.
