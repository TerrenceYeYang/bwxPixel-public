# bwxPixel

**bwxPixel** is an ultra-low-latency pixel-processing library optimized for NVIDIA GPUs (including the Blackwell `sm_120` architecture) and modern x86-64 CPUs. This repository contains public documentation, benchmark evidence, and prebuilt Windows x64 binaries only. The implementation source code is proprietary and is not included.

## Download

Download the current Windows x64 package (`v0.2.0`) from the [latest GitHub Release](https://github.com/TerrenceYeYang/bwxPixel-public/releases/latest).

The release provides:

- `bwxPixel.dll` — standalone runtime library (`KERNEL32.dll` only; statically linked CUDA and C/C++ runtimes);
- `bwxPixel.lib` — MSVC import library;
- `bwxpixel_api.h` — stable C ABI header (ABI version `1`);
- `SHA256SUMS.txt` — SHA-256 checksums for integrity verification.

## RGBX / RGBA to Grayscale — Verified Benchmarks (`v0.2.0`)

`bwxPixel` converts four-byte RGBX or RGBA pixels to one-byte grayscale pixels using exact fixed-point integer math (`((77*R + 150*G + 29*B) >> 8)`, fourth byte ignored), verified 100% bit-exact across all **16,777,216 (`256³`) RGB combinations**, prime/odd tail lengths, and unaligned ROI pointers.

Benchmark platform: **NVIDIA GeForce RTX 5080** (`SM 12.0`, 84 SMs, 64 MiB L2, 960.1 GB/s theoretical GDDR7 peak) + **Intel Core i7-14700KF** (20 cores / 28 threads, DDR5), Windows 11 Pro x64, CUDA 13.1 / MSVC 19.44. Effective logical traffic is `5 Bytes/pixel` (`4B` read + `1B` write); figures below are medians across 9 independent batches.

| Resolution (Frame Traffic) | Legacy CPU 16MB LUT | CPU Scalar OpenMP | **bwxPixel CPU (SIMD)** | **bwxPixel GPU (Cold GDDR7 >512 MiB Ring)** | **bwxPixel GPU (Hot L2 Stream)** | **bwxPixel GPU (Hot L2 CUDA Graph)** |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **720P** (`4.61 MB`) | `0.1626 ms` (28.3 GB/s) | `0.0377 ms` (122.3 GB/s) | **`0.0067 ms` (686.7 GB/s)** | `0.0082 ms` (564.3 GB/s) | `0.0062 ms` (739.2 GB/s) | **`1.74 µs` (2,654.9 GB/s)** |
| **1MP** (`5.24 MB`) | `0.1874 ms` (28.0 GB/s) | `0.0424 ms` (123.7 GB/s) | **`0.0073 ms` (718.3 GB/s)** | `0.0082 ms` (637.6 GB/s) | `0.0062 ms` (844.1 GB/s) | **`1.80 µs` (2,914.0 GB/s)** |
| **1080P** (`10.37 MB`) | `0.3670 ms` (28.3 GB/s) | `0.0831 ms` (124.8 GB/s) | **`0.0136 ms` (760.9 GB/s)** | `0.0135 ms` (767.0 GB/s) *(Graph: **903.1 GB/s**)* | `0.0064 ms` (1,627.9 GB/s) | **`2.82 µs` (3,676.7 GB/s)** |
| **2K** (`18.43 MB`) | `0.6750 ms` (27.3 GB/s) | `0.1463 ms` (126.0 GB/s) | **`0.0247 ms` (744.9 GB/s)** | **`0.0204 ms` (902.2 GB/s · 94.0% peak)** | `0.0080 ms` (2,308.8 GB/s) | **`4.35 µs` (4,236.4 GB/s)** |
| **4K** (`41.47 MB`) | `1.5120 ms` (27.4 GB/s) | `0.3321 ms` (124.9 GB/s) | **`0.1040 ms` (398.9 GB/s)** | **`0.0469 ms` (884.1 GB/s · 92.1% peak)** | `0.0103 ms` (4,041.1 GB/s) | **`8.47 µs` (4,898.9 GB/s)** |
| **8K** (`165.89 MB`) | `6.1917 ms` (26.8 GB/s) | `3.3741 ms` (49.2 GB/s) | **`2.5875 ms` (64.1 GB/s)** | **`0.1890 ms` (877.6 GB/s · 91.4% peak)** | `0.1852 ms` (895.9 GB/s) | **`183.82 µs` (902.5 GB/s)** |
| **64Mi** (`335.54 MB`) | `12.7551 ms` (26.3 GB/s) | `6.8133 ms` (49.3 GB/s) | **`5.2240 ms` (64.2 GB/s)** | **`0.3850 ms` (871.5 GB/s · 90.8% peak)** | `0.3874 ms` (866.1 GB/s) | **`386.81 µs` (867.5 GB/s)** |

Full methodology, memory-hierarchy boundary notes (Hot L2 vs. Cold GDDR7 vs. PCIe round-trip), and raw batch CSV are available in [docs/BENCHMARK_RTX5080.md](docs/BENCHMARK_RTX5080.md) and [docs/benchmark_rtx5080_results.csv](docs/benchmark_rtx5080_results.csv).

## Requirements

- Windows x64;
- an NVIDIA GPU with compute capability 7.5 or later (`sm_75`, `sm_80`, `sm_86`, `sm_89`, `sm_90`, `sm_100`, `sm_120`, plus PTX forward-compatibility);
- a display driver compatible with CUDA 13.1-generated code.

The DLL statically links its CUDA and C/C++ runtime components (`KERNEL32.dll` is its sole OS import) and accepts arbitrary byte-aligned ROI device pointers safely.

## C API Example

Add `bwxpixel_api.h` to your include path, link against `bwxPixel.lib`, and place `bwxPixel.dll` beside your executable or on the DLL search path.

```cpp
#include "bwxpixel_api.h"

// d_input:  four bytes per pixel (R, G, B, ignored)
// d_output: one byte per pixel
int status = bwxPixelConvertRGBXToGrayGPU(
    d_input,
    d_output,
    num_pixels,
    nullptr); // default CUDA stream

if (status == 0) {
    status = bwxPixelSynchronize(nullptr);
}

if (status != 0) {
    const char* message = bwxPixelGetCudaErrorString(status);
    // Handle the CUDA error.
}
```

The binary reports library version `0.2.0` and ABI version `1` through `bwxPixelGetVersion()` and `bwxPixelGetAbiVersion()`.

## Integrity Verification

On PowerShell, compare the DLL hash with `SHA256SUMS.txt`:

```powershell
Get-FileHash .\bwxPixel.dll -Algorithm SHA256
```

## License & Contact

This repository distributes proprietary binaries and documentation. No implementation source-code license is granted.

For licensing, evaluation, or technical inquiries:
- **Homepage:** <https://terrenceyeyang.github.io/>
- **Email:** <mailto:terrence.ye.yang@gmail.com>

Copyright © 2026 Terrence Ye Yang. All rights reserved.
