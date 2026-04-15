+++
date = '2026-04-15T14:47:10-03:00'
draft = false
title = 'Docbert v0.4'
+++

[docbert](https://github.com/cfcosta/docbert) is a local document search tool. You point it at your files, it indexes them, and you get search results back from the terminal, a web app, or through MCP inside your AI agent. Nothing leaves your machine.

v0.4 is a stability release. No new features, but a lot of things that should have worked before now actually do.

**Important:** this release changes how document IDs are generated. After upgrading, run `docbert rebuild` to reindex your collections.

## What changed

- **Document IDs no longer collide.** The old 24-bit short IDs had realistic collision odds at moderate corpus sizes. IDs are now full blake3 hashes, with short prefixes kept only for display (like git SHAs). This is what requires the rebuild.
- **Short IDs auto-extend to stay unique.** When two documents share a prefix, docbert now extends it to the shortest unambiguous length instead of giving up.
- **PDFs work everywhere.** Several code paths were silently failing on PDF files because they tried to read them as text. PDFs now go through proper content extraction in search, retrieval, the CLI, and MCP.
- **Batch uploads are atomic.** If one document in a multi-file upload fails, all previously committed documents in that batch are rolled back, including restoring any overwritten files to their original contents.
- **Deletes and syncs no longer leave orphans.** Operations that touch the index, embeddings, and metadata now run in a safe order so a failure at any step leaves everything consistent.
- **Rebuilding embeddings no longer destroys metadata.** `docbert rebuild --embeddings-only` used to wipe all document and user metadata. It now preserves them.
- **Removing a collection cleans up properly.** Stale Merkle snapshots were left behind, causing re-added collections to report "up to date" even though all indexed data was gone.
- **File extensions are case-insensitive.** Files like `README.MD` or `paper.PDF` were silently skipped during discovery.
- **Ambiguous bare paths are rejected.** If multiple collections contain a file with the same name, docbert now asks you to use `collection:path` syntax instead of silently picking one.
- **Symlinks outside the collection root are rejected.** Previously, symlinks pointing outside the root could be indexed but never read, creating ghost documents in search results.
- **Path traversal is blocked everywhere.** CLI and MCP document access now share the same path safety checks as the web interface.
- **Sync deletes actually work.** Deleted files were not being removed from the search index due to an ID format mismatch between the sync code and Tantivy.

## Upgrading

```
docbert rebuild
```

That's it. Your collections and settings are preserved, only the index and embeddings are regenerated.
