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
Latest 2026-06-22 retained real Wayland arbitrary-entry direct gates on
`WAYLAND_DISPLAY=wayland-1`, `HDMI-A-1`, 3840x2160@240:
H.264 `/tmp/gilder-vulkan-h264-barrier-tightened-final` passed
`decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=9`, `loop_boundary_reset_count=8`,
`h264_packet_queue_bootstrap_discarded_access_units=155`,
`h264_packet_queue_loop_skip_access_units=155`,
`h264_packet_queue_retained_payload_bytes=0`,
`h264_display_handoff_strategy=gpu-copy-to-dual-slot-nv12-display-ring`,
`h264_display_copy_count=2400`, `h264_decode_ahead_submit_count=2399`,
`bitstream_ring_wrap_count=214`, and `average_present_fps=207.34187751641383`;
retained smaps evidence in
`/tmp/gilder-vulkan-h264-barrier-tightened-final/performance` reported
`RSS/PSS/USS/Private_Dirty max=102956/88739/84188/27236 KiB`, average CPU
`14.36%`, and NVIDIA process GPU memory `154 MiB`. H.265
`/tmp/gilder-vulkan-h265-after-h264-barrier-tightened` passed
`decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=9`, `loop_boundary_reset_count=8`,
`h265_packet_queue_bootstrap_discarded_access_units=153`,
`h265_packet_queue_loop_skip_access_units=153`,
`h265_packet_queue_retained_payload_bytes=0`, `bitstream_ring_wrap_count=57`,
and `average_present_fps=239.82864245894595`; retained smaps evidence in
`/tmp/gilder-vulkan-h265-after-h264-barrier-tightened/performance` reported
`RSS/PSS/USS/Private_Dirty max=102456/88200/83636/24684 KiB`, average CPU
`11.35%`, and NVIDIA process GPU memory `152 MiB`.

The H.264 dual-slot display ring removes the previous DPB/read hazard: the same
complex stream now submits every decode-ahead candidate (`2399/2399`) instead of
skipping reference hazards. A follow-up present-overlap run moved H.264 decode
ahead after `vkQueuePresentKHR` and then replaced the per-frame scoped present
thread with a scoped persistent present worker. The retained 2026-06-22 real
Wayland evidence `/tmp/gilder-vulkan-h264-present-worker` on `HDMI-A-1` reports
`decoded_frame_count=2400`, `presented_frame_count=2400`,
`average_present_fps=234.53720838404902`, `h264_decode_ahead_submit_count=2399`,
average `queue_present_us=3975`, average `present_us=4252`,
p50/p90/p99 present `4224/4415/4995us`, and no retained packet payload. Smaps
evidence in `/tmp/gilder-vulkan-h264-present-worker/performance` reports
`RSS/PSS/USS/Private_Dirty max=104972/76817/60048/28580 KiB`, average CPU
`14.77%`, and NVIDIA process GPU memory `128 MiB`.

This is a real improvement over `/tmp/gilder-vulkan-h264-barrier-tightened-final`
(`207.34187751641383fps`, average `queue_present_us=4509`) and keeps the
per-frame thread version's pacing (`/tmp/gilder-vulkan-h264-present-overlap`,
`234.4142733415641fps`) while removing its extra scheduling cost. It does not
yet make complex H.264 4K/240 fully stable at 240fps: the remaining gap is still
the FIFO present/display-copy submit chain, and the dual-slot display ring costs
about `25.6MB` of Vulkan image memory. Treat it as a correctness/perf
experiment, not as the final zero-copy memory target.

Follow-up H.264 ownership work borrowed the same high-level shape used by mature
hardware-video paths: fixed frame/surface pools, explicit slot ownership, command
rings, descriptors updated only when bindings change, and timeline/fence based
handoff. Local Sunshine reference points are its Vulkan hardware-frame path
(`AVHWFramesContext` pool, DMABuf import with explicit modifiers, command ring,
and timeline semaphore handoff to FFmpeg), but Gilder still owns decode and
presentation directly. The 2026-06-22 real Wayland `HDMI-A-1` smoke
`/tmp/gilder-vulkan-h264-display-slot-fence-4k240-ref1` added per-frame acquire
semaphores/fences plus display-slot reuse fences and reports
`decoded/presented=480/480`, `average_present_fps=230.31172461134605`,
`h264_present_result_wait_elapsed_us=1929885`, average fence wait about `0.89us`,
and no retained packet payload. This is a stability/ownership step, not an FPS
breakthrough.

The longer performance run
`/tmp/gilder-vulkan-h264-display-slot-fence-4k240-perf` reports
`decoded/presented=1200/1200`, `average_present_fps=232.89863472099296`,
`RSS/PSS/USS/Private_Dirty max=106000/90291/84616/27544 KiB`, and NVIDIA process
GPU memory `116 MiB`. Its raw CPU average is `36.84%` because the first 0s sample
is `100%`; the following samples are `28.6/20.9/18.0/16.7%`. Treat the memory
result as unchanged relative to the previous display-ring path.

Two deeper present experiments should remain negative evidence: with
`GILDER_H264_ASYNC_PRESENT_DEPTH=2`,
`/tmp/gilder-vulkan-h264-per-frame-fence-depth2-4k240-short-seq` completed but
fell to `219.4879316010344fps` because single-queue present blocking moved into
`avg_submit_us=4175.98`; with `GILDER_H264_PRESENT_QUEUE_COUNT=2`,
`/tmp/gilder-vulkan-h264-per-frame-fence-dual-present-4k240-short-seq` timed out
after 20s and produced empty runtime/stderr. Keep H.264 default at single present
queue/depth 1 until the binary-semaphore path is replaced by a real timeline
frame-pool scheduler.

H.265 Main10 remains the stable control after these H.264-only changes:
`/tmp/gilder-vulkan-h265-main10-after-h264-framepool-fence-4k240` reports
`decoded/presented=480/480`, `average_present_fps=240.3833285970556`, P010
`G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`, and average `queue_present_us=3859`.

The next H.264 low-level experiment is resource-image layout control. With
`GILDER_H264_RESOURCE_LAYOUT=general`, H.264 keeps the decode resource image in
`GENERAL` and copies from `GENERAL` back to `GENERAL`, avoiding the repeated
`VIDEO_DECODE_DPB_KHR -> TRANSFER_SRC_OPTIMAL -> VIDEO_DECODE_DPB_KHR` source
layout churn during display-copy. The runtime and summary now expose
`h264_resource_image_layout`. Real Wayland evidence
`/tmp/gilder-vulkan-h264-resource-general-4k240-ref1` passed `480/480` and
reached `233.11475907497862fps` with lower average decode/record timings, but
the follow-up telemetry run
`/tmp/gilder-vulkan-h264-resource-general-layout-field-4k240-ref1` passed at
`232.52402677308388fps`. Treat `GENERAL` layout as a valid but noisy H.264
synchronization experiment, not as the 240fps fix.

Main10 and AV1 were also rechecked after the H.264 layout work. H.265 Main10
visible 4K/240 evidence
`/tmp/gilder-vulkan-h265-main10-after-h264-general-layout-4k240` reports
`decoded/presented=480/480`, `average_present_fps=239.76366459616204`, and P010
`G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`. AV1 Main10 4K direct first-frame
evidence `/tmp/gilder-vulkan-av1-main10-after-h264-general-layout-4k` reports
`first-frame-decode-output-sampled-and-readback-completed`,
`first_frame_decode_codec=av1-main-10`, P010 readback bytes `24883200`, and
RGBA sampling with `rgba_unique_values=256`.

AV1 has now moved past the old inter-header hard stop. The native parser emits
`reference_order_hints`, `frame_refs_short_signaling`, `last_frame_idx`,
`gold_frame_idx`, and 7 `ref_frame_indices` in
`NativeVulkanAv1FrameSubmitSnapshot`. Real Wayland Main10 evidence
`/tmp/gilder-vulkan-av1-inter-ref-telemetry-main10` keeps first-frame P010 direct
decode/sampling valid and shows later temporal units as inter frames with
`order_hint` and `ref_frame_indices`, while still marking them non-submit-ready
until reference-name slot planning exists. H.265 Main10 visible 4K/240 was
rechecked after that parser change:
`/tmp/gilder-vulkan-h265-main10-after-av1-inter-ref-telemetry-4k240` reports
`decoded/presented=480/480`, `average_present_fps=240.30745235839046`, P010, and
`h265_packet_queue_retained_payload_bytes=0`.

The next AV1 blocker was a real-stream `show_existing_frame` temporal unit using
a standalone frame-header OBU with no tile-group OBU. That is now parsed as a
display-handoff planning item instead of a malformed split frame:
`/tmp/gilder-vulkan-av1-show-existing-split-fix-main10` reports first-frame
Main10 P010 direct decode/sampling and later temporal units with
`show_existing_frame=true`, `frame_to_show_map_idx=2/5`, and unsupported reason
`display handoff needs reference slot planning`. H.265 Main10 was rechecked
again after this parser change:
`/tmp/gilder-vulkan-h265-main10-after-av1-show-existing-fix-4k240` reports
`decoded/presented=480/480`, `average_present_fps=240.157162809936`, P010, and
`h265_packet_queue_retained_payload_bytes=0`.

Stricter AV1 readback diversity checks later exposed a false positive: the AV1
ready-prefix runtime could present at target cadence while decoded inter-frame
content still repeated. The root cause was the native AV1 frame-header parser
reading `allow_warped_motion` in the wrong order. It must be consumed after
`skip_mode_present` and before `reduced_tx_set`; reading it earlier shifted the
following inter fields and made the runtime submit stale picture/reference
state. After fixing the bit order and matching the GStreamer/FFmpeg parser
shape, real `WAYLAND_DISPLAY=wayland-1` evidence
`/tmp/gilder-av1-10s-warped-regression` reports `decoded_frame_count=2400`,
`presented_frame_count=2400`, `average_present_fps=240.20825729224006`,
`readback_y_distinct=5`, `readback_uv_distinct=5`, and `loop_count=79` on the
Main8 640x368 AV1 inter stream.

A second hidden-reference-chain regression was then traced to reference order:
`StdVideoDecodeAV1PictureInfo.OrderHints` and saved DPB `SavedOrderHints` must
be submitted in AV1 reference-name order (`INTRA`, `LAST`, `LAST2`, ...,
`ALTREF`), not in the internal reference-map slot order. Real Wayland reruns
after that fix are `/tmp/gilder-av1-main8-reference-name-order-hints-rerun`
(`decoded=40`, `hidden_decoded=26`, `presented=64`,
`average_present_fps=240.55662367081612`, `readback_y_distinct=5`,
`readback_uv_distinct=5`) and
`/tmp/gilder-av1-main10-reference-name-order-hints-rerun` (`decoded=40`,
`hidden_decoded=26`, `presented=64`,
`average_present_fps=244.68053337771838`, P010,
`readback_y_distinct=5`, `readback_uv_distinct=5`). AV1 continuous direct is
no longer blocked on the old repeated-frame failures; remaining work is broader
Main8/Main10 stream coverage, longer process sampling, lower-memory DPB/output
handling, and audio/clock integration.

The follow-up 10-second observation runs also pass separately:
`/tmp/gilder-av1-main8-observe-reference-name-order-10s` presents 2400 Main8
frames at `average_present_fps=239.9047972118651` with
`readback_y_distinct=10` and `readback_uv_distinct=10`;
`/tmp/gilder-av1-main10-observe-reference-name-order-10s` presents 2400
Main10/P010 frames at `average_present_fps=239.99269927809237` with
`readback_y_distinct=10` and `readback_uv_distinct=10`.

Native-resolution observation separates quality from codec coverage. The old
640x368 smoke source is useful for parser/debug turnaround but looks soft when
scaled to the 2560x1600 output. Libaom low-delay 2560x1600@240 sources look
much better and pass readback: `/tmp/gilder-av1-main8-native-res-libaom-lowdelay-observe-10s`
reports `presented=2400`, `average_present_fps=235.13213456630402`,
`readback_y_distinct=16`, `readback_uv_distinct=16`; the Main10/P010 rerun
`/tmp/gilder-av1-main10-native-res-libaom-lowdelay-observe-10s` reports
`presented=2400`, `average_present_fps=230.54892214299622`,
`readback_y_distinct=16`, `readback_uv_distinct=16`. These prove the direct
path can render native-resolution AV1 correctly, but they also expose the next
performance target because neither run sustains a full 240fps average at this
resolution.

SVT-AV1 random-access has moved from repeated-frame failure to visible continuous
decode. The 2560x1600@240 SVT source at
`/tmp/gilder-av1-observe-native-res-source/av1-main8-2560x1600-240fps-240frames.mkv`
still decodes to distinct frames with FFmpeg framehash, and the direct Vulkan
run now matches FFmpeg's compact single-tile payload sizes for the first frames
(`97725`, `109775`, `85111`, `67245`). The short correctness gate
`/tmp/gilder-av1-svt-leading-zero-default-ring-readback` reports
`presented=64`, `readback_y_distinct=9`, and `readback_uv_distinct=9`. The longer
default 8-slot bitstream-ring run
`/tmp/gilder-av1-svt-leading-zero-default-ring-20s` reports `presented=4800`,
`decoded_frame_count=2420`, `hidden_decoded_frame_count=2380`,
`displayed_handoff_frame_count=2380`, `average_present_fps=238.2264888256383`,
and 19 clean source loops. This makes SVT random-access a performance/coverage
target rather than the old correctness blocker.

AV1 repeated-frame root cause and fix notes:

- Failure mode: present cadence and decoded-frame counters were normal, but
  readback hashes collapsed to the key-frame/gray output on libaom-style hidden
  alt-ref streams. This made early FPS-only smokes false positives.
- Parser bug: `allow_warped_motion` was inferred before `reduced_tx_set` but
  not actually consumed after `skip_mode_present`, shifting the following AV1
  inter-frame fields. The fix consumes `skip_mode_present`, then
  `allow_warped_motion`, then `reduced_tx_set`, matching the AV1 bitstream order.
- Reference bug: the runtime filled Vulkan AV1 `OrderHints` from internal
  reference-map slot order. Vulkan/FFmpeg expect AV1 reference-name order
  (`INTRA`, `LAST`, `LAST2`, `LAST3`, `GOLDEN`, `BWDREF`, `ALTREF2`, `ALTREF`).
  The visible symptom in diagnostics was hidden frame 2 submitting
  `[0,29,0,0,0,0,0,0]` where the correct reference-name array was
  `[0,0,0,0,0,29,0,0]`.
- Code fix: `native_vulkan_av1_picture_order_hints_for_submit` now submits
  `reference_name_order_hints` directly, and saved DPB `SavedOrderHints` are
  kept in the same reference-name order. The old NVIDIA/order-hint offset path
  is disabled by default and only enabled explicitly with
  `GILDER_VULKAN_AV1_ORDER_HINT_OFFSET`.
- Streaming fix: when the packet queue loops back to the source, the AV1
  streaming reference planner is recreated before planning the first temporal
  unit of the new loop, so stale reference maps do not leak across loop
  boundaries.
- SVT tile-payload fix: SVT random-access inter frame OBUs expose a leading
  zero byte before the actual single-tile entropy payload at the parser's
  previous tile boundary. FFmpeg/Vulkan submits the compact tile payload one
  byte later. `native_vulkan_av1_tile_group_offsets_from_payload` now skips that
  byte only for inter, single-tile, 1x1 tile layouts whose first tile byte is
  zero; key frames and non-zero tile starts remain unchanged. The regression
  test is `trims_av1_single_tile_inter_leading_zero_for_tile_payload_window`.
- Performance fix: AV1 streaming bitstream rings now default to 8 slots while
  H.264/H.265 stay at 2 slots. On the same SVT source this reduced ring wraps
  and improved the no-readback 10s observation from roughly 236fps to a
  238-239fps range, with `GILDER_VULKAN_BITSTREAM_RING_SLOTS` still available
  for explicit override.
- Diagnostics added: runtime snapshots now include submitted picture
  `OrderHints`, setup/reference `SavedOrderHints`, reference frame types, sign
  bias, frame-size flags, and hidden-frame diagnostics so future false positives
  can be checked without relying on visual inspection alone.

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
temporal units, the native parser extracts sequence-header and first-frame STD
fields, Vulkan accepts the resulting `StdVideoAV1SequenceHeader`, and the first
shown key frame is submitted through `vkCmdDecodeVideoKHR`. The AV1 smoke now
defaults to `--decode-first-frame`, allocates video resource images, and requires
non-zero decode-output readback. The 2026-06-22 readback fix makes the output
layout format-aware instead of hard-coding NV12: Main8 reports
`G8_B8R8_2PLANE_420_UNORM`, while Main10 reports
`G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16` with P010-sized Y/UV planes.
Current real `WAYLAND_DISPLAY=wayland-1` decode/readback evidence:
`/tmp/gilder-vulkan-av1-smoke-first-frame-main8`,
`/tmp/gilder-vulkan-av1-smoke-first-frame-main10`, and
`/tmp/gilder-vulkan-av1-smoke-first-frame-main10-4k`; all report
`result=first-frame-decode-and-output-readback-completed`,
`first_frame_decode.completed=true`, and non-zero Y/UV readback. The 4K Main10
run reads back 24,883,200 bytes as P010-like output.
P010 shader sampling is now verified as well: AV1 Main10 4K script evidence
`/tmp/gilder-vulkan-av1-p010-sampling-script` reports
`result=first-frame-decode-output-sampled-and-readback-completed`,
`first_frame_decode.codec=av1-main-10`, `output_sampling.rendered=true`,
`rgba_unique_values=256`; H.265 Main10 evidence
`/tmp/gilder-vulkan-h265-main10-p010-sampling.CGax7L` reports the same result
shape with `first_frame_decode.codec=h265-main-10`.

The continuous input layer is partly prepared: AV1 temporal units can use the
same generic streaming packet queue as H.264/H.265, sequence header is a
bootstrap parameter set, frame-only temporal units can derive first-frame submit
snapshots from that active sequence header, and packet timeline metadata keeps
access-unit index, source-loop index, PTS and duration for later audio/clock
integration. The remaining direct AV1 work is continuous inter/reference frame
headers, reference-name slot planning, and visible playback. Main10 now has
format-aware P010 plane views and first-frame shader sampling; the remaining
visible path is continuous DPB/display handoff and swapchain present, not plane
view creation.

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

Current 2026-06-22 Main10/P010 direct Vulkan evidence:

- H.265 Main10 visible ready-prefix:
  `/tmp/gilder-vulkan-h265-main10-final-regression-4k240`,
  `requested_codec=h265-main-10`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`,
  `decoded_frame_count=480`, `presented_frame_count=480`,
  `average_present_fps=240.71777490911953`,
  `h265_packet_queue_loop_count=1`,
  `h265_packet_queue_loop_skip_access_units=156`,
  `h265_packet_queue_retained_payload_bytes=0`,
  `video_resource_memory_bytes=75104256`,
  `session_memory_bytes=46309376`.
  This completes the arbitrary-entry direct Vulkan Video Main10 decode + P010
  shader present gate on the 4K/240 real Wayland path. It is not yet the full
  video-wallpaper playback contract:
  long-duration process sampling, broader real-world Main10 streams, continuous
  demux/loop, audio and clock integration still need separate evidence.
  Renderer descriptor-set expansion was regression-tested with
  `/tmp/gilder-vulkan-h265-main10-renderer-regression-4k240`: `decoded=480`,
  `presented=480`, `average_present_fps=240.2474194054933`, same P010 picture
  format.
- AV1 visible ready-prefix:
  `/tmp/gilder-vulkan-av1-main10-dpb9-regression-4k240`,
  `requested_codec=av1-main-10`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`,
  `decoded_frame_count=259`, `displayed_handoff_frame_count=221`,
  `presented_frame_count=480`, `average_present_fps=239.94913990040843`,
  `stream_dpb_slots=9`, `session_max_dpb_slots=9`, `driver_max_dpb_slots=16`,
  `av1_packet_queue_retained_payload_bytes=0`,
  `bitstream_buffer_strategy=fixed-capacity-persistent-mapped-ring`,
  `video_resource_memory_bytes=225312768`, `session_memory_bytes=14143488`.
  Main8 was rechecked with 10 seconds of 4K/240 playback at
  `/tmp/gilder-vulkan-av1-main8-observe-10s-dpb9-v3`: `decoded=1305`,
  `handoff=1095`, `presented=2400`, `average_present_fps=239.6313194270436`,
  `stream_dpb_slots=9`, all displayed layers `0..8`, and
  `av1_packet_queue_retained_payload_bytes=0`. This supersedes the earlier
  single-DPB-slot AV1 evidence, which could pass submit/present counters while
  visibly flashing gray because inter/show-existing output reused active
  reference layers. A stricter Main8 parser/readback regression
  `/tmp/gilder-av1-10s-warped-regression` now also proves content changes across
  inter frames: `decoded=2400`, `presented=2400`,
  `average_present_fps=240.20825729224006`, `readback_y_distinct=5`,
  `readback_uv_distinct=5`. Remaining AV1 work is broader stream coverage,
  Main10 long-duration readback/present evidence, physical DPB slot compaction
  or a lower-memory display handoff, process sampling, audio/clock integration
  and replacing the synthetic libaom smoke source with more real wallpaper
  samples. The latest hidden-reference-chain reruns also pass for both Main8
  and Main10: `/tmp/gilder-av1-main8-reference-name-order-hints-rerun` and
  `/tmp/gilder-av1-main10-reference-name-order-hints-rerun` both report
  `readback_y_distinct=5` and `readback_uv_distinct=5`. Separate 10-second
  observation runs `/tmp/gilder-av1-main8-observe-reference-name-order-10s` and
  `/tmp/gilder-av1-main10-observe-reference-name-order-10s` both present 2400
  frames at roughly 240fps with `readback_y_distinct=10` and
  `readback_uv_distinct=10`. Native-resolution low-delay AV1 is also visible
  and readback-valid, but Main8 averages about 235fps and Main10/P010 about
  230fps at 2560x1600@240. SVT-AV1 random-access is no longer the repeated-frame
  correctness blocker after the single-tile leading-zero fix:
  `/tmp/gilder-av1-svt-leading-zero-default-ring-readback` reports
  `readback_y_distinct=9` and `readback_uv_distinct=9`, while
  `/tmp/gilder-av1-svt-leading-zero-default-ring-20s` presents 4800 frames at
  `average_present_fps=238.2264888256383`.
- H.264 4K/240 remains the current performance debt:
  `/tmp/gilder-vulkan-h264-telemetry-default-4k240-ref1`,
  `decoded_frame_count=480`, `presented_frame_count=480`,
  `average_present_fps=230.37179368303578`,
  `h264_present_queue_count=1`, `h264_async_present_depth=1`,
  `h264_display_handoff_strategy=gpu-copy-to-dual-slot-nv12-display-ring`,
  `h264_packet_queue_retained_payload_bytes=0`.
  Direct sampled DPB output removes the extra 25.6MB display ring but regresses
  to about 212fps on the same host, so H.264 240fps work should stay focused on
  decode/display/present scheduling rather than packet queue retention.
  The latest H.264 display-ring optimization prebinds one descriptor set per
  display slot. Real Wayland evidence
  `/tmp/gilder-vulkan-h264-prebound-descriptor-4k240-perf` shows
  `decoded=1200`, `presented=1200`, `average_present_fps=233.90643962520952`,
  `avg_descriptor_update_us=0`, and process sampling
  `RSS/PSS/USS/Private_Dirty max=106000/91369/86404/27424 KiB`, CPU avg
  `15.60%`. This is a measurable CPU-side cleanup, not a completed 240fps fix.

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
