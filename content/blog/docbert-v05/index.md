+++
date = '2026-04-16T16:25:54-03:00'
draft = false
title = 'Docbert v0.5'
+++

[docbert](https://github.com/cfcosta/docbert) is a local document search tool. You point it at your files, it indexes them, and you search from the terminal, a web app, or an MCP-connected agent. Nothing leaves your machine.

v0.5 is mostly a retrieval rewrite. The BM25-then-ColBERT cascade is gone, and semantic search finally has a real index.

## RRF instead of cascade

In v0.4, hybrid search was cascaded: BM25 pulled the top 1000 candidates, ColBERT reranked them. BM25 was the only leg that ever looked at the whole corpus. A document that was semantically close but shared no keywords with the query would never make BM25's top 1000, so ColBERT never reranked it.

v0.5 runs both legs in parallel. BM25 returns its top 100, the semantic leg returns its top 100, and the two lists get fused with Reciprocal Rank Fusion:

```
score(d) = sum over lists L: 1 / (k + rank_L(d))
```

With `k = 60`. A document in both lists adds the contributions from each. A document in only one list still counts, just without the double-dip. `min_score` is only honored in `--bm25-only` mode now, because RRF scores aren't on the BM25 scale. If you want pure keyword search, `--bm25-only` still skips the semantic leg.

## PLAID index for semantic search

The old semantic leg ran ColBERT MaxSim over every embedded document in the collection. That doesn't scale.

v0.5 introduces `docbert-plaid`, a new crate implementing PLAID from Santhanam et al. (2022). Same algorithm as LightOn's Python [fast-plaid](https://github.com/lightonai/fast-plaid), rewritten from scratch in Candle so docbert doesn't have to pull in libtorch.

PLAID clusters every token embedding with k-means to pick coarse centroids. Each token is then stored as `(centroid_id, quantized residual)`, plus an inverted file that maps each centroid back to the documents whose tokens landed in it. At query time, each query token probes its 8 nearest centroids, the inverted file pulls the candidate documents, and MaxSim only runs over those candidates.

The Rust implementation keeps everything on Candle tensors. K-means assignment runs as a single matmul instead of a scalar per-point loop. Search-time MaxSim is also a batched matmul over every IVF candidate at once. Codec tables live on the same device as the embeddings, so there's no PCIe ping-pong per query. CPU and CUDA both work; on Mac, model inference goes through Metal via pylate-rs and PLAID stays on CPU.

Some numbers from the criterion bench suite, scalar Rust vs Candle CPU vs CUDA:

- `assign_points`, n=50k, k=256: 747ms → 71ms → 498µs
- `kmeans/fit`, n=5000, 10 iterations: 596ms → 26ms → 1.7ms
- `search/end_to_end`, 1000 docs × 100 tokens: 170ms → 41ms → 34ms
- Rebuild on a 6M-token corpus with k=256: ~30 minutes → ~3 minutes → ~5 seconds

`docbert sync` updates the PLAID index incrementally: only the chunks that were re-embedded get re-encoded against the existing codec, and the IVF rebuilds in O(n_tokens). The trained centroids and codec tables are reused byte-for-byte. `docbert rebuild` still retrains from scratch if that's what you want.

## Getting started

Download a binary from [GitHub releases](https://github.com/cfcosta/docbert/releases/tag/v0.5.0). Prebuilt binaries for Linux (x86_64, aarch64) and macOS (Apple Silicon), CPU-only and CUDA.

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

After upgrading, run a rebuild:

```bash
docbert rebuild
```

v0.5 requires a PLAID index for all semantic and hybrid search. Without one, docbert returns a "PLAID index missing" error pointing you at `sync` or `rebuild`. There's also a bug fix in `DocumentId`: the numeric id is now masked to 48 bits so it fits the chunk-family bit space the embedding code expects. In v0.4, `ssearch` was returning zero hits on real corpora because the metadata lookup was comparing a masked key against an unmasked one. Existing databases were written with the old 64-bit keys and need the full rebuild to pick up the fix.

Collections and settings are preserved. Only the index and embeddings get regenerated.
