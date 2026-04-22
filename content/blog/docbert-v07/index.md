+++
date = '2026-04-22T10:42:40-03:00'
draft = false
title = 'Docbert v0.7'
+++

[docbert](https://github.com/cfcosta/docbert) is a local document search tool. You point it at your files, it indexes them, and you search from the terminal, a web app, or an MCP-connected agent. Nothing leaves your machine.

v0.7 is mostly perf work on the PLAID build, plus a defaults fix that had a bigger impact than most of the algorithm changes combined.

## The whole PLAID build is on the GPU now

v0.6's PLAID build assigned tokens to centroids on the GPU, then fell off a cliff. Bucketize, bit-pack, and farthest-first k-means init all ran as scalar loops on the host. On a 6.78M-token corpus that was roughly 6.78M × 128 scalar iterations plus as many per-token `Vec<u8>` allocations.

The farthest-first (Gonzalez) k-means init was O(k · n · dim) across the whole corpus, so that phase scaled with corpus size. v0.7 follows the ColBERTv2 recipe instead: Lloyd runs on a random `k * 256` sample, seeded from the first `k` rows of that shuffle. Training complexity becomes independent of corpus size. A 6.78M-token corpus now trains in the same time as a 65k-token one, and the farthest-first seeder is deleted entirely.

Residual encode was the bigger one. `ResidualCodec::batch_encode_tokens_on_tensor` now does the residual subtraction, quantile bucketize, and LSB-first bit-pack as tensor ops. Bucketize is iterative `broadcast_ge` + cast + add against each cutoff, because candle doesn't expose a direct `bucketize`. Bit-pack is a sum-reduce against shift weights like `[1, 4, 16, 64]` for `nbits = 2`, which is fast-plaid's `packbits` matmul trick with LSB-first ordering.

On CUDA, search-time decode also stays on the device. §4.5 of the PLAID paper says the decompression belongs on the GPU when one's available, but v0.6 was doing a host round-trip per search anyway. v0.7 splits decode behind a device dispatch: CPU still uses the 256-entry LUT walk (L1-resident, beats candle's `index_select` kernel launch overhead at that scale), CUDA uploads the centroid bank and LUT once per search and stays on-device through the final GEMM.

End-to-end reindex on a 6.78M-token corpus:

- v0.6: 17.1s PLAID build
- After subsample k-means: 10.1s
- After GPU residual encode: 4.3s

About 4× faster overall, and `build_index` at 1000 docs × 100 tokens × k=256 went from 2.74s to 311ms. The build also stopped blowing up VRAM. The old code did one `Tensor::from_slice(&pool, ...)` at the top of `build_index_from_pool`. That's 40 GB for the 6.78M-token LateOn corpus, more than an RTX 3080 Ti holds. v0.7 walks the host pool one 128 MiB tile at a time instead, so peak VRAM is bounded by codec state plus one live tile regardless of corpus size.

## Pruning was quietly turned off

This one is embarrassing. v0.6 shipped the full four-stage PLAID cascade but `docbert-core` was calling `SearchParams` with `centroid_score_threshold=None`, which silently disabled Stage 2 pruning. The ablation in Figure 6 of the paper credits pruning with most of the CPU speedup, and docbert had it off by default.

v0.7 replaces the hardcoded defaults with a lookup from Table 2 of the paper:

| top_k bucket | nprobe | t_cs | ndocs |
|---|---|---|---|
| ≤ 10 | 1 | 0.50 | 256 |
| ≤ 100 | 2 | 0.45 | 1024 |
| > 100 | 4 | 0.40 | 4096 |

`SearchParams::paper_defaults(top_k)` encodes this and clamps `ndocs` to at least `4 * top_k` so Stage 3's `ndocs/4` shortlist can still return what the caller asked for at large `top_k`. Property tests in `hegel_properties.rs` pin the invariants: pruning always on, `t_cs ∈ [0, 1]`, `ndocs ≥ 4 * top_k`, and monotonicity (larger `top_k` never decreases `nprobe` or `ndocs`, never increases `t_cs`).

On the same 5k × 100 bench from the v0.6 post, the default search path dropped from around 12.6ms (v0.6 default, interaction only) to 3.3ms (v0.7 default, interaction plus pruning). That's roughly 3.8× faster with no recall loss on the existing semantic-search tests. The per-stage bench numbers didn't change. What changed is which row `docbert-core` lands on when it calls PLAID with default params.

v0.7 also L2-normalizes decoded tokens before MaxSim, the same as fast-plaid's `decompress_residuals` tail. ColBERT was trained with unit-norm embeddings, so MaxSim is really cosine similarity, but residual quantization drifts decoded vectors slightly off 1.0 and skews the ranking toward whichever tokens happened to decompress longer.

## New default model: LateOn

docbert's default ColBERT model switched from [`lightonai/ColBERT-Zero`](https://huggingface.co/lightonai/ColBERT-Zero) to [`lightonai/LateOn`](https://huggingface.co/lightonai/LateOn). Same family, both from LightOn, both built on ModernBERT-base at 149M parameters with 128-dim embeddings. LateOn is their newer state-of-the-art supervised checkpoint, trained on a fully open Apache-2.0-compatible dataset that they also [released](https://huggingface.co/datasets/lightonai/embeddings-fine-tuning).

On BEIR (14 datasets), LateOn averages 57.22 NDCG@10, up from ColBERT-Zero's 55.39 and GTE-ModernColBERT-v1's 54.75 despite sharing the same backbone. On the decontaminated BEIR split (12 datasets, with train/eval overlap stripped from the mGTE and LightOn training sets) it reaches 60.36 and holds first place overall. It does this at 149M parameters, beating Jina ColBERT v2 (559M) and Arctic Embed L v2 (568M), both almost 4× its size.

Switching models means the existing embedding database is stale: vectors produced by ColBERT-Zero don't live in the same space as LateOn's. Run `docbert rebuild` to re-embed everything, or set `DOCBERT_MODEL=lightonai/ColBERT-Zero` if you'd rather stay on the old model.

## The rest

A new top-level `docbert reindex` command rebuilds only the PLAID index from whatever's in the embedding database. No re-embedding, no Tantivy work. Useful after any change to the PLAID builder itself (centroid count, codec bit-width, k-means iters) when the stored embeddings are still valid. The build also prints `Pool: N tokens × D dim = X MiB (device: F MiB free / T MiB total)` before uploading, so you see whether the corpus fits before CUDA returns an opaque OOM.

The MCP `docbert_get` and `docbert_multi_get` ranges are breaking. The old `fromLine` / `maxLines` / `maxBytes` are gone, replaced by two mutually-exclusive range pairs: `startLine`/`endLine` (1-indexed inclusive) and `startByte`/`endByte` (0-indexed inclusive, rounded down to UTF-8 boundaries). Line and byte ranges can't be mixed in the same call. `maxBytes` used to reject oversize files; the new API slices them and appends a `[... N more bytes remaining]` footer.

pylate-rs is vendored now as `docbert-pylate` in the workspace. The old external crate had Python bindings, a WASM target, and an autoresearch harness that docbert didn't need; the vendored copy strips all of that and versions in lockstep with the rest of the workspace. Along the way the `Index` storage got flattened from `Vec<Vec<EncodedVector>>` to three contiguous buffers (`doc_centroid_ids`, `doc_residual_bytes`, `doc_offsets`), mirroring fast-plaid's `StridedTensor` layout. Search decode is now `extend_from_slice` per candidate instead of walking nested Vecs.

## Getting started

Download a binary from [GitHub releases](https://github.com/cfcosta/docbert/releases/tag/v0.7.0). Prebuilt binaries for Linux (x86_64, aarch64) and macOS (Apple Silicon), CPU-only and CUDA.

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

The PLAID persistence format didn't change, so v0.6 indexes still load. But if you were on the default model (ColBERT-Zero) and want to pick up LateOn, you need to re-embed:

```bash
docbert rebuild
```

If you had `DOCBERT_MODEL` set to something other than the default, or if you want to keep ColBERT-Zero explicitly (`DOCBERT_MODEL=lightonai/ColBERT-Zero`), the embeddings on disk are still valid. You only need the faster build path and the new default centroid count, which `reindex` handles without touching embeddings or Tantivy:

```bash
docbert reindex
```

Collections, embeddings, and settings are preserved by `reindex`. `rebuild` regenerates embeddings too.
