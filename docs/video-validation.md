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
scripts/native-vulkan-h264-bitstream-smoke.sh --no-build
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-bitstream-smoke.sh --no-build --width 3840 --height 2160 --rate 240 --level 5.2 --samples 8
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-first-frame-smoke.sh --no-build --width 3840 --height 2160 --rate 240 --level 5.2 --samples 8
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-idr-prefix-smoke.sh --no-build --width 3840 --height 2160 --rate 240 --level 5.2 --decode-prefix 8 --samples 8
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-ready-prefix-smoke.sh --no-build --width 3840 --height 2160 --rate 240 --level 5.2 --decode-prefix 8 --samples 8
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-ready-prefix-video-smoke.sh --no-build --output-name HDMI-A-1 --decode-prefix 240 --playback-frames 240 --refs 2
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-ready-prefix-video-smoke.sh --no-build --output-name HDMI-A-1 --width 1280 --height 720 --target-fps 60 --level 4.2 --refs 2 --bframes 2 --decode-prefix 180 --playback-frames 180
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --no-build --output-name HDMI-A-1 --decode-prefix 240 --playback-frames 240
scripts/native-vulkan-av1-bitstream-smoke.sh --no-build
scripts/native-vulkan-av1-bitstream-smoke.sh --no-build --bit-depth 10
scripts/native-vulkan-h265-main10-bitstream-smoke.sh --no-build
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-visible-video-smoke.sh --no-build --output-name HDMI-A-1 --target-fps 240 --duration 2
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-av1-visible-video-smoke.sh --no-build --output-name HDMI-A-1 --target-fps 60 --duration 3
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h265-main10-visible-video-smoke.sh --no-build --output-name HDMI-A-1 --target-fps 60 --duration 3
scripts/native-vulkan-h265-first-frame-video-smoke.sh --output-name HDMI-A-1
scripts/native-vulkan-surface-video-queue-smoke.sh --output-name HDMI-A-1
```

The ready-prefix smoke is a decode/present/resource gate. When
`--playback-frames` is set and `--decode-prefix` is not set, it now generates and
decodes a continuous source long enough for that playback window. This keeps the
first Vulkan Video route comparable with the second GStreamer/appsink route's
continuous 4K/240 source. Passing an explicit shorter `--decode-prefix` keeps
the old loop-window diagnostic mode; loop boundaries can visibly jump unless the
source is authored to be seamless. For full playback validation, H.264/H.265
direct smokes now use bounded demux/parser streaming queues; the next performance
gate is decode/present decoupling plus fence/timeline-managed bitstream range
reuse, not ready-prefix spool.
The current visible H.265 path uses a fixed-capacity persistent-mapped bitstream
ring, so valid evidence should report
`bitstream_buffer_strategy=fixed-capacity-persistent-mapped-ring`,
non-zero `bitstream_ring_capacity_bytes`, and increasing/wrapping
`frames[].src_buffer_offset` / `frames[].bitstream_ring_wrap_count`.
The direct H.264 ready-prefix visible path follows the same native Vulkan
presentation contract: GStreamer only supplies parsed H.264 AU buffers, while
Vulkan Video owns `vkCmdDecodeVideoKHR` and native Vulkan owns the Wayland
swapchain. Valid H.264 evidence should include non-zero `presented_frame_count`,
`max_reference_count`/`requested_reference_count` for P frames, DPB slot reuse,
and the fixed-capacity bitstream ring telemetry. H.264/H.265 visible direct
smokes now always use the bounded parser/appsink streaming packet queue;
`--streaming-queue` is only a compatibility no-op and ready-prefix spool is no
longer a maintained input mode. Valid evidence should report
`h264_input_mode=streaming-queue`, non-zero
`h264_packet_queue_pulled_count`, and
`h264_packet_queue_retained_payload_bytes=0` at shutdown.
Latest 2026-06-22 real Wayland arbitrary-entry direct gates on
`WAYLAND_DISPLAY=wayland-1`, `HDMI-A-1`, 3840x2160@240:
H.264 `/tmp/gilder-vulkan-h264-arbitrary-continuous-mmco-wrap` passed with
`decoded_frame_count=480`, `presented_frame_count=480`, `b_frames=317`,
`max_reference_count=4`, `h264_packet_queue_bootstrap_discarded_access_units=155`,
`h264_packet_queue_loop_skip_access_units=155`,
`bitstream_ring_wrap_count=43`, and `average_present_fps=195.617`; H.265
`/tmp/gilder-vulkan-h265-arbitrary-continuous-regression` passed with
`decoded_frame_count=480`, `presented_frame_count=480`, `b_frames=317`,
`max_reference_count=4`, `h265_packet_queue_bootstrap_discarded_access_units=153`,
`h265_packet_queue_loop_skip_access_units=153`, `bitstream_ring_wrap_count=11`,
and `average_present_fps=240.976`. These are decode/present functional gates;
H.264 still needs separate long-duration pacing and memory/CPU sampling before
calling stable 240fps complete.

Latest retained performance snapshots for the same arbitrary-entry functional
sources are:
H.264 `/tmp/gilder-vulkan-h264-arbitrary-performance-keep` passed
`decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=9`, `loop_boundary_reset_count=8`,
`h264_packet_queue_bootstrap_discarded_access_units=155`,
`h264_packet_queue_loop_skip_access_units=155`,
`h264_packet_queue_retained_payload_bytes=0`, `bitstream_ring_wrap_count=214`,
and `average_present_fps=197.51976491979758`; retained smaps evidence in
`/tmp/gilder-vulkan-h264-arbitrary-performance-keep/performance` reported
`RSS/PSS/USS/Private_Dirty max=105144/70095/58636/26924 KiB`, average CPU
`13.30%`, and NVIDIA process GPU memory `130 MiB`. H.265
`/tmp/gilder-vulkan-h265-arbitrary-performance-keep` passed
`decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=9`, `loop_boundary_reset_count=8`,
`h265_packet_queue_bootstrap_discarded_access_units=153`,
`h265_packet_queue_loop_skip_access_units=153`,
`h265_packet_queue_retained_payload_bytes=0`, `bitstream_ring_wrap_count=57`,
and `average_present_fps=240.1502442126708`; retained smaps evidence in
`/tmp/gilder-vulkan-h265-arbitrary-performance-keep/performance` reported
`RSS/PSS/USS/Private_Dirty max=103088/68051/56592/24660 KiB`, average CPU
`10.90%`, and NVIDIA process GPU memory `152 MiB`. The current gate for moving
on to AV1/scene wallpaper work is full H.264/H.265 arbitrary continuous
decode/present with streaming queue, zero retained AU payload, ring reuse,
EOS replay, cleanup, and recorded RSS/PSS/USS/private dirty evidence; H.264
4K/240 stable pacing remains the known performance gap.

The visible codec smokes are native Wayland + native Vulkan presentation gates:
GStreamer owns demux/decode/appsink and may output GPU memory, but it does not
own a display sink or Wayland surface. They validate importer, shader sampling,
swapchain present, output selection and visible pacing. They are not direct
Vulkan Video picture-info decode evidence.

Current visible codec evidence from 2026-06-21:

- H.264 720p/240: `/tmp/gilder-vulkan-visible-h264.dqQnsN`,
  `frames_rendered=480`, `average_render_fps=239.99340618116517`,
  `last_sample_format=NV12`, decoder `nvh264dec`.
- H.264 4K/240 source: `/tmp/gilder-vulkan-visible-h264.K0XXrj`,
  `frames_rendered=240`, `average_render_fps=239.98185473198185`,
  `last_sample_size=[3840,2160]`, decoder `nvh264dec`.
- AV1 640x368/60: `/tmp/gilder-vulkan-visible-av1.fBQmOz`,
  `frames_rendered=180`, `average_render_fps=59.99921519026557`,
  `last_sample_format=NV12`, decoder `nvav1dec`.
- AV1 4K/60 source: `/tmp/gilder-vulkan-visible-av1.yAKhDg`,
  `frames_rendered=60`, `average_render_fps=59.996364880248265`,
  `last_sample_size=[3840,2160]`, decoder `nvav1dec`.
- H.265 Main10 640x368/60: `/tmp/gilder-vulkan-visible-h265-main10.GxYmkr`,
  `frames_rendered=180`, `average_render_fps=59.99883480262852`,
  `last_sample_format=P010_10LE`.
- H.265 Main10 4K/60 source: `/tmp/gilder-vulkan-visible-h265-main10.0nZH7D`,
  `frames_rendered=60`, `average_render_fps=59.99589508085857`,
  `last_sample_size=[3840,2160]`, `last_sample_format=P010_10LE`.

The H.264 first-frame smoke is not a visible playback test, but it is now a real
direct Vulkan Video decode gate. `qtdemux ! h264parse ! appsink` produces Annex-B
access units, the native parser extracts SPS/PPS and IDR slice headers, the
selected AU is uploaded into a `VIDEO_DECODE_SRC_KHR` buffer, Vulkan accepts
`StdVideoH264SequenceParameterSet`/`StdVideoH264PictureParameterSet` via
`VkVideoSessionParametersKHR`, and the first IDR is submitted through
`vkCmdDecodeVideoKHR` with NV12 output readback. Current evidence from
2026-06-21:

- H.264 720p/60 direct bitstream/session-parameters:
  `/tmp/gilder-vulkan-h264-bitstream.iVMCh1`,
  `session_parameters_created=true`, `profile_idc=100`, `level_idc=42`.
- H.264 4K/240 direct bitstream/session-parameters:
  `/tmp/gilder-vulkan-h264-bitstream.fs7CCw`,
  `session_parameters_created=true`, `profile_idc=100`, `level_idc=52`,
  `framerate=240`, `codec_max_level=5.2`.
- H.264 720p/60 direct first-frame decode/readback:
  `/tmp/gilder-vulkan-h264-first-frame.AYMakX`,
  `first_frame_decode.completed=true`, `slice_count=11`,
  `y_plane_nonzero_bytes=921600`, `uv_plane_nonzero_bytes=460800`.
- H.264 4K/240 direct first-frame decode/readback:
  `/tmp/gilder-vulkan-h264-first-frame.lQiwMa`,
  `first_frame_decode.completed=true`, `slice_count=20`,
  `src_buffer_range=217600`, `y_plane_nonzero_bytes=8294400`,
  `uv_plane_nonzero_bytes=4147200`.
- H.264 720p/60 direct first-frame decode plus NV12 shader sampling:
  `/tmp/gilder-vulkan-h264-first-frame.GJildG`,
  `result=first-frame-decode-output-sampled-and-readback-completed`,
  `sample_copied=true`.
- H.264 720p/60 direct all-IDR multi-frame decode/readback:
  `/tmp/gilder-vulkan-h264-idr-prefix.kKR6lh`, `decoded_frame_count=8`,
  `frame_offsets=[0,35072,57088,79104,101376,123648,145920,168192]`,
  `reset_control_count=8`.
- H.264 4K/240 direct all-IDR multi-frame decode/readback:
  `/tmp/gilder-vulkan-h264-idr-prefix.7H4DV3`, `decoded_frame_count=8`,
  `frame_offsets=[0,217600,329216,441600,553984,666624,779264,892160]`,
  `y_plane_nonzero_bytes=8294400`, `uv_plane_nonzero_bytes=4147183`.
- H.264 720p/60 direct ready-prefix visible:
  `/tmp/gilder-vulkan-h264-ready-prefix-video.faL4eZ`,
  `decoded_frame_count=8`, `presented_frame_count=8`,
  `max_reference_count=2`, `stream_dpb_slots=3`.
- H.264 4K/240 direct ready-prefix visible:
  `/tmp/gilder-vulkan-h264-ready-prefix-video.Jy9iXF`,
  `decoded_frame_count=240`, `presented_frame_count=240`,
  `source_extent=[3840,2160]`, `bitstream_buffer_bytes=435200`,
  `video_resource_memory_bytes=37552128`.
- H.264 4K/240 direct ready-prefix visible loop:
  `/tmp/gilder-vulkan-h264-ready-prefix-video.S305L5`,
  `decoded_frame_count=480`, `presented_frame_count=480`,
  `playback_loop_count=2`, `loop_boundary_reset_count=1`,
  `max_reference_count=2`. Average present is still about 212fps, so this is a
  direct visible functionality gate, not yet the final 240fps pacing gate.

The H.264 IDR-prefix smoke proves multiple direct decode submits and aligned
bitstream windows, but it deliberately uses all-IDR input. The H.264
ready-prefix visible smoke now covers IPPP P-frame reference tracking and real
Wayland presentation. The remaining H.264 direct gates are B/reference-list
features, arbitrary continuous GOP supply, audio/clock integration and stable
240fps pacing.
AV1 verifies the next codec front-end stage: demux/parser/appsink produces AV1
temporal units, the native parser extracts sequence-header fields, and Vulkan
accepts the resulting `StdVideoAV1SequenceHeader` via
`VkVideoSessionParametersKHR`. It also requires the selected temporal unit to be
a decode candidate: sequence header plus a frame OBU, or sequence header plus
frame-header/tile-group OBUs.

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

Ubuntu CI for `native-vulkan-renderer` also needs `libxkbcommon-dev`; without
it `smithay-client-toolkit` cannot find `xkbcommon.pc` while building the
Wayland host.

Current direct Vulkan Video streaming evidence from 2026-06-21 on
`WAYLAND_DISPLAY=wayland-1`, output `HDMI-A-1`, 3840x2160@240:

- H.264 direct Vulkan Video streaming queue:
  `/tmp/gilder-vulkan-h264-ci-fix-smoke`,
  `decoded_frame_count=2400`, `presented_frame_count=2400`,
  `average_present_fps=214.29452814312305`, `queue_retained=0`.
  Matching smaps evidence:
  `/tmp/gilder-vulkan-h264-ci-fix-smaps-keep/performance`,
  `RSS/PSS/USS/Private_Dirty max=112080/78517/61032/29176 KiB`,
  average CPU `13.48%`.
- H.265 direct Vulkan Video streaming queue:
  `/tmp/gilder-vulkan-h265-ci-fix-smoke`,
  `decoded_frame_count=2400`, `presented_frame_count=2400`,
  `average_present_fps=238.60528994743973`, `queue_retained=0`.
  Matching smaps evidence:
  `/tmp/gilder-vulkan-h265-ci-fix-smaps-keep/performance`,
  `RSS/PSS/USS/Private_Dirty max=112800/79293/61652/29836 KiB`,
  average CPU `15.84%`.

The same run shows H.264 is still present-limited, not packet-retention limited:
`vkQueuePresentKHR` averages about `4373us` for H.264 versus about `3831us` for
H.265, while both paths report zero retained packet payload and the same
1,036,800-byte bitstream ring.

Current arbitrary-entry H.264/H.265 smokes can capture the same process evidence
inline with the visible Wayland run:

```sh
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-ready-prefix-video-smoke.sh \
  --no-build --output HDMI-A-1 --source /tmp/loop-h264.mp4 \
  --target-fps 240 --decode-prefix 240 --playback-frames 2400 \
  --arbitrary-entry-offset 0.35 --require-loop-skip-replay \
  --performance-snapshot --performance-duration 8 --performance-interval 1
env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h265-ready-prefix-video-smoke.sh \
  --no-build --output HDMI-A-1 --source /tmp/loop-h265.mp4 \
  --target-fps 240 --decode-prefix 240 --playback-frames 2400 \
  --arbitrary-entry-offset 0.35 --require-loop-skip-replay \
  --performance-snapshot --performance-duration 8 --performance-interval 1
```

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
