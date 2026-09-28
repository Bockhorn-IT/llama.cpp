# Experimental Radeon Pro 5600M mat-vec tuning

Branch: `perf/metal-5600m-mul-mv`, based on `fix/metal-fa-scalar-5600m` for the tested-device property. The incremental change contains mat-vec selection and its tests, not buffer fixes or loading policy.

Based on ai-janitor (hung), [llama.cpp #19600](https://github.com/ggml-org/llama.cpp/pull/19600). That earlier PR reports a different GPU and is not a substitute for this fork's measurements.

The existing small-batch `mul_mv_ext` kernel is also selected for batches 14 through 512 on the validated Intel/5600M combination. Batches 9-13 and batches above 512 retain upstream selection. Correctness tests include 13/14 and 512/513 to cover both dispatch boundaries. This selection uses a hardware property, not the enablement state of Flash Attention.

## Measured evidence

Historical comparisons used Q4_K x F32 with 3584 x 3584 weights. Relative gains over the fallback were approximately 4 percent at batch 14, 12 percent at 16, and 15 percent at 512. Thirteen cases from batch 9 through 512 passed CPU comparison before the upper-bound case was added.

The publication snapshot passed 14/14 selected Q4_K cases against CPU, including batch 513, with Metal API Validation enabled. This verifies the selected correctness cases, not throughput for all eligible types.

These measurements do not establish a universal crossover for other matrix shapes or the other eligible types (including F32, F16, BF16, and other quantizations). This branch remains experimental. It is not ready to be represented as a generally validated optimization or silently enabled on additional hardware.

```sh
./build/bin/test-backend-ops -b MTL0 -o MUL_MAT -p 'type_a=q4_K,type_b=f32,m=3584,n=(9|10|11|12|13|14|15|16|32|64|128|256|512|513),k=3584'
./build/bin/test-backend-ops perf -b MTL0 -o MUL_MAT -p 'type_a=q4_K,type_b=f32,m=3584'
```

For new performance claims compare against the same integration tree with this change removed, alternate run order, control CPU/GPU temperature and background load, and report distributions. No new temperature-controlled throughput measurements were made during branch separation.
