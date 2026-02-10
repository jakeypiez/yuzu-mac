# Copilot Instructions — yuzu macOS (Apple Silicon)

This is a fork of the yuzu Nintendo Switch emulator targeting **macOS on Apple Silicon** (arm64).
The upstream repo was archived; this fork (`jakeypiez/yuzu-mac`) contains all macOS-specific fixes.

## Build

### Prerequisites

- macOS 14+ on Apple Silicon (M1–M5)
- Xcode Command Line Tools
- CMake 3.22+, Ninja
- Qt 5.15 (`brew install qt@5`)
- Homebrew dependencies: `brew install pkg-config sdl2 opus lz4 zstd`
- Do **not** rely on `brew install molten-vk` — the build downloads a known-good universal MoltenVK from the `jakeypiez/MoltenVK` fork automatically.

### Configure & Build

```bash
mkdir -p build && cd build
cmake .. -GNinja -DCMAKE_BUILD_TYPE=Release \
  -DYUZU_USE_BUNDLED_VCPKG=OFF \
  -DUSE_SYSTEM_MOLTENVK=OFF
ninja -j$(sysctl -n hw.ncpu)    # uses all available cores
```

Key CMake options:
- `USE_SYSTEM_MOLTENVK=OFF` (default): Downloads the yuzu-compatible universal MoltenVK binary from `jakeypiez/MoltenVK` releases. This is the recommended setting — homebrew's MoltenVK may be single-arch or an incompatible version.
- `USE_SYSTEM_MOLTENVK=ON`: Uses `find_library()` to locate a system-installed MoltenVK. Only use this if you have a known-good universal build installed.

### Packaging the App Bundle

After building, the app is at `build/bin/yuzu.app`. To package for distribution:

```bash
cd build

# 1. Clean stale Qt frameworks (critical — stale frameworks cause "two sets of Qt binaries" crashes)
rm -rf bin/yuzu.app/Contents/Frameworks/Qt* \
       bin/yuzu.app/Contents/PlugIns \
       bin/yuzu.app/Contents/Resources/qt.conf

# 2. Run macdeployqt (bundles Qt frameworks with correct install names)
/opt/homebrew/opt/qt@5/bin/macdeployqt bin/yuzu.app

# 3. Verify MoltenVK is still the universal binary (macdeployqt should not touch it)
lipo -info bin/yuzu.app/Contents/Frameworks/libMoltenVK.dylib
# Should show: x86_64 arm64

# 4. Ad-hoc codesign
codesign --force --deep --sign - bin/yuzu.app
```

**Critical**: Always clean Qt frameworks before running `macdeployqt`. If you skip step 1, `macdeployqt` will skip already-existing files and leave stale homebrew paths, causing a `createPlatformIntegration()` crash at runtime.

### Creating a DMG

```bash
hdiutil create -volname yuzu -srcfolder build/bin/yuzu.app \
  -ov -format UDZO yuzu-mac.dmg
```

## Testing

Always test before committing or releasing. Use the fullscreen CLI launcher:

```bash
# Nordic Ashes (exercises JIT edge cases, fastmem recovery)
build/bin/yuzu.app/Contents/MacOS/yuzu -f -g "/path/to/Nordic Ashes.nsp"

# TOTK (exercises shader compilation, MoltenVK compatibility)
build/bin/yuzu.app/Contents/MacOS/yuzu -f -g "/path/to/TOTK.xci"
```

Verify:
- Process stays alive for 45+ seconds (`ps aux | grep yuzu`)
- No new crash reports in `~/Library/Logs/DiagnosticReports/yuzu-*.ips`
- stderr is clean (only MoltenVK info boilerplate expected)

Parse crash reports with: `python3 /tmp/parse_crash.py ~/Library/Logs/DiagnosticReports/yuzu-<date>.ips`

## MoltenVK

MoltenVK is the Vulkan-to-Metal translation layer. This project uses a specific binary hosted at `jakeypiez/MoltenVK` release `v1.4.0-yuzu`.

**Why not upstream MoltenVK?**
- Homebrew's `molten-vk` formula installs a single-architecture binary (arm64-only on Apple Silicon). The app bundle needs a universal (arm64 + x86_64) fat binary.
- The official KhronosGroup v1.4.0 release binary causes `sampler attribute parameter is out of bounds` Metal shader compilation errors and `RescaleRenderTargets()` crashes in TOTK.
- The `v1.4.0-yuzu` release contains the specific universal binary that works with yuzu.

The CMake download in `CMakeModules/DownloadExternals.cmake` fetches MoltenVK from `jakeypiez/MoltenVK` and the binary path is set directly in `src/yuzu/CMakeLists.txt` (bypassing `find_library` to avoid accidentally picking up homebrew).

## Architecture & Key Fixes (macOS Fork)

### macOS Apple Silicon build fixes (`10aecdce8a`)
- Initial CMake/build adaptations for arm64 macOS.

### Vulkan/MoltenVK initialization (`ea77787ef7`, `915f62a658`)
- Fixed Vulkan instance creation and device selection on MoltenVK.
- Enabled `VK_KHR_portability_subset` extension required by MoltenVK.

### MoltenVK hardening (`4342d5db07`)
- Non-throwing Vulkan pipeline creation (avoids crashes on shader compile failures).
- Exception-safe shader cache loading.

### svcGetInfo AliasRegionExtraSize (`5f195ab0d5`)
- Implemented `svcGetInfo` id `0x1C` (`AliasRegionExtraSize`), returning 0.
- Fixes crash in Nordic Ashes and other games that query this info type.

### Dynarmic JIT: VectorRotateWholeVectorRight (`6a2fc996c8`)
- Removed `ASSERT` in `emit_arm64_vector.cpp` for the unimplemented `VectorRotateWholeVectorRight` IR opcode.
- Allows the JIT to gracefully fail and fall back to the interpreter for blocks using this opcode.

### FastmemCallback crash fix (`6ce51bca50`)
- In the Mach exception handler context, the fastmem fail path now returns to the dispatcher instead of calling `ASSERT_FALSE`.
- The `ASSERT_FALSE` called `std::terminate()` which crashed the process; the Mach handler can't unwind normally.

### Dynarmic assert → throw (uncommitted, in submodule)
- Changed `mcl::detail::assert_terminate_impl` from `std::terminate()` to `throw std::runtime_error(...)`.
- This allows `GetOrEmit`'s try/catch to catch JIT emission failures (e.g., unimplemented opcodes) instead of aborting.

### Failed JIT block caching (uncommitted, in submodule)
- `GetOrEmit` now caches failed blocks in `block_entries` pointing to `return_to_dispatcher`.
- Prevents infinite retry loops that previously caused millions of fastmem segfault recoveries per second.

### FastmemCallback exception safety (uncommitted, in submodule)
- Wrapped the `FastmemCallback` lambda in a try/catch.
- If an exception propagates during a Mach exception handler callback, it's caught and the dispatcher is returned to instead of crashing.

### Fastmem fail path log silencing (uncommitted, in submodule)
- Removed the two `fmt::print(stderr, ...)` lines in the `FastmemCallback` fail path.
- These were flooding stderr with millions of messages when a failed JIT block's memory accesses hit the fastmem handler.

### GPU/DMA exception safety (`17e9cf4464`)
- Wrapped `ScaleUp()`/`ScaleDown()` in `texture_cache.h` with try-catch to handle `vk::Exception` from failed `vmaCreateImage()` calls (returns `false` instead of propagating).
- Wrapped GPU thread dispatch loop in `gpu_thread.cpp` with try-catch to prevent uncaught exceptions from calling `std::terminate()`.
- Wrapped DMA pusher `Step()` loop in `dma_pusher.cpp` with try-catch.
- Fixes `RescaleRenderTargets()` crash in Nordic Ashes and other games that trigger Vulkan image creation failures.

### Dynarmic JIT: VectorMin/MaxU64/S64 (`17e9cf4464`, in submodule)
- Implemented `VectorMinU64`, `VectorMaxU64`, `VectorMinS64`, `VectorMaxS64` in `emit_arm64_vector.cpp`.
- Uses ARM64 NEON `CMHI` (unsigned) or `CMGT` (signed) + `BSL` (bitwise select) on `D2`/`B16` arrangements.
- Fixes Nordic Ashes infinite loading hang — the game's code uses `VectorMinU64`, which was previously unimplemented, causing the JIT to cache the block as `return_to_dispatcher` and spin at 100% CPU without advancing.

### Dynarmic JIT: opcode emission diagnostics (`17e9cf4464`, in submodule)
- Added try-catch in the main IR emission loop (`emit_arm64.cpp`) that re-throws with the opcode name.
- Error messages now include the specific unimplemented opcode (e.g., `Unimplemented opcode: VectorMinU64`) instead of just `Unimplemented`.

### MoltenVK download from fork (uncommitted)
- Updated `DownloadExternals.cmake` to download from `jakeypiez/MoltenVK` instead of `KhronosGroup/MoltenVK`.
- Updated `src/yuzu/CMakeLists.txt` to use version `v1.4.0-yuzu` and set `MOLTENVK_LIBRARY` directly from the download path (bypasses `find_library`).

## Submodule Workflow

The `externals/dynarmic` submodule contains local changes (in `externals/mcl/src/assert.cpp`, `src/dynarmic/backend/arm64/address_space.cpp`, `src/dynarmic/backend/arm64/emit_arm64.cpp`, and `src/dynarmic/backend/arm64/emit_arm64_vector.cpp`). To commit submodule changes:

```bash
# 1. Commit inside the submodule
cd externals/dynarmic
git add -A && git commit -m "description"

# 2. Update the submodule ref in the parent
cd ../..
git add externals/dynarmic
git commit -m "Update dynarmic submodule"
```

## Repository

- **Fork**: [jakeypiez/yuzu-mac](https://github.com/jakeypiez/yuzu-mac)
- **MoltenVK fork**: [jakeypiez/MoltenVK](https://github.com/jakeypiez/MoltenVK) (release `v1.4.0-yuzu`)
- **Branch**: `macos-build` (pushed to remote `main`)
- **Release**: v1.0.2 DMG on GitHub Releases
