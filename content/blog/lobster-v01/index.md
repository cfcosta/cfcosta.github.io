+++
date = '2026-04-15T15:43:00-03:00'
draft = false
title = 'Lobster v0.1'
+++

[Lobster](https://github.com/cfcosta/lobster) gives Claude Code persistent, per-repo memory. It watches your coding sessions, captures decisions and context, and brings them back when you open the repo again. Everything lives in a `.lobster/` directory at the repo root.

Claude Code forgets everything between sessions. You spend half a conversation explaining your architecture, make a bunch of decisions, and next time it's a blank slate. Lobster keeps that context around.

## Hooks and recall

Lobster plugs into Claude Code via two hooks: `UserPromptSubmit` and `PostToolUse`. Each hook writes the event to a staging directory as JSON and exits. No database locks, no waiting.

On each prompt, recall runs too. Lobster opens a read-only snapshot of the database, searches for relevant memories, and injects a few high-confidence items as a system message. Your top still-valid decisions are always included. Anything matching what you just asked about gets added on top. If the whole thing takes longer than 500ms, it returns nothing rather than blocking you.

## Ingestion

The MCP server is a long-lived process that watches the staging directory via inotify and ingests events into a [redb](https://www.redb.org/) database. It groups events into episodes based on idle gaps and repo transitions, then runs LLM calls (Anthropic or OpenAI) to produce summaries, detect decisions, and extract entities into a semantic graph built on [Grafeo](https://github.com/cfcosta/grafeo).

If the ColBERT model is installed (via [pylate-rs](https://github.com/cfcosta/pylate-rs)), ingestion also generates embeddings for vector search. Without it, retrieval falls back to BM25.

A background loop runs every 60 seconds to retry failed extractions, mine workflow patterns, and flag decisions that have been superseded by newer ones.

## MCP tools

Hooks are the passive side. Claude can also query memory directly through MCP:

- `memory_context` returns ranked decisions, summaries, and entities for a query
- `memory_search` does free-text search with confidence scores
- `memory_recent` lists the newest artifacts
- `memory_decisions` returns the decision timeline with rationale
- `memory_neighbors` walks the semantic graph from an entity node

So Claude gets automatic hints on every prompt, and can dig deeper when it needs the full story behind a decision or wants to trace how things connect.

## Getting started

Grab a binary from [GitHub releases](https://github.com/cfcosta/lobster/releases/tag/v0.1.0). Prebuilt binaries for Linux (x86_64, aarch64) and macOS (Apple Silicon), CPU-only and CUDA.

Or install through Nix or Cargo:

```bash
# Nix
nix profile install github:cfcosta/lobster

# Nix, for CUDA support (NVIDIA gpus)
nix profile install github:cfcosta/lobster-cuda

# Nix, for Metal support on Mac OS
nix profile install github:cfcosta/lobster-metal

# Cargo
cargo install --git https://github.com/cfcosta/lobster

# Cargo, for CUDA support (NVIDIA gpus)
cargo install --git https://github.com/cfcosta/lobster --features cuda

# Cargo, for Metal support on Mac OS
cargo install --git https://github.com/cfcosta/lobster --features metal
```

Set an LLM API key:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
# or
export OPENAI_API_KEY=sk-...
```

Initialize in your repo:

```bash
cd /path/to/your/repo
lobster init
```

This creates `.lobster/`, adds hooks to `.claude/settings.json`, registers the MCP server in `.mcp.json`, and updates `.gitignore`. Existing config is preserved -- `lobster init` only adds its own entries.

Optionally, download the ColBERT model for vector search:

```bash
lobster install
```

Restart Claude Code. From there, every session builds memory that the next one can use.
