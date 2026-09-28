# AMD Metal fork: branch map and validation status

This fork separates general buffer corrections from hardware workarounds and experimental performance tuning. The tested base is upstream `187664b537aabee60f10ca2e3e791010433b3876`, not a claim of validation against every later upstream revision. `master` remains unchanged.

| Branch | Parent | Incremental scope |
| --- | --- | --- |
| `fix/metal-private-buffer-io` | tested upstream base | staging lifetime, fill ranges, transfer completion, buffer regression |
| `fix/metal-non-uma-load-mode` | tested upstream base | automatic loading prefers private VRAM on non-UMA devices |
| `fix/metal-non-uma-dispatch` | tested upstream base | ai-janitor-derived serial dispatch workaround for non-UMA devices |
| `fix/metal-amd-fa-vector` | tested upstream base | float-precision reduction of vector attention masks |
| `fix/metal-fa-scalar-5600m` | vector branch | scalar FA, device gating, wide-head corrections and tests |
| `perf/metal-5600m-mul-mv` | scalar branch | experimental multi-column mat-vec selection and boundary tests |
| `integration/amd-metal-5600m` | merge of the above | combined local test configuration and this branch map |

The integration branch is deliberately not called `develop`: it identifies a specific hardware experiment, not the fork's general development policy. Compare stacked branches against their listed parent to see only that branch's contribution.

## General changes versus device workarounds

The private-buffer API fixes have no 5600M allowlist. The automatic mmap capability follows unified-memory topology: host-pointer import remains available, and explicit mapped loading is not removed. Private loading can consume more VRAM, so residency must still be measured.

Serial dispatch remains a broad non-UMA workaround inherited from [ai-janitor #19527](https://github.com/ggml-org/llama.cpp/pull/19527), not proof that every non-UMA GPU lacks concurrent-dispatch support. Its scope and performance tradeoff require upstream review. The log now calls it a workaround rather than an unsupported hardware feature.

The new scalar FA paths are device-restricted; the common float mask reduction is not. The mat-vec optimization is explicitly experimental and has a 512-column upper bound. General correctness fixes must not inherit the performance branch's weaker evidence.

## Publication checks

Hardware: Intel Mac with Radeon Pro 5600M, macOS 26.7 (25G229). CPU reference backend used for operator comparisons. Metal API Validation enabled for targeted tests. Vulkan was not loaded during these checks. Production server configuration and binaries were not replaced.

- Release build of the updated test target: passed.
- Buffer regression against the unchanged base Metal library: failed as expected.
- Buffer regression and four scalar 512/512 edge cases in the corrected integration tree: 5/5 passed.
- `git diff --check`: passed before branch creation.
- Independently rebuilt buffer-only Metal library: 1/1 buffer regression passed, without other AMD changes.
- Wide-head F16 FA: 383/383 selected cases and 240/240 vector slices passed against CPU.
- Long-KV 256/256 FA: 10 selected cases repeated three times, 30/30 passed against CPU.
- Q4_K mat-vec boundary matrix: 14/14 passed against CPU, including batch 513.

See [buffer details](metal-private-buffer-io.md), [kernel scope](metal-amd-kernels.md), and [performance limitations](metal-amd-performance.md). Earlier model smoke tests and the larger FA matrix are historical evidence, not a substitute for qualification of each new revision. Apple Silicon runtime testing, full CI, perplexity, a broader mat-vec performance matrix, and fault injection remain open. These branches are for collaboration and testing, not a claim that every component is upstream-ready.

No upstream pull request, issue, discussion post, or reviewer reply is created by this publication. The user handles those communications personally. New comments and documentation are in English. Original human provenance and AI assistance are retained explicitly.
