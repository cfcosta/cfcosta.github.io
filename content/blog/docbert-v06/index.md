+++
date = '2026-04-19T14:10:49-03:00'
draft = false
title = 'Docbert v0.6'
+++

[docbert](https://github.com/cfcosta/docbert) is a local document search tool. You point it at your files, it indexes them, and you search from the terminal, a web app, or an MCP-connected agent. Nothing leaves your machine.

v0.6 is mostly PLAID work. The index got the two middle stages from the paper, the residual codec is properly packed, and the decode path uses a lookup table now.

## The full PLAID cascade

Santhanam et al. (2022) describe PLAID as a four-stage cascade. What landed in v0.5 was only stages 1 and 4: probe the IVF, then run exact MaxSim on everything it returned. The two middle stages are where PLAID's speedups actually live, so that's what this release wires in.

The pipeline:

1. Probe the top-`n_probe` centroids per query token. Drop any whose best-any-query score falls below `t_cs` (centroid pruning).
2. Score candidates with approximate MaxSim over only the centroids each token landed in, with the pruned mask applied. Keep the top `ndocs`.
3. Re-score the survivors without the mask. Keep `ndocs/4`, which the paper found empirically to be a good heuristic.
4. Decode residuals and run exact MaxSim on what's left. Return the top `k`.

Decoding residuals is the expensive step. Before this release, every candidate the IVF turned up went through full decode. Now the two approximate passes filter the candidate set down to `ndocs/4` before anything touches residual storage. On a 5k × 100 synthetic bench, end-to-end search drops from 210ms (all candidates decoded) to around 3ms with centroid interaction and pruning both on.

Pruning also bites at the token level inside the approximate scorer: tokens whose centroid is masked don't contribute. A doc that touches both a strong and a weak centroid gets the weak one suppressed rather than dominating the per-query max, which keeps the shortlist ordering closer to what the exact scorer would produce.

## Packed codes and table-based decode

The residual codec was the biggest place docbert was cheating. Tokens stored each dimension's residual as one `u8`, regardless of how many bits the bucket code actually used. v0.6 packs them properly, the way §3.1 of the PLAID paper describes: 1- or 2-bit codes into the LSBs of a byte buffer. At `nbits = 2` and dim 128, each token's residual goes from 128 bytes to 32. A 4× cut that was just sitting there.

Supported widths are now `{1, 2, 4, 8}`, the ones that divide 8 cleanly. `nbits = 3/5/6/7` are rejected at validation. The persistence format bumped to version 2, and v0.5 indexes are rejected with a clear error pointing at `docbert rebuild`.

Decoding got faster too. A `DecodeTable` precomputes every possible unpacked weight sequence for all 256 byte values as a flat `256 × codes_per_byte` f32 table. At `nbits = 2` the whole table is 4 KiB and fits in L1. Per-byte decode becomes one table load plus a centroid add. No bit shifting, no masking, no bucket-weights lookup. The search path builds the table once per call and reuses it across every candidate.

`batch_maxsim` stopped padding. The old code built a `[n_c, max_len, dim]` padded tensor and an additive mask so short docs didn't win. Now every candidate's decoded tokens go into a single `[total_tokens, dim]` buffer; one GEMM against `query.T` produces `[total_tokens, n_q]`, and MaxSim is a scalar reduction over each doc's slice of rows. Ragged corpora don't pay padding overhead anymore.

## Error handling and property tests

The old PLAID code had unwraps and expects scattered across production paths. Any candle tensor failure, codec validation miss, or malformed index on disk would panic the process. The new `PlaidError` covers all of it, with variants for tensor errors (`#[from] candle_core::Error`), invalid codec state, invalid index, and IO failures. The codec, persistence, build, update, and search APIs all return `Result<T, PlaidError>` now, and the internal unwraps on things that were supposed to be infallible got replaced with explicit validation.

The test suite grew along with it. About 40 new property tests, written with [hegeltest](https://docs.rs/hegeltest), cover the distance primitives, k-means, the quantizer, the codec, index construction, search, persistence roundtrips, and incremental update. Composite generators produce valid ColBERT-shape unit-norm corpora and codec dims that pack cleanly, so the tests exercise the real hot paths instead of toy shapes.

Two invariants worth calling out:

- `prop_doc_token_shuffle_preserves_score`: shuffling a document's tokens doesn't change its MaxSim score. MaxSim is a max over positions, so this should hold, but the packed codec touches tokens in sequence and a shuffle-sensitivity would reveal a packing bug.
- `prop_centroid_pruning_result_subset_of_unfiltered`: results from the pruned path are always a subset of the unpruned path. Pruning is allowed to drop recall, but it can't invent documents the unpruned path wouldn't return.

## Getting started

Download a binary from [GitHub releases](https://github.com/cfcosta/docbert/releases/tag/v0.6.0). Prebuilt binaries for Linux (x86_64, aarch64) and macOS (Apple Silicon), CPU-only and CUDA.

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

The persistence format bumped to version 2 for the packed token layout. v0.5 indexes (written in the old unpacked format) are rejected with a clear error asking you to rebuild. Collections and settings are preserved. Only the index and embeddings get regenerated.
