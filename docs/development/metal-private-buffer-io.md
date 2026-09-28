# Metal private buffer I/O

This branch fixes private-buffer transfers and fill ranges independently of AMD kernel changes. Its base is `187664b537aabee60f10ca2e3e791010433b3876`.

## Scope

- Allocate owned shared staging buffers for uploads and downloads instead of wrapping arbitrary caller pointers with `newBufferWithBytesNoCopy`.
- Keep staging buffers alive until the GPU transfer completes, then release them.
- Require successful command completion before returning or reading back staged data.
- Pass a length, not an end offset, to `NSMakeRange`.
- Return immediately for empty transfers and fills.

The existing shared-buffer path is unchanged. Staging adds a host copy and temporary allocation; this is a correctness change, not a claim that staging is always faster. There is no GPU-name allowlist: the API requirements apply to private buffers generally.

Related independent report: [llama.cpp issue #26949](https://github.com/ggml-org/llama.cpp/issues/26949). That report describes the page-alignment requirement of `newBufferWithBytesNoCopy` on other Intel/AMD configurations. It is not a test result for this branch.

## Regression test

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DGGML_METAL=ON -DGGML_METAL_EMBED_LIBRARY=ON -DGGML_BACKEND_DL=ON -DLLAMA_BUILD_TESTS=ON
cmake --build build --target test-backend-ops -j 4
MTL_DEBUG_LAYER=1 ./build/bin/test-backend-ops -b MTL0 -o BUFFER_IO
```

The test checks a private tensor at a nonzero buffer offset, an unaligned host pointer, a partial upload, a partial fill, zero-length operations, host guard bytes, and preservation of the preceding tensor. It explicitly reports `SKIPPED` if the default Metal buffer is not private; such a run does not validate this fix. Check both the device name and the executed test count, not just the process exit code.

On 2026-09-28, Intel Mac / Radeon Pro 5600M / macOS 26.7 (25G229): the same new test executable reported `BUFFER_IO: FAIL` with the unmodified base Metal library, and passed with the combined fixed library. Both runs used Metal API Validation. The negative control demonstrated a data mismatch, not an allocation assertion; it does not independently attribute the failure to every changed transfer operation.

The independent buffer-only branch also passed 1/1 with a separately rebuilt Metal library and the same test executable. The log confirmed upstream's concurrent-dispatch setting remained enabled, and that no scalar FA library was present. The fix therefore does not depend on the AMD kernel or serial-dispatch branches.

To reproduce the negative control in a disposable checkout, start at the base revision, copy only `tests/test-backend-ops.cpp` from this branch, rebuild, and run the same command. Do not copy the modified Metal implementation into the negative-control checkout. No Apple Silicon run, allocation-failure injection, or GPU-timeout injection has been performed.
