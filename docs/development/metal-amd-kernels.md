# Experimental Intel/AMD Metal kernels

Base: `187664b537aabee60f10ca2e3e791010433b3876`. This branch builds on `fix/metal-amd-fa-vector`. It does not contain private-buffer fixes, automatic loading policy changes, the serial-dispatch workaround, or mat-vec tuning. Use `integration/amd-metal-5600m` for the complete tested configuration.

## Provenance and selection

The scalar attention implementation and AMD workarounds originate from ai-janitor (hung), notably [#19527](https://github.com/ggml-org/llama.cpp/pull/19527) and [#19601](https://github.com/ggml-org/llama.cpp/pull/19601). The original kernel cites Apple MLX `sdpa_vector.h`. This fork adapts the implementation, adds wide-head correctness fixes, and narrows its device scope. AI assistance is disclosed in commit trailers; the human contributor must review and understand the code before any upstream submission.

- The common vector mask reduction uses float instead of half precision. This small change also affects Apple Silicon; absence of a measured regression there is not established locally.
- Additional FA support requires x86_64, the exact `AMD Radeon Pro 5600M` name, SIMD-group reductions, no SIMD-group matrix multiply, and non-unified memory.
- `supports_op` and scalar dispatch use the same device restriction. Other non-MMA GPUs retain upstream's support decision rather than gaining unvalidated vector paths.
- Scalar F16 attention supports K/V head sizes 64/64, 128/128, 256/256, 512/512, and 576/512.
- Dense 576/512, unmasked 512/512, and long-KV multi-query 256/256 forms use scalar accumulation. Normal vector decode and the tested sparse path remain separate.
- Scalar output offsets use 64-bit arithmetic before multiplication. The 32-lane, 16-SIMD-group layout is validated on this device only.

The device flag describes hardware, not a general capability or a promise of support for every AMD GPU. Expanding it requires checking execution width, threadgroup limits, operator correctness, and performance on the added device.

## Evidence and limits

Earlier combined-build tests passed 2,784 supported `FLASH_ATTN_EXT` cases and 240 vector slices against the CPU reference. Before scalar correction, unmasked 512/512 had roughly 2.3-3.3 percent NMSE; long-KV 256/256 also had intermittent failures. These are historical combined-build results, not independent validation of this standalone branch.

After the branch-split hardening, 383/383 selected wide-head F16 cases (512/512 and 576/512) and 240/240 vector slices passed CPU comparison with Metal API Validation. The four 512/512 edge cases (KV lengths 1, 2, 32, 33) also passed separately with a logged `kernel_flash_attn_ext_scalar_dk512_dv512` dispatch. The complete historical matrix has not been rerun for this snapshot.

Ten selected 256/256, KV=4096 cases (query batches 1, 8, 12-19) passed three consecutive repetitions: 30/30 with Metal API Validation. This is a targeted check of the previously intermittent failure region, not a proof that intermittent failures are impossible.

```sh
MTL_DEBUG_LAYER=1 ./build/bin/test-backend-ops -b MTL0 -o FLASH_ATTN_EXT
MTL_DEBUG_LAYER=1 ./build/bin/test-backend-ops -b MTL0 -o FLASH_ATTN_EXT -p 'hsk=(512|576),hsv=512,.*type_K=f16,type_V=f16'
```

Run this command on the integration branch, which includes the private-buffer and synchronization fixes needed for a meaningful end-to-end test on this Mac. CPU reference comparison is part of the test. Unsupported or skipped cases are not passing coverage. No Apple Silicon hardware test, full local CI, or final perplexity qualification is claimed.
