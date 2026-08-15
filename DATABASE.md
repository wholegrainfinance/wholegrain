# The database

Everything the platform knows lives in one PostgreSQL database. Modules — the API,
the frontends, extraction workers, agents — all read and write the same tables;
none of them owns private state. **The database is the contract**, which is why it
is worth documenting properly rather than inferring from ORM models.

40 tables, 21 functions, 15 tables under row-level security, 15 policies, 15
triggers, 126 indexes, 4 extensions — created by a single migration and verified
to build from an empty PostgreSQL 16 + pgvector.

## Contents

- [Three schemas](#three-schemas) · [The domain](#the-domain-ledger) ·
  [Documents](#documents) · [Access control](#access-control-and-rls) ·
  [Search](#search) · [Operational](#operational-tables) ·
  [Indexes](#indexes-and-what-they-cost) · [Corrections](#corrections-made-in-this-baseline)

---

## Three schemas

| Schema | Holds |
|---|---|
| `public` | every table |
| `app` | 20 functions — tenant scoping, workspace resolution, search |
| `audit` | `audit.log_change()`, the generic change-capture trigger |

Extensions: **`vector`** (0.7.0, embeddings), **`pg_trgm`** (fuzzy filename match),
**`btree_gin`** (composite GIN), `plpgsql`.

Four enums: `fileextractionstatusenum` (PREVIEW · PENDING · EXTRACTED · ERROR ·
INDEXING · INDEXED), `kvp_status` (extracted · human_validated · rejected ·
needs_review · labelling), `syncjobstatusenum`, `syncjobtypeenum`.

---

## The domain ledger

Four tables carry the investment domain. See the [ontology](README.md#the-ontology)
for the type vocabularies; this covers how they are wired.

```
entities ──────subject_id/object_id──► transactions ──event_id──► events
       ▲                                                              │
       └──────────────── measurements ────────────────────────────────┘
                                    entity_groups (netting)
```

**`entities`** — 20 columns. Every party, asset and account: the thing you own, the
thing you owe, the counterparty you pay, the bank account the money sits in. A
**text** primary key (a readable slug like `acme_holding_gmbh`), not a UUID — so
ledger rows are legible without joins. `group_id` → `entity_groups`, for netting a
property against its mortgage.

> Not `legal_entities`: a bank account, a property and a loan balance are none of
> them legal persons, and accounts are the *most* trafficked rows here — every bank
> transaction names one as its subject. The table holds anything a transaction can
> point at; `type` and `structure_type` say what each one is.

> **Type is what a thing is, not the part it plays.** A law firm is a
> `service_provider`; it *acts as* a counterparty when you pay it, and as a subject
> when it invoices. Role is already expressed by which foreign key points at the
> row, so it must not leak into `type` — that is how a vocabulary silently drifts
> until cross-entity reporting stops working.

**`transactions`** — 21 columns. Two legs, each saying *how much* of *what unit*
(`outflow_*` / `inflow_*`). Both `subject_id` and `object_id` are `RESTRICT`
foreign keys into `entities`, so an entity cannot be deleted while any
transaction references it. `event_id` is `SET NULL` — losing the economic event
must not destroy the cash record.

**`events`** — the economic thing that happened. One event owns several
transactions; that is how an investment's cash leg and share leg stay tied.

**`measurements`** — 22 columns. Periodic numeric facts. `instrument_type` is the
discriminator: `NULL` ⇒ a company KPI, set ⇒ an instrument valuation.

---

## Documents

```
workspaces ──► folders ──► files ──► file_extractions ──► file_chunks
                             └─────► key_value_pairs
```

**`folders`** — self-referencing via `parent_folder_id`, plus a denormalised
`ancestor_ids` array maintained by trigger (`app.set_folder_ancestors`,
`app.resync_descendant_ancestors`). The array exists so permission checks are a
single array-overlap test instead of a recursive walk — Postgres-managed `ltree`
was not available on the managed instance.

**`files`** — 140k rows. Identity is `hashcode` (SHA-256 of the original bytes),
so moves and renames are detected and nothing re-extracts needlessly. Non-native
files are stored **twice**: `original_blob_name` holds the bytes as uploaded and
`pdf_blob_name` the PDF rendition, so "give me the file" and "give me a PDF of
it" are each one column.

**`file_extractions`** — 148k rows, **10 GB**. One row per extraction *run*, not
per file: re-extraction appends. `md_format` is the searchable body;
`json_format` holds the page/coordinate map that chunking needs; `md_tsv` is a
generated bilingual tsvector.

**`file_chunks`** — 3.0M rows, **44 GB**, the largest object by far. Chunk text
plus a 1536-dimension embedding, tied to the extraction that produced it.

**`key_value_pairs`** — 353k rows. Labelled facts (doctype, summary, readability)
with a `kvp_status` lifecycle from machine-extracted to human-validated.

---

## Access control and RLS

Tenant isolation is enforced **in the database**, not in application code. Fifteen
tables carry row-level security; the policies are uniform:

```sql
USING (app.tenant_visible(workspace_id, CURRENT_USER))
```

`app.tenant_visible(ws, role)` is `SECURITY DEFINER` and resolves visibility two
ways: an agent role is pinned to exactly one workspace via `app.role_workspace()`,
while a normal user's membership is looked up. Because it is `SECURITY DEFINER`,
each policy check runs with the owner's privileges and can use the fast index
path rather than being re-planned per caller.

Tables that do not carry `workspace_id` directly resolve it through a helper:

| Table | Scoped via |
|---|---|
| `file_chunks` | `app.file_workspace(file_id)` |
| `files` | `app.folder_workspace(folder_id)` |
| `file_extractions` | `workspace_id`, backfilled by trigger |

**Dataroom sharing** widens this deliberately. `files`, `folders` and `datarooms`
add a second branch — `folder_id = ANY(app.visible_folder_ids())`, and for folders
an overlap against `ancestor_ids` — so an external user granted one folder sees
that subtree and nothing else. `app.is_external_user()` and
`app.reject_external_membership()` keep those users out of normal membership.

> **This is the part a schema rebuild must not get wrong.** Tables can be
> recreated from ORM models; RLS policies and `SECURITY DEFINER` functions cannot.
> A baseline that restores tables without policies produces a database that looks
> correct and silently lets every workspace read every other workspace's data.

---

## Search

One function, `app.search()`, so the API and direct-SQL callers rank identically.
It fuses three legs with Reciprocal Rank Fusion (k=60) and returns one row per
file:

| Leg | Mechanism |
|---|---|
| Full text | `file_extractions.md_tsv` — German **and** English stemmings side by side, so German documents answer English queries |
| Vector | `file_chunks.embedding` — HNSW, best chunk per file |
| Filename | `pg_trgm` word similarity on `files.file_name` |

RRF uses only each leg's *rank*, never its score — cosine distance, `ts_rank` and
trigram similarity are not comparable numbers, and a leg returning nothing
degrades gracefully.

The **embedding is always computed by the caller**, never in the database: it
needs an API key that deliberately does not live in Postgres. Callers without one
get lexical-only hybrid, which still works.

Two hard-won constraints, both easy to reintroduce:

- **Never put a window function at the same query level as `ORDER BY embedding <=> … LIMIT`.** Windows are evaluated before `ORDER BY`, so the HNSW index cannot be used and Postgres distance-sorts every chunk. Isolate the ANN in its own CTE.
- **Raise `hnsw.ef_search` whenever the ANN is filtered.** pgvector post-filters, so a workspace predicate discards most of what the scan found. Pinned to 400 as a function-level `SET`.

---

## Operational tables

Not domain data — the record of what the workers did. The frontend reads these to
show pipeline health, which is the whole point of workers writing their state to
the database rather than to logs.

| Table | Holds |
|---|---|
| `tasks` | per-item agent work; `add_data` carries request facets |
| `pipeline_run_log` | one row per nightly step (`sync`, `extraction`, `labelling`, `qa`, `chunking`) |
| `pipeline_compute_instances` | leased compute, so a second run does not start while one is live |
| `qa_reports` | nightly reconciliation buckets |
| `folder_sync_logs`, `pipeline_file_logs` | per-file audit, pruned on a retention window |
| `audit_logs` | generic change capture via `audit.log_change()` |
| `tokens_usage` | LLM spend |

---

## Indexes and what they cost

121 indexes. The distribution is extremely lopsided:

| Index | Size | Kind |
|---|---:|---|
| `idx_file_chunks_embedding_hnsw` | **13 GB** | HNSW |
| `idx_file_extractions_md_trgm` | 656 MB | GIN |
| `idx_fe_ws_md_trgm` | 620 MB | GIN |
| `idx_fe_md_tsv` | 411 MB | GIN |
| `file_chunks_pkey` | 136 MB | unique |

The HNSW index alone is larger than every table except `file_chunks` itself.
Two consequences worth internalising:

- **Writes are expensive, not reads.** Inserting a chunk updates the HNSW graph.
  Deletes are cheap by comparison — pgvector filters dead tuples at scan time and
  `VACUUM` reclaims them, so bulk deletion is I/O-bound rather than index-bound.
- **`maintenance_work_mem` governs rebuild feasibility.** At 154 MB against a
  13 GB index, a `REINDEX` spills to disk and takes hours with search degraded
  throughout. Plan around never needing one.

---

## Corrections made in this baseline

This schema was built from a running reference deployment, and five things were
fixed on the way in rather than inherited. Recorded because each had already cost
someone a day.

**The two blob columns were named backwards.** `blob_name` held the *converted
PDF* while `file_name` held the *original blob key* — and `file_name` means "the
human filename" everywhere else. Now `original_blob_name` and `pdf_blob_name`,
each a direct link, and `file_name` is only ever the human filename.

**Two overlapping trigram indexes** on `file_extractions.md_format` cost 656 MB
and 620 MB for near-identical lookups. The workspace-composite one survives.

**The filename index could not be used.** `app.search` filtered on
`coalesce(file_name, blob_name)`, which no index on `file_name` can serve. With
the rename above the coalesce is unnecessary, so the filename leg finally hits
`idx_files_file_name_trgm`.

**Two scratch tables** — `tmp_sheet_chunk_purge` (67 MB, 840k rows) and
`_pipeline_diag` — were left over from one-off operations and still being indexed
and backed up. Not carried over.

**`file_extractions` had no way to describe several extractions of one file.**
Running a cheap extractor over everything and better ones on top is the design, so
it gained `attempt`, `content_chars`, `cost_micros` and `extractor_version`: which
try this was, what it achieved, what it cost, and which build produced it.
Deliberately *no* column marks a winner — that judgement belongs to the workers,
which is why the schema records facts rather than verdicts.

One inherited property is worth stating plainly, because it is a trap rather than
a wart: **`file_extractions` is one row per extraction run, not per file.**
Anything ranking over it must `GROUP BY file_id` first, or a re-extracted file
occupies several result slots.
