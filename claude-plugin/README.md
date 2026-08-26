# Wholegrain plugin for Claude Code

Query a Wholegrain workspace from Claude: entities, events, transactions,
measurements, and the documents they were extracted from. **Read-only** — no
tool in this plugin can modify a workspace.

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
to approve the connection. There is no API key to paste.

## Skills

| skill | for |
|---|---|
| `/wholegrain:connect` | point at a workspace, diagnose a missing connection |
| `/wholegrain:portfolio-review` | holdings, invested capital, latest valuations |
| `/wholegrain:trace-figure` | a number back to its source document and page |
| `/wholegrain:data-gaps` | what is missing before you rely on a report |

## Tools

`get_workspace_info`, `search_documents`, `list_folders`, `list_files`,
`get_file_chunks`, `query_entities`, `query_events`, `query_transactions`,
`query_measurements`, `query_tasks`.

`search_documents` is semantic and needs an embeddings provider configured on
the deployment; without one it degrades to lexical matching.
