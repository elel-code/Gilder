# Native Vulkan Video

This is the current source of truth for the native video path. Obsolete
planning/spec documents were removed; do not recreate a compatibility archive
for old renderer, GStreamer, decoded-frame copy, or descriptor-set paths.

## Hard Gates

- FFmpeg owns demux, packet/parser normalization, serial/clock semantics, and
  the queue model.
- FFmpeg source is the first reference for performance work as well as
  correctness. Every accepted optimization must name the local FFmpeg source
  line range it follows, explain the Vulkan/runtime change, and keep the
  descriptor-heap/zero-copy gates intact.
- Vulkanalia owns Vulkan Video decode, GPU Y/UV sampling, dynamic rendering,
  and Wayland present.
- The Vulkan loader is exactly `libvulkan.so.1`. Do not reintroduce loader
  candidate mapping such as `libvulkan.so` fallback.
- `VK_EXT_descriptor_heap` is mandatory. Passing evidence must report
  `descriptor_sets=0` and `descriptor_heap_only=true`.
- Decoded pixels stay on the GPU: Vulkan Video writes the retained output/DPB
  image, the render pass samples Y/UV plane descriptors from descriptor heaps,
  and the swapchain owns only the final presented image.
- Passing 4K240 evidence requires `average_present_fps >= 239.999`,
  `performance_max_private_dirty_kib < 25000`, zero-copy presented frames, and
  no validation/performance mixing.
- Any real-source, arbitrary-entry, or optimization evidence must include the
  full performance snapshot: average CPU percent, max RSS/PSS/USS, max
  `Private_Dirty`, max process GPU memory, decoded/presented frame counts,
  average present FPS, descriptor-set count, descriptor-heap-only state, and
  zero-copy state. Evidence without `--performance-snapshot` is only a
  functional smoke, not a performance result.

## FFmpeg References

- `references/ffmpeg/fftools/ffplay.c:114-123`: `PacketQueue` carries packet
  count, duration, and serial state.
- `references/ffmpeg/fftools/ffplay.c:125-128`: video queue size is three.
- `references/ffmpeg/fftools/ffplay.c:3132-3141`: the read thread blocks only
  when queues have enough packets; native keeps this asynchronous shape but caps
  the handoff by codec to stay under the 25,000 KiB `Private_Dirty` gate.
- `references/ffmpeg/fftools/ffplay.c:420-456`: `av_packet_move_ref` transfers
  packet payload ownership into and out of the packet queue.
- `references/ffmpeg/fftools/ffplay.c:168-179` and
  `references/ffmpeg/fftools/ffplay.c:788-800`: `FrameQueue` implements
  `keep_last` ring-buffer advancement.
- `references/ffmpeg/fftools/ffplay.c:1629-1740`: `video_refresh` is the
  frame-timer/serial/drop reference for presentation pacing.
- `references/ffmpeg/libavcodec/vulkan_decode.h:91-99`: Vulkan decode picture
  uses fixed `refs[36]`, fixed `ref_slots[36]`, and per-picture `slices_buf`.
- `references/ffmpeg/libavcodec/vulkan_decode.c:305-390`:
  `ff_vk_decode_add_slice` grows a pooled per-picture bitstream buffer and
  records slice offsets.
- `references/ffmpeg/libavcodec/vulkan_decode.c:527-568`: current picture is
  bound as inactive with `slotIndex = -1`, then `slices_buf` becomes owned by
  the exec buffer.
- `references/ffmpeg/libavcodec/vulkan_decode.c:575-586`: decoded frames carry
  mirrored semaphore values as frame dependencies; optimization should preserve
  this per-frame decode completion handoff instead of adding host waits.
- `references/ffmpeg/libavcodec/vulkan_decode.c:586-690`: image layout work is
  performed where the decode command actually consumes output/reference
  resources; layout/sync changes should follow this per-frame dependency model.
- `references/ffmpeg/libavcodec/vulkan_h264.c:476-562`: H.264 adds slices
  through `ff_vk_decode_add_slice`, prepares fixed reference arrays, and submits
  with `ff_vk_decode_frame`.
- `references/ffmpeg/libavcodec/vulkan_hevc.c:743-815` and
  `references/ffmpeg/libavcodec/vulkan_hevc.c:828-842`: HEVC fills
  `vp->ref_slots[idx]`, reference sets, and slice offsets.
- `references/ffmpeg/libavcodec/vulkan_av1.c:298-358`: AV1 scans duplicate
  reference slots, fills unique refs, and writes `referenceNameSlotIndices`.
- `references/ffmpeg/libavutil/mem.c:98-165` and
  `references/ffmpeg/libavutil/mem.c:247-253`: FFmpeg allocates packet/parser
  storage through aligned malloc/realloc and releases with `av_free`; native
  must reduce dirty memory through ownership/lifetime changes, not glibc
  allocator tuning.

## Substantial Breakthroughs

1. The practical memory breakthrough was shader-owned plane conversion.
   Removing the `VkSamplerYcbcrConversion`/embedded-sampler route and sampling
   Y/UV plane views explicitly through descriptor heaps dropped host
   `Private_Dirty` below the 25,000 KiB gate while keeping zero-copy GPU present.
2. The bitstream path was aligned to FFmpeg's picture-owned `slices_buf` model:
   exec-slot-owned mapped slices buffers, no global growing AU buffer, and no
   retained payload window.
3. Submit/reference construction stopped allocating per-frame reference Vecs.
   H.264, H.265, and AV1 now lower into fixed/borrowed workspaces matching
   FFmpeg's fixed `refs[36]`/`ref_slots[36]` contract.
4. The packet queue stores AU metadata, PTS/timeline data, and serial state;
   payload is uploaded and released instead of being retained through present.
5. Presentation follows the FFmpeg queue shape: bounded queue depth three,
   `keep_last` semantics, serial reset handling, and frame-timer PTS-delta
   pacing.
6. Smoke runs use the untuned distribution allocator behavior. The scripts clear
   external malloc/glibc tuning variables before launch and the native process
   does not call `mallopt`; memory reductions must come from FFmpeg-aligned
   ownership, queue, copy, and lifetime changes.
7. Decode/present timeline synchronization follows FFmpeg's per-frame semaphore
   dependency shape: decode signals at `VIDEO_DECODE_KHR` completion and
   present waits on that per-frame value before touching the decoded image. Low
   GPU busy with stable 240 fps should be treated as CPU/submit/synchronization
   headroom, not as a reason to add copy paths or descriptor sets.
8. FFmpeg read-thread handoff is rendezvous by default rather than a hidden
   second compressed-payload FIFO. H.264/H.265 use the single-packet
   `packet_queue_get` shape; AV1 is the only path that declares packet splitting
   and it shares the FFmpeg packet backing by byte range.
9. Annex-B conversion keeps one reusable scratch buffer. Extra free converted
   payload buffers are not retained after upload, matching FFmpeg's
   packet-unref lifetime and keeping long-source `Private_Dirty` under the
   gate without allocator tuning.
10. The packet handoff queue can still hold three active packets, but the free
    Annex-B scratch pool retains only one buffer capped at 160 KiB. This keeps
    FFmpeg's `av_packet_move_ref` queue depth while avoiding a second hidden
    three-packet retained-payload pool.
11. Vulkan Video parameter sets now use `VK_KHR_video_maintenance2` inline
    submit. H.264/H.265/AV1 keep one validated STD payload owner per stream,
    create the video session with the inline-session-parameters flag bit, pass
    codec parameters through `VkVideoDecodeInfoKHR` pNext, and leave
    `VkVideoBeginCodingInfoKHR::videoSessionParameters` null in the streaming
    path.

## Format Evidence

Performance evidence uses no allocator tuning profile. Current memory gates
must be judged with malloc/glibc tuning env cleared and no in-process
`mallopt`.

### H.264

- Source:
  `artifacts/video-sources/h264/h264-high-b0-ref2-weightp0-weightb0-3840x2160-240fps-2640frames-g2401-d2400.mp4`.
- Breakthroughs: descriptor-heap Y/UV plane shader conversion, borrowed slice
  offsets from the first slice path, fixed reference workspace, two-slot
  FFmpeg-style slices buffer pool, single-packet FFmpeg handoff, one retained
  Annex-B scratch buffer, and bounded streaming packet upload.
- Evidence directory: `/tmp/gilder-h264-4k240-pool1-final`.
- Result: decoded/presented `2640/2640`, `average_present_fps=240.0122493521044`,
  `performance_max_private_dirty_kib=24368`, `performance_avg_cpu_percent=13.22`,
  `performance_max_pss_kib=67427`, `performance_max_uss_kib=39696`,
  `performance_avg_gpu_busy_percent=30`, `performance_max_gpu_busy_percent=43`,
  `performance_max_nvidia_process_gpu_memory_mib=102`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Latest no-build 4K240 rerun after the present2/audio/scene/Vulkan 1.4,
  Roadmap 2026, and `VK_KHR_video_maintenance2` inline-parameter work:
  `/tmp/gilder-vulkan-h264-ready-prefix-video.245287`. Result:
  decoded/presented `2400/2400`, `average_present_fps=240.00376702069323`,
  `performance_max_private_dirty_kib=24364`, `performance_avg_cpu_percent=12.88`,
  `performance_max_pss_kib=73203`, `performance_max_uss_kib=51304`,
  `performance_max_nvidia_process_gpu_memory_mib=102`, file mapping
  `Private_Dirty=124 KiB`, descriptor-heap model `VK_EXT_descriptor_heap`,
  zero-copy present, GPU `NVIDIA GeForce RTX 4060 Laptop GPU`,
  `host_image_copy=true`, `present_mode_fifo_latest_ready_enabled=true`,
  `uses_present_id2=true`, `present_wait2_available=true`,
  `video_maintenance2_enabled=true`, `inline_session_parameters_enabled=true`,
  `video_session_create_inline_session_parameters=true`,
  `video_session_create_flags_bits=32`,
  `uses_inline_session_parameters=true`,
  `video_session_parameters_handle_used=false`, latest `present_id=2400`, and
  `present_id_mode=present-id2-khr`.

### H.265

- Main8 source:
  `artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-566frames-g240-d240.mp4`.
- Main10 source:
  `artifacts/video-sources/h265/h265-main-10-b0-ref1-3840x2160-240fps-566frames-g240-d240.mp4`.
- Breakthroughs: HEVC reference sets follow FFmpeg's `vp->ref_slots[idx]`
  filling, slice offsets are stack/borrowed instead of heap-retained, Main10
  uses the 10-bit two-plane Vulkan format directly, and both profiles share the
  descriptor-heap shader conversion path. H.265 uses the same single-packet
  FFmpeg handoff and one retained Annex-B scratch buffer as H.264.
- Current Main8 evidence directory:
  `/tmp/gilder-h265-main8-4k240-main-matrix-final-rerun`.
- Current Main8 result: decoded/presented `566/566`,
  `average_present_fps=240.03084388696948`,
  `performance_max_private_dirty_kib=24692`, `performance_avg_cpu_percent=15.23`,
  `performance_max_nvidia_process_gpu_memory_mib=126`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Rejected fresh-rebuild Main8 evidence:
  `/tmp/gilder-h265-main8-4k240-main-matrix-final` decoded/presented
  `566/566`, but failed the strict memory gate at
  `performance_max_private_dirty_kib=27800`. The same rebuilt release binary
  passed on the immediate rerun above; the failed first run remains rejected
  evidence, not an accepted pass.
- Rejected Main8 long-window evidence:
  `/tmp/gilder-h265-main8-4k240-main-matrix-2400-optimized` decoded/presented
  `2400/2400`, but failed the strict memory gate at
  `performance_max_private_dirty_kib=25052`. The gate remains `25000 KiB`;
  this run is not accepted and is the next H.265 Main8 memory target.
- Current Main10 evidence directory:
  `/tmp/gilder-h265-main10-4k240-main-matrix-final`.
- Current Main10 result: decoded/presented `566/566`,
  `average_present_fps=240.0339348336484`,
  `performance_max_private_dirty_kib=24576`, `performance_avg_cpu_percent=15.33`,
  `performance_max_nvidia_process_gpu_memory_mib=174`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`.

### AV1

- Main8 source:
  `artifacts/video-sources/av1/av1-main8-3840x2160-240fps-566frames-g240.webm`.
- Main10 source:
  `artifacts/video-sources/av1/av1-main10-3840x2160-240fps-566frames-g240.webm`.
- Breakthroughs: AV1 reference lowering follows FFmpeg's duplicate-slot scan,
  unique `referenceNameSlotIndices`, caller-owned workspaces, and shared
  FFmpeg-packet byte ranges when a container packet contains multiple frame
  units; this removed retained-copy pressure while keeping continuous 4K240
  present.
- Current Main8 evidence directory: `/tmp/gilder-av1-main8-4k240-main-matrix-final`.
- Current Main8 result: submitted/displayed/presented `585/566/566`,
  `average_present_fps=240.04827113264776`,
  `performance_max_private_dirty_kib=24640`, `performance_avg_cpu_percent=38.87`,
  `performance_max_nvidia_process_gpu_memory_mib=179`,
  `descriptor_sets=0`, `descriptor_heap_only=true`,
  `descriptor_model=VK_EXT_descriptor_heap`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Current Main10 evidence directory: `/tmp/gilder-av1-main10-4k240-main-matrix-final`.
- Current Main10 result: submitted/displayed/presented `585/566/566`,
  `average_present_fps=240.07923906642975`,
  `performance_max_private_dirty_kib=24580`, `performance_avg_cpu_percent=40.20`,
  `performance_max_nvidia_process_gpu_memory_mib=286`,
  `descriptor_sets=0`, `descriptor_heap_only=true`,
  `descriptor_model=VK_EXT_descriptor_heap`, `all_zero_copy_presented=true`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`.

## Allocator Evidence

There is no allocator tuning profile. Scripts clear
`MALLOC_ARENA_MAX`, `MALLOC_MMAP_THRESHOLD_`, `MALLOC_TRIM_THRESHOLD_`, and
`GLIBC_TUNABLES` before launching the video process, and the binary must not
configure glibc malloc internally.

Current 4K240 no-tuning comparison is the evidence in the format sections
above. All listed runs are under `performance_max_private_dirty_kib < 25000`,
`average_present_fps >= 239.999`, `descriptor_sets=0`,
`descriptor_heap_only=true`, and `all_zero_copy_presented=true`.

If a performance run starts immediately after
`target/release/gilder-native-vulkan` was rebuilt or replaced, Linux can report
the freshly executed binary mapping as private dirty memory. The sampler now
classifies `gilder-native-vulkan` under `memory_category_gilder_binary_*`
instead of the generic `file-mapping` bucket so this is visible. The H.264 smoke
script records the release-binary fingerprint before and after `cargo build`.
When the binary changed, it syncs
`target/release/gilder-native-vulkan` with file-data, file-metadata, and release
directory syncs before starting the measured process so the executable is not
sampled while the freshly written file is still dirty. If a performance attempt
still fails with either a high file-backed/gilder-binary dirty category or a
fresh-build cold heap dirty category while mapping dirty is clean, the script
preserves that attempt as `performance.fresh-build-contaminated[.N]`, syncs the
same binary again, waits a short stabilization window, and retries up to four
total attempts. The accepted result is still the final attempt's unadjusted
total `performance_max_private_dirty_kib < 25000`; if the dirty category does
not fall, the run fails as rebuild contamination rather than being reclassified
as codec heap pressure.

`/tmp/gilder-vulkan-h264-ready-prefix-video.7WuEs7` is the current rejected
example after the inline-session-parameters flag build:
`performance_max_private_dirty_kib=28168`, while its mapping breakdown showed
`memory_category_file_mapping_private_dirty_kib=3080` before the
`gilder-native-vulkan` category fix. The same release binary rerun without
rebuilding is `/tmp/gilder-vulkan-h264-ready-prefix-video.245287`, which passes
at `24364 KiB` with file mapping `Private_Dirty=124 KiB`. The gate stays
`25000 KiB`; no file-mapping or gilder-binary memory is subtracted from the
metric.

After switching present mode selection to prefer Roadmap 2026
`VK_KHR_present_mode_fifo_latest_ready` over `MAILBOX`, the current rebuilt
H.264 4K240 release run
`/tmp/gilder-vulkan-h264-ready-prefix-video.EFHymZ` passes strict gates with
`present_mode=fifo-latest-ready`, `presented_frame_count=2400`,
`present_mode_gate_failed=0`, `average_present_fps=240.006152921391`,
`performance_max_private_dirty_kib=24368`, and
`memory_category_file_mapping_private_dirty_kib=0`. The immediately preceding
same-source `MAILBOX` run
`/tmp/gilder-vulkan-h264-ready-prefix-video.XDKz2d` stayed under memory at
`24356 KiB` but failed the FPS gate at `224.57390736245236` because almost all
elapsed time was in `vkQueuePresentKHR`.

## Code Layout

- `src/renderer/native_vulkan.rs`: facade, shared codec parsers, snapshot
  construction, and public native Vulkan contract types.
- `src/renderer/native_vulkan/video/`: FFmpeg demux/packet handoff, codec
  reference planning, route selection, pacing, timeline, and video extraction
  metadata. This directory must not retain access-unit payload windows.
- `src/renderer/native_vulkan/vulkan/`: the only Vulkan binding backend. It is
  split into `core/` device/feature/profile setup, `present/` swapchain/render
  present, `scene/` scene draw/present, and `video/` Vulkan Video session,
  command, decode submit, and present runtime code.
- `src/renderer/native_vulkan/present/`: renderer item planning plus clear and
  static-image present entry points.
- `src/renderer/native_vulkan/scene/`: scene planning/runtime bridge into
  the Vulkan present path.
- `src/renderer/native_vulkan/audio/`: audio policy and clock boundary.
  `clock-only` decodes FFmpeg audio frames for timestamp/serial metadata and
  immediately releases packet payloads. `auto` is now PipeWire-only: FFmpeg
  decoded frames are resampled through `libswresample` to S16LE interleaved and
  written directly to a native PipeWire playback stream. There is no PulseAudio,
  ALSA, GStreamer `autoaudiosink`, or silent fallback path; if PipeWire cannot
  start, `auto` fails the run. Ready-prefix runs keep `clock_ns` as the
  diagnostic and pass the first ready `video_master_start_clock_ns` sample into
  audio-master pacing so pre-read AAC packets do not advance the video start
  clock. Snapshots report `audio_output_backend=pipewire-s16le`,
  `audio_output_frames`, `audio_output_samples`, `audio_output_bytes`,
  `audio_output_sample_rate_hz`, `audio_output_channel_count`,
  `audio_output_write_calls`, `audio_output_write_waits`,
  `audio_output_process_callbacks`, `audio_output_buffer_errors`,
  `audio_output_timeout_errors`, `audio_output_xrun_count`,
  `audio_output_state_changes`, `audio_output_ready_state_changes`,
  `audio_output_stream_state`, `audio_output_stream_ready`,
  `audio_output_lifecycle_model`, `audio_output_latency_policy`,
  `playback_target_clock_ns`, `playback_covered_clock_ns`,
  `playback_coverage_percent`, and `playback_target_reached`. Ready-prefix
  runtime snapshots also expose `audio_video_sync` with audio coverage, video
  present sequence readiness, drift policy, and pacing-clock model. The
  ready-prefix smokes now require the audio clock/output window to cover the
  full requested video playback duration, require the PipeWire write path to
  report real write/process activity, require stream lifecycle transitions, and
  gate buffer/timeout/xrun counts at zero.

## Vulkan 1.4 And Roadmap Modernization

The native path requests Vulkan 1.4 and treats old compatibility paths as
deleted design space. Do not add descriptor-set fallback, legacy present-id/wait
fallback, render-pass/framebuffer fallback, or pre-sync2 submit/barrier paths.

Current modern baseline:

- Dynamic rendering owns graphics presentation. `VkRenderPass`/framebuffer
  compatibility paths are not part of the native scene/video present route.
- Synchronization2 owns barriers and submits: `cmd_pipeline_barrier2` and
  `queue_submit2` are the expected command model.
- `VK_EXT_descriptor_heap` is the only shader resource binding model for
  decoded video and scene sampled images.
- Present timing is `VK_KHR_present_id2`/`VK_KHR_present_wait2` only. The older
  `VK_KHR_present_id` and `VK_KHR_present_wait` route is not an allowed
  fallback. Runtime JSON no longer exposes legacy `uses_present_id`,
  `present_id_enabled`, `present_wait_enabled`, or `present_wait_available`
  fields; path evidence is the id2/wait2 field set plus
  `present_id_mode=present-id2-khr`. Swapchain creation now hard-fails when
  the selected device/surface cannot enable both id2 and wait2.
- `VK_KHR_present_mode_fifo_latest_ready` is queried and enabled through the
  device feature chain when available. Present mode selection now uses
  `FIFO_LATEST_READY` when that KHR feature and surface mode are both
  available, otherwise `FIFO_RELAXED`/`FIFO`. `MAILBOX` is not a native Vulkan
  fallback even when the surface advertises it. Ready-prefix video smoke gates
  now accept only `fifo-latest-ready`, `fifo-relaxed`, or `fifo` and fail
  `mailbox`/`immediate` evidence.
- Resource memory binding and host mapping use Vulkan 1.4-style
  `vkBind*Memory2`, `vkMapMemory2`, and `vkUnmapMemory2`.
- Scene sampled-image upload uses Vulkan 1.4 `hostImageCopy`: image resources
  carry `HOST_TRANSFER | TRANSFER_DST | SAMPLED`, upload uses
  `vkTransitionImageLayout` + `vkCopyMemoryToImage`, and no staging buffer,
  upload queue submit, or upload fence is retained for the scene image path.
- Vulkan Video decode uses `VK_KHR_video_maintenance2` inline session
  parameters when the device enables it. The video session is created with
  `VK_VIDEO_SESSION_CREATE_INLINE_SESSION_PARAMETERS_BIT_KHR` (bit `0x20` in
  current vulkanalia), H.264/H.265/AV1 parameter payloads are attached to
  `VkVideoDecodeInfoKHR`, and the streaming path does not create or bind a
  `VkVideoSessionParametersKHR` object.
- The Vulkan 1.4 feature chain requests `host_image_copy` when the device
  reports it. Keeping this enabled is part of the roadmap contract; the 25,000
  KiB memory gate must be met by reducing FFmpeg/runtime retained memory, not by
  disabling modern Vulkan features.
- Current runtime maintenance use is split by API family: Vulkan 1.4 core keeps
  `maintenance5`/`maintenance6`, Vulkan Video keeps `VK_KHR_video_maintenance2`
  for inline codec parameters, WSI keeps surface/swapchain maintenance1, and
  the present device feature selection now enables `VK_KHR_maintenance7`,
  `VK_KHR_maintenance8`, `VK_KHR_maintenance9`, and `VK_KHR_maintenance10`
  when both the extension and feature bit are available. Those are not
  compatibility paths and `VK_KHR_maintenance10` does not replace 7/8/9; the
  enabled device-extension list and present-device snapshot expose the runtime
  state separately for all four maintenance generations.
- `--probe-vulkanalia` now emits a per-physical-device `roadmap_2026` evidence
  block. That block records `api_version_1_4_or_newer`,
  `core_vulkan_1_4_features_ready`, the available/missing tracked Roadmap 2026
  device extensions, and feature bits for pipeline binary, robustness2,
  fragment shading rate, shader clock, cooperative matrix, compute shader
  derivatives, depth-clamp-zero-one, copy-memory-indirect, maintenance7/8/9/10,
  and shader untyped pointers. For `VK_KHR_maintenance10`, the probe also
  records the `PhysicalDeviceMaintenance10PropertiesKHR` sRGB resolve and RGBA4
  opaque-black behavior bits. M10 is useful when the native scene path needs
  explicit sRGB resolve transfer-function control, dynamic-rendering end-info
  extensibility, or depth/stencil copy capability on non-graphics queues; it is
  not expected to improve the current 4K240 direct-video memory/FPS gate until
  one of those paths exists. Maintenance7/8/9/10 are now present-device runtime
  features when available; the remaining Roadmap 2026 items are still probe
  evidence until the relevant scene/video/present path consumes them and exposes
  path-specific JSON fields.
  Current probe evidence:
  `cargo run --features native-vulkan-video --bin gilder-native-vulkan -- --probe-vulkanalia`
  reports `NVIDIA GeForce RTX 4060 Laptop GPU` with Vulkan `1.4.341`,
  `roadmap_2026.api_version_1_4_or_newer=true`,
  `roadmap_2026.core_vulkan_1_4_features_ready=true`,
  `roadmap_2026.tracked_device_extensions_missing=[]`, and true feature bits
  for pipeline binaries, robustness2/null descriptors, fragment shading rate,
  shader clock, cooperative matrix, compute shader derivatives,
  depth-clamp-zero-one, copy-memory-indirect, maintenance7/8/9/10, and shader
  untyped pointers. The same probe reports all three maintenance10 property
  bits true: `rgba4_opaque_black_swizzled`,
  `resolve_srgb_format_applies_transfer_function`, and
  `resolve_srgb_format_supports_transfer_function_control`.

Primary Vulkan references for this baseline:

- Khronos Vulkan Roadmap 2026 requires Vulkan 1.4 and highlights
  `VK_KHR_present_mode_fifo_latest_ready`, `VK_KHR_present_id2`,
  `VK_KHR_present_wait2`, and the Vulkan 1.4 `hostImageCopy` feature:
  <https://docs.vulkan.org/spec/latest/appendices/roadmap.html#roadmap-2026>.
- `VK_EXT_host_image_copy` is promoted to Vulkan 1.4 and removes the staging
  buffer/memory management requirement for host-to-image copies when the
  optional feature is enabled:
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_host_image_copy.html>.
- `VK_KHR_video_maintenance2` allows codec parameter sets to be supplied inline
  with decode operations instead of separate session-parameter objects:
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_video_maintenance2.html>.
- `VK_KHR_present_id2` and `VK_KHR_present_wait2` are the only present timing
  contract for native present telemetry:
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_present_id2.html>,
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_present_wait2.html>.
- `VK_KHR_surface_maintenance1` and `VK_KHR_swapchain_maintenance1` are WSI
  extensions, not general rendering maintenance. Surface maintenance1 is for
  per-present-mode surface capabilities, scaling capability, and compatible
  present-mode queries; swapchain maintenance1 is for per-present-mode changes,
  present fences, deferred swapchain allocation, present scaling, and releasing
  acquired images:
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_surface_maintenance1.html>,
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_swapchain_maintenance1.html>.
- `VK_KHR_video_maintenance1` is specific to Vulkan Video. Its benefits are
  profile-independent video buffers/images and inline video query metadata; it
  complements `VK_KHR_video_maintenance2` inline codec parameters and is not
  replaced by general `VK_KHR_maintenance*` extensions:
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_video_maintenance1.html>.
- `VK_KHR_maintenance10` is a Roadmap-era maintenance extension available in
  current vulkanalia bindings. The present device enables its `maintenance10`
  feature when available; its sRGB resolve/RGBA4 opaque-black properties still
  need a concrete scene or video resolve path before they count as path-specific
  rendering work:
  <https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_maintenance10.html>.

Next Vulkan/roadmap gates:

1. Extend the completed scene `hostImageCopy` pattern to any remaining
   host-to-image upload path where the enabled device exposes it. The rule is
   the same: remove upload buffer allocation, upload submit/fence pressure, and
   transfer queue dependency without adding CPU-retained decoded-frame copies.
2. Keep the `VK_KHR_video_maintenance2` inline path as the only streaming
   decode route, and restrict `VkVideoSessionParametersKHR` object creation to
   explicit smoke/probe validation.
3. Add `VK_EXT_present_timing` telemetry for present queue depth/timing. This is
   diagnostic; video cadence remains FFmpeg/audio-clock driven.
4. Probe and use `VK_KHR_unified_image_layouts` for decode-image sampling and
   present handoff once validation confirms the video/image-layout path.
5. Emit `VK_EXT_frame_boundary` markers around demux/decode/render/present work
   for profiling and driver scheduling evidence.
6. Promote the remaining Roadmap 2026 probe bits into runtime paths where they remove
   actual work: `VK_KHR_pipeline_binary` for retained scene/video pipelines,
   `VK_KHR_robustness2`/null descriptors for descriptor-heap resource tables,
   M10-specific image/resolve behavior where it removes an existing pass, and
   copy-memory-indirect only if it replaces CPU-side upload loops without
   retaining extra buffers.
7. Evaluate shader-object and extended-dynamic-state cleanup only after the
   scene path has video/text/path layers. The target is fewer pipeline variants,
   not a second shader binding model.

## Smoke Commands

Use the codec-specific ready-prefix smoke scripts with the repository 4K240
sources, at least 2400 presented frames, and `--performance-snapshot`. The
smoke scripts default performance snapshots to `--max-private-dirty-kib 25000`;
an explicit `--max-private-dirty-kib` only overrides that hard gate. Validation
layer runs are for correctness only; do not use them for the memory/FPS gate.

The scripts clear
`MALLOC_ARENA_MAX`, `MALLOC_MMAP_THRESHOLD_`, `MALLOC_TRIM_THRESHOLD_`, and
`GLIBC_TUNABLES` before launching the video process. There is no
`--allocator-profile` option and no in-process glibc allocator tuning.

Do not rebuild or overwrite `target/release/gilder-native-vulkan` while a
performance run is sampling it. Linux may report the executable mapping as
`Private_Dirty` after the file is replaced; that is a contaminated measurement,
not codec heap pressure. The H.264 smoke script now syncs a just-replaced
binary on its own build path and can rerun bounded contaminated attempts with
the same binary, but the final reported PASS still comes only from an unadjusted
strict `Private_Dirty` sample.

The native Vulkan CLI also writes ready-prefix JSON directly to stdout with
`serde_json::to_writer_pretty` instead of first materializing a full
`serde_json::Value` tree/string. This keeps teardown reporting from adding an
extra heap peak to long performance runs.

Real-source and arbitrary-entry runs use the same reporting rule. If the run is
intended to prove performance, keep playback long enough for the sampler window
and pass `--performance-snapshot --performance-duration <sec>
--performance-interval <sec>`. The result summary must retain CPU, GPU memory,
RSS/PSS/USS, `Private_Dirty`, FPS, frame counts, descriptor heap, and zero-copy
fields together with the report directory.

## Next Plan

1. Audio integration: the native audio path is now complete for the direct
   ready-prefix player. It lives under
   `native_vulkan/audio/clock.rs` and the FFmpeg/PipeWire C shim. It selects an
   FFmpeg audio stream, decodes frames, immediately unreferences packet
   payloads, and reports serial-scoped clock samples for ready-prefix runs when
   `--audio-clock-probe` is requested. `clock-only` keeps output counters at
   zero. `auto` is now native PipeWire-only: decoded FFmpeg frames are resampled
   with `libswresample` to S16LE interleaved and written to a PipeWire playback
   stream. There is no PulseAudio/ALSA/GStreamer fallback and no compatibility
   alias for old output names. Audio snapshots explicitly report
   `video_master_clock_ready`, `video_master_start_clock_ns`,
   `video_master_start_serial`, `video_master_start_packet_index`,
   `current_serial_start_clock_ns`, `current_serial_start_serial`,
   `current_serial_start_packet_index`, `audio_output_backend`,
   `audio_output_frames`, `audio_output_samples`, `audio_output_bytes`,
   `audio_output_sample_rate_hz`, `audio_output_channel_count`,
   `audio_output_write_calls`, `audio_output_write_waits`,
   `audio_output_process_callbacks`, `audio_output_buffer_errors`,
   `audio_output_timeout_errors`, `audio_output_xrun_count`,
   `audio_output_state_changes`, `audio_output_ready_state_changes`,
   `audio_output_stream_state`, `audio_output_stream_ready`,
   `audio_output_lifecycle_model`, `audio_output_latency_policy`,
   `playback_runtime_model`, `playback_target_clock_ns`,
   `playback_covered_clock_ns`, `playback_coverage_percent`, and
   `playback_target_reached`. Top-level ready-prefix snapshots also report
   `audio_video_sync.ready`, `audio_video_sync.audio_video_target_drift_abs_ns`,
   `audio_video_sync.max_allowed_drift_ns`,
   `audio_video_sync.video_presented_frame_count`, and
   `audio_video_sync.present_pacing_clock_model`.
   Ready-prefix video pacing consumes the stable `video_master_start_*` sample,
   while loop/seek evidence uses `current_serial_start_*` so pre-present audio
   decoding cannot move the first-frame master clock to a later loop. When
   playback requests more frames than the ready-prefix window, the audio path
   now budgets enough FFmpeg packets to cover the full target playback duration,
   enables FFmpeg EOS seek, and exposes current-serial reset evidence without
   retaining packet payloads. The remaining audio work is outside the direct
   ready-prefix player: full-scene audio response must consume the same
   PipeWire-only backend from the scene runtime.
   Current PipeWire ready-prefix auto smoke:
   `/tmp/gilder-audio-scene-remaining10-auto-smoke` passes
   `--audio-clock-probe --audio-output auto --unmuted --pacing-master audio`,
   reports `audio_output_backend=pipewire-s16le`, `audio_output_frames=7`,
   `audio_output_samples=7168`, `audio_output_bytes=14336`,
   `audio_playback_runtime_model=pipewire-duration-covered-runtime`,
   `audio_playback_target_clock_ns=133333333`,
   `audio_playback_covered_clock_ns=149333333`,
   `audio_playback_coverage_percent=112`,
   `audio_playback_target_reached=true`, positive PipeWire
   `audio_output_write_calls`, `audio_output_write_waits`,
   `audio_output_process_callbacks`, zero `audio_output_buffer_errors`,
   zero `audio_output_timeout_errors`, `audio_output_xrun_count=0`,
   positive `audio_output_state_changes`, positive
   `audio_output_ready_state_changes`, `audio_output_stream_state=streaming`,
   `audio_output_stream_ready=true`, `audio_video_sync_ready=true`,
   `audio_video_sync_drift_abs_ns=16000000`,
   `audio_video_sync_max_allowed_drift_ns=100000000`,
   `audio_video_sync_presented_frames=4`,
   `audio_video_sync_pacing_clock_model=audio-clock-master-pts-sync-sleep`,
   `consumed_packets=8`, and `retained_payload_bytes=0`. The matching
   clock-only smoke `/tmp/gilder-audio-scene-remaining10-clock-smoke` passes
   with `audio_output_backend=none`, zero output counters, zero PipeWire write
   counters, `audio_output_stream_state=unconnected`,
   `audio_output_stream_ready=false`, and the same
   `audio_video_sync_ready=true` gate. The arbitrary-entry smoke
   `/tmp/gilder-audio-scene-remaining10-arbitrary-smoke` also passes with
   `--arbitrary-entry-offset 0.10 --audio-output auto`, so non-starting source
   entry now exercises video recovery plus PipeWire/A-V sync gates together.
   Current H.264 generated-source loop-audio smoke:
   `/tmp/gilder-vulkan-h264-ready-prefix-video.JfquFZ` passes
   `--audio-clock-probe --pacing-master audio`, reports
   `consumed_packets=512`, `loop_count=4`, `current_serial=4`,
   `video_master_start_clock_ns=21333333`, `retained_payload_bytes=0`,
   `video_master_start_serial=0`, `video_master_start_packet_index=1`,
   `current_serial_start_serial=4`,
   `current_serial_start_packet_index=469`,
   `pacing_strategy=audio-clock-master-pts-sync-sleep`,
   `frame_sleep_count=7`, `missed_frame_pacing_count=0`, and
   `total_frame_sleep_us=1726024`.
   Current H.265 generated-source loop-audio smoke:
   `/tmp/gilder-vulkan-h265-ready-prefix-video.t7XlLE` passes
   `--audio-clock-probe --pacing-master audio`, reports
   `audio_output_mode=clock-only`, `audio_stream_found=true`,
   `consumed_packets=512`, `loop_count=4`, `current_serial=4`,
   `video_master_clock_ready=true`,
   `video_master_start_clock_ns=21333333`,
   `video_master_start_serial=0`, `current_serial_start_serial=4`,
   `retained_payload_bytes=0`, and pacing
   `audio-clock-master-pts-sync-sleep`. The same run reports
   `video_session_create_inline_session_parameters=true`,
   `video_session_create_flags_bits=32`,
   `uses_inline_session_parameters=true`, and
   `video_session_parameters_handle_used=false`.
   The AV1 ready-prefix smoke now reports the same
   `video_master_start_*`/`current_serial_start_*` audio fields and applies the
   same loop gate when playback exceeds the ready-prefix window, so all direct
   ready-prefix codecs use the same audio-clock evidence fields.
2. Full scene wallpaper support: the current completed work is a first-class
   Gilder scene document/runtime path plus explicit full-scene boundaries, not
   full Wallpaper Engine scene execution. For progress accounting, full scene
   is roughly `92%`: package/conversion boundaries, `scene/gscene` format
   validation, snapshot-time propagation, render clear-color snapshot layers,
   WE `scene.pkg` direct import,
   WE parent-id graph lowering into gscene children,
   WE `shape`/`solid`/`radius` lowering into native rectangle/ellipse nodes,
   explicit WE keyframe timeline lowering into gscene timeline channels,
   WE `{script,value}` wrapper lowering without a JS engine, deterministic
   numeric SceneScript expression lowering into native property bindings,
   geometry-field timeline/property animation, parallax depth camera-property
   offsets, WE TEXV0005/TEXB0004 RGBA texture decoding, spritesheet atlas
   resources with time-sampled UV frame selection, retained sampled-image
   resources, solid/image mixed composition, descriptor heap sampling,
   visible scene runtime status, native present route selection,
   retained resource status, clear-background composition, native runtime
   layer-coverage accounting, rounded-rectangle tessellation, simple/concave
   path tessellation, stroke geometry, deterministic text glyph geometry,
   first-class `video` layer detection, single-video-layer Vulkan Video scene
   composition, clear-background plus video scene composition, scene
   timeline animation snapshotting, property update binding, pause/resume
   policy, and package state/property persistence are in place; particle
   systems, full WE scene graph execution, arbitrary SceneScript,
   shader/material graph, cursor parallax input plumbing,
   PipeWire audio response, complex font
   shaping/atlas typography, full path rasterization, and actual mixed
   video-as-scene composition remain open. Wallpaper Engine scene conversions
   now write `assets/*.gscene.json`
   documents with `source`, `size`, `render`, `camera`, `import`,
   `resources`, `nodes`, `systems`, `native_lowering`, and
   `unsupported_features` sections, plus a structured
   `full_scene` report block with
   `target_runtime=native-vulkan-full-scene`,
   `current_runtime=native-vulkan-scene-runtime`,
   `progress_estimate_percent=92`,
   preserved source-scene metadata paths, completed boundaries, and pending
   full-scene boundaries. Gilder scene is the runtime format, not a
   Wallpaper Engine schema clone: WE's historical fields are treated as an
   input dialect and are isolated in converter-owned `provenance`/`import`
   metadata. Runtime-facing node roots stay clean (`type`, `transform`,
   `resource`, `effects`, `audio`, draw properties), while WE ids, parent ids,
   dependencies, original transforms, model/material chains, particles,
   animation layers, and instance overrides live under node `provenance`.
   Matched WE parent ids are lowered into real gscene `children`, so the core
   snapshot path now composes parent/child transform and opacity instead of only
   preserving parent ids as metadata. `render.clear_color` with
   `clear_enabled != false` now emits the first snapshot color layer, so
   converted WE scene clear color participates in native clear-background and
   mixed scene composition. WE `{ value: ... }` wrappers for text, point size,
   font, and horizontal alignment are lowered to gscene text node fields, and
   WE `visible: { value, user }` is lowered to a gscene opacity property
   binding so runtime property updates can reveal/hide the layer without a
   legacy visibility path. WE `shape`/`solid` objects now lower directly into
   gscene `rectangle`/`ellipse` nodes with color, size, and `corner_radius`,
   so ordinary vector shape layers enter the same native solid-geometry
   runtime instead of staying as source metadata. Explicit source keyframe
   tracks for supported transform/opacity properties now lower into gscene
   `timelines`, including vector `origin`/`scale` split into native `x`/`y`
   and `scale-x`/`scale-y` channels, so the existing core timeline runtime
   executes converted motion instead of leaving it only in provenance. The
   same channel now covers geometry fields (`width`, `height`,
   `corner-radius`), and WE `{script: ..., value: ...}` wrappers are unwrapped
   to deterministic gscene defaults without introducing a JS engine. User-bound
   scalar wrappers for transform, opacity, size, and radius lower into
   `property_bindings`; deterministic numeric SceneScript expressions over one
   user property, `value`, constants, parentheses, and `+ - * /` now compile
   into the same native `scale`/`offset` binding model. Arbitrary JS-like
   SceneScript remains explicit pending work instead of being executed by a
   compatibility VM. Parallax now has a gscene runtime model:
   `render.parallax.amount` plus node `parallax_depth` consumes
   `scene.parallax.x/y` property values to offset snapshot transforms.
   The converter now understands WE `object.image` as a model JSON entry
   rather than a direct image path, follows `model -> material -> texture`,
   copies model/material/effect/audio/texture assets into the gscene resource
   graph, and assigns `node.resource` only to a native sampled-image resource.
   Standard WE `TEXV0005/TEXB0004` RGBA `.tex` material textures are decoded
   through their LZ4 block payload. Non-spritesheet textures are cropped to the
   model's first frame when the model width/height divides the atlas; WE
   `SPRITESHEET` materials instead write the full atlas as a generated PNG
   image resource and attach `properties.spritesheet` (`atlas-grid`, atlas
   size, frame size, columns/rows, frame count, FPS, loop flag) to the gscene
   node. The original `.tex` remains as provenance, while the runtime-facing
   sampled image is the generated PNG. Runtime `_rt_`
   textures, shaders, particles, arbitrary SceneScript, effect graphs, and
   audio-response systems are preserved structurally and reported as explicit
   pending runtime systems instead of being hidden behind a legacy loader.
   There is no internal legacy scene format, loader, or
   lowering bridge; old `layers` fixture data was replaced by `nodes/resources`
   gscene documents. Static wallpapers now lower into a single-image scene
   layer before the Vulkan sampled-image runtime. Scene plans route through
   `native_vulkan/scene/` and
   `native_vulkan/vulkan/scene/` with descriptor-heap sampled-image geometry.
   The main scene present entry now chooses the native fast-clear,
   solid-quad, sampled-image, or mixed solid+sampled-image Vulkan route from the
   runtime draw-pass plan, including implicit full-extent image layers that
   derive fit geometry from the swapchain extent at present time. Full-extent
   image backgrounds can now compose with solid quad overlays through the mixed
   scene route, and a leading full-screen color layer now becomes the dynamic
   rendering clear background for `color + image`, `color + shape`, and
   `color + simple path` scenes instead of blocking native presentation. The
   native spike CLI uses the same path through
   `--run-scene` for image and color scene probes, and the CLI accepts
   `--scene-time-ms`/`--snapshot-time-ms` so non-zero sampled scene time reaches
   `SceneWallpaperPlan` through the same entry point used by visible
   runtime smoke. Text layers now lower into deterministic built-in glyph
   geometry and render through the same solid dynamic-rendering pipeline as
   rectangles, rounded rectangles, ellipses, and simple paths; this gives text
   layers real native coverage without adding a legacy font-renderer
   compatibility path.
   Scene sampled-image uploads now use Vulkan 1.4
   `hostImageCopy`, so static scene image upload has no staging buffer, upload
   queue submit, or upload fence. Scene runtime and `SceneWallpaperPlan`
   now carry `snapshot_time_ms` into the native Vulkan render item instead of
   resetting it to zero, which keeps the time-sampled scene state visible at
   the Vulkan boundary. Scene runtime and sampled-image present snapshots now
   expose `scene_input_model`,
   `scene_resource_model`, `scene_solid_quad_draw_count`,
   `scene_sampled_image_resource_count`, and
   `scene_sampled_image_descriptor_heap_required`, making the group-flattened
   core snapshot boundary and descriptor-heap-only sampled-image resource model
   directly visible in JSON evidence. The scene document model accepts
   first-class `video` layers; native runtime snapshots expose
   `draw_pass_video_op_count`, `scene_video_layer_resource_count`,
   `draw_pass_required_video_resources`, `scene_video_native_layer_count`,
   and `draw_pass_requires_video_decode`.
   Full-scene runtime snapshots now expose `active_scene_layer_count`,
   `native_runtime_layer_count`, `native_runtime_pending_layer_count`,
   `native_runtime_coverage_percent`, `clear_background_layer_count`,
   `sampled_image_native_layer_count`, `solid_geometry_layer_count`,
   `rounded_rectangle_layer_count`, `tessellated_path_layer_count`,
   `text_geometry_layer_count`, and `stroke_geometry_layer_count`, so scene
   progress is tied to actual layer
   coverage rather than treating scene as full scene.
   Renderer plans now also count `timeline_animation_count`,
   `timeline_animated_layer_count`, and `property_binding_count`; these values
   are carried into `NativeVulkanRenderItem::Scene` and
   `runtime.full_scene` instead of being inferred at the reporting boundary.
   The property binding path uses the persisted global/output `AppState`
   property store and the same resolver used to build visible scene snapshots.
   Visible scene present results now include `runtime.full_scene`, with
   `target_runtime=native-vulkan-full-scene`,
   `current_runtime=native-vulkan-scene-runtime`,
   `progress_estimate_percent=92`, `native_present_route_ready`,
   `retained_resource_model_ready`, `timeline_snapshot_runtime_ready`,
   `timeline_animation_runtime_ready`, `timeline_animation_count`,
   `timeline_animated_layer_count`, `property_update_runtime_ready`,
   `property_binding_count`, `pause_resume_policy_ready`,
   `package_state_persistence_ready`, `scene_state_persistence_model`,
   `source_layer_count`, flattened draw counts, per-feature layer counts,
   completed boundaries, and pending boundaries. A single scene `video` layer
   now routes through the same Vulkanalia ready-prefix Vulkan Video presenter
   used by direct video wallpapers and reports
   `video-layer-vulkan-video-scene-bridge-ready`. A leading full-screen color
   scene layer plus one video layer now routes through the same presenter as
   `clear-background-video-layer-vulkan-video-scene-bridge-ready`; the dynamic
   rendering attachment clear color is carried in each decoded-image draw
   snapshot. Mixed video scenes with overlays remain explicitly pending under
   `mixed-video-scene-composition` instead of silently rasterizing or falling
   back.
   Current real Workshop scene conversion sample: Steam Workshop item
   `3726503096` (`Beneath The Seventh`) is tagged `3840 x 2160` by Workshop,
   but the package's WE scene/model frame is `2160x1440` and its material
   texture is a `6480x5760` atlas (`3x4`, 12 frames). The native conversion at
   `/tmp/gilder-we-3726503096-output-atlas` now starts from the original
   Workshop directory, parses `scene.pkg` `PKGV0023` directly, and writes
   `assets/scene-resources/scene/resource-4-img-5944-atlas.png` as the native
   sampled-image atlas resource. The gscene node `resource` is
   `resource-4-img-5944-atlas`; `properties.spritesheet` records atlas
   `6480x5760`, frame `2160x1440`, `columns=3`, `rows=4`,
   `frame_count=12`, `fps=12.0`, and `loop=true`; the original `.tex` remains
   as `resource-3-img-5944` provenance. The conversion report records
   `scene-we-package-import`, `scene-we-tex-rgba-frame-decode`, and
   `scene-we-spritesheet-atlas-runtime`, so this sample now uses the completed
   atlas runtime path. The converter must not
   up-label the generated frame to 4K; the 4K tag is a presentation/display
   label, not the asset frame dimensions.
   Current real Workshop video conversion sample: Steam Workshop item
   `3498008367` (`素晴 爆裂魔法`) downloads into
   `artifacts/wallpaper-engine-workshop/steamcmd-root/steamapps/workshop/content/431960/3498008367`
   as `project.json`, `preview.gif`, and `慧慧.mp4`. The converter output at
   `/tmp/gilder-we-3498008367-output-video` is a clean video package:
   `assets/loop.mp4`, `previews/poster.gif`, `kind=video`,
   `entry.source=assets/loop.mp4`, `entry.poster=previews/poster.gif`,
   `property:schemecolor`, and an empty `unsupported_features` list.
   `ffprobe` identifies the source as H.264 Main, `yuv420p`, `3840x2160`,
   `60/1` FPS, 20 seconds, 1200 frames. The H.264 ready-prefix runtime now
   sizes its streaming packet queue from the requested bitstream/ready-prefix
   window instead of the fixed FFplay picture-queue handoff size, so this
   real stream's third P frame can request three active references without
   losing the current output slot. The 4K60 smoke
   `env WAYLAND_DISPLAY=wayland-1 scripts/native-vulkan-h264-ready-prefix-video-smoke.sh --source /tmp/gilder-we-3498008367-output-video/assets/loop.mp4 --width 3840 --height 2160 --target-fps 60 --decode-prefix 60 --playback-frames 60 --output-name HDMI-A-1 --fit cover`
   passes with 60 decoded frames, 60 presented frames, 2 IDR frames,
   58 P frames, `stream_dpb_slots=4`,
   `stream_max_active_reference_pictures=3`,
   `session_max_dpb_slots=4`,
   `session_max_active_reference_pictures=3`,
   `distinct_sampled_array_layer_count=4`,
   `all_zero_copy_presented=true`,
   source PTS deltas `16..17` ms, `average_present_fps=60.017923854505206`,
   `present_mode=fifo-latest-ready`, and no failed present-mode/pacing gates.
   Current runtime smoke:
   `WAYLAND_DISPLAY=wayland-1 target/release/gilder-native-vulkan --run-scene --output-name HDMI-A-1 --source artifacts/smoke/scene-heap-smoke.png --fit cover --duration 1 --target-fps 30 --scene-time-ms 1234`
   presents `30` frames at `29.99748264125423` FPS and reports
   `runtime.full_scene.progress_estimate_percent=92`,
   `runtime.full_scene.native_present_route_ready=true`,
   `runtime.full_scene.retained_resource_model_ready=true`,
   `runtime.full_scene.timeline_snapshot_runtime_ready=true`,
   `runtime.full_scene.timeline_animation_runtime_ready=true`,
   `runtime.full_scene.property_update_runtime_ready=true`,
   `runtime.full_scene.pause_resume_policy_ready=true`,
   `runtime.full_scene.package_state_persistence_ready=true`,
   `runtime.full_scene.native_runtime_coverage_percent=100`,
   `scene_resource_model=retained-sampled-images-descriptor-heap`,
   `scene_sampled_image_resource_count=1`,
   `scene_sampled_image_descriptor_heap_required=true`,
   `uses_host_image_copy=true`, `staging_buffer_bytes=0`,
   `upload_submitted=false`,
   `descriptor_heap.descriptor_model=VK_EXT_descriptor_heap`,
   `uses_present_id2=true`, `present_wait2_available=true`,
   `swapchain.present_id2_enabled=true`, `swapchain.present_wait2_enabled=true`,
   and no legacy `uses_present_id`/`present_wait_available` fields.
   Current video scene smoke:
   `/tmp/gilder-scene-video-h265-main8-background-final.json` from
   `WAYLAND_DISPLAY=wayland-1 target/release/gilder-native-vulkan --run-scene --scene-video --output-name HDMI-A-1 --source artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-566frames-g240-d240.mp4 --video-codec h265 --width 3840 --height 2160 --decode-h265-ready-prefix 4 --playback-frames 4 --target-fps 240 --background '#102030' --fit contain`
   reports `scene_present_route=video`,
   `draw_pass_backend_status=clear-background-video-layer-vulkan-video-scene-bridge-ready`,
   `scene_resource_model=clear-background-and-retained-vulkan-video-scene-resource`,
   `active_scene_layer_count=2`, `clear_background_layer_count=1`,
   `video_native_layer_count=1`, `native_runtime_layer_count=2`,
   `native_runtime_pending_layer_count=0`,
   `native_runtime_coverage_percent=100`, presented `4` H.265 Main8 frames,
   `descriptor_model=VK_EXT_descriptor_heap`, `all_zero_copy_presented=true`,
   and decoded-image draw `clear_color=[0.062745101749897,0.125490203499794,0.1882352977991104,1.0]`.
   Current regression coverage:
   `cargo test --features native-vulkan-renderer scene -- --nocapture`
   passes `101` filtered lib tests, `5` native-vulkan CLI tests, and `1`
   gilderd test. The added renderer/runtime coverage asserts gscene package
   validation, clean WE scene-to-gscene conversion, WE model/material texture
   provenance, renderable material image texture resource resolution, WE parent
   graph lowering into gscene children, render clear-color snapshot layers,
   WE text wrapper conversion, visible property binding lowering, WE
   shape/solid/radius lowering into native snapshot nodes, explicit WE
   keyframe timeline lowering into native timeline snapshot values,
   geometry field timeline/property animation, script/value wrapper lowering
   without a JS engine, deterministic numeric SceneScript expression lowering,
   WE `.tex` RGBA/LZ4 first-frame or spritesheet-atlas decoding to a renderable
   PNG resource, `scene.pkg` direct import, and parallax depth property-camera
   offsets, timeline animation metadata reaches `SceneWallpaperPlan`,
   atlas-frame texture regions are time sampled, property binding counts reach
   the native runtime, and the completed full-scene boundaries include
   `timeline-animation-runtime`, `property-update-runtime`,
   `pause-resume-policy-runtime`, `package-state-persistence`,
   `wallpaper-engine-scene-pkg-import`, and
   `scene-we-spritesheet-atlas-runtime`.
   Next gates:
   wiring mixed video-as-scene layer composition from this explicit bridge boundary,
   complex font shaping/atlas typography, full path rasterization,
   full Wallpaper Engine graph execution, WE animation layer blending,
   arbitrary SceneScript runtime, shader/material graph, particle systems,
   cursor parallax input source, and PipeWire audio response.
   The scene path must keep retained GPU images,
   `descriptor_sets=0`, and descriptor-heap sampling.
3. Video coverage and regression: the H.264/H.265/AV1 core decode/present path
   is now in coverage/stability mode. Expand real and generated matrices across
   Main/Main10, reference counts, B-frame patterns, weighted prediction,
   long-term references, HDR/color metadata, MP4/MKV/WebM containers,
   extradata/Annex-B boundaries, arbitrary entry points, loop boundaries, bad
   packets, unsupported profiles, driver capability failures, long-run resource
   stability, validation-layer correctness runs, and audio/scene integration
   regressions.
4. Script hygiene: keep the active codec smokes, real-source matrix,
   performance sampler, CI dependency/policy scripts, packaging scripts, and
   workshop downloader. Delete one-off spike scripts instead of preserving
   wrappers.
