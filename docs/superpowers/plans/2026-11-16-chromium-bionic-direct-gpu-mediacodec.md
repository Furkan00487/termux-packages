# Chromium ARM64 Bionic Direct GPU & MediaCodec Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an `aarch64` Termux Bionic Chromium `157.0.8103.78` `.deb` package in Codespaces (`/tmp`) that uses the phone's native Mali-G68 Vulkan/AHardwareBuffer GPU pipeline directly and decodes H.264/HEVC/VP8 video via Android `AMediaCodec` (`libmediandk.so`).

**Architecture:** Use an isolated git worktree of `Furkan00487/termux-packages` on `feat/chromium-bionic-direct-gpu`, redirect all heavy Termux build directories, Android SDK/NDK, and `/data` onto `/tmp` (108 GiB free), and patch Chromium's GPU init, Ozone X11 Vulkan/AHardwareBuffer (`1255`) surface path, and `media/gpu/termux` `AMediaCodec` decoder before compiling with `concurrent_links = 1` for `aarch64`.

**Tech Stack:** Termux `build-package.sh`, Android NDK r30 (`aarch64-linux-android24`), Chromium `157.0.8103.78` (`gn`, `ninja`, `lld`), Vulkan `libvulkan.so`, `AHardwareBuffer` (`libandroid.so`), `AMediaCodec` (`libmediandk.so`), DRI3 modifier `1255`.

**Spec:** `docs/superpowers/specs/2026-11-16-chromium-bionic-direct-gpu-mediacodec-design.md`

## Global Constraints

- Target device: `SM-A256E` (Exynos 1280, ARM Mali-G68 MP4, Android 16, `aarch64`).
- Do NOT install anything onto the user's phone during development/build.
- All large directories (`TERMUX_TOPDIR=/tmp/termux-build`, `ANDROID_HOME=/tmp/android-sdk`, `NDK=/tmp/android-ndk-r30`, `/data -> /tmp/termux-data`) MUST reside under `/tmp` (`/dev/sda1`, 108 GiB free), NOT `/workspaces` or `/var/lib/docker` (`/dev/loop4`, 19 GiB free).
- Container has `CapEff=0` (no `swapon`) and ~14 GiB available RAM: `args.gn` MUST include `concurrent_links = 1` and final link targets must run with bounded `-j` so `lld` does not OOM.
- `x11-packages/chromium/build.sh` version MUST match `x11-packages/chromium-host-tools/build.sh` (`157.0.8103.78`).
- All Android NDK API >= 26/28 symbols (`AHardwareBuffer_*`, `AImageReader_*`, `AMediaCodec_*`) must be loaded via `dlopen`/`dlsym` (from `/system/lib64/libandroid.so` and `/system/lib64/libmediandk.so`) so API 24 linkage never fails.

## Review Focus

- `lld` OOM during `chrome` / `libblink` linking under 15 GiB RAM with zero swap: verify `concurrent_links = 1` and `symbol_level = 0` in generated `out/Release/args.gn` before running full `ninja`.
- `dlopen` path/namespace mismatch on Android 16: verify `libvulkan.so`, `libandroid.so`, and `libmediandk.so` load by soname without hardcoding `/vendor/lib64/...` paths that Bionic linker namespaces block inside apps.
- `AMediaCodec` unsupported codec (VP9/AV1 on Exynos 1280): ensure `TermuxMediaCodecVideoDecoder::IsSupportedConfig` returns `false` for VP9/AV1 so `dav1d`/`FFmpegVideoDecoder` handles them cleanly without playback failure.
- `AHardwareBuffer` DRI3 modifier `1255` handshake deadlock if client/server socket closes early: verify `1`-byte ack check and `fds` cleanup on error paths in the X11 Vulkan surface patch.
- Version guard & patch conflicts against `157.0.8103.78`: run `termux_step_post_get_source` patch application check (`patch --dry-run` / full extract + patch) before starting `ninja`.

---

### Task 1: Configure `/tmp` Build Environment & Sync Chromium `157.0.8103.78`

**Files:**
- Modify: `x11-packages/chromium/build.sh` (sync version to `157.0.8103.78` and SHA256 `435562dbe041c96e7df02791798b2aa0489ed793627e2a9791045c5e5b9f9408`)
- Host environment (Codespace `/tmp`): `/tmp/termux-build`, `/tmp/termux-data`, `/tmp/android-sdk`, `/tmp/android-ndk-r30`

**Interfaces:**
- Produces: Bootstrapped Ubuntu build toolchain, Android SDK/NDK r30 on `/tmp`, `/data` symlinked to `/tmp/termux-data`, extracted & base-patched Chromium `157.0.8103.78` source tree in `/tmp/termux-build/chromium/src`.

- [ ] **Step 1: Symlink `/data` and configure `/tmp` SDK/NDK/build directories**
  Run `sudo mkdir -p /tmp/termux-data /tmp/termux-build /tmp/android-sdk /tmp/android-ndk-r30`, point `/data -> /tmp/termux-data`, export `TERMUX_TOPDIR=/tmp/termux-build`, `ANDROID_HOME=/tmp/android-sdk`, `NDK=/tmp/android-ndk-r30`.
- [ ] **Step 2: Run Termux Ubuntu & Android SDK/NDK setup scripts**
  Run `./scripts/setup-ubuntu.sh` and `./scripts/setup-android-sdk.sh` inside the Codespace worktree, verifying `df -h /workspaces /tmp` keeps `/workspaces` usage low and `clang` + NDK r30 exist in `/tmp/android-ndk-r30`.
- [ ] **Step 3: Sync `x11-packages/chromium/build.sh` to `157.0.8103.78`**
  Update `TERMUX_PKG_VERSION="157.0.8103.78"` and `TERMUX_PKG_SHA256=435562dbe041c96e7df02791798b2aa0489ed793627e2a9791045c5e5b9f9408` in `x11-packages/chromium/build.sh`.
- [ ] **Step 4: Build/stage `chromium-host-tools` and extract `chromium` source**
  Build `chromium-host-tools` (or verify `build-package.sh -I -a aarch64 x11-packages/chromium-host-tools` and `termux_step_get_source` for `chromium`) into `/tmp/termux-build/chromium/src` and verify existing `cr-patches` apply cleanly.
- [ ] **Step 5: Commit version sync**
  `git add x11-packages/chromium/build.sh && git commit -m "chore(chromium): sync version to 157.0.8103.78"`

---

### Task 2: Inspect `157.0.8103.78` GPU/X11 Sources & Write Direct Mali GPU Patch Set

**Files:**
- Create: `x11-packages/chromium-host-tools/cr-patches/1030-chromium-termux-direct-gpu-init.patch`
- Create: `x11-packages/chromium-host-tools/cr-patches/1031-chromium-x11-ahb-vulkan-present.patch`
- Modify: `x11-packages/chromium-host-tools/cr-patches/1601-angle-no-android-jni.patch` (or add `1602-angle-termux-vulkan-dlopen.patch`)
- Modify: `x11-packages/chromium/build.sh` (`args.gn` + packaged Vulkan `.so` handling)

**Interfaces:**
- Consumes: Extracted Chromium `157.0.8103.78` source tree in `/tmp/termux-build/chromium/src`.
- Produces: Cleanly applying patches enabling direct system `libvulkan.so` (`Mali-G68`), bypassing `/dev/dri/renderD*` rejection in `gpu_init.cc`/`gpu_info_collector_linux.cc`, and `AHardwareBuffer` (`1255`) DRI3 Vulkan swapchain presentation in `ui/ozone/platform/x11/`.

- [ ] **Step 1: Inspect exact `157.0.8103.78` source lines**
  Read `gpu/ipc/service/gpu_init.cc`, `gpu/config/gpu_info_collector_linux.cc`, `ui/ozone/platform/x11/x11_surface_factory.cc`, `ui/ozone/platform/x11/vulkan_implementation_x11.cc`, `ui/ozone/platform/x11/vulkan_surface_x11.cc`, and `third_party/angle/src/libANGLE/renderer/vulkan/DisplayVk.cpp`.
- [ ] **Step 2: Create `1030-chromium-termux-direct-gpu-init.patch`**
  Patch `gpu_info_collector_linux.cc` and `gpu_init.cc` under `#if defined(__TERMUX__)` so missing `/dev/dri/renderD*` or PCI IDs do not force software SwiftShader when system `libvulkan.so` enumerates a physical GPU (`Mali-G68`).
- [ ] **Step 3: Create `1031-chromium-x11-ahb-vulkan-present.patch` and ANGLE patch**
  Patch X11 Vulkan presentation (`vulkan_surface_x11.cc` / `vulkan_implementation_x11.cc`) to load system `libvulkan.so` (not bundled `libvk_swiftshader.so`) and support `AHardwareBuffer` (`1255`) DRI3 presentation (`xcb_dri3_pixmap_from_buffers` + `AHardwareBuffer_sendHandleToUnixSocket` via `dlsym("libandroid.so")`) when `VK_KHR_xcb_surface` is unavailable on Lorie X11.
- [ ] **Step 4: Update `x11-packages/chromium/build.sh`**
  Add `concurrent_links = 1`, `use_vulkan = true`, `angle_enable_vulkan = true`, and avoid making `vk_swiftshader_icd.json` the default ICD in the packaged `lib/chromium/` directory (or remove `libvulkan.so.1` override so system/Termux `libvulkan.so` resolves).
- [ ] **Step 5: Dry-run patches & compile `ui/ozone/platform/x11` + `gpu/ipc/service` targets**
  Run `gn gen` and targeted `ninja -C out/Release gpu/ipc/service ui/ozone/platform/x11` to verify zero compile errors, then commit patches.

---

### Task 3: Implement `TermuxMediaCodecVideoDecoder` (`libmediandk.so`) for H.264 / HEVC / VP8

**Files:**
- Create: `x11-packages/chromium-host-tools/cr-patches/1035-chromium-media-termux-mediacodec-decoder.patch`
- Modify: `media/gpu/BUILD.gn`, `media/renderers/default_decoder_factory.cc` (via patch)
- Create (via patch): `media/gpu/termux/termux_mediacodec_video_decoder.h`, `media/gpu/termux/termux_mediacodec_video_decoder.cc`

**Interfaces:**
- Consumes: `dlopen("/system/lib64/libmediandk.so")` & `dlopen("/system/lib64/libandroid.so")`.
- Produces: `std::unique_ptr<media::VideoDecoder> CreateTermuxMediaCodecVideoDecoder(scoped_refptr<base::SequencedTaskRunner> task_runner)` registered ahead of `FFmpegVideoDecoder` for `kH264`, `kHEVC`, `kVP8` (returning unsupported for `kVP9`/`kAV1`).

- [ ] **Step 1: Inspect `media/renderers/default_decoder_factory.cc` and `media/gpu/BUILD.gn` in `157.0.8103.78`**
  Check exact `VideoDecoder` interface signatures (`Initialize`, `Decode`, `Reset`, `GetDecoderType`, `NeedsBitstreamConversion`) in `media/base/video_decoder.h`.
- [ ] **Step 2: Write `TermuxMediaCodecVideoDecoder` patch (`1035-*`)**
  Implement dynamic `AMediaCodec` wrapper (`AMediaCodec_createDecoderByType`, `AMediaFormat`, `queueInputBuffer`, `dequeueOutputBuffer` -> NV12/I420 `VideoFrame` or `AImageReader`/`AHardwareBuffer` output) for `video/avc`, `video/hevc`, and `video/x-vnd.on2.vp8` with clean fallback if codec init fails.
- [ ] **Step 3: Add host/target unit test or compile check for `media/gpu/termux`**
  Compile `ninja -C out/Release media/gpu media/renderers` and verify zero link/compile errors on `aarch64-linux-android24`.
- [ ] **Step 4: Commit MediaCodec patch**
  `git add x11-packages/chromium-host-tools/cr-patches/1035-* && git commit -m "feat(chromium): add Termux AMediaCodec hardware video decoder patch"`

---

### Task 4: Update Launcher, Full `aarch64` Build, and Verify Output `.deb`

**Files:**
- Modify: `x11-packages/chromium/chromium-launcher.sh.in`
- Modify: `x11-packages/chromium/build.sh`

**Interfaces:**
- Consumes: All patches from Tasks 1–3.
- Produces: `/tmp/termux-build/debs/chromium_157.0.8103.78_aarch64.deb` (and `chromium-host-tools`).

- [ ] **Step 1: Update `chromium-launcher.sh.in` default flags**
  Set default `CHROMIUM_FLAGS` (overridable by `/etc/chromium/*.conf` and `CHROMIUM_USER_FLAGS`) with `--ozone-platform=x11 --use-gl=angle --use-angle=vulkan --enable-features=Vulkan --ignore-gpu-blocklist --enable-gpu-rasterization --no-sandbox`.
- [ ] **Step 2: Run full `aarch64` build in Codespace (`/tmp`)**
  Run `TERMUX_TOPDIR=/tmp/termux-build ./build-package.sh -I -a aarch64 x11-packages/chromium` with controlled parallelism (`-j3` / `-j2` for final link).
- [ ] **Step 3: Verify produced `.deb` and ELF binaries**
  Inspect `dpkg -c` on the `.deb`, check `readelf -d chrome | grep NEEDED` (confirm `aarch64`, Bionic `libc.so`/`libdl.so`, no glibc `libc.so.6`, no forced `libvk_swiftshader.so` dependency), and verify `chromium-launcher.sh` syntax (`sh -n`).
- [ ] **Step 4: Commit & report**
  Commit final recipe changes on `feat/chromium-bionic-direct-gpu`, report exact artifact paths in Codespace and optional install steps for the user.
