---
name: connect
description: Point this plugin at a Wholegrain workspace, or check which one it is connected to. Use when the user asks to connect to Wholegrain, switch workspace, says the Wholegrain tools are missing or erroring, or asks which workspace they are querying.
user-invocable: true
allowed-tools:
  - Read
  - Bash(grep *)
  - Bash(echo *)
---

# /wholegrain:connect — point the plugin at a workspace

A Wholegrain MCP endpoint is **per workspace**, not per server:

```
https://<api-host>/mcp/<workspace-id>
```

So the plugin reads its URL from `WHOLEGRAIN_MCP_URL` rather than hardcoding one.
Switching workspace means changing that variable and restarting Claude Code.

## Check what is set

```bash
echo "${WHOLEGRAIN_MCP_URL:-<unset>}"
```

If it is unset, the `wholegrain` MCP server will fail to start and none of its
tools appear. That is the usual cause of "the Wholegrain tools are missing".

## Set it

The workspace id is the UUID in the dashboard URL after the workspace is
selected, or from `get_workspace_info` if another workspace is already connected.

```bash
export WHOLEGRAIN_MCP_URL="https://api.example.com/mcp/<workspace-id>"
```

Put it somewhere durable — `~/.zshrc`, `~/.bashrc`, or the `env` block in
`~/.claude/settings.json`:

```json
{ "env": { "WHOLEGRAIN_MCP_URL": "https://api.example.com/mcp/<workspace-id>" } }
```

**Restart Claude Code afterwards.** MCP servers are started once at launch, so a
variable exported into a running session does not reach the server.

## First connection

On first use the server answers `401` with a `WWW-Authenticate` challenge, and
Claude walks the OAuth discovery chain automatically — protected-resource
metadata, then the authorization server, then dynamic client registration. No
API key to paste: you are sent to your own Wholegrain login in a browser and
approve the connection there. Tokens are short-lived and refresh silently.

## Confirming it worked

Call `get_workspace_info`. It returns the workspace this connection is bound to
— which is also how you verify you are pointed at the workspace you meant, not a
different one that happens to be configured.

## When it does not work

| symptom | cause |
|---|---|
| No `wholegrain` tools at all | `WHOLEGRAIN_MCP_URL` unset, or set after launch |
| `401` that never resolves | the workspace id in the URL is not one this account can reach |
| `421` | the API is being reached on a hostname missing from `MCP_EXTRA_ALLOWED_HOSTS` server-side |
| Tools present, every query empty | connected to the wrong workspace — check `get_workspace_info` |

All tools are **read-only**. Nothing in this plugin can modify a workspace.
