# Video Validation

The current visible video direction is native Wayland + native Vulkan.
GStreamer remains the codec/container/audio frontend, but display sinks do not
own the Wayland surface. Do not use the retired GTK or native
`playbin/waylandsink` smoke paths for current validation.

## Codec Reference Priority

FFmpeg is the first engineering reference for direct codec behavior. For AV1,
H.264 and H.265 work, use FFmpeg's mature packet/frame model, parser resilience,
bitstream resynchronization, reference/DPB handling, reorder semantics, loop/seek
behavior and low-copy buffer ownership as the primary comparison point. A Gilder
direct Vulkan Video path is not considered broadly usable just because a narrow
closed-GOP generated stream passes; it must move toward the same practical
behavior users expect from FFmpeg-backed playback: arbitrary continuous input,
arbitrary entry points, malformed or partial leading data skipped until a
decodable boundary, and steady-state playback without retaining long compressed
payload windows.

GStreamer is the second reference and the active integration frontend. It should
continue to provide container demux, parser/appsink handoff, timestamp/segment
behavior, audio/clock integration and practical pipeline diagnostics. When
FFmpeg and GStreamer expose different symptoms, treat FFmpeg as the primary
codec semantics reference and GStreamer as the frontend contract that must be
adapted or instrumented. Vulkan Video specifications, driver capabilities and
real Wayland evidence remain the final API and hardware validation boundary.

The next direct-video phase is therefore AV1, H.264 and H.265 arbitrary
continuous/arbitrary-bitstream usability before audio integration. Validation
must explicitly cover non-zero entry offsets, loop replay after EOS/seek, codec
state rebuild after skipped leading data, bounded packet queues, fixed-capacity
bitstream rings and present/decode overlap. Increasing timeouts, accepting empty
runtime evidence or only optimizing generated happy paths is not sufficient.

## Pipeline Boundary Contract

The native Vulkan video contract exposed by `gilder-native-vulkan --contract`
now names the runtime boundaries explicitly: render-plan source selection,
frontend demux, parser normalization, bounded packet queue, codec state/DPB,
Vulkan Video decode, decoded-image handoff, render, present and separate audio
clock. FFmpeg is the first reference for packet/frame/clock semantics at each
boundary; GStreamer is the second reference and active demux/parser/audio
frontend. This is the implementation split target: demux/parser code may remain
GStreamer-backed, but decode, render, present and audio-clock telemetry must stay
independently attributable and compressed payload retention must remain bounded
by the packet queue and bitstream ring. The active split is now centered on
`src/renderer/native_vulkan/demux.rs`, `demux_gst.rs`, `vulkanalia_extract.rs`
and `vulkanalia_backend/*`: GStreamer supplies parser-normalized packets, while
Vulkanalia/native Vulkan owns video session setup, decode submission, decoded
image handoff, render and present. `audio_runtime.rs`, `audio_worker.rs` and
`audio_telemetry.rs` own the separate audio clock boundary. The renderer session
sends video clock samples over a command channel and reads shared telemetry; the
audio worker owns the GStreamer probe and coalesces queued clock samples so only
the latest video clock is processed.

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

## Real Source Corpus

Local real-source coverage should use scripts instead of hand-written one-off
commands. `scripts/native-vulkan-real-source-matrix.sh` scans a user-owned media
directory or a local Wallpaper Engine Workshop directory, records `ffprobe`
codec/profile/extent/fps/audio metadata, and can run the matching H.264/H.265/AV1
direct Vulkan Video smoke for each supported source. Probe-only mode is the
default and does not require a Wayland session:

```sh
scripts/native-vulkan-real-source-matrix.sh \
  --source-dir /path/to/videos \
  --report-dir /tmp/gilder-real-source-matrix \
  --duration 10
```

Wallpaper Engine Workshop items can be downloaded into an ignored local corpus
with SteamCMD. The downloader keeps third-party assets under
`artifacts/wallpaper-engine-workshop/` and never adds them to the tracked repo.
Many Workshop items require a Steam account that owns Wallpaper Engine; anonymous
download is not guaranteed to work.

```sh
scripts/wallpaper-engine-workshop-download.sh \
  --item-id 123456789 \
  --install-steamcmd \
  --probe-after-download \
  -- --duration 10
```

For a real Wayland run, pass matrix arguments after `--`:

```sh
GILDER_STEAM_USER=<steam-user> scripts/wallpaper-engine-workshop-download.sh \
  --item-list /path/to/workshop-ids.txt \
  --install-steamcmd \
  --probe-after-download \
  -- --run-video --output-name HDMI-A-1 --audio-clock-probe --duration 10
```

`--install-steamcmd` installs Valve's SteamCMD into
`artifacts/tools/steamcmd/`, which is ignored by git and survives restarts.
Omit it when a system `steamcmd` is already available or pass `--steamcmd` for a
custom executable. To install the tool without downloading any Workshop item:

```sh
scripts/wallpaper-engine-workshop-download.sh --install-steamcmd-only
```

Do not commit downloaded Workshop media or generated matrix reports. The
expected long-term flow is: download or subscribe locally, probe/classify the
corpus, then run direct-video smokes for supported codecs. Unsupported codecs
such as VP8/VP9 should remain classified evidence until a corresponding runtime
path exists.

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

Latest 2026-06-23 FFmpeg-aligned arbitrary-entry loop gates reused complete
4K/240 sources and ran without an external `timeout` wrapper. In this Wayland
session, GNU `timeout` could produce an empty runtime with `Could not find
wayland compositor`; treat that as invalid environment evidence, not codec
evidence. The current 2400-frame single-run matrix on `WAYLAND_DISPLAY=wayland-1`,
`HDMI-A-1`, 3840x2160@240 is:
H.264 `/tmp/gilder-h264-arbitrary-loop-replay-4k240-solo-2400-no-timeout-a`
passed `decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=7`, `loop_boundary_reset_count=6`,
`h264_packet_queue_bootstrap_discarded_access_units=126`,
`h264_packet_queue_loop_skip_access_units=126`,
`h264_packet_queue_retained_payload_bytes=0`, `first_frame_recovery=true`,
`loop_first_unrecovered_count=0`, `bitstream_ring_wrap_count=219`,
`average_present_fps=239.52339060880522`,
`average_present_result_fps=239.5585093449288`, and
`average_present_result_drop_first_60_fps=239.9394191434932`. H.265 Main8
`/tmp/gilder-h265-main8-arbitrary-loop-replay-4k240-solo-2400-no-timeout-a`
passed `decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=8`, `loop_boundary_reset_count=7`,
`h265_packet_queue_bootstrap_discarded_access_units=156`,
`h265_packet_queue_loop_skip_access_units=156`,
`h265_packet_queue_retained_payload_bytes=0`, `first_frame_idr=true`,
`loop_first_non_idr_count=0`, `bitstream_ring_wrap_count=253`, and
`average_present_fps=239.30586018474193`. H.265 Main10
`/tmp/gilder-h265-main10-arbitrary-loop-replay-4k240-solo-2400-no-timeout-a`
passed `decoded_frame_count=2400`, `presented_frame_count=2400`,
`playback_loop_count=8`, `loop_boundary_reset_count=7`,
`h265_packet_queue_bootstrap_discarded_access_units=156`,
`h265_packet_queue_loop_skip_access_units=156`,
`h265_packet_queue_retained_payload_bytes=0`, `first_frame_idr=true`,
`loop_first_non_idr_count=0`, `bitstream_ring_wrap_count=245`, and
`average_present_fps=239.09677180017684`. AV1 Main8
`/tmp/gilder-av1-main8-arbitrary-loop-replay-4k240-solo-2400-no-timeout-a`
passed `presented_frame_count=2400`, `processed_temporal_unit_count=3574`,
`decoded_frame_count=1309`, `hidden_decoded_frame_count=1174`,
`displayed_handoff_frame_count=1091`, `playback_loop_count=8`,
`loop_boundary_reset_count=7`, `av1_packet_queue_retained_payload_bytes=0`,
`bitstream_ring_wrap_count=7`, `average_present_fps=239.99982482412787`,
`average_present_result_fps=239.91585843824276`, and
`average_present_result_drop_first_60_fps=240.0008208664245`. AV1 Main10
`/tmp/gilder-av1-main10-arbitrary-loop-replay-4k240-solo-2400-no-timeout-a`
passed `presented_frame_count=2400`, `processed_temporal_unit_count=3574`,
`decoded_frame_count=1309`, `hidden_decoded_frame_count=1174`,
`displayed_handoff_frame_count=1091`, `playback_loop_count=8`,
`loop_boundary_reset_count=7`, `av1_packet_queue_retained_payload_bytes=0`,
`bitstream_ring_wrap_count=5`, `average_present_fps=239.99886607735746`,
`average_present_result_fps=239.91641028246332`, and
`average_present_result_drop_first_60_fps=239.99865379121943`.

2026-06-24 tightened the arbitrary-entry definition to match the FFmpeg seek
contract more closely: after a decoder reset, the first visible H.264/H.265 AU
must be a recovery point (currently IDR), and AV1 must start from a shown key
frame. A real H.264 Main + AAC MP4 shifted by `--arbitrary-entry-offset 0.35`
previously exposed the bug: the queue accepted a non-IDR P frame because the
reset DPB was empty. The fixed gate
`/tmp/gilder-h264-real-kamen-2-arbitrary-entry-loop-1440p60-900-b` now passes
with `decoded_frame_count=900`, `presented_frame_count=900`,
`h264_packet_queue_bootstrap_discarded_access_units=9`,
`h264_packet_queue_loop_skip_access_units=9`, first visible AU `9` as IDR,
loop replay first AU `850` as IDR, `loop_first_non_idr_count=0`,
`first_frame_recovery=true`, `runtime_elapsed_ms=14991`, and
`average_present_result_drop_first_60_fps=59.9793482310882`. The same stricter
bootstrap was regression-tested on generated 4K/240 arbitrary sources:
H.265 Main8 `/tmp/gilder-h265-main8-arbitrary-recovery-bootstrap-4k240-480-b`
passed `decoded/presented=480/480`, `bootstrap_discarded=156`, `loop_skip=156`,
`first_frame_idr=true`, `loop_first_non_idr_count=0`, and
`average_present_fps=239.4167640037135`; H.265 Main10
`/tmp/gilder-h265-main10-arbitrary-recovery-bootstrap-4k240-2400-b` passed
`decoded/presented=2400/2400`, `playback_loop_count=8`,
`loop_boundary_reset_count=7`, `bootstrap_discarded=156`, `loop_skip=156`,
`first_frame_idr=true`, `loop_first_non_idr_count=0`, and
`average_present_fps=238.9120674801146`, so correctness is stable but Main10
4K/240 pacing still needs scheduler work. AV1 Main8
`/tmp/gilder-av1-main8-arbitrary-recovery-bootstrap-4k240-480-b` and Main10
`/tmp/gilder-av1-main10-arbitrary-recovery-bootstrap-4k240-480-b` both passed
with `presented_frame_count=480`, `processed_temporal_unit_count=718`,
`decoded_frame_count=260`, `hidden_decoded_frame_count=238`,
`displayed_handoff_frame_count=220`, `first_frame_key=true`,
`loop_first_non_key_count=0`, `arbitrary_entry_demux_dropped_prefix=yes`, and
warmup-dropped present-result FPS of `239.93820016572343` and
`239.96829440387066`.

Audio/clock work started after the arbitrary-entry gates were fixed. The first
probe is intentionally audio-only rather than mixed into the Vulkan present loop:
`scripts/native-vulkan-audio-clock-probe.sh` records ffprobe stream/packet timing
and runs an explicit GStreamer AAC chain (`qtdemux.audio_0 ! aacparse !
avdec_aac ! fakesink`) so the evidence is not polluted by video decode. Real
source `/tmp/gilder-audio-clock-probe-kamen-2-aac-10s-c` reports AAC LC stereo
at 48 kHz, `audio_packet_count=469`, `audio_first_pts=0.000000`,
`audio_last_pts=9.984000`, packet PTS delta `0.021333..0.021334`, one
GStreamer clock, one stream-start message, and playing-state messages from the
audio-only chain. This establishes the first measurable audio-clock frontend
before making audio the playback master clock.

The next step moved that audio-only frontend into the native Vulkan H.264
visible runtime behind `--audio-clock-probe`. Following the ffplay clock model,
the runtime no longer lets the audio pipeline free-run during video setup:
playback starts on the first video sample, loop reset advances an audio clock
serial, stale positions/samples from the old serial are ignored, and telemetry
reports a monotonic audio-master estimate. Real Wayland evidence
`/tmp/gilder-h264-real-kamen-2-audio-clock-ffplay-serial-1440p60-900-f` passes
from the same shifted Main + AAC MP4 with `decoded/presented=900/900`,
`bootstrap_discarded=9`, `loop_skip=9`, `first_frame_recovery=true`,
`loop_first_unrecovered_count=0`, and
`average_present_result_drop_first_60_fps=59.9944410395156`. The parallel audio
pipeline is explicit AAC only: `audio_decoders=["avdec_aac"]`,
`audio_video_decoders=[]`, `audio_sample_rate=48000`, `audio_channels=2`,
`audio_buffer_count=701`, `audio_position_query_count=900`,
`audio_position_query_hit_count=897`, `audio_clock_serial=2`,
`audio_loop_restart_count=1`, `audio_position_stale_count=0`,
`audio_sample_stale_count=0`, and `audio_reached_clocked_playback=true`.
The old loop-reset drift issue is no longer present in this gate:
`audio_video_master_clock_drift_latest_ns=-61777` and
`audio_video_master_clock_drift_abs_max_ns=856739`.

The native Vulkan runtime now also has an output policy branch. `--audio-output
plan` follows the effective video plan muted state produced from
`entry.muted || !runtime.allow_audio`: muted resolves to `clock-only`, unmuted
resolves to `auto`. Explicit `--audio-output auto` still tees the AAC chain into
both the telemetry appsink and `autoaudiosink`, preserving the ffplay-style clock
probe while allowing audible output when the system has an audio sink. Short
Wayland script evidence `/tmp/gilder-h264-audio-output-auto-script-60` reports
`decoded/presented=60/60`, pipeline
`qtdemux-aacparse-avdec_aac-tee-appsink-autoaudiosink`,
`audio_output_mode=auto`, `audio_output_sinks=["autoaudiosink","jackaudiosink"]`,
`audio_output_sink_count=2`, `audio_decoders=["avdec_aac"]`,
`audio_video_decoders=[]`, and `audio_reached_clocked_playback=true`. The
H.264/H.265/AV1 ready-prefix smokes now expose `--muted/--unmuted` and
`--audio-output`, record `audio_output_expected_mode`, and fail the audio gate
when an `auto` run does not report an output sink. Runtime snapshot policy also
distinguishes muted plans (`clock-only`) from unmuted plans (`auto`). The
plan-following smoke `/tmp/gilder-h264-audio-output-policy-module-plan-unmuted-60`
passes with `audio_output=plan`, `audio_plan_muted=false`,
`audio_output_expected_mode=auto`, `audio_output_mode=auto`, and
`audio_output_sink_count=2`. The shared policy now lives in the native Vulkan
audio boundary (`NativeVulkanAudioOutputPolicy`) rather than the CLI wrapper.
Manifest-backed `VideoWallpaperPlan` runtime snapshots report
`audio_output_policy=plan` and resolve the same effective muted state, so the
native Vulkan renderer now starts the actual plan-following audio runtime from
the same policy path. Muted plans start the same worker in `clock-only` mode for
ffplay-style clock telemetry without an audible sink; unmuted plans start it in
`auto` mode and tee to `autoaudiosink`. A 2026-06-24 real Wayland
`--run-video --unmuted` check on
`artifacts/video-sources/h264/audio-loop/kamen-h264-aac-2s-loop.mp4` rendered
`60` frames in one second and reported
`audio_runtime_status=clocked-playback-active`,
`audio_runtime_buffer_count=42`, `audio_runtime_output_sink_count=2`,
`audio_runtime_position_query_hit_count=60`, and
`audio_runtime_last_error=null`. The snapshot now separates policy
(`audio_output_policy/mode/status`) from actual runtime telemetry
(`audio_runtime_*`), including audio clock serial, segment start/elapsed,
loop seek/restart counts, stale sample/position counts, sampled video frame
count, position query hit ratio, master-clock estimate and latest audio/video
drift fields. This follows ffplay's packet-queue serial model: obsolete segment
positions and samples are counted at the generic video runtime boundary instead
of remaining hidden inside the audio frontend. This is the boundary to keep
while splitting video demux, decode, render and present code, and it is the
evidence surface used by the audio-clock default pacing master when the audio
clock probe is enabled. `GILDER_VIDEO_PACING_MASTER=target` remains the explicit
fallback for target-fps pacing comparisons.

Follow-up AV1 copy-cost work on 2026-06-23 made `show_existing_frame` presentation
sample the decoded DPB image directly by default instead of copying those handoff
frames into the display ring again. `GILDER_VULKAN_AV1_SHOW_EXISTING_DIRECT_DPB=0`
keeps the old display-copy fallback available. Default real Wayland gates passed:
AV1 Main8
`/tmp/gilder-av1-main8-show-existing-direct-dpb-default-4k240-2400-a`
reported `presented_frame_count=2400`, `displayed_handoff_frame_count=1091`,
`av1_display_handoff_strategy=video-queue-early-keep-last-copy-display-ring+show-existing-direct-dpb`,
`av1_display_copy_count=1309`, `av1_show_existing_direct_dpb_count=1091`,
`av1_present_frame_queue_record_elapsed_us=24878`,
`average_present_fps=240.015153212689`, and
`average_present_result_drop_first_60_fps=240.03158277892948`. AV1 Main10
`/tmp/gilder-av1-main10-show-existing-direct-dpb-default-4k240-2400-a`
reported `presented_frame_count=2400`, `displayed_handoff_frame_count=1091`,
`av1_display_handoff_strategy=video-queue-early-keep-last-copy-display-ring+show-existing-direct-dpb`,
`av1_display_copy_count=1309`, `av1_show_existing_direct_dpb_count=1091`,
`av1_present_frame_queue_record_elapsed_us=27584`,
`average_present_fps=239.87471039235322`, and
`average_present_result_drop_first_60_fps=240.0080365239079`. Compared with the
previous 2400-frame AV1 matrix, these runs keep 4K/240 pacing while avoiding
1091 display-copy operations per 2400 presented frames on this stream.

The later 2026-06-23 AV1 direct-DPB pass supersedes the show-existing-only
optimization for the default path. AV1 now treats FFmpeg-style decoded-frame
ownership as the primary model: displayed frames sample the decoded DPB resource
in `GENERAL` layout, frame contexts keep the sampled resource live until present
retire, and decode only waits when a later output would overwrite that DPB slot.
`GILDER_VULKAN_AV1_DISPLAYED_DIRECT_DPB=0` keeps the old display-ring fallback.
Real Wayland 4K/240 gates passed with no AV1 display ring and no display copy:
Main8 `/tmp/gilder-av1-main8-displayed-direct-dpb-general-4k240-2400-b`
reported `presented_frame_count=2400`,
`av1_display_handoff_strategy=direct-sampled-dpb-general-layout+frame-context-retire`,
`av1_display_ring_slot_count=0`, `av1_display_ring_memory_bytes=0`,
`av1_display_copy_count=0`, `av1_displayed_direct_dpb_count=2400`,
`av1_show_existing_direct_dpb_count=1091`,
`average_present_fps=240.0128447194083`, and
`average_present_result_drop_first_60_fps=240.00899700250415`. Main10
`/tmp/gilder-av1-main10-displayed-direct-dpb-general-4k240-2400-a`
reported `presented_frame_count=2400`, P010
`G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`,
`av1_display_ring_slot_count=0`, `av1_display_ring_memory_bytes=0`,
`av1_display_copy_count=0`, `av1_displayed_direct_dpb_count=2400`,
`av1_show_existing_direct_dpb_count=1091`,
`average_present_fps=239.8255409247888`, and
`average_present_result_drop_first_60_fps=240.001288762799`. Short readback
diversity gates confirmed the direct-DPB sampled content changes: Main8
`/tmp/gilder-av1-main8-displayed-direct-dpb-general-readback-4k240-480-a`
and Main10
`/tmp/gilder-av1-main10-displayed-direct-dpb-general-readback-4k240-480-a`
both reported `readback_y_distinct=9`, `readback_uv_distinct=9`,
`av1_display_copy_count=0`, and `av1_displayed_direct_dpb_count=480`.

The next AV1 present-cost pass tested H.264/H.265-style 2-frame present-frame
clear preroll but kept it opt-in because the result was mixed. Set
`GILDER_VULKAN_AV1_PRESENT_FRAME_CLEAR_PREROLL=1` to enable it; the default
enabled count remains 2 and can be overridden with
`GILDER_VULKAN_AV1_PRESENT_FRAME_CLEAR_PREROLL_COUNT`. Real Wayland 4K/240
validation stayed zero-copy with preroll enabled: Main8
`/tmp/gilder-av1-main8-direct-dpb-clear-preroll-4k240-2400-a` reported
`av1_present_frame_preroll_count=2`, `av1_display_copy_count=0`,
`av1_displayed_direct_dpb_count=2400`, `average_present_fps=239.9417797107689`,
and `average_present_result_drop_first_60_fps=239.9933347725409`; Main10
`/tmp/gilder-av1-main10-direct-dpb-clear-preroll-4k240-2400-a` reported
`av1_present_frame_preroll_count=2`, `av1_display_copy_count=0`,
`av1_displayed_direct_dpb_count=2400`, `average_present_fps=239.93205426537594`,
and `average_present_result_drop_first_60_fps=239.99924480784875`.
Process sampling on the same path shows the new zero-copy baseline rather than
the historical display-ring path: Main8
`/tmp/gilder-av1-main8-direct-dpb-clear-preroll-performance-4k240-2400-a`
reported `average_present_fps=239.9059074396761`, average CPU `12.95%`,
RSS/PSS/USS/Private_Dirty max `117112/80418/67100/36912 KiB`, NVIDIA process
GPU memory `181 MiB`, `av1_display_copy_count=0`, and
`av1_present_frame_preroll_count=2`; Main10
`/tmp/gilder-av1-main10-direct-dpb-clear-preroll-performance-4k240-2400-a`
reported `average_present_fps=239.9695573179769`, average CPU `12.46%`,
RSS/PSS/USS/Private_Dirty max `116776/80249/66892/36688 KiB`, NVIDIA process
GPU memory `289 MiB`, `av1_display_copy_count=0`, and
`av1_present_frame_preroll_count=2`.

Three AV1 present-side tuning probes were intentionally not made default. The
clear-preroll probe improved the Main10 no-sampling average versus the earlier
direct-DPB run but slightly reduced Main8, so the default direct-DPB path keeps
`av1_present_frame_preroll_count=0`. `GILDER_VULKAN_AV1_READY_CONTEXT_SELECTION=1`
passed
`/tmp/gilder-av1-main8-direct-dpb-ready-context-preroll-4k240-2400-a`, but the
ready-probe path only found 143 ready contexts out of 2400 and increased
missed-vblank-after-warmup to 7, so round-robin remains the default. Forcing
`GILDER_VULKAN_AV1_PRESENT_FRAME_QUEUE_DEPTH=2` in
`/tmp/gilder-av1-main10-direct-dpb-preroll-depth2-4k240-2400-a` reduced decode
slot wait to `58611us` but moved the cost into
`av1_present_result_wait_elapsed_us=9692746`, dropping average FPS to
`239.80400159488644`; the default depth remains 4.

On 2026-06-24 the ready-prefix smokes were tightened to a shared
FFmpeg-aligned timestamp validation shape. The Rust runtime now computes
`pts_delta_expected_min_ms`, `pts_delta_expected_max_ms`, and
`pts_delta_in_expected_range` from `target_max_fps` in
`src/renderer/native_vulkan/timeline.rs`, while the shell smokes reuse
`scripts/native-vulkan-ready-prefix-video-common.sh` for source-cache paths and
the same expected range check. H.264, H.265 and AV1 smokes now fail if the
runtime PTS delta range is missing or outside the target frame-period bounds.
Generated AV1 sources are cached under
`artifacts/video-sources/av1/` by default, with `--source-cache-dir` available
for overrides; reports can still live in `/tmp`. Cache-reuse Wayland checks
passed for Main8 `/tmp/gilder-av1-main8-pts-cache-reuse-640x368-240-480-a` and
Main10 `/tmp/gilder-av1-main10-pts-cache-reuse-640x368-240-480-a`. Both used
repo-local cached sources, reported `pts_delta_min_ms=4`,
`pts_delta_max_ms=4`, `av1_present_frame_preroll_count=0`,
`av1_display_copy_count=0`, and `av1_displayed_direct_dpb_count=480`. These are
timestamp/default-path checks. The same cache path was then populated for 4K
sources and validated with 480-frame Wayland gates: Main8
`/tmp/gilder-av1-main8-pts-cache-4k240-480-a` used
`artifacts/video-sources/av1/av1-main8-3840x2160-240fps-242frames-g240.webm`,
reported `pts_delta_min_ms=4`, `pts_delta_max_ms=4`,
`av1_present_frame_preroll_count=0`, `av1_display_copy_count=0`,
`av1_displayed_direct_dpb_count=480`, and
`average_present_result_drop_first_60_fps=239.99770885242145`. Main10
`/tmp/gilder-av1-main10-pts-cache-4k240-480-a` used
`artifacts/video-sources/av1/av1-main10-3840x2160-240fps-242frames-g240.webm`,
reported the same PTS/preroll/copy/direct-DPB counts and
`average_present_result_drop_first_60_fps=239.59697387306508`. The 2400-frame
matrix above remains the stronger long-run performance evidence.

The same 2026-06-24 pass moved H.264/H.265 generated sources into repo-local
ignored caches and validated the stricter PTS range gate at 4K/240, 480 frames.
H.264 `/tmp/gilder-h264-pts-range-cache-4k240-480-a` used
`artifacts/video-sources/h264/h264-high-b0-ref2-weightp0-weightb0-3840x2160-240fps-242frames-g241-d240.mp4`,
reported `presented_frame_count=480`, `pts_delta_min_ms=4`,
`pts_delta_max_ms=5`, expected range `4..5`,
`pts_delta_in_expected_range=true`, and
`average_present_result_drop_first_60_fps=240.04308171776893`. H.265 Main8
`/tmp/gilder-h265-main8-pts-range-cache-4k240-480-a` used
`artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-242frames-g240-d240.mp4`,
reported `presented_frame_count=480`, NV12 format, expected PTS range `4..5`,
and `pts_delta_in_expected_range=true`. H.265 Main10
`/tmp/gilder-h265-main10-pts-range-cache-4k240-480-a` used
`artifacts/video-sources/h265/h265-main-10-b0-ref1-3840x2160-240fps-242frames-g240-d240.mp4`,
reported `presented_frame_count=480`, P010 format, expected PTS range `4..5`,
and `pts_delta_in_expected_range=true`. AV1 was rerun with the stricter runtime
range fields in `/tmp/gilder-av1-main8-pts-range-cache-4k240-480-a` and
`/tmp/gilder-av1-main10-pts-range-cache-4k240-480-a`; both kept zero display
copy, direct-DPB display for all 480 frames, default `av1_present_frame_preroll_count=0`,
and `pts_delta_in_expected_range=true`.

The real MP4 validation set under
`/home/yk/Documents/mpv/动态视频MP4-假面骑士/` exposed two 2026-06-24 correctness
issues that synthetic High-profile sources did not cover. First, the direct
H.264 Vulkan session/profile selection was still hardcoded around High-profile
streams; it now maps SPS `profile_idc` 66/77/100 to Baseline/Main/High Vulkan
STD profiles and accepts 8-bit 4:2:0 Main streams. The first real file
`1.假面骑士555-Kamen Rider Faiz（试作255）.mp4` now passes as
`h264_stream_profile=main`, `h264_stream_profile_idc=77`, and
`h264_vulkan_std_profile_idc=77`. Second, sampled video appeared vertically
inverted because the fullscreen-triangle shader path treated raw UV Y as a
bottom-left screen coordinate while decoded video frames follow top-left origin
semantics like FFmpeg/GStreamer. `src/renderer/native_vulkan/sampling.rs` now
folds the vertical flip into the fit push constants while preserving cover
crop. The follow-up pacing fix is also root behavior, not a timeout change:
FIFO present no longer substitutes for source/target FPS pacing when a 60fps
stream is shown on a 240Hz output. With `--target-fps 60 --decode-prefix 600
--playback-frames 600`, real Wayland evidence
`/tmp/gilder-h264-real-kamen-1-orientation-pacing-10s-1440p60-600` reported
`runtime_elapsed_ms=9988`, `presented_frame_count=600`,
`present_mode=fifo`, `pacing_strategy=target-fps-cpu-sleep-with-fifo-present`,
`frame_sleep_count=599`, `missed_frame_pacing_count=0`,
`average_present_result_drop_first_60_fps=59.98248822941043`,
`pts_delta_min_ms=16`, `pts_delta_max_ms=17`, and
`pts_delta_in_expected_range=true`.

After that pacing change, the 4K/240 generated-source gates were rerun against
the same FIFO Wayland output. H.264 long-run evidence
`/tmp/gilder-h264-pacing-plan-4k240-2400-a` passed `decoded_frame_count=2400`,
`presented_frame_count=2400`, `runtime_elapsed_ms=10012`,
`pacing_strategy=target-fps-cpu-sleep-with-fifo-present`,
`average_present_fps=239.69567237903803`, and
`average_present_result_drop_first_60_fps=239.9397391191329`. The shorter
480-frame H.265/AV1 matrix also passed with the same expected pacing strategy:
H.265 Main8 `/tmp/gilder-h265-main8-pacing-plan-4k240-480-a` reported
`average_present_fps=240.10211819204144`; H.265 Main10
`/tmp/gilder-h265-main10-pacing-plan-4k240-480-a` reported
`average_present_fps=239.89495455740342`; AV1 Main8
`/tmp/gilder-av1-main8-pacing-plan-4k240-480-a` reported
`average_present_result_drop_first_60_fps=239.96389660133235`,
`av1_display_copy_count=0`, and `av1_displayed_direct_dpb_count=480`; AV1
Main10 `/tmp/gilder-av1-main10-pacing-plan-4k240-480-a` reported
`average_present_result_drop_first_60_fps=239.9814887787176`,
`av1_display_copy_count=0`, and `av1_displayed_direct_dpb_count=480`.

The next 2026-06-24 pacing pass moved the repeated per-loop
`next_frame += interval` sleep blocks into `src/renderer/native_vulkan/pacing.rs`
as `NativeVulkanVideoClockPacer`, matching the ffplay-style frame timer shape
more closely: target deadlines are accumulated with integer nanoseconds,
sleeping stops before a configurable spin margin, short lateness is allowed to
catch up on later frames, and only larger lateness resynchronizes the timer. The
first step still uses target FPS as the master clock, so it is a safe bridge
toward later audio-clock pacing rather than an audio policy change. Real
Wayland evidence stayed stable or improved: the same real 1440p60 H.264 Main
file in `/tmp/gilder-h264-real-kamen-1-ffmpeg-clock-pacer-10s-1440p60-600`
reported `runtime_elapsed_ms=9988`,
`average_present_result_drop_first_60_fps=59.98499818598243`,
`frame_sleep_count=599`, and `missed_frame_pacing_count=0`. H.264 4K/240
long-run evidence `/tmp/gilder-h264-ffmpeg-clock-pacer-4k240-2400-a` reported
`runtime_elapsed_ms=10005`, `average_present_fps=239.8605765305128`,
`average_present_result_drop_first_60_fps=240.0756582168383`,
`missed_frame_pacing_count=9`, and `max_frame_pacing_late_us=641`. H.265 Main8
`/tmp/gilder-h265-main8-ffmpeg-clock-pacer-4k240-480-a` and Main10
`/tmp/gilder-h265-main10-ffmpeg-clock-pacer-4k240-480-a` reported
`average_present_fps=240.1629163155045` and `240.10727993267392`. AV1 Main8
`/tmp/gilder-av1-main8-ffmpeg-clock-pacer-4k240-480-a` and Main10
`/tmp/gilder-av1-main10-ffmpeg-clock-pacer-4k240-480-a` kept direct-DPB display
with `av1_display_copy_count=0`; their warmup-dropped present-result FPS values
were `239.95496406116047` and `240.01269375010384`.

Follow-up H.264 copy-cost work on 2026-06-23 made the
`GILDER_H264_DISPLAY_HANDOFF=direct-sampled-dpb-output` path use a persistent
direct-DPB present worker instead of doing acquire/record/submit/present on the
decode loop. The direct path now prebinds one sampled descriptor set per decoded
DPB layer, keeps descriptor updates at zero during playback, uses a 2-frame clear
present preroll to remove FIFO startup cost from measured decode/present
throughput, and requests two present queues by default when the selected present
queue family exposes them. Real Wayland 4K/240 gates passed with no display ring:
`/tmp/gilder-h264-direct-sampled-dpb-default-q2-preroll-4k240-2400-a` reported
`decoded_frame_count=2400`, `presented_frame_count=2400`,
`h264_present_frame_preroll_count=2`, `h264_present_queue_count=2`,
`h264_async_present_depth=2`,
`h264_display_handoff_strategy=direct-sampled-dpb-output`,
`h264_display_ring_slot_count=0`, `h264_display_ring_memory_bytes=0`,
`h264_display_copy_count=0`, `h264_packet_queue_retained_payload_bytes=0`,
`descriptor_update_sum=0`, `average_present_fps=239.12677751832481`,
`average_present_result_fps=239.15832221115875`, and
`average_present_result_drop_first_60_fps=239.39705921336318`. A manual
comparison with `GILDER_H264_PRESENT_QUEUE_COUNT=2` before making that the direct
default, `/tmp/gilder-h264-direct-sampled-dpb-present-worker-preroll-presentq2-4k240-2400-a`,
reported `average_present_fps=239.7187247637892`,
`average_present_result_drop_first_60_fps=239.9289342600875`,
`h264_display_copy_count=0`, and `h264_display_ring_memory_bytes=0`. The older
no-preroll direct worker gate
`/tmp/gilder-h264-direct-sampled-dpb-present-worker-4k240-2400-a` was already
correct and zero-copy (`h264_display_copy_count=0`) but only averaged
`237.71277173477333fps` because the first FIFO frames were slow; after warmup it
was `239.5859741310655fps`. The new default turns that observation into the
direct-DPB presentation contract instead of loosening the gate.

The same 2026-06-23 present-worker/default-preroll/descriptor-prebind pattern was
then moved to H.265 Main8/Main10. H.265 already sampled the decoded resource
directly; the follow-up removed per-frame descriptor updates, moved
acquire/record/submit/present to a persistent worker, defaulted to two present
queues when available, and added H.265 present telemetry. Real Wayland 4K/240
gates passed:
H.265 Main8 `/tmp/gilder-h265-main8-present-worker-preroll-q2-4k240-2400-a`
reported `decoded_frame_count=2400`, `presented_frame_count=2400`,
`picture_format=G8_B8R8_2PLANE_420_UNORM`,
`h265_present_frame_preroll_count=2`, `h265_present_queue_count=2`,
`h265_async_present_depth=2`, `h265_acquire_not_ready_count=0`,
`h265_packet_queue_retained_payload_bytes=0`, `bitstream_ring_wrap_count=253`,
`descriptor_update_sum=0`, and `average_present_fps=240.1206840555046`.
H.265 Main10 `/tmp/gilder-h265-main10-present-worker-preroll-q2-4k240-2400-a`
reported `decoded_frame_count=2400`, `presented_frame_count=2400`,
`picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`,
`h265_present_frame_preroll_count=2`, `h265_present_queue_count=2`,
`h265_async_present_depth=2`, `h265_acquire_not_ready_count=0`,
`h265_packet_queue_retained_payload_bytes=0`, `bitstream_ring_wrap_count=245`,
`descriptor_update_sum=0`, and `average_present_fps=240.07289114727573`.

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
after 20s and produced empty runtime/stderr. These are historical negative
results for the old per-frame fence/display-ring branch; the 2026-06-23
direct-DPB present worker supersedes them for zero-copy H.264 validation.

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

AV1 arbitrary-entry visible direct correctness is now gated in the same style as
H.264/H.265, with one AV1-specific distinction: WebM/`av1parse` may discard the
broken pre-key prefix before packets reach Gilder's streaming queue. The AV1
smoke therefore records either runtime queue discard or demux/parser prefix
discard as explicit evidence, and requires loop replay, first-frame key restart,
zero retained payload, 8-slot bitstream ring, DPB/session consistency, and
optional readback diversity. Real `WAYLAND_DISPLAY=wayland-1`, `HDMI-A-1`
evidence:

- Main8 640x368/60 arbitrary-entry loop/readback:
  `/tmp/gilder-av1-arbitrary-main8-script-gate`, `presented=120`,
  `playback_loop_count=2`, `loop_boundary_reset_count=1`,
  `arbitrary_entry_demux_dropped_prefix=yes`, `first_key_pts=0.650000`,
  `readback_y_distinct=5`, `readback_uv_distinct=5`.
- Main10/P010 640x368/60 arbitrary-entry loop/readback:
  `/tmp/gilder-av1-arbitrary-main10-script-gate`, `presented=120`,
  `playback_loop_count=2`, `loop_boundary_reset_count=1`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`,
  `readback_y_distinct=5`, `readback_uv_distinct=5`.
- Main8 4K/240 arbitrary-entry loop/readback:
  `/tmp/gilder-av1-main8-arbitrary-4k240-script-gate`, `presented=480`,
  `decoded=260`, `hidden_decoded=238`, `displayed_handoff=220`,
  `playback_loop_count=2`, `readback_y_distinct=5`,
  `average_present_fps=214.3091083253833`.
- Main10/P010 4K/240 arbitrary-entry loop/readback:
  `/tmp/gilder-av1-main10-arbitrary-4k240-script-gate`, `presented=480`,
  `decoded=260`, `hidden_decoded=238`, `displayed_handoff=220`,
  `playback_loop_count=2`, `readback_y_distinct=5`,
  `average_present_fps=195.08401023440956`.

Historical longer no-readback 4K/240 performance samples kept the same
arbitrary-entry loop correctness but still missed stable 240fps before the AV1
displayed direct-DPB pass. Main8
`/tmp/gilder-av1-main8-arbitrary-4k240-performance` reports
`presented=2400`, `playback_loop_count=8`, `loop_boundary_reset_count=7`,
`average_present_fps=212.34285202161269`, RSS/PSS/USS/Private_Dirty
max `105732/72353/58468/29380 KiB`, average CPU `23.41%`, NVIDIA process GPU
memory `180 MiB`. Main10/P010
`/tmp/gilder-av1-main10-arbitrary-4k240-performance` reports
`presented=2400`, `playback_loop_count=8`, `loop_boundary_reset_count=7`,
`average_present_fps=211.02778884327637`, RSS/PSS/USS/Private_Dirty
max `108404/74981/61104/30172 KiB`, average CPU `10.09%`, NVIDIA process GPU
memory `288 MiB`. These are superseded for the synthetic arbitrary-entry path by
the direct-DPB general-layout evidence above: Main8/Main10 now keep 2400-frame
4K/240 pacing with zero AV1 display copy and no display ring. Current status:
arbitrary-entry continuous correctness and synthetic 4K/240 present performance
are usable for Main8/Main10; remaining AV1 work is broader real-wallpaper stream
coverage, longer process sampling, audio/clock integration, and codec-specific
DPB/output memory compaction.

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

- FFmpeg is the first behavioral reference for packet/frame/clock semantics. A
  shallow local FFmpeg checkout is kept at `references/ffmpeg/` (ignored from
  Git; current reference commit `dc953a1`). Primary files for direct-video
  alignment are `fftools/ffplay.c` (`PacketQueue`, `FrameQueue`, `Clock`,
  serial invalidation, audio/video refresh) plus codec parser/decoder state
  under `libavcodec/`.
- GStreamer is the second reference and current frontend implementation, but it
  is replaceable. The stable contract is packet/audio/clock output into Gilder,
  not `gst::Pipeline` ownership inside decode/render/present.
- Route 1 is the bitstream-native-decode route: GStreamer/libav may provide
  demux/parser access units, but H.264/H.265/AV1 picture decode is owned by
  Gilder's native Vulkan Video path. This is not a full-pipeline zero-copy
  route: compressed AU/TU payloads are still copied from the frontend packet
  buffer into Gilder's Vulkan Video bitstream ring. Its zero-copy claim is
  scoped to decoded-frame display handoff when direct-DPB/display telemetry
  proves zero display copies.
- Route 2 is the decoded-frame-frontend route: the provider decodes frames and
  Gilder imports provider-neutral decoded samples for render/present. `gst-dma`
  belongs here as the DMABuf/VA memory route, not as a display-sink bypass.
  This route may claim zero-copy only after caps, memory type and importer
  telemetry confirm a DMABuf/DRM-PRIME Vulkan import contract; hardware decode,
  GPU caps or `CUDAMemory`/`VAMemory` labels alone are not sufficient.
- The current native Vulkan runtime still has `ash` compatibility code, but
  `vulkanalia` is now the primary migration surface inside
  `native-vulkan-renderer`. The gates are still incremental: Vulkan 1.4
  instance/device capability probing, Wayland/swapchain parity, Vulkan Video
  H.264/H.265/AV1 profile+format parity, session/resource migration, one direct
  H.265 submit path, then present/import parity. Binding choice is not itself
  zero-copy evidence; zero-copy still requires extension/capability/import
  telemetry on the selected Vulkan device.
- Native wallpaper-visible Vulkanalia validation must use `background` or
  `bottom` layer-shell surfaces. `top`/`overlay` remain foreground debug only and
  should not be cited as wallpaper smoke evidence. The current
  `--probe-vulkanalia-video-present-session` gate creates a retained runtime
  boundary (`video-present-session-retained-resource`) that owns the Wayland
  surface, Vulkan 1.4 instance, video+present device, swapchain, video session,
  session memory and coincident sampled DPB/output image until the probe drops
  the runtime; H.264 high8, H.265 main8/main10 and AV1 main8/main10 have all
  passed this retained resource gate on `background`/`bottom`.
- H.265 has moved one gate beyond retained resource creation:
  `--run-vulkanalia-ready-prefix-video` now emits
  `h265_retained_video_present_decode` for Main8/Main10. This submits the
  ready-prefix decode commands to the retained video-present session and writes
  into the retained coincident DPB/output sampled image on queue family `3`,
  with the image created for queue families `[3,0]`. Current evidence covers
  240 submitted frames for Main8/NV12 and Main10/P010 on `background`. This is
  still not the final presented zero-copy gate; `decoded_image_zero_copy_presented`
  remains false until the graphics present pass samples that retained decoded
  image into the swapchain.
- The retained H.265 image now also has Vulkanalia graphics sampling resources.
  `src/renderer/native_vulkan/vulkanalia_backend/render_present.rs` creates a
  `VkSamplerYcbcrConversion`, immutable YCbCr sampler, converted 2D image view
  and combined-image-sampler descriptor set for the retained decoded image after
  the video+present device enables `samplerYcbcrConversion`. Real Wayland
  `background` smoke confirms this for Main8/NV12 and Main10/P010 with
  `retained_submitted=240`, `decoded_image_present_sampler.route=
  decoded-image-ycbcr-sampler-present-resource` and no sampler error. The next
  validation gate is recording the dynamic-rendering fullscreen draw and
  presenting that sampled decoded image, so `decoded_image_zero_copy_presented`
  remains false for this step.
- `src/renderer/native_vulkan/demux.rs` owns the frontend-agnostic packet queue,
  access-unit timeline, loop serial and bootstrap window. The current
  GStreamer provider lives in `src/renderer/native_vulkan/demux_gst.rs`;
  future libav/FFmpeg or native demux providers should implement the same
  packet frontend boundary.
- `src/renderer/native_vulkan/timeline.rs` owns codec-neutral timeline checks
  derived from ffplay's queue-serial model: loop-boundary detection and stale
  frame serial rejection must be shared by H.264/H.265/AV1 instead of repeated
  as ad hoc `source_loop_index` comparisons inside codec loops.
- The old decoded-frame appsink/importer route has been retired from the active
  code path. Future provider work should attach at the demux/packet boundary or
  at an explicit helper texture handoff, not by reintroducing render-owned
  `gst::Sample` handling.
- Web wallpaper support should follow the same provider boundary: no `gtk-rs`
  or WebKitGTK dependency should enter native Vulkan core. A web provider may
  be an external process, WPE/CEF/headless engine, or native Wayland surface
  producer as long as Gilder receives a bounded render handoff.
- High-performance web wallpaper requires GPU handoff (`DMABuf`, `EGLImage`,
  Vulkan external image, or an independently composited Wayland surface).
  CPU screenshot/RGBA frames are only an explicit fallback path because they
  reintroduce the same copy cost that the video path is removing.
- `src/renderer/native_vulkan/interop.rs` owns the stable external interop
  policy surface for decoded video and future Web/helper texture handoff:
  target memory flow, Vulkan binding migration policy, accepted frame sources and designs
  that must not enter the native Vulkan core.
- `src/renderer/native_vulkan/audio_frontend.rs` owns the provider-neutral
  audio clock runtime wrapper and reports `audio_runtime_provider` into video
  runtime telemetry. The current implementation is still the GStreamer AAC
  clock probe in `audio_clock.rs`, but pacing/render code should depend on the
  wrapper contract, not directly on the GStreamer probe type.
- The generic video session forwards decoded-frontend loop/segment boundaries
  into the audio runtime. Segment-done growth triggers
  `seek_for_video_loop(loop_start_position_ms)` on the audio frontend, and the
  audio worker keeps loop seek commands ahead of ordinary video-clock samples
  when coalescing queued work. This follows the FFmpeg/ffplay clock-serial rule:
  loop/seek boundaries must reset audio clock state to the actual segment start
  instead of letting stale samples drift across segments.
- H.264/H.265/AV1 direct Vulkan Video runtime JSON now reports unified
  `decoded_frame_zero_copy_scope` and `decoded_frame_zero_copy_status` fields.
  The scope is explicitly decoded-frame display handoff; bitstream-ring upload
  remains a separate copy scope. H.264/AV1 can classify confirmed direct-DPB
  no-display-copy paths from display-copy/direct-DPB counters; H.265 now reports
  the same display-copy count, display-ring memory and displayed direct-DPB
  count from its direct sampled output path.
- `src/renderer/native_vulkan/direct_runtime.rs` owns codec-neutral direct
  runtime summary calculations: elapsed time, average present FPS and
  decoded-frame zero-copy classification. H.264/H.265/AV1 adapters still own
  codec-specific parser, reference and DPB state, but common display-handoff
  evidence should flow through this helper so performance comparisons do not
  drift across codecs.
- H.265 direct runtime now reports the same present-result interval family used
  by H.264/AV1 (`average_present_result_fps`, drop-first variants, over-budget
  and missed-vblank counters) and exposes `GILDER_H265_ASYNC_PRESENT_DEPTH` for
  bounded present-worker backpressure tuning. This makes H.265 performance
  diagnosis attributable before deeper decode-ahead changes.
- The H.264, H.265 and AV1 direct runtimes now all use the shared
  `native_vulkan_direct_present_result_summary` helper for present-result FPS
  and missed-vblank classification, keeping cross-codec performance telemetry
  on one calculation path.
- `src/renderer/native_vulkan/render_item.rs` owns render-sync-plan to
  `NativeVulkanRenderItem` mapping. It is the thin integration boundary for
  wallpaper/control sources and must not accumulate decode, render or present
  state.
- `src/renderer/scene_lite_display.rs` owns scene-lite display fallback
  decisions such as direct color clear eligibility, snapshot renderability and
  fallback background selection. This keeps scene display policy out of the
  render sync coordinator while preserving backend-neutral `SceneLiteDisplayPlan`
  output for native Vulkan and future backends.
- `src/renderer/native_vulkan/render_plan.rs` owns render-item to native Vulkan
  session setup and draw planning decisions such as static upload source
  selection, scene-lite color clear fallback and native scene-lite draw-op
  classification. Scene-lite wallpaper work should extend this planning
  boundary instead of adding scene-specific branches to the Vulkan session.
- Pure scene-lite color surfaces, including a single opaque full-target
  rectangle without stroke/corner/transform effects, are planned as
  `SceneLiteDisplayPlan::Color` instead of generated SVG snapshots. Native
  Vulkan can clear the swapchain directly and avoid an unnecessary snapshot
  file, image decode and upload.
- A single opaque, untransformed scene-lite image layer is planned as a direct
  `SceneLiteDisplayPlan::Image` source instead of a generated SVG snapshot.
  Scene-lite image resource accounting de-duplicates that display source from
  the layer source so cache telemetry matches the actual resource set.
- `src/renderer/native_vulkan/scene_lite_runtime.rs` owns scene-lite native draw
  plan telemetry. It reports whether the current deterministic scene snapshot
  can be taken over by native Vulkan draw passes, plus fallback availability and
  unsupported layer reasons, without making the session parse scene manifests.
- GStreamer may provide demux/parser/appsink/audio/clock.
- GStreamer display sinks must not own the visible surface.
- Native Wayland owns layer-shell surface/output/scale/viewport/dmabuf feedback.
- Native Vulkan owns import/decode/render/present.
- NVIDIA importer work may use CUDA interop, but CUDA is not the cross-GPU
  abstraction. AMD/Intel work should target VA/DMABuf -> Vulkan external image.
- Historical native-wgpu and GTK numbers may be used as comparison baselines,
  but those backends are no longer buildable paths.

After the 2026-06-24 `demux_gst.rs` provider split, a real
`--probe-video-session --extract-bitstream` check on the repo-local H.265
Main10 3840x2160@240 source reported `samples=4`,
`h265_decode_ready_prefix_count=4`, `stream_format=byte-stream`,
`alignment=au`, `mapped_write_source=extracted-encoded-video-unit`, and
`picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`. This verifies that
GStreamer provider lifecycle and bitstream pipeline construction moved out of
the frontend-agnostic queue without breaking H.265 Main10 bitstream extraction.

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
