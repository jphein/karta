# Karta MCP Server Design

**Date:** 2026-04-11
**Status:** Approved

## Goal

Build a native Rust MCP server in the `karta-cli` crate, exposing Karta's memory system to Claude Code as 7 MCP tools over stdio JSON-RPC. Connected to Azure OpenAI gpt-5.4-pro.

## Architecture

```
Claude Code  <--stdio JSON-RPC-->  karta-cli binary  -->  karta-core
                                        |
                                        +-- reads .env (Azure creds)
                                        +-- inits Karta::with_defaults()
                                        +-- data at ~/Projects/karta/.karta/
```

### Transport

MCP v2024-11-05 over stdio. JSON-RPC 2.0. No SSE, no HTTP — matches existing MCP servers in JP's stack (speech-to-cli, mempalace).

Hand-rolled protocol handler (~200 lines). Methods to implement:
- `initialize` — return server info + capabilities
- `notifications/initialized` — no-op ack
- `tools/list` — return tool schemas
- `tools/call` — dispatch to karta-core methods
- `ping` — keepalive

### Karta Initialization

On startup:
1. Parse CLI args (`--data-dir` optional, defaults to `~/Projects/karta/.karta/`)
2. Set CWD to the project root (`~/Projects/karta/`) so `dotenvy::dotenv()` finds `.env` regardless of how Claude Code launches the binary
3. Build `KartaConfig` with data_dir
4. Call `Karta::with_defaults(config)` which loads `.env` via dotenvy, initializes LanceDB + SQLite + Azure OpenAI provider
5. Enter stdio JSON-RPC read loop

### Credential Flow

`.env` in project root (`~/Projects/karta/.env`). The binary sets CWD to its project root on startup so dotenvy finds it reliably. Contents:
```
AZURE_OPENAI_API_KEY=<from vault>
AZURE_OPENAI_ENDPOINT=https://claud-assistant-resource.cognitiveservices.azure.com
AZURE_OPENAI_API_VERSION=2025-04-01-preview
AZURE_OPENAI_CHAT_MODEL=gpt-5.4-pro
AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
```

Picked up automatically by `Karta::with_defaults()` → `dotenvy::dotenv()`.

## MCP Tools (7)

### karta_add_note
Ingest a piece of knowledge. Triggers LLM attribute extraction, embedding, ANN linking, retroactive evolution, atomic fact decomposition.

**Parameters:**
- `content` (string, required) — the text to ingest
- `session_id` (string, optional) — groups notes into episodes for thematic clustering

**Returns:** note ID, context, keywords, tags, link count, atomic fact count

**Maps to:** `Karta::add_note()` or `Karta::add_note_with_session()` when session_id provided

### karta_ask
Query with full synthesis. Classifies query mode, does ANN search, episode drilldown, fact retrieval, reranking, multi-hop graph traversal, and LLM synthesis.

**Parameters:**
- `query` (string, required) — the question
- `top_k` (integer, optional, default 5) — max notes to consider

**Returns:** synthesized answer, query mode used, cited note IDs, contradiction flag, abstention flag

**Maps to:** `Karta::ask()`

### karta_search
Raw retrieval without synthesis. Returns scored notes directly.

**Parameters:**
- `query` (string, required) — search query
- `top_k` (integer, optional, default 5) — max results

**Returns:** array of {note_id, content, context, score, provenance, keywords}

**Maps to:** `Karta::search()`

### karta_dream
Run background reasoning over the knowledge graph. Produces new inferred notes via deduction, induction, abduction, consolidation, contradiction detection, episode digests.

**Parameters:**
- `scope_type` (string, optional, default "workspace") — dream scope
- `scope_id` (string, optional, default "default") — scope identifier

**Returns:** dreams attempted, dreams written, types produced, total tokens used

**Maps to:** `Karta::run_dreaming()`

### karta_get_note
Retrieve a single note by ID.

**Parameters:**
- `id` (string, required) — note UUID

**Returns:** full note (content, context, keywords, tags, provenance, confidence, status, timestamps) or null

**Maps to:** `Karta::get_note()`

### karta_get_links
Get all notes linked to a given note.

**Parameters:**
- `note_id` (string, required) — note UUID

**Returns:** array of linked note IDs

**Maps to:** `Karta::get_links()`

### karta_note_count
Total number of notes in the knowledge graph.

**Parameters:** none

**Returns:** integer count

**Maps to:** `Karta::note_count()`

## Data Directory

Default: `~/Projects/karta/.karta/` (within synced/backed-up Projects folder, already gitignored).

Override: `--data-dir <path>` CLI flag.

Contains:
- `karta.db` — SQLite graph store
- `lancedb/` — LanceDB vector store

## Dependencies

Added to karta-cli/Cargo.toml:
- `serde_json` — JSON-RPC message parsing/building
- `serde` with derive — tool parameter deserialization

Already present:
- `karta-core` (path dependency)
- `tokio` (async runtime)
- `tracing-subscriber` (logging)
- `clap` (CLI args for --data-dir)

No external MCP framework crate. Protocol is simple enough to hand-roll.

## Claude Code Configuration

Add to `~/.claude/settings.local.json` under `mcpServers`:
```json
{
  "karta": {
    "command": "/home/jp/Projects/karta/target/release/karta-cli",
    "args": [],
    "env": {}
  }
}
```

## File Changes

| File | Change |
|------|--------|
| `crates/karta-cli/Cargo.toml` | Add serde, serde_json deps |
| `crates/karta-cli/src/main.rs` | MCP server: CLI args, Karta init, stdio JSON-RPC loop, tool dispatch |
| `~/Projects/karta/.env` | Azure OpenAI credentials (populated from vault) |
| `~/.claude/settings.local.json` | Add karta MCP server entry |

## Not Building

- SSE/HTTP transport — Claude Code uses stdio only
- Streaming tool results — MCP stdio doesn't support it
- Web UI — out of scope
