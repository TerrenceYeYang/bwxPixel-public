# bwxPixel

**bwxPixel** is a low-latency pixel-processing library optimized for NVIDIA GPUs, including the Blackwell architecture. This repository contains public documentation and prebuilt Windows binaries only. The implementation source code is proprietary and is not included.

## Download

Download the current Windows x64 package from the [latest GitHub Release](https://github.com/TerrenceYeYang/bwxPixel-public/releases/latest).

The release provides:

- `bwxPixel.dll` — the runtime library;
- `bwxPixel.lib` — the MSVC import library;
- `bwxpixel_api.h` — the stable C ABI header;
- `SHA256SUMS.txt` — checksums for release verification.

## RGBX/RGBA to grayscale

The first binary release provides CPU and CUDA entry points for converting four-byte RGBX or RGBA pixels to one-byte grayscale pixels. The fourth input byte is ignored.

Reported benchmark results for 60 frames at 1024 × 1024 pixels:

| Implementation | Time per frame | Reported frame rate |
| --- | ---: | ---: |
| bwxPixel CUDA, internal test | 0.012 ms | 80,500 FPS |
| bwxPixel CUDA, third-party platform | 0.036 ms | 27,960 FPS |
| OpenCV CUDA, third-party platform | 0.180 ms | 5,555 FPS |
| OpenCV CPU, third-party platform | 1.205 ms | 830 FPS |

Performance depends on the GPU, driver, memory layout, transfer strategy, and workload. These figures describe the original benchmark configuration and are not a guarantee for every system.

## Requirements

- Windows x64;
- an NVIDIA GPU with compute capability 7.5 or later;
- a sufficiently recent NVIDIA display driver for CUDA 13.1-generated code.

The DLL statically links its CUDA and C/C++ runtime components and has no redistributable DLL dependency beyond Windows itself. The consuming application is still responsible for providing valid CUDA device pointers to the GPU API, typically through its own CUDA runtime or driver integration.

The release contains native CUDA code for `sm_75`, `sm_80`, `sm_86`, `sm_89`, `sm_90`, `sm_100`, and `sm_120`, plus a PTX fallback.

## C API example

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

GPU conversion is asynchronous with respect to the host. Synchronize through `bwxPixelSynchronize`, the application's CUDA stream, or another appropriate CUDA synchronization mechanism before reading the output.

The binary reports library version `0.1.0` and ABI version `1` through `bwxPixelGetVersion()` and `bwxPixelGetAbiVersion()`.

## Integrity verification

On PowerShell, compare the DLL hash with the release checksum file:

```powershell
Get-FileHash .\bwxPixel.dll -Algorithm SHA256
```

## License

This repository distributes proprietary binaries and documentation. No implementation source-code license is granted. Contact the repository owner for licensing and redistribution terms.

Copyright © 2026 TerrenceYeYang. All rights reserved.
