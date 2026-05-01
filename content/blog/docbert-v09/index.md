+++
date = '2026-05-01T11:00:00-03:00'
draft = false
title = 'Docbert v0.9'
+++

[docbert](https://github.com/cfcosta/docbert) is a local document search tool. You point it at your files, it indexes them, and you search from the terminal, a web app, or an MCP-connected agent. Nothing leaves your machine.

v0.9 swaps the storage backend from redb to LMDB so several docbert processes can share one data dir, moves to content-derived chunk identifiers that make the embedding database portable between machines, and tightens the Nix flake so a `bun install` or a `cargo build` doesn't force the next `nix build` to start from scratch.

## Multiple processes can share one data dir now

Until v0.9, only one docbert process could open a given data directory at a time. The underlying storage (redb) was single-process by design: opening the same database files from a second binary while the first was running failed with "already open". That ruled out the multi-binary cases: running `docbert mcp` while `docbert web` was already serving the UI, or pointing two `docbert mcp` instances at the same corpus from two different editors.

v0.9 swaps the storage backend to LMDB. LMDB takes a filesystem-level lock and lets several readers and a single writer share the same data files across processes, which is the property the multi-binary case needs.

The upgrade is in-place. The first time v0.9 opens an old data file it migrates the contents over and renames the original with a `.redb-bak` extension, so an upgrade that turns out broken can be reverted by hand. Once you've confirmed everything works, the backups are safe to delete.

## Embeddings are content-addressed now

The way docbert identified each chunk in `embeddings.db` used to depend on where the chunk lived: the id was derived from the document's path and the chunk's position within the document. So the same paragraph appearing in two different files got embedded twice. Copying `embeddings.db` to another machine was useless too, because the ids on disk wouldn't match anything the destination would compute.

v0.9 rewrites the id as a hash of the chunk's text and the model name. The same chunk in two documents now lands at the same entry, so an embedding gets computed once across the corpus instead of once per document. Edits to a long document only re-encode the chunks whose text changed. And `embeddings.db` is portable now: rsync it to another machine and the destination gets a cache hit on every chunk you've already encoded somewhere else.

The cache is also persistent across deletes. Removing a document drops its bookkeeping but leaves the embedding matrices on disk, so the next document that contains the same chunk text gets a cache hit instead of re-running the encoder.

The on-disk format from v0.8 is keyed under the old scheme, so v0.9 needs to re-chunk and re-embed everything once on upgrade. Install steps below.

## The Nix flake stops trashing the cache on every dev change

Two unrelated bits of `flake.nix` had been treating the whole repo as the source for their derivations. Any local mutation (a `cargo build` filling `target/`, a `bun install` walking through `node_modules/`, a `jj` commit touching the op log) changed the source store path and busted the cache. For the rust derivation that meant the next `nix build` recompiled all 50+ workspace deps from scratch, which on a cold cache is 10-30 minutes.

Both source trees now go through `lib.fileset.toSource` whitelists pinned to the files cargo and bun actually read. Local development stops changing the derivation hash, so a `nix build` after a dev-mode UI build or a `cargo test` is back to a cache hit instead of a full rebuild.

## Getting started

Download a binary from [GitHub releases](https://github.com/cfcosta/docbert/releases/tag/v0.9.0). Prebuilt for Linux (x86_64, aarch64) and macOS (Apple Silicon), CPU-only and CUDA.

Or install through Nix or Cargo:

```bash
# Nix
nix profile install github:cfcosta/docbert

# Nix, for CUDA support (NVIDIA gpus)
nix profile install github:cfcosta/docbert#docbert-cuda

# Nix, for Metal support on Mac OS
nix profile install github:cfcosta/docbert#docbert-metal

# Cargo
cargo install --git https://github.com/cfcosta/docbert

# Cargo, for CUDA support (NVIDIA gpus)
cargo install --git https://github.com/cfcosta/docbert --features cuda
```

If you're upgrading from v0.8, run a rebuild:

```bash
docbert rebuild
```

That one command handles both migrations: the storage swap runs automatically the first time docbert opens its data files (the originals get backed up alongside the new ones with a `.redb-bak` extension), and the rebuild re-chunks and re-embeds everything under the new content-derived ids. Collections, document metadata, and settings are preserved.
