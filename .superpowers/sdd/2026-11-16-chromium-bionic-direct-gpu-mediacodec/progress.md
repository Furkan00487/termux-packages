# SDD ledger — plan: docs/superpowers/plans/2026-11-16-chromium-bionic-direct-gpu-mediacodec.md

Pre-flight: Task 1 (extracted 157.0.8103.78 src + sysroot) -> Task 2 (gpu_init/ozone x11/angle) -> Task 3 (media/gpu/termux) -> Task 4 (launcher + full deb). No interface conflicts; Task 2 & 3 require exact 157.0.8103.78 source inspection before patch generation.
Task 1: Ruling: Installed selective Ubuntu build packages instead of full setup-ubuntu.sh (which pulls texlive/erlang/php onto 18G overlay) and relocated Android SDK/NDK r30 + /data + TERMUX_TOPDIR to /tmp (91G free) — saves ~6 GiB on / overlay without affecting Chromium ARM64 build.
Task 1: complete (commits 17753cf692..58f7c75199, tests: cr-patches --dry-run on 157.0.8103.78 -> fail=0)
Task 2: complete (commits 58f7c75199..29eb4e6622, tests: ninja libEGL libGLESv2 x11 -> EXIT=0)
Task 3: complete (commit $(git rev-parse --short HEAD), patch 1035 verified: filters_jumbo_2.o, default_decoder_factory.o, termux_mediacodec_video_decoder_unittest.o, host/base_jumbo_3.o -> EXIT=0)
Task 4: recipe ready (build.sh amd64-sysroot symlink + no bundled SwiftShader ICD + chromium-launcher.sh.in direct Vulkan X11 flags committed; full build launching in tmux)
