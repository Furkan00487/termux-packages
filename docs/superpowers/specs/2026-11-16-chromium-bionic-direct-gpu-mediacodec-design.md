# Chromium ARM64 Bionic Direct GPU (Mali Vulkan + AHardwareBuffer) & MediaCodec Design

**Date:** 2026-11-16
**Target Device:** Samsung Galaxy A25 5G (`SM-A256E`, Exynos 1280, ARM Mali-G68 MP4, Android 16 / SDK 36, aarch64)
**Build Environment:** GitHub Codespaces `chromium-bionic-gpu-wr5qrj7r4xxrfg9xw` (`standardLinux32gb`: 4 vCPU, 15 GiB RAM, 108 GiB free on `/tmp`, 19 GiB free on `/workspaces`)
**Repository / Branch:** `Furkan00487/termux-packages` -> `.worktrees/feat-chromium-bionic-direct-gpu` (`feat/chromium-bionic-direct-gpu`)

## 1. Intent & Why

Stock Termux Chromium (`x11-packages/chromium`) compiles for `aarch64-linux-android` (Bionic), but falls back to CPU SwiftShader (`libvk_swiftshader.so`) and software video decoding (`FFmpegVideoDecoder` / `dav1d`), causing UI/video lag. Live RAM probes on the target `SM-A256E` proved that Termux Bionic processes can directly load and execute:
- `/system/lib64/libvulkan.so` -> `Mali-G68` (`vendor=0x13b5`, `Vulkan=1.1.177`)
- `/system/lib64/libandroid.so` -> `AHardwareBuffer_allocate` (`ret=0`)
- `/system/lib64/libmediandk.so` -> `c2.exynos.h264.decoder` (`configure=0, start=0`), `c2.exynos.hevc.decoder`, and `c2.exynos.vp8.decoder`

This design modifies the Termux Chromium recipe and `cr-patches` so Chromium `157.0.8103.78` builds for ARM64 Bionic with direct hardware GPU acceleration and hardware `AMediaCodec` video decoding without installing intermediate packages on the phone during development.

## 2. Goals, Non-Goals, and Anti-Goals

### Goals
- Sync `x11-packages/chromium/build.sh` with `x11-packages/chromium-host-tools/build.sh` (`157.0.8103.78`).
- Configure the Codespace build layout so all multi-GB caches, NDK/SDK, `/data`, and Chromium build/out artifacts reside on `/tmp` (108 GiB free), avoiding `/workspaces` (19 GiB) exhaustion and limiting link concurrency (`concurrent_links = 1`) to fit inside 15 GiB RAM (0 swap).
- Patch Chromium GPU initialization and X11/Vulkan presentation (`gpu/ipc/service/gpu_init.cc`, `gpu/config/gpu_info_collector_linux.cc`, `ui/ozone/platform/x11/`, `third_party/angle`) to load Android system `libvulkan.so` (Mali-G68) directly and support zero-copy `AHardwareBuffer` (`DRI3` modifier `1255` socket transport) / direct GPU compositing instead of SwiftShader.
- Patch Chromium `media/gpu/` to add a Termux Bionic `AMediaCodec` (`libmediandk.so` via `dlopen`/`dlsym`) hardware `VideoDecoder` for H.264, HEVC, and VP8 (`c2.exynos.*`), keeping clean fallback to `dav1d`/FFmpeg NEON for VP9 and AV1 (since `c2.exynos.vp9/av1` are absent on Exynos 1280).
- Update `chromium-launcher.sh.in` with clean default flags (`--ozone-platform=x11`, `--use-gl=angle`, `--use-angle=vulkan`, `--enable-features=Vulkan`, `--ignore-gpu-blocklist`, `--enable-gpu-rasterization`, `--no-sandbox`) while allowing user overrides via `/etc/chromium/*.conf` and `CHROMIUM_USER_FLAGS`.

### Non-Goals
- Modifying `modern-termux-advanced` (explicitly deferred by user).
- Modifying or building VS Code / Electron (deferred until Chromium is validated).
- Installing any test packages or `.deb` artifacts onto the user's phone automatically.

### Anti-Goals
- Silently falling back to `libvk_swiftshader.so` by default when Mali Vulkan initialization can work.
- Depending on `proot-distro` (glibc), `VirGL`, or `Zink`.

## 3. Constraints
- **Storage:** `/workspaces` and `/var/lib/docker` share `/dev/loop4` (19 GiB free). All large build directories (`TERMUX_TOPDIR=/tmp/termux-build`, `ANDROID_HOME=/tmp/android-sdk`, `NDK=/tmp/android-ndk-r30`, `/data -> /tmp/termux-data`) MUST live under `/tmp` (`/dev/sda1`, 108 GiB free).
- **RAM / No Swap:** `CapEff=0` disables `swapon` inside the Codespace container. Available RAM is ~14 GiB. `args.gn` must constrain `concurrent_links = 1` (and build `ninja` with `-j3` or `-j2` at final link targets) to avoid `lld` OOM.
- **API Level (`TERMUX_PKG_API_LEVEL=24`):** `AHardwareBuffer` (`API 26`) and `AImageReader` (`API 24+` / `API 28` for `newWithUsage`) symbols used by Termux patches MUST be resolved dynamically via `dlopen("/system/lib64/libandroid.so")` / `dlopen("/system/lib64/libmediandk.so")` and `dlsym`, or guarded cleanly so `minSdk`/API 24 compilation succeeds without link-time errors.
- **Version Guard:** `x11-packages/chromium/build.sh` fails if `TERMUX_PKG_VERSION` differs from `x11-packages/chromium-host-tools/build.sh` (`157.0.8103.78`).

## 4. Approaches Considered & Chosen Architecture

### Approaches Considered
1. **External Wrapper Reliance (Wait for `vulkan-wsi-layer` + CPU video):** Uses stock Termux patches without touching `media/gpu/` or `x11` swapchain. Rejected because videos remain CPU-bound and external wrappers are required.
2. **Direct In-Tree Chromium Bionic Patches (Chosen):**
   - Keep `libvulkan.so` runtime loading targeting Android `/system/lib64/libvulkan.so` (Mali-G68) instead of SwiftShader ICD.
   - Patch `ui/ozone/platform/x11` / Vulkan surface presentation to allocate `AHardwareBuffer` and present via DRI3 modifier `1255` (`AHARDWAREBUFFER_SOCKET_FD`) compatible with Termux:X11 (`Lorie`), while allowing `gpu_init.cc` / `gpu_info_collector_linux.cc` to succeed without `/dev/dri/renderD*`.
   - Implement a focused `TermuxMediaCodecVideoDecoder` (`media/gpu/termux/`) using `dlopen("libmediandk.so")` for H.264, HEVC, and VP8, emitting `VideoFrame` buffers (zero-copy `AHardwareBuffer` or NV12/I420 mapped frames) into Chromium's media pipeline, falling back to `FFmpegVideoDecoder`/`dav1d` on unsupported codecs (VP9/AV1) or runtime errors.
   - Build via `termux-packages` Ubuntu setup (`scripts/setup-ubuntu.sh` + `scripts/setup-android-sdk.sh` into `/tmp`, avoiding a heavy Docker image eating `/dev/loop4`'s 19 GiB) + `./build-package.sh -I -a aarch64 x11-packages/chromium` (after building/installing `chromium-host-tools` or pre-staging dependencies).

### Why This Approach
- Direct `dlopen` of `/system/lib64/libvulkan.so`, `/system/lib64/libandroid.so`, and `/system/lib64/libmediandk.so` was empirically verified on the user's `SM-A256E`.
- Building natively on the Ubuntu 22.04 Codespace host with `TERMUX_TOPDIR=/tmp/termux-build` and `/data -> /tmp/termux-data` preserves all 108 GiB of `/tmp` for Chromium source + build artifacts instead of crowding the 19 GiB `/dev/loop4` Docker storage.

## 5. Component Breakdown & Verification Plan
1. **Build Environment & Version Sync:** Update `x11-packages/chromium/build.sh` to `157.0.8103.78` (matching `chromium-host-tools`), set up `/tmp` workspace/SDK/NDK/data symlinks, and verify `chromium-host-tools` & prerequisites build/stage cleanly.
2. **GPU & Ozone X11 Direct Mali Patch Set (`1030-*`, `1602-*`):** Patch `gpu_info_collector_linux.cc`, `gpu_init.cc`, `ui/ozone/platform/x11/`, and `third_party/angle` so Android `libvulkan.so` + `AHardwareBuffer` (`1255`) DRI3 presentation works without `/dev/dri/renderD*` or SwiftShader default.
3. **MediaCodec Hardware Video Decoder Patch (`1035-*`):** Add `media/gpu/termux/termux_mediacodec_video_decoder.{h,cc}` wired into `DefaultDecoderFactory` for `kH264`, `kHEVC`, `kVP8`, using `dlopen("libmediandk.so")`.
4. **Build & Launcher Verification:** Update `chromium-launcher.sh.in` and `args.gn` (`concurrent_links = 1`, `use_vulkan = true`, `angle_enable_vulkan = true`), build `aarch64` `.deb` in `/tmp`, verify ELF `NEEDED` (`aarch64` Bionic, no glibc deps, `libvulkan`/`libmediandk` symbol resolution), and run host unit/compile tests for the patched targets before full package assembly.
