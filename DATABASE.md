# The database

Everything the platform knows lives in one PostgreSQL database. Modules — the API,
the frontends, the pipeline or an external document service, agents — all read and
write the same tables; none of them owns private state. **The database is the
contract**, which is why it is worth documenting properly rather than inferring
from ORM models.

The schema is built by Alembic: a squashed baseline (`0001_initial_schema`, from
`migrations/schema.sql` — 40 tables, 15 of them under row-level security, verified
to build from an empty PostgreSQL 16 + pgvector) followed by migrations `0002`–`0025`,
which add 13 tables, split every policy into read and write halves, and introduce
the RBAC matrix. The backend container applies them on start
(`alembic upgrade heads`). Column-level reference: `docs/DATABASE_SCHEMA.md` in the
backend repository.

## Contents

- [Three schemas](#three-schemas) · [The domain](#the-domain-ledger) ·
  [Beside the domain](#beside-the-domain) · [Documents](#documents) ·
  [Access control](#access-control-and-rls) · [Search](#search) ·
  [Operational](#operational-tables) · [Indexes](#indexes-and-what-they-cost) ·
  [Corrections](#corrections-made-in-the-baseline)

---

## Three schemas

| Schema | Holds |
|---|---|
| `public` | every table except the access-control ones |
| `app` | tenant scoping, workspace resolution, search, triggers; and three access-control tables (`agent_workspace_roles`, `dataroom_permissions`, `entity_members`) |
| `audit` | `audit.log_change()`, the generic change-capture trigger |

Extensions: **`vector`** (embeddings), **`pg_trgm`** (fuzzy filename match),
**`btree_gin`** (composite GIN), `plpgsql`.

Four enums: `fileextractionstatusenum` (PREVIEW · PENDING · EXTRACTED · ERROR ·
INDEXING · INDEXED), `kvp_status` (extracted · human_validated · rejected ·
needs_review · labelling), `syncjobstatusenum`, `syncjobtypeenum`.

---

## The domain ledger

Four tables carry the investment domain. See the [ontology](README.md#the-ontology)
for the type vocabularies and the unit/sign conventions; this covers how they are
wired.

```
entities ──────subject_id/object_id──► transactions ──event_id──► events
       ▲                                                              │
       └──────────────── measurements ────────────────────────────────┘
                                    entity_groups (netting)
```

**`entities`** — every party, asset and account: the thing you own, the thing you
owe, the counterparty you pay, the bank account the money sits in. A **text**
primary key per workspace (`(workspace_id, id)`, a readable slug like
`northwind_holding_gmbh`), not a UUID — so ledger rows are legible without joins.
`group_id` → `entity_groups`, for netting a property against its mortgage.
`contacts` is `jsonb` with a fixed shape the API enforces (people, channels,
portals, `how_to_reach`; migration 0021). `accounting_system_ref` is the account
number in the firm's bookkeeping (0013) — a string, indexed, deliberately not
unique.

> Not `legal_entities`: a bank account, a property and a loan balance are none of
> them legal persons, and accounts are the *most* trafficked rows here. The table
> holds anything a transaction can point at; `type` and `structure_type` say what
> each one is.

> **Type is what a thing is, not the part it plays.** A law firm is a
> `service_provider`; it *acts as* a counterparty when you pay it, and as a subject
> when it invoices. Role is already expressed by which foreign key points at the
> row, so it must not leak into `type` — that is how a vocabulary silently drifts
> until cross-entity reporting stops working.

**`transactions`** — two legs, each saying *how much* of *what unit*
(`outflow_*` / `inflow_*`), amounts signed from the subject's point of view. Both
`subject_id` and `object_id` are composite `RESTRICT` foreign keys into
`entities`, so an entity cannot be deleted while any transaction references it.
`event_id` is `SET NULL` — losing the economic event must not destroy the record.
Amounts are `numeric(28,8)`, reporting amounts `numeric(20,4)`.

**`events`** — the economic thing that happened. One event owns several
transactions; that is how an investment's cash leg and share leg stay tied.

**`measurements`** — periodic numeric facts. `instrument_type` is the
discriminator: `NULL` ⇒ a company KPI, set ⇒ an instrument valuation, whose
`value` is the price of **one** unit and whose `unit` is a bare currency.
Migration 0019 converted every existing valuation to that form and refuses to
commit while any valuation unit is not a three-letter currency; totals that could
not be divided were renamed `<metric>_total`.

Tasks, events, transactions and measurements also carry a `group_id` into the
cross-type `groups` table (below). `entities.group_id` is the older, unrelated
asset grouping into `entity_groups`; no table carries both.

### Controlled vocabularies

Some type columns are enforced by `CHECK` constraints, and one deliberately is
not. The difference is whether the column is an extension point.

| Column | Constraint |
|---|---|
| `entities.type` | a list: the canonical eleven (`private_company` … `individual`), `other`, and four legacy spellings real data arrived with (`portfolio_company`, `account`, `counterparty`, `entity`) |
| `entities.structure_type` | `holding` · `asset` |
| `events.type` | a list (see the README), including `dividend`, `external_financing`, `invoice` |
| `transactions.inflow_type` / `outflow_type` | a pattern: `^[A-Z]{3}$` (a currency) or `^[a-z][a-z0-9_]*$` (an instrument / share class), plus a few legacy words |
| `transactions.type` | **none** — free text, except that `bank_wire` is refused |

`transactions.type` lost its list in migration 0010, on purpose. It is where a
module names a kind of movement it needs to tell apart, and a list in a shared
table means every module has to widen a shared constraint — and every deployment
that widens it carries a migration the next upstream revision overwrites. Meaning
belongs where rows are read: a report that asks for its own `type` gets exactly
its own rows, and a misspelt type produces a query that returns nothing, which is
visible. The single refusal, `bank_wire`, exists because bank lines moved to their
own table (`cashflows`, 0016) and must not be written back.

The share-class column is a pattern rather than a list for the same reason in
miniature: a class is named by whoever issues it, and the next financing round
invents another. The pattern still rejects what the constraint is actually for —
free text, prose, stray capitals.

The widened lists (0009) were added `NOT VALID`: they check what is written from
now on and do not re-validate existing rows, so an install whose history holds a
value the list does not know still migrates instead of refusing to start.

Three kinds of value are deliberately not types at all:

| Not a type | Where it goes |
|---|---|
| **role** — a law firm *is* a `service_provider`, it *acts as* a counterparty | which FK points at the row |
| **rail** — wire, SEPA, card: how money moved, not why | `add_data.rail` |
| **firm-specific label** | `transactions.classification` |

**`NULL` passes every constraint, on purpose.** A row may exist before it has been
classified. `other` means "classified, and none of these fit"; `NULL` means "not
classified yet". Keeping them apart is what makes an unclassified backlog
countable.

---

## Beside the domain

| Table | Holds |
|---|---|
| `cashflows` | bank statement lines: one **signed** `amount` per line on one `account_id` (an entity), `booking_date` and `value_date`, `counterparty_raw` as printed plus an optional `counterparty_id` link, `booking_text` and `purpose` kept apart, import provenance (`import_batch`, `source_sheet`, `source_row`, `dedupe_hash`). No `event_id`, no instrument leg. |
| `cashflow_settlements` | many-to-many `cashflow ↔ transaction`, with the `amount` attributed and a free-text `basis` |
| `entity_files` · `event_files` · `transaction_files` · `measurement_files` · `cashflow_files` | source documents per record: `file_id` + `ref_name` (where in the document) |
| `groups` | cross-type labels `(workspace_id, id)` with a slug id, a name, and an optional ordered `columns` list for request-list display (0022) |
| `comments` | an append-only thread on a task; `author_channel` says whether it came through the API or MCP |
| `tasks` | the per-item work ledger: `agent_name` · `type` · `status` · `result`, `dedupe_key` for idempotent registration, one level of `parent_task_id`, `folder_id`, `group_id`; due dates in `add_data.due_date` with an expression index |
| `task_files` | documents attached to a task, with a `role` (`source` · `attachment` · `output`) |

**Bank lines are admin-write.** `cashflows` is its own table rather than a
transaction `type` because permissions are granted per resource: only a separate
resource can say "anyone may record an investment, only an admin may record a
bank line". The RBAC seed grants `cashflows` create/update/delete to `admin` and
read to everyone else — including agents, which carry a member's permissions.

**Source links are real foreign keys.** Before 0020 a record's citations lived in
`add_data.references`, a JSON list nothing checked. Now each record type has its
own link table (one per type, because a foreign key can point at exactly one
table), and a trigger refuses a link whose file belongs to a different workspace
than the record — a plain FK to `files` cannot see workspaces, since files carry
theirs through their folder. Deleting the file or the record removes the link.
Citations that could not be resolved during the backfill were kept in
`add_data.references_unresolved` as evidence of what the author cited.

**Comments are append-only by grant.** No update or delete path exists and the
grants withhold both; a correction is another comment. A comment is an internal
note, so unlike `tasks` its policy does not admit dataroom guests.

---

## Documents

```
workspaces ──► folders ──► files ──► file_extractions ──► file_chunks
                             └─────► key_value_pairs
```

These are the tables an ingestion pipeline writes — the built-in pipeline image or
an external document service — and the API reads.

**`folders`** — self-referencing via `parent_folder_id`, plus a denormalised
`ancestor_ids` array maintained by trigger (`app.set_folder_ancestors`,
`app.resync_descendant_ancestors`). The array exists so permission checks are a
single array-overlap test instead of a recursive walk — `ltree` is not available
on every managed PostgreSQL.

**`files`** — identity is `hashcode` (SHA-256 of the original bytes), so moves and
renames are detected and nothing re-extracts needlessly. Non-native files are
stored **twice**: `original_blob_name` holds the bytes as uploaded and
`pdf_blob_name` the PDF rendition, so "give me the file" and "give me a PDF of
it" are each one column. `marked_extractable` says whether extraction should run.

**`origin`** on `folders` and `files` (`'disk'` or `'api'`) records who created a
row. A sync mirrors a directory tree and archives what has vanished from it, but
only among rows it created: anything uploaded through the API — and everything
below an API-created folder — is out of its reach, and triggers refuse to move a file or folder into or out of
API-owned storage.

**`file_extractions`** — one row per extraction *run*, not per file:
re-extraction appends. `md_format` is the searchable body; `json_format` holds the
page/coordinate map that chunking needs; `md_tsv` is a generated bilingual
tsvector (German and English stemming side by side). `attempt`, `content_chars`,
`cost_micros` and `extractor_version` record which try this was, what it
achieved, what it cost and which build produced it. Status `PREVIEW` marks a
provisional extraction (for example an agent that read a PDF directly); members
may write only such rows, and the regular extractor still runs over the file and
supersedes it.

**`file_chunks`** — chunk text plus a 1536-dimension embedding, tied to the
extraction that produced it. Optional: without chunks, search is lexical-only
and readers fall back to the extraction text.

**`key_value_pairs`** — labelled facts (doctype, summary, readability) with a
`kvp_status` lifecycle from machine-extracted to human-validated.

In a reference deployment `files` is ~140k rows, `file_extractions` ~10 GB and
`file_chunks` ~3M rows and 44 GB — the largest object by far.

---

## Access control and RLS

Three independent questions decide every request, and they are kept apart:

| | Question | Where |
|---|---|---|
| **RBAC** | *what* may this role do to this kind of resource? | `role_permissions` + the API's authorize middleware |
| **RLS** | *which rows* may this principal see and write? | Postgres policies |
| **Ownership** | is this user an owner of this workspace? | in the handful of owner-only routes |

### The RBAC matrix

`role_permissions(role_id, resource, can_create, can_read, can_update,
can_delete)` — one row per role and resource, absent row = deny. It is global
configuration, not under RLS. Resource keys live in code (`core/rbac/resources.py`,
with the route → resource map in `core/rbac/registry.py`); an authenticated route
that maps to no resource is refused. The seed (0011, extended by 0015 and 0024):

| role | holds |
|---|---|
| `admin` | everything |
| `member` = `default` = `agent` | full CRUD on workspace data and feature views; read on search, stats, activity, extraction and ad-hoc SQL; read-only on `cashflows`; nothing on roles, users or the operations dashboards |
| `external` | create/read/update on `datarooms`, nothing else |

`default` is the role every registered account receives. **`agent` is seeded
identical to `member`**: an agent is a user with a role, and nothing in the code
branches on whether a principal is human. Narrowing a bot is editing its role's
rows. Only the `admin` *role* may change roles or the matrix — holding the `roles`
permission alone is not enough, since that permission lives in the matrix being
edited.

RBAC never widens *which rows*: that is RLS, identically for every role.

### Row-level security

Tenant isolation is enforced **in the database**, not in application code. Every
workspace-scoped table carries policies built on two functions:

```sql
FOR SELECT                USING      (app.tenant_visible (workspace_id, CURRENT_USER))
FOR INSERT                WITH CHECK (app.tenant_writable(workspace_id, CURRENT_USER))
FOR UPDATE  USING (visible / writable) WITH CHECK (app.tenant_writable(...))
FOR DELETE                USING      (app.tenant_writable(workspace_id, CURRENT_USER))
```

`app.tenant_visible(ws, role)` is `SECURITY DEFINER` and decides in this order:

1. a **per-workspace agent role** (`agent_ws_<id>`, mapped in
   `app.agent_workspace_roles`) sees exactly its one workspace;
2. a member of **`app_trusted`** (the pipeline's `app_worker`) sees everything;
3. otherwise the row is visible if its workspace equals the session's
   **`app.portal_workspace_id`** (a single pinned workspace — portal links, MCP
   connections, workspace-pinned SQL), or the user in **`app.current_user_id`**
   is a member of it in `user_workspaces`.

`app.tenant_writable` is the same function with one extra condition on the
membership branch: the membership must not be **`read_only`**. The original
single `FOR ALL` policies were split into these four (0003), because a `FOR ALL`
policy uses one expression for SELECT and DELETE alike and so cannot permit the
read while refusing the write. Tables created later are created split, and 0006,
0007 and 0023 converted the ones that had been created on parallel branches with
the old shape. The UPDATE policies use the *visible* expression in `USING` and the
*writable* one in `WITH CHECK` (0004), so a refused update raises instead of
quietly reporting "0 rows updated" while the user believes the change was saved.
A refused DELETE still matches zero rows silently — RLS gives it no
`WITH CHECK` — so a friendly refusal for it belongs in the API layer, with the
policy remaining the actual boundary.

Tables that do not carry `workspace_id` directly resolve it through a helper:

| Table | Scoped via |
|---|---|
| `file_chunks`, `key_value_pairs` | `app.file_workspace(file_id)` |
| `files` | `app.folder_workspace(folder_id)` |
| `file_extractions` | `workspace_id`, filled by trigger |

**Dataroom sharing** widens this deliberately. `files`, `folders`, `tasks`,
`task_files` and `datarooms` add a second branch — the row's folder is in
`app.visible_folder_ids()`, or for folders an overlap against `ancestor_ids` — so
an external user granted one folder sees that subtree and nothing else. The
grant lookups run inside `SECURITY DEFINER` functions (folders since 0025), so the
access-control tables never appear in the caller's own query plan.
`app.is_external_user()` and a trigger on `user_workspaces` keep those users out
of normal membership.

**What carries no policy:** `users`, the login/refresh/OAuth token tables,
`workspaces`, `user_workspaces`, `roles`, `user_roles`, `role_permissions`, the
sync / pipeline logs, and some feature-module tables. They are reached only
through routes that name them, so for them the application's checks are the only
boundary. That is why ad-hoc SQL runs under its own role (below), and why a new
workspace-scoped table should get its four policies in the migration that
creates it.

### Session settings (GUCs)

The API sets these per transaction (`set_config(..., true)`); nothing else should:

| Setting | Read by | Meaning |
|---|---|---|
| `app.current_user_id` | policies | the logged-in user; RLS resolves their memberships |
| `app.portal_workspace_id` | policies | pin the session to exactly one workspace |
| `app.actor_user_id` | the audit trigger only | who acted, when the session is pinned rather than user-scoped |
| `app.actor_channel` | audit trigger, `comments` | `api`, `mcp`, `sql`, … |
| `app.change_reason` | audit trigger | why — required on every MCP / CLI write |

Identity for the decision and identity for the record are deliberately separate
settings: `tenant_visible` ORs the pinned workspace with the user's memberships,
so setting `current_user_id` on a pinned session would let it read every
workspace the user belongs to. `actor_user_id` gives the audit trail a name
without widening anything. All of them are read with `current_setting(..., true)`,
so a session that sets none (a migration, `psql`) records nothing rather than
failing — and, for the policies, sees nothing.

### Database roles

| Role | Login | Used by | RLS |
|---|---|---|---|
| owner | yes | migrations only | bypass (owner; no table uses `FORCE`) |
| `app_api` | yes | the API | subject to RLS; DML on the schema, RLS scopes the rows |
| `app_sql_ro` | no | `POST /query/sql`, via `SET LOCAL ROLE` | subject to RLS; SELECT on workspace-scoped tables only |
| `app_worker` | yes | pipeline / document service | bypass, via `app_trusted`; grants on an explicit table list (documents, sync logs, compute leases) |
| `app_trusted` | no (group) | — | the marker `tenant_visible` tests for |
| `agent_ws_<id>` | yes | direct-SQL agents, optional | pinned to one workspace |

`app_trusted` and `app_sql_ro` are created by migrations (0002, 0005); the login
roles carry secrets and are created by `db/sql/rls_roles_setup.sql`.

> **This is the part a schema rebuild must not get wrong.** Tables can be
> recreated from ORM models; RLS policies, `SECURITY DEFINER` functions and roles
> cannot. A baseline that restores tables without policies produces a database
> that looks correct and silently lets every workspace read every other
> workspace's data.

### Ad-hoc SQL

`POST /query/sql` (the CLI's `wholegrain sql`, MCP's `run_sql`) hands the caller's
statement to the database. Three independent layers make that safe:

1. **Read-only by Postgres.** The statement runs in `SET TRANSACTION READ ONLY`,
   so writes fail in the executor — including a write hidden in a CTE — rather
   than being guessed at by a parser. One statement per call; `;` is refused.
2. **Privileges by role.** The statement runs as `app_sql_ro`, which holds SELECT
   on the workspace-scoped business tables and nothing else, so the credential
   tables are unreachable by absence of grant. No default privileges are set: a
   new table is invisible to ad-hoc SQL until a migration grants it. The caller's
   identity survives the role switch (`current_user_id` is a GUC), so RLS still
   filters rows for the same person.
3. **A plan check.** Before running, the API `EXPLAIN`s the statement and refuses
   it if the plan names `users`, `app_login_pins`, `app_refresh_tokens`,
   `oauth_clients`, `oauth_refresh_tokens`, `dataroom_permissions`,
   `entity_members`, `agent_workspace_roles`, `mirror_tokens` or
   `mirror_imported` — base relations, so a view, CTE or alias does not hide one.

Who may call it at all is the `query_sql` row of the RBAC matrix (read for
members, not for `external`). Joins against `user_workspaces` and `workspaces` do
not work in ad-hoc SQL; that is the price of not exposing the membership graph.

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
pass `NULL` and get lexical-only hybrid, which still works.

Three hard-won constraints, all easy to reintroduce:

- **Never put a window function at the same query level as `ORDER BY embedding <=> … LIMIT`.** Windows are evaluated before `ORDER BY`, so the HNSW index cannot be used and Postgres distance-sorts every chunk. Isolate the ANN in its own CTE.
- **Raise `hnsw.ef_search` whenever the ANN is filtered.** pgvector post-filters, so a workspace predicate discards most of what the scan found. Pinned to 400 as a function-level `SET`.
- **Group `file_extractions` by `file_id` before ranking.** It holds one row per extraction run, so a re-extracted file otherwise occupies several result slots.

---

## Operational tables

Not domain data — the record of what the workers did. The frontend reads these to
show pipeline health, which is the whole point of workers writing their state to
the database rather than to logs.

| Table | Holds |
|---|---|
| `sync_sources` | one row per way data enters a workspace, with two independent freshness signals: is the source reachable now (`alive`, three-valued — `NULL` means nothing observes it), and did the mover actually run |
| `pipeline_run_log` | one row per pipeline step (`sync`, `extraction`, `labelling`, `qa`, `chunking`) |
| `pipeline_compute_instances` | leased compute, so a second run does not start while one is live |
| `qa_reports` | nightly reconciliation buckets |
| `folder_sync_logs`, `sync_audit_report`, `pipeline_file_logs` | per-file audit of what sync and extraction did, pruned on a retention window |
| `audit_logs` | generic change capture via `audit.log_change()`: old and new row, actor, channel, reason |
| `tokens_usage` | LLM spend |
| `mirror_tokens`, `mirror_imported` | the workspace-mirror module: hashed per-workspace tokens, and which mirrored rows came from which source |

`audit.log_change()` is attached per table — entities, events, transactions,
measurements, tasks, task_files, files, groups, datarooms. Cashflows, the
`*_files` link tables and the document tables below `files` carry none.

---

## Indexes and what they cost

The distribution is extremely lopsided. In a reference deployment:

| Index | Size | Kind |
|---|---:|---|
| `idx_file_chunks_embedding_hnsw` | **13 GB** | HNSW |
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

## Corrections made in the baseline

The baseline was built from a running reference deployment, and five things were
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
the rename above the coalesce is unnecessary. Note that a database migrated
across the rename kept the *old* function body — which then failed on the
missing column — until `0012` re-created it from the baseline; the lesson is that
a rename must re-create every function that names the column.

**Two scratch tables** left over from one-off operations were still being indexed
and backed up. Not carried over.

**`file_extractions` had no way to describe several extractions of one file.**
Running a cheap extractor over everything and better ones on top is the design, so
it gained `attempt`, `content_chars`, `cost_micros` and `extractor_version`.
Deliberately *no* column marks a winner — that judgement belongs to the workers,
which is why the schema records facts rather than verdicts.

Two later corrections are worth knowing about because they changed meaning, not
just shape: **valuations became unit prices** (0019 — a bare-currency `unit` on a
total had been the only thing distinguishing a price from a total), and **bank
lines left `transactions`** (0015/0016 — one signed amount per line, with the
signed totals per account checked to match before the old rows were deleted).
