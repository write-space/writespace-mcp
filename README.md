# Writespace MCP Server

A remote, hosted [Model Context Protocol](https://modelcontextprotocol.io) server for [Writespace](https://writespace.io) — a real-time collaborative document editor. Give Claude, ChatGPT, Gemini, Cursor, or any MCP-speaking agent a shared docs workspace it can read, write, organise and search alongside you.

**Endpoint:** `https://app.writespace.io/mcp` (HTTP transport, Bearer auth)
**Free tier:** yes — up to 10 documents, no card required → [sign up](https://app.writespace.io/signup)

## What it does

Writespace is a markdown-first doc editor for humans with an MCP server built in for agents. Every action you can take in the app is exposed as a tool your model can call:

- **Read** — walk the workspace tree, fetch any doc as lossless JSON _or_ markdown, run ranked full-text search (drop it straight into your agent as RAG retrieval)
- **Write** — create docs, append content, replace a single section by its heading, shallow-merge structured metadata
- **Organise** — create/rename/move folders, move docs, query docs by metadata (`{ status: "accepted" }`)

Agent writes broadcast over Realtime, so open editors reload live. Humans and agents work on the same canvas — no export step, no format negotiation.

## Quick start

Three steps, two minutes:

1. **Get a token** — sign in at [app.writespace.io](https://app.writespace.io), open **Settings → API tokens**, create a token (shown once, revocable any time).
2. **Endpoint** — one hosted endpoint for all accounts: `https://app.writespace.io/mcp`
3. **Configure your client** — snippets below.

### Claude Desktop

`~/Library/Application Support/Claude/claude_desktop_config.json`

Claude Desktop's config supports only **local (stdio)** servers, so a remote HTTP
server is bridged through [`mcp-remote`](https://www.npmjs.com/package/mcp-remote),
which Claude Desktop launches locally and which forwards to the endpoint:

```json
{
  "mcpServers": {
    "writespace": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://app.writespace.io/mcp",
        "--header",
        "Authorization:${AUTH_HEADER}"
      ],
      "env": {
        "AUTH_HEADER": "Bearer ws_pat_YOUR_TOKEN_HERE"
      }
    }
  }
}
```

Notes:

- The token goes in the `AUTH_HEADER` env var (and `--header Authorization:${AUTH_HEADER}` with **no space** after the colon) because `mcp-remote` mishandles spaces in `--header` arguments.
- Claude Desktop launches with a minimal `PATH`. If `npx` isn't found, use its absolute path (e.g. `/usr/local/bin/npx`, or your nvm path) — and add a matching `"PATH"` entry to `env` if it relies on a versioned `node`.
- Fully quit and reopen Claude Desktop (Cmd-Q) after editing — the config is read only at launch.

### Claude Code (CLI)

```bash
claude mcp add --transport http writespace \
  https://app.writespace.io/mcp \
  --header "Authorization: Bearer ws_pat_YOUR_TOKEN_HERE"
```

### Cursor

Settings → MCP → Add server

```
Type:    HTTP
URL:     https://app.writespace.io/mcp
Header:  Authorization: Bearer ws_pat_YOUR_TOKEN_HERE
```

### Gemini CLI

`~/.gemini/settings.json`

```json
{
  "mcpServers": {
    "writespace": {
      "httpUrl": "https://app.writespace.io/mcp",
      "headers": {
        "Authorization": "Bearer ws_pat_YOUR_TOKEN_HERE"
      }
    }
  }
}
```

Any other spec-compliant MCP client works the same way: HTTP transport + `Authorization: Bearer` header.

## Tools

Every tool except `list_workspaces` takes a `workspace_id` — call `list_workspaces` first to discover them.

| Tool              | Description                                                               |
| ----------------- | ------------------------------------------------------------------------- |
| `list_workspaces` | List every workspace you belong to                                        |
| `list_folders`    | List folders, optionally nested under a parent                            |
| `list_docs`       | List documents, optionally scoped to a folder; returns metadata per row   |
| `get_doc`         | Fetch a doc by ID — canonical Tiptap JSON + markdown rendering + metadata |
| `search`          | Ranked full-text search with `<<match>>` snippets                         |
| `query_docs`      | Find docs whose metadata contains a JSON object (Postgres JSONB `@>`)     |
| `create_doc`      | Create a doc from Tiptap JSON or markdown                                 |
| `update_doc`      | Update title, content, or metadata (shallow-merge; `null` deletes a key)  |
| `upsert_doc`      | Create-or-update by title within an optional folder — idempotent          |
| `append_to_doc`   | Append markdown / Tiptap JSON to an existing doc                          |
| `replace_section` | Replace a section's body by its heading text — surgical edits             |
| `delete_doc`      | Delete a doc by ID                                                        |
| `move_doc`        | Move a doc to another folder (`folder_id: null` → root)                   |
| `create_folder`   | Create a folder, optionally nested                                        |
| `rename_folder`   | Rename a folder — IDs stay stable, links don't break                      |
| `delete_folder`   | Delete a folder and its contents                                          |

Full reference with format notes and troubleshooting: [writespace.io/connect](https://writespace.io/connect)

## Design notes

- **Lossless content, two formats** — reads return both `content` (Tiptap JSON) and `content_markdown`; writes accept either. Markdown round-trips for everything the editor supports, including tables, task lists, and mermaid diagrams.
- **Lean write responses** — write tools return only metadata by default, saving 80–95% of the payload on long docs. Pass `include_content: true` to echo the body.
- **Structured metadata** — every doc carries a JSONB `metadata` field. Treat it as queryable frontmatter: write it, shallow-merge it, query by containment.
- **Stable cross-doc links** — reference docs as `writespace://doc/<id>`; IDs survive renames and moves.
- **Live reload** — MCP writes notify open editors over Realtime; they refetch in the same tab.

## Use cases

- [MCP docs server](https://writespace.io/mcp-docs-server) — a docs backend your agents treat as a tool
- [Shared docs for Claude](https://writespace.io/docs-for-claude) — persistent memory and notes for Claude
- [Shared docs for ChatGPT](https://writespace.io/docs-for-chatgpt) — same workspace, different model
- [MCP knowledge base](https://writespace.io/mcp-knowledge-base) — searchable team knowledge any agent can query

## Auth & security

- Per-agent personal access tokens, issued and revoked from Settings — scoped to your account, easy to rotate
- Tokens are shown once at creation
- Standard `Authorization: Bearer ws_pat_…` header; spec-compliant HTTP transport

## Links

- Website: [writespace.io](https://writespace.io)
- App: [app.writespace.io](https://app.writespace.io)
- Setup & full tool reference: [writespace.io/connect](https://writespace.io/connect)
- Contact: hello@writespace.io

---

© 2026 Writespace · Pro is $5/user/month, free tier available.
