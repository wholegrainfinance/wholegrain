# Wholegrain plugin for Claude Code

Query a Wholegrain workspace from Claude: entities, events, transactions,
measurements, tasks, and the documents they were extracted from. The four skills
below only read. The server also offers a small set of **write tools** for
coordinating work; every one of them requires a reason, is checked against your
role, is recorded on the audit trail, and none of them can delete.

## Install

```
/plugin marketplace add wholegrainfinance/wholegrain
/plugin install wholegrain
```

## Configure

The MCP endpoint is per workspace, so set the URL before launching:

```bash
export WHOLEGRAIN_MCP_URL="https://your-api-host/mcp/<workspace-id>"
```

or in `~/.claude/settings.json`:

```json
{ "env": { "WHOLEGRAIN_MCP_URL": "https://your-api-host/mcp/<workspace-id>" } }
```

Then restart Claude Code and run `/wholegrain:connect` to verify.

Authentication is OAuth: on first use you are sent to your own Wholegrain login
to approve the connection. There is no API key to paste. A connection is bound to
its one workspace, even if your account belongs to several.

## Skills

| skill | for |
|---|---|
| `/wholegrain:connect` | point at a workspace, diagnose a missing connection |
| `/wholegrain:portfolio-review` | holdings, invested capital, latest valuations |
| `/wholegrain:trace-figure` | a number back to its source document and page |
| `/wholegrain:data-gaps` | what is missing before you rely on a report |

## Tools

**Read:** `get_workspace_info`, `search_documents`, `list_folders`, `list_files`,
`get_file_chunks`, `query_entities`, `query_events`, `query_transactions`,
`query_measurements`, `query_tasks`, and `run_sql` — one read-only `SELECT`
(joins, aggregates) under the same row-level security; the transaction is
read-only in the database itself.

**Write** (each takes a required `reason`): `create_process`,
`update_process_stage`, `update_task_status`, `assign_to_process`,
`add_request`, and `call_api` — any API endpoint as you, with your own
permissions, except `DELETE` and the `/external`, `/auth`, `/oauth`,
`/workspace` and `/settings` surfaces.

The `query_*` tools return table columns only. A record's source documents
(`references`) and resources without a typed tool — `cashflows`, `groups`,
settlements — are read with `call_api` (`GET /transactions/<id>?workspace_id=…`)
or `run_sql` (`select * from transaction_files where transaction_id = …`).

`search_documents` is vector search over document chunks: it needs an
embeddings provider on the deployment and chunked documents, and returns nothing
without them. `get_file_chunks` falls back to a file's full extracted text when
the file has no chunks.
