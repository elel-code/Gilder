# Video Validation

The current visible video direction is native Wayland + native Vulkan.
GStreamer remains the codec/container/audio frontend, but display sinks do not
own the Wayland surface. Do not use the retired GTK or native
`playbin/waylandsink` smoke paths for current validation.

## Validation Layers

Use the checks in this order:

1. Codec smoke: confirms ffmpeg/GStreamer demux and decode availability without
   a Wayland session.
2. Native Vulkan capability probes: confirms Wayland surface, Vulkan present,
   Vulkan Video decode capability, session memory, bitstream buffers and image
   resources.
3. Native Vulkan real Wayland video smokes: confirms decode/import/render/present
   on an actual compositor output.
4. Process sampling: measures CPU, RSS/PSS/USS/private dirty, GPU memory and
   renderer telemetry while a scenario is running.

## Codec Smoke

Run the codec smoke:

```sh
scripts/video-codec-smoke.sh
```

Useful variants:

```sh
scripts/video-codec-smoke.sh --preflight --report-dir /tmp/gilder-video-codec-preflight
scripts/video-codec-smoke.sh --install-missing --work-dir /tmp
scripts/video-codec-smoke.sh --report-dir /tmp/gilder-video-codec-smoke
scripts/video-codec-smoke.sh --allow-missing
scripts/video-codec-smoke.sh --no-convert
scripts/video-codec-smoke.sh --keep
```

This path still uses `playbin` with a headless `fakesink` so it can run in CI
without a compositor. It validates package loading, decode availability,
pipeline lifecycle basics and converter preview generation. It is not visible
presentation evidence.

## Native Vulkan Wayland Smoke

Run inside a real niri, Hyprland or other Wayland session:

```sh
scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --output-name HDMI-A-1 --playback-frames 4800 --target-fps 240
```

Useful variants:

```sh
scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --no-build --source /tmp/loop-h265.mp4 --output-name HDMI-A-1
scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --no-build --output-name HDMI-A-1 --decode-prefix 240 --playback-frames 4800
scripts/native-vulkan-h265-first-frame-video-smoke.sh --output-name HDMI-A-1
scripts/native-vulkan-surface-video-queue-smoke.sh --output-name HDMI-A-1
```

The ready-prefix smoke is a decode/present/resource gate. When
`--playback-frames` is set and `--decode-prefix` is not set, it now generates and
decodes a continuous source long enough for that playback window. This keeps the
first Vulkan Video route comparable with the second GStreamer/appsink route's
continuous 4K/240 source. Passing an explicit shorter `--decode-prefix` keeps
the old loop-window diagnostic mode; loop boundaries can visibly jump unless the
source is authored to be seamless. For full playback validation, the next gate is
continuous demux/parser handoff into the same native Vulkan importer/present path.

## Current Architecture Gates

- GStreamer may provide demux/parser/appsink/audio/clock.
- GStreamer display sinks must not own the visible surface.
- Native Wayland owns layer-shell surface/output/scale/viewport/dmabuf feedback.
- Native Vulkan owns import/decode/render/present.
- NVIDIA importer work may use CUDA interop, but CUDA is not the cross-GPU
  abstraction. AMD/Intel work should target VA/DMABuf -> Vulkan external image.
- Historical native-wgpu and GTK numbers may be used as comparison baselines,
  but those backends are no longer buildable paths.

## Runtime Packages

Ubuntu-like codec smoke packages:

- `ffmpeg`
- `gstreamer1.0-tools`
- `gstreamer1.0-libav`
- `gstreamer1.0-plugins-base`
- `gstreamer1.0-plugins-good`
- `gstreamer1.0-plugins-bad`
- `gstreamer1.0-plugins-ugly`

Arch-like codec smoke packages:

- `ffmpeg`
- `gstreamer`
- `gst-libav`
- `gst-plugin-dav1d`
- `gst-plugins-base`
- `gst-plugins-good`
- `gst-plugins-bad`
- `gst-plugins-ugly`

Native Vulkan Wayland video also needs the host Wayland/Vulkan driver stack and
GStreamer parser/decoder plugins. Arch-like systems typically need:

- `pkgconf`
- `vulkan-headers`
- `vulkan-icd-loader`
- `wayland-protocols`

## Performance Sampling

For an already running daemon, collect resource evidence with:

```sh
scripts/performance-snapshot.sh --label active-video --duration 30 --interval 1 --keep
scripts/performance-snapshot.sh --label paused-video --duration 30 --interval 1 --keep
```

The sampler writes process CPU/RSS/PSS/USS/private dirty/shared summaries,
`memory-mapping-summary.txt`, `memory-mapping-categories.csv`, status snapshots,
decision CSV, telemetry CSV and video-runtime CSV when `gilderctl` is available.
Prefer `Private_Dirty`, USS/private and PSS for process-private memory
regression work; RSS includes shared mappings at full size.

Useful gates include:

```sh
scripts/performance-snapshot.sh \
  --duration 20 \
  --interval 1 \
  --expect-max-private-dirty-kib-at-most 163840 \
  --expect-max-uss-kib-at-most 430080 \
  --expect-video-position-progress \
  --keep
```

On NVIDIA hosts, also use `--expect-max-nvidia-process-gpu-memory-mib-at-most`
when `nvidia-smi` exposes the sampled process.

## Historical Evidence

Historical GTK, native-wgpu and native `playbin/waylandsink` measurements are
kept only as baselines for judging native Vulkan regressions. They are recorded
in `docs/vulkan-migration.md` and the archived M8 note. They are not current
commands and should not be used for new visible-video validation.
