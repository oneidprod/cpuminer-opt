# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Fork of cpuminer-opt (by JayDDee) supporting 80+ mining algorithms for x86_64 and aarch64. This fork adds ARM64/Android (Termux) build compatibility via a clang forward-declaration fix.

## Building on Android/Termux (aarch64)

```bash
./autogen.sh
CFLAGS="-O2 -march=armv8-a+crypto+sha2+aes -Wall -flax-vector-conversions" \
./configure --with-curl
make -j$(nproc)
```

The existing `build-armv8.sh` script is for cross-compilation from x86 and will not work on Termux directly (it passes `--host`/`--build`/`--target` flags for cross-compiling).

## Building on Linux x86_64

```bash
./autogen.sh
CFLAGS="-O3 -march=native -Wall" ./configure --with-curl
make -j$(nproc)
```

Or use `build.sh` / `build-avx2.sh` for common presets.

## Running

```bash
./cpuminer -a ALGO -o stratum+tcp://POOL:PORT -u WALLET -p x
./cpuminer -c config.json
```

## Key ARM64/Termux Fix Applied

### clang strict mode (algo-gate-api.c)
clang treats implicit function declarations as errors; GCC tolerated them as warnings. Added `extern` forward declarations for all 89 `register_*_algo` functions before `register_algo_gate()`. This fix is backward-compatible — the declarations are redundant but harmless on GCC/x86. If new algorithms are added to the switch in `register_algo_gate()`, add a corresponding `extern bool register_NEWNAME_algo(algo_gate_t* gate);` declaration in the same block.

## Architecture

Each algorithm lives in `algo/<name>/` and exposes a `register_<name>_algo(algo_gate_t*)` function that populates function pointers (hash, scanhash, etc.) into a gate struct. `algo-gate-api.c` dispatches to the correct gate via `register_algo_gate()`.

SIMD optimization layers:
- x86: SSE2 → SSSE3 → AVX → AVX2 → AVX512, selected at runtime via CPU feature detection
- ARM: NEON + AES + SHA2, enabled via `-march=armv8-a+crypto+sha2+aes`

`simd-utils/` contains the portable SIMD abstraction used across algo implementations. `simd-utils/simd-int.h` has ARM-specific inline asm for byte-swap operations.
