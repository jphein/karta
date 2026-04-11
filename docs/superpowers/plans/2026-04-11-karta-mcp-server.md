# Karta MCP Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a native Rust MCP server in `karta-cli` exposing 7 tools over stdio JSON-RPC, connected to Azure OpenAI gpt-5.4-pro.

**Architecture:** Hand-rolled MCP protocol handler in `main.rs` reads JSON-RPC from stdin, dispatches to `Karta` methods, writes JSON-RPC responses to stdout. Single binary, no external MCP framework.

**Tech Stack:** Rust 2024, tokio, serde_json, karta-core, clap

---

## File Structure

| File | Responsibility |
|------|---------------|
| `crates/karta-cli/Cargo.toml` | Add serde, serde_json dependencies |
| `crates/karta-cli/src/main.rs` | CLI args parsing, Karta init, MCP stdio event loop |
| `crates/karta-cli/src/mcp.rs` | JSON-RPC message types, protocol handler (initialize, tools/list, tools/call, ping) |
| `crates/karta-cli/src/tools.rs` | Tool schemas (JSON) and dispatch: maps tool names to karta-core calls, formats responses |
| `~/Projects/karta/.env` | Azure OpenAI credentials |
| `~/.claude/settings.local.json` | MCP server registration |

---

### Task 1: Add dependencies to karta-cli

**Files:**
- Modify: `crates/karta-cli/Cargo.toml`

- [ ] **Step 1: Add serde and serde_json to Cargo.toml**

Replace the full contents of `crates/karta-cli/Cargo.toml` with:

```toml
[package]
name = "karta-cli"
version.workspace = true
edition.workspace = true
license.workspace = true
description = "MCP server for Karta memory system"

[dependencies]
karta-core = { path = "../karta-core" }
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
dotenvy = "0.15"
```

- [ ] **Step 2: Verify it compiles**

Run: `cargo check -p karta-cli`
Expected: compiles with no errors

- [ ] **Step 3: Commit**

```bash
git add crates/karta-cli/Cargo.toml
git commit -m "feat(mcp): add serde deps to karta-cli"
```

---

### Task 2: MCP protocol types and handler

**Files:**
- Create: `crates/karta-cli/src/mcp.rs`

- [ ] **Step 1: Create mcp.rs with JSON-RPC types and protocol handler**

Create `crates/karta-cli/src/mcp.rs`:

```rust
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};

/// Incoming JSON-RPC 2.0 request.
#[derive(Debug, Deserialize)]
pub struct JsonRpcRequest {
    pub jsonrpc: String,
    pub id: Option<Value>,
    pub method: String,
    #[serde(default)]
    pub params: Value,
}

/// Outgoing JSON-RPC 2.0 response.
#[derive(Debug, Serialize)]
pub struct JsonRpcResponse {
    pub jsonrpc: String,
    pub id: Value,
    pub result: Value,
}

/// Outgoing JSON-RPC 2.0 error response.
#[derive(Debug, Serialize)]
pub struct JsonRpcError {
    pub jsonrpc: String,
    pub id: Value,
    pub error: JsonRpcErrorBody,
}

#[derive(Debug, Serialize)]
pub struct JsonRpcErrorBody {
    pub code: i64,
    pub message: String,
}

impl JsonRpcResponse {
    pub fn new(id: Value, result: Value) -> Self {
        Self {
            jsonrpc: "2.0".to_string(),
            id,
            result,
        }
    }
}

impl JsonRpcError {
    pub fn new(id: Value, code: i64, message: String) -> Self {
        Self {
            jsonrpc: "2.0".to_string(),
            id,
            error: JsonRpcErrorBody { code, message },
        }
    }
}

/// Handle the `initialize` method.
pub fn handle_initialize(id: Value) -> Value {
    let resp = JsonRpcResponse::new(
        id,
        json!({
            "protocolVersion": "2024-11-05",
            "capabilities": {
                "tools": {}
            },
            "serverInfo": {
                "name": "karta",
                "version": env!("CARGO_PKG_VERSION")
            }
        }),
    );
    serde_json::to_value(resp).unwrap()
}

/// Handle `ping`.
pub fn handle_ping(id: Value) -> Value {
    let resp = JsonRpcResponse::new(id, json!({}));
    serde_json::to_value(resp).unwrap()
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cargo check -p karta-cli`
Expected: warning about unused module (not wired into main yet), but no errors

- [ ] **Step 3: Commit**

```bash
git add crates/karta-cli/src/mcp.rs
git commit -m "feat(mcp): add JSON-RPC types and protocol handlers"
```

---

### Task 3: Tool schemas and dispatch

**Files:**
- Create: `crates/karta-cli/src/tools.rs`

- [ ] **Step 1: Create tools.rs with tool schemas and dispatch logic**

Create `crates/karta-cli/src/tools.rs`:

```rust
use std::sync::Arc;
use serde_json::{json, Value};
use karta_core::Karta;
use tracing::error;

/// Return the JSON array of tool schemas for `tools/list`.
pub fn tool_schemas() -> Value {
    json!({
        "tools": [
            {
                "name": "karta_add_note",
                "description": "Ingest a piece of knowledge into Karta. Triggers LLM attribute extraction, embedding, ANN linking, retroactive evolution, and atomic fact decomposition.",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "content": {
                            "type": "string",
                            "description": "The text to ingest as a memory note"
                        },
                        "session_id": {
                            "type": "string",
                            "description": "Optional session ID to group notes into episodes for thematic clustering"
                        }
                    },
                    "required": ["content"]
                }
            },
            {
                "name": "karta_ask",
                "description": "Query Karta with full synthesis. Classifies query mode, does ANN search, episode drilldown, fact retrieval, reranking, multi-hop graph traversal, and LLM synthesis.",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "The question to ask"
                        },
                        "top_k": {
                            "type": "integer",
                            "description": "Max notes to consider (default 5)"
                        }
                    },
                    "required": ["query"]
                }
            },
            {
                "name": "karta_search",
                "description": "Raw retrieval without synthesis. Returns scored notes directly.",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "Search query"
                        },
                        "top_k": {
                            "type": "integer",
                            "description": "Max results (default 5)"
                        }
                    },
                    "required": ["query"]
                }
            },
            {
                "name": "karta_dream",
                "description": "Run background reasoning over the knowledge graph. Produces new inferred notes via deduction, induction, abduction, consolidation, contradiction detection, and episode digests.",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "scope_type": {
                            "type": "string",
                            "description": "Dream scope type (default 'workspace')"
                        },
                        "scope_id": {
                            "type": "string",
                            "description": "Scope identifier (default 'default')"
                        }
                    },
                    "required": []
                }
            },
            {
                "name": "karta_get_note",
                "description": "Retrieve a single note by ID.",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "id": {
                            "type": "string",
                            "description": "Note UUID"
                        }
                    },
                    "required": ["id"]
                }
            },
            {
                "name": "karta_get_links",
                "description": "Get all note IDs linked to a given note.",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "note_id": {
                            "type": "string",
                            "description": "Note UUID"
                        }
                    },
                    "required": ["note_id"]
                }
            },
            {
                "name": "karta_note_count",
                "description": "Total number of notes in the knowledge graph.",
                "inputSchema": {
                    "type": "object",
                    "properties": {},
                    "required": []
                }
            }
        ]
    })
}

/// Format a MemoryNote as JSON for tool responses (omit embedding, evolution_history).
fn format_note(note: &karta_core::note::MemoryNote) -> Value {
    json!({
        "id": note.id,
        "content": note.content,
        "context": note.context,
        "keywords": note.keywords,
        "tags": note.tags,
        "links": note.links,
        "provenance": format!("{:?}", note.provenance),
        "confidence": note.confidence,
        "status": format!("{:?}", note.status),
        "created_at": note.created_at.to_rfc3339(),
        "updated_at": note.updated_at.to_rfc3339(),
    })
}

/// Dispatch a `tools/call` request to the appropriate Karta method.
pub async fn dispatch(karta: &Arc<Karta>, name: &str, args: &Value) -> Value {
    match name {
        "karta_add_note" => {
            let content = match args["content"].as_str() {
                Some(c) => c,
                None => return tool_error("content is required"),
            };
            let result = match args["session_id"].as_str() {
                Some(sid) => karta.add_note_with_session(content, sid).await,
                None => karta.add_note(content).await,
            };
            match result {
                Ok(note) => tool_ok(json!({
                    "note_id": note.id,
                    "context": note.context,
                    "keywords": note.keywords,
                    "tags": note.tags,
                    "links_count": note.links.len(),
                })),
                Err(e) => {
                    error!(error = %e, "add_note failed");
                    tool_error(&format!("add_note failed: {e}"))
                }
            }
        }

        "karta_ask" => {
            let query = match args["query"].as_str() {
                Some(q) => q,
                None => return tool_error("query is required"),
            };
            let top_k = args["top_k"].as_u64().unwrap_or(5) as usize;
            match karta.ask(query, top_k).await {
                Ok(result) => tool_ok(json!({
                    "answer": result.answer,
                    "query_mode": result.query_mode,
                    "notes_used": result.notes_used,
                    "note_ids": result.note_ids,
                    "has_contradiction": result.has_contradiction,
                    "contradiction_injected": result.contradiction_injected,
                })),
                Err(e) => {
                    error!(error = %e, "ask failed");
                    tool_error(&format!("ask failed: {e}"))
                }
            }
        }

        "karta_search" => {
            let query = match args["query"].as_str() {
                Some(q) => q,
                None => return tool_error("query is required"),
            };
            let top_k = args["top_k"].as_u64().unwrap_or(5) as usize;
            match karta.search(query, top_k).await {
                Ok(results) => {
                    let notes: Vec<Value> = results
                        .iter()
                        .map(|r| {
                            let mut v = format_note(&r.note);
                            v.as_object_mut().unwrap().insert(
                                "score".to_string(),
                                json!(r.score),
                            );
                            v
                        })
                        .collect();
                    tool_ok(json!({ "results": notes }))
                }
                Err(e) => {
                    error!(error = %e, "search failed");
                    tool_error(&format!("search failed: {e}"))
                }
            }
        }

        "karta_dream" => {
            let scope_type = args["scope_type"].as_str().unwrap_or("workspace");
            let scope_id = args["scope_id"].as_str().unwrap_or("default");
            match karta.run_dreaming(scope_type, scope_id).await {
                Ok(run) => {
                    let types: Vec<String> = run
                        .dreams
                        .iter()
                        .filter(|d| d.would_write)
                        .map(|d| d.dream_type.as_str().to_string())
                        .collect();
                    tool_ok(json!({
                        "dreams_attempted": run.dreams_attempted,
                        "dreams_written": run.dreams_written,
                        "notes_inspected": run.notes_inspected,
                        "types_produced": types,
                        "total_tokens_used": run.total_tokens_used,
                    }))
                }
                Err(e) => {
                    error!(error = %e, "dream failed");
                    tool_error(&format!("dream failed: {e}"))
                }
            }
        }

        "karta_get_note" => {
            let id = match args["id"].as_str() {
                Some(id) => id,
                None => return tool_error("id is required"),
            };
            match karta.get_note(id).await {
                Ok(Some(note)) => tool_ok(format_note(&note)),
                Ok(None) => tool_ok(json!({ "note": null })),
                Err(e) => {
                    error!(error = %e, "get_note failed");
                    tool_error(&format!("get_note failed: {e}"))
                }
            }
        }

        "karta_get_links" => {
            let note_id = match args["note_id"].as_str() {
                Some(id) => id,
                None => return tool_error("note_id is required"),
            };
            match karta.get_links(note_id).await {
                Ok(links) => tool_ok(json!({ "linked_note_ids": links })),
                Err(e) => {
                    error!(error = %e, "get_links failed");
                    tool_error(&format!("get_links failed: {e}"))
                }
            }
        }

        "karta_note_count" => {
            match karta.note_count().await {
                Ok(count) => tool_ok(json!({ "count": count })),
                Err(e) => {
                    error!(error = %e, "note_count failed");
                    tool_error(&format!("note_count failed: {e}"))
                }
            }
        }

        _ => tool_error(&format!("unknown tool: {name}")),
    }
}

fn tool_ok(content: Value) -> Value {
    json!({
        "content": [{
            "type": "text",
            "text": serde_json::to_string_pretty(&content).unwrap()
        }]
    })
}

fn tool_error(message: &str) -> Value {
    json!({
        "content": [{
            "type": "text",
            "text": message
        }],
        "isError": true
    })
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cargo check -p karta-cli`
Expected: warnings about unused (not wired into main yet), but no errors

- [ ] **Step 3: Commit**

```bash
git add crates/karta-cli/src/tools.rs
git commit -m "feat(mcp): add tool schemas and dispatch for 7 karta tools"
```

---

### Task 4: Main entry point — CLI args, Karta init, stdio loop

**Files:**
- Modify: `crates/karta-cli/src/main.rs`

- [ ] **Step 1: Replace main.rs with full MCP server**

Replace the full contents of `crates/karta-cli/src/main.rs` with:

```rust
mod mcp;
mod tools;

use std::io::{self, BufRead, Write};
use std::path::PathBuf;
use std::sync::Arc;

use clap::Parser;
use serde_json::{json, Value};
use tracing::{debug, error, info};

use karta_core::{Karta, config::KartaConfig};

#[derive(Parser)]
#[command(name = "karta-cli", about = "MCP server for Karta memory system")]
struct Args {
    /// Data directory for LanceDB + SQLite storage
    #[arg(long, default_value = "~/Projects/karta/.karta")]
    data_dir: String,
}

fn expand_tilde(path: &str) -> PathBuf {
    if let Some(rest) = path.strip_prefix("~/") {
        if let Some(home) = dirs_home() {
            return home.join(rest);
        }
    }
    PathBuf::from(path)
}

fn dirs_home() -> Option<PathBuf> {
    std::env::var_os("HOME").map(PathBuf::from)
}

#[tokio::main]
async fn main() {
    // Log to stderr so stdout stays clean for JSON-RPC
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
                .add_directive("karta_cli=info".parse().unwrap())
                .add_directive("karta_core=info".parse().unwrap()),
        )
        .with_writer(io::stderr)
        .init();

    let args = Args::parse();
    let data_dir = expand_tilde(&args.data_dir);

    // Load .env from next to the binary or from data_dir's parent.
    // Don't rely on CWD — Claude Code may launch us from anywhere.
    let env_path = std::env::current_exe()
        .ok()
        .and_then(|p| {
            // binary is at target/release/karta-cli, project root is 3 levels up
            p.ancestors().nth(3).map(|a| a.join(".env"))
        })
        .filter(|p| p.exists())
        .unwrap_or_else(|| data_dir.parent().unwrap_or(&data_dir).join(".env"));
    let _ = dotenvy::from_path(&env_path);

    info!(data_dir = %data_dir.display(), "Starting Karta MCP server");

    // Build config and init Karta
    let mut config = KartaConfig::default();
    config.storage.data_dir = data_dir.to_string_lossy().to_string();

    let karta = match Karta::with_defaults(config).await {
        Ok(k) => Arc::new(k),
        Err(e) => {
            error!(error = %e, "Failed to initialize Karta");
            std::process::exit(1);
        }
    };

    info!("Karta initialized, entering MCP stdio loop");

    // Stdio JSON-RPC loop
    let stdin = io::stdin();
    let stdout = io::stdout();
    let mut stdout = stdout.lock();

    for line in stdin.lock().lines() {
        let line = match line {
            Ok(l) => l,
            Err(e) => {
                error!(error = %e, "Failed to read stdin");
                break;
            }
        };

        let line = line.trim();
        if line.is_empty() {
            continue;
        }

        debug!(line = %line, "Received JSON-RPC message");

        let request: mcp::JsonRpcRequest = match serde_json::from_str(line) {
            Ok(r) => r,
            Err(e) => {
                error!(error = %e, "Invalid JSON-RPC");
                let err = mcp::JsonRpcError::new(
                    Value::Null,
                    -32700,
                    format!("Parse error: {e}"),
                );
                let _ = writeln!(stdout, "{}", serde_json::to_string(&err).unwrap());
                let _ = stdout.flush();
                continue;
            }
        };

        let id = request.id.clone().unwrap_or(Value::Null);

        // Notifications (no id) don't get responses
        if request.id.is_none() {
            debug!(method = %request.method, "Notification (no response needed)");
            continue;
        }

        let response = match request.method.as_str() {
            "initialize" => mcp::handle_initialize(id),

            "ping" => mcp::handle_ping(id),

            "tools/list" => {
                let resp = mcp::JsonRpcResponse::new(id, tools::tool_schemas());
                serde_json::to_value(resp).unwrap()
            }

            "tools/call" => {
                let tool_name = request.params["name"]
                    .as_str()
                    .unwrap_or("");
                let arguments = &request.params["arguments"];
                let result = tools::dispatch(&karta, tool_name, arguments).await;
                let resp = mcp::JsonRpcResponse::new(id, result);
                serde_json::to_value(resp).unwrap()
            }

            _ => {
                let err = mcp::JsonRpcError::new(
                    id,
                    -32601,
                    format!("Method not found: {}", request.method),
                );
                serde_json::to_value(err).unwrap()
            }
        };

        let response_str = serde_json::to_string(&response).unwrap();
        debug!(response = %response_str, "Sending response");
        let _ = writeln!(stdout, "{}", response_str);
        let _ = stdout.flush();
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `cargo check -p karta-cli`
Expected: compiles with no errors

- [ ] **Step 3: Build the release binary**

Run: `cargo build --release -p karta-cli`
Expected: binary at `target/release/karta-cli`

- [ ] **Step 4: Commit**

```bash
git add crates/karta-cli/src/main.rs
git commit -m "feat(mcp): implement MCP stdio server with Karta init and JSON-RPC loop"
```

---

### Task 5: Populate .env with Azure credentials

**Files:**
- Create: `~/Projects/karta/.env`

- [ ] **Step 1: Unlock vault if needed**

Run: `bw status` — if locked, unlock in a Ghostty window.

- [ ] **Step 2: Fetch Azure credentials and write .env**

```bash
# Get the API key from vault
AZURE_KEY=$(bw get password "Azure AI Foundry - Claude")

# Write .env
cat > ~/Projects/karta/.env << EOF
AZURE_OPENAI_API_KEY=${AZURE_KEY}
AZURE_OPENAI_ENDPOINT=https://claud-assistant-resource.cognitiveservices.azure.com
AZURE_OPENAI_API_VERSION=2025-04-01-preview
AZURE_OPENAI_CHAT_MODEL=gpt-5.4-pro
AZURE_OPENAI_EMBEDDING_MODEL=text-embedding-3-small
EOF
```

- [ ] **Step 3: Verify .env is gitignored**

Run: `git status` — `.env` should NOT appear in untracked files (already in `.gitignore`).

---

### Task 6: Smoke test the MCP server

**Files:** none (testing only)

- [ ] **Step 1: Test initialize handshake**

Run:
```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | timeout 30 ./target/release/karta-cli 2>/dev/null | head -1 | python3 -m json.tool
```

Expected: JSON response with `protocolVersion`, `capabilities.tools`, `serverInfo.name = "karta"`.

- [ ] **Step 2: Test tools/list**

Run:
```bash
printf '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}\n{"jsonrpc":"2.0","method":"notifications/initialized"}\n{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}\n' | timeout 30 ./target/release/karta-cli 2>/dev/null | tail -1 | python3 -m json.tool
```

Expected: JSON with 7 tools listed (karta_add_note, karta_ask, karta_search, karta_dream, karta_get_note, karta_get_links, karta_note_count).

- [ ] **Step 3: Test note_count (should be 0 on fresh db)**

Run:
```bash
printf '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}\n{"jsonrpc":"2.0","method":"notifications/initialized"}\n{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"karta_note_count","arguments":{}}}\n' | timeout 30 ./target/release/karta-cli 2>/dev/null | tail -1 | python3 -m json.tool
```

Expected: Response containing `"count": 0`.

- [ ] **Step 4: Commit everything**

```bash
git add crates/karta-cli/
git commit -m "feat(mcp): complete karta MCP server with 7 tools"
```

---

### Task 7: Wire into Claude Code

**Files:**
- Modify: `~/.claude/settings.local.json`

- [ ] **Step 1: Add karta MCP server to Claude Code settings**

Read `~/.claude/settings.local.json`, then add a `mcpServers.karta` entry:

```json
{
  "karta": {
    "command": "/home/jp/Projects/karta/target/release/karta-cli",
    "args": [],
    "env": {}
  }
}
```

If `mcpServers` key doesn't exist yet, create it. Preserve all existing content in the file.

- [ ] **Step 2: Verify Claude Code can see the tools**

Restart Claude Code (or start a new session). The karta tools should appear in the deferred tools list. Test by running:

```
/mcp
```

Expected: `karta` server listed with 7 tools.

- [ ] **Step 3: Commit the settings change**

This file is outside the repo, so no git commit needed. Just verify it works.

---

### Task 8: End-to-end test via Claude Code

**Files:** none (manual verification)

- [ ] **Step 1: Test add_note via MCP tool**

In Claude Code, call `karta_note_count` — should return 0.

Then call `karta_add_note` with content: `"The homelab firewall is gatekeeper at 10.0.6.1 running OpenWrt"`.

Expected: returns note ID, context, keywords, tags, links_count.

- [ ] **Step 2: Test ask via MCP tool**

Call `karta_ask` with query: `"What is the firewall IP?"`.

Expected: synthesized answer mentioning 10.0.6.1 and gatekeeper.

- [ ] **Step 3: Test note_count**

Call `karta_note_count` — should return 1 (or more if dreaming created notes).

- [ ] **Step 4: Tag release and push**

```bash
git push origin main
```
