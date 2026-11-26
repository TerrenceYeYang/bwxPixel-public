# bwxPixel v0.2.0 — RTX 5080 & Intel Core i7-14700KF Benchmark Report

## 1. Platform & Measurement Boundaries

- **System:** Windows 11 Pro x64 native execution
- **GPU:** NVIDIA GeForce RTX 5080 (Blackwell `SM 12.0`, 84 SMs, 64.0 MiB L2 Cache, 256-bit GDDR7 @ 15001 MHz, **960.1 GB/s theoretical VRAM peak**)
- **CPU:** Intel Core i7-14700KF (8P + 12E, 20 cores / 28 threads, 28 MiB L2 + 33 MiB L3, dual-channel DDR5)
- **Toolchain:** CUDA Toolkit 13.1 + Visual Studio 2022 MSVC 19.44
- **Formula:** Exact Q8 fixed-point `((77 * R + 150 * G + 29 * B) >> 8)` (fourth byte ignored)
- **Verification (100% Pass):**
  - Exhaustive bit-exact check across all **16,777,216 (`256³`) RGB combinations** with non-zero alpha bytes and 64-byte pre/post canary guard regions.
  - 24 boundary and prime tail lengths (`0, 1, 2, 3, 31, 33, 63, 65, 1023, 1025, 1000037`) and 16 combinations of misaligned input/output ROI byte offsets, plus C ABI DLL export tests.
- **Methodology:** Every configuration undergoes dedicated warmup followed by **9 independent batches**; reported latency and effective logical throughput (`5 Bytes × numPixels / time`) use the **9-batch median**. Raw batch records are provided in [benchmark_rtx5080_results.csv](benchmark_rtx5080_results.csv).

---

## 2. Summary Results (9-Batch Median)

| Resolution (Traffic) | Legacy CPU 16MB LUT | CPU Scalar OpenMP | **bwxPixel CPU (AVX2)** | **bwxPixel GPU (Cold GDDR7 >512 MiB Ring)** | **bwxPixel GPU (Hot L2 Stream)** | **bwxPixel GPU (Hot L2 CUDA Graph)** |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **720P** (`4.61 MB`) | `0.1626 ms` (28.3 GB/s) | `0.0377 ms` (122.3 GB/s) | **`0.0067 ms` (686.7 GB/s)** | `0.0082 ms` (564.3 GB/s) | `0.0062 ms` (739.2 GB/s) | **`1.74 µs` (2,654.9 GB/s)** |
| **1MP** (`5.24 MB`) | `0.1874 ms` (28.0 GB/s) | `0.0424 ms` (123.7 GB/s) | **`0.0073 ms` (718.3 GB/s)** | `0.0082 ms` (637.6 GB/s) | `0.0062 ms` (844.1 GB/s) | **`1.80 µs` (2,914.0 GB/s)** |
| **1080P** (`10.37 MB`) | `0.3670 ms` (28.3 GB/s) | `0.0831 ms` (124.8 GB/s) | **`0.0136 ms` (760.9 GB/s)** | `0.0135 ms` (767.0 GB/s) *(Graph: **903.1 GB/s**)* | `0.0064 ms` (1,627.9 GB/s) | **`2.82 µs` (3,676.7 GB/s)** |
| **2K** (`18.43 MB`) | `0.6750 ms` (27.3 GB/s) | `0.1463 ms` (126.0 GB/s) | **`0.0247 ms` (744.9 GB/s)** | **`0.0204 ms` (902.2 GB/s · 94.0% peak)** | `0.0080 ms` (2,308.8 GB/s) | **`4.35 µs` (4,236.4 GB/s)** |
| **4K** (`41.47 MB`) | `1.5120 ms` (27.4 GB/s) | `0.3321 ms` (124.9 GB/s) | **`0.1040 ms` (398.9 GB/s)** | **`0.0469 ms` (884.1 GB/s · 92.1% peak)** | `0.0103 ms` (4,041.1 GB/s) | **`8.47 µs` (4,898.9 GB/s)** |
| **8K** (`165.89 MB`) | `6.1917 ms` (26.8 GB/s) | `3.3741 ms` (49.2 GB/s) | **`2.5875 ms` (64.1 GB/s)** | **`0.1890 ms` (877.6 GB/s · 91.4% peak)** | `0.1852 ms` (895.9 GB/s) | **`183.82 µs` (902.5 GB/s)** |
| **64Mi** (`335.54 MB`) | `12.7551 ms` (26.3 GB/s) | `6.8133 ms` (49.3 GB/s) | **`5.2240 ms` (64.2 GB/s)** | **`0.3850 ms` (871.5 GB/s · 90.8% peak)** | `0.3874 ms` (866.1 GB/s) | **`386.81 µs` (867.5 GB/s)** |

---

## 3. Regime Interpretation & Honest Boundaries

1. **Hot L2 Cache Regime (`4.90 TB/s` at 4K):**
   Single-frame sizes up to 4K (`41.47 MB`) fit within RTX 5080's `64 MiB` L2 cache. Reusing device-resident buffers represents chained GPU pipelines (e.g., NVDEC decode → grayscale → downstream vision/inference kernel) and reaches **4,898.9 GB/s** (`8.47 µs/frame`) under CUDA Graph replay or **4,041.1 GB/s** (`10.26 µs/frame`) via standard stream submission. These figures reflect L2 throughput, not external GDDR7 bandwidth.
2. **Cold GDDR7 VRAM Regime (`871.5 – 903.1 GB/s`, `90.8% – 94.1%` of Peak):**
   Cycling across a multi-buffer ring exceeding `512 MiB` (`> 8×` L2 capacity) measures sustained physical GDDR7 bandwidth, reaching **902.2 GB/s** (`94.0%` of the `960.1 GB/s` hardware limit) on 2K/1080P and **871.5 – 884.1 GB/s** on 4K–64Mi frames.
3. **Honest CPU Baseline (`760.9 GB/s` Cache / `64.2 GB/s` Cold DDR5):**
   Eliminating the legacy 16 MB lookup table (which suffered severe L1/L2/L3 random-access misses) in favor of register-resident AVX2 SIMD + non-temporal streaming stores above L3 raises CPU throughput by up to **27.3×**, providing a realistic baseline: GPU leads by **10.1× – 12.3×** at 4K L2-resident and **13.5× – 14.1×** on 8K/64Mi DRAM-bound workloads.
4. **PCIe Host-to-Host Round-Trip (`26.2 – 41.7 GB/s`):**
   When frames originate in system RAM and must be copied back immediately after a standalone grayscale pass (`H2D + Kernel + D2H`, 4-stream pinned pipeline), PCIe transfer dominates (`1.055 ms` at 4K vs. `0.104 ms` in-place on CPU). Use the GPU entry point when frames already reside on the device or remain on the GPU for subsequent stages.
