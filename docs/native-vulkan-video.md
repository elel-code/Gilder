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
    Annex-B scratch pool retains only one buffer capped at 224 KiB. This keeps
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
  `artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-2402frames-g240-d2400.mp4`.
- Main10 source:
  `artifacts/video-sources/h265/h265-main-10-b0-ref1-3840x2160-240fps-566frames-g240-d240.mp4`.
- Breakthroughs: HEVC reference sets follow FFmpeg's `vp->ref_slots[idx]`
  filling, slice offsets are stack/borrowed instead of heap-retained, Main10
  uses the 10-bit two-plane Vulkan format directly, and both profiles share the
  descriptor-heap shader conversion path. H.265 uses the same single-packet
  FFmpeg handoff and one retained Annex-B scratch buffer as H.264.
- Main8 evidence directory: `/tmp/gilder-h265-main8-4k240-pool1-2400`.
- Main8 result: decoded/presented `2400/2400`,
  `average_present_fps=240.00573524668013`,
  `performance_max_private_dirty_kib=24064`, `performance_avg_cpu_percent=15.72`,
  `performance_max_pss_kib=66545`, `performance_max_uss_kib=38956`,
  `performance_avg_gpu_busy_percent=31`, `performance_max_gpu_busy_percent=34`,
  `performance_max_nvidia_process_gpu_memory_mib=126`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Main10 evidence directory: `/tmp/gilder-h265-main10-pool1-rerun`.
- Main10 result: decoded/presented `2400/2400`,
  `average_present_fps=240.00463578108003`,
  `performance_max_private_dirty_kib=24048`, `performance_avg_cpu_percent=15.88`,
  `performance_max_pss_kib=66580`, `performance_max_uss_kib=39000`,
  `performance_avg_gpu_busy_percent=32`, `performance_max_gpu_busy_percent=38`,
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
- Main8 evidence directory: `/tmp/gilder-av1-main8-pool1`.
- Main8 result: displayed/presented `2400/2400`,
  `average_present_fps=240.03187096557068`,
  `performance_max_private_dirty_kib=24004`, `performance_avg_cpu_percent=14.22`,
  `performance_max_pss_kib=66591`, `performance_max_uss_kib=39012`,
  `performance_avg_gpu_busy_percent=30`, `performance_max_gpu_busy_percent=33`,
  `performance_max_nvidia_process_gpu_memory_mib=179`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Main10 evidence directory: `/tmp/gilder-av1-main10-pool1`.
- Main10 result: displayed/presented `2400/2400`,
  `average_present_fps=240.04099556802595`,
  `performance_max_private_dirty_kib=23836`, `performance_avg_cpu_percent=14.68`,
  `performance_max_pss_kib=66580`, `performance_max_uss_kib=38912`,
  `performance_avg_gpu_busy_percent=33`, `performance_max_gpu_busy_percent=44`,
  `performance_max_nvidia_process_gpu_memory_mib=286`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
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
  present, `scene/` scene-lite draw/present, and `video/` Vulkan Video session,
  command, decode submit, and present runtime code.
- `src/renderer/native_vulkan/present/`: renderer item planning plus clear and
  static-image present entry points.
- `src/renderer/native_vulkan/scene/`: scene-lite planning/runtime bridge into
  the Vulkan present path.
- `src/renderer/native_vulkan/audio/`: audio policy and clock boundary.
  Clock-only audio now probes FFmpeg audio packets as timestamp/serial metadata
  and immediately releases packet payloads. Ready-prefix runs perform the probe
  before video present when requested, keep `clock_ns` as the probe diagnostic,
  and pass the first ready `video_master_start_clock_ns` sample into muted
  audio-master pacing so pre-read AAC packets do not advance the video start
  clock. Audible output is still deliberately out of scope until daemon
  mute/pause/device lifecycle is wired.

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
  fallback.
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
  for inline codec parameters, and present keeps surface/swapchain maintenance1.
  Those are not compatibility fallbacks and are not replaced by
  `VK_KHR_maintenance10`. Roadmap 2026 still requires
  `VK_KHR_maintenance7`, `VK_KHR_maintenance8`, and `VK_KHR_maintenance9`; keep
  probing them. `VK_KHR_maintenance10` is tracked as an additional modern
  ratified extension, not as a replacement for 7/8/9.
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
  one of those paths exists. This is probe evidence only: an extension is not
  considered runtime-used until the relevant scene/video/present path consumes it
  and exposes path-specific JSON fields.
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
  current vulkanalia bindings. Track both its `maintenance10` feature and its
  sRGB resolve/RGBA4 opaque-black properties before promoting it into scene or
  video resolve paths:
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
6. Promote the new Roadmap 2026 probe bits into runtime paths where they remove
   actual work: `VK_KHR_pipeline_binary` for retained scene/video pipelines,
   `VK_KHR_robustness2`/null descriptors for descriptor-heap resource tables,
   maintenance7/8/9 where they simplify image/memory limits, `VK_KHR_maintenance10`
   where it simplifies image/resolve behavior, and
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

1. Audio integration: the first clock-only boundary exists under
   `native_vulkan/audio/clock.rs`. It selects an FFmpeg audio stream, consumes
   packet PTS/duration/serial metadata, immediately unreferences payloads, and
   reports a muted audio-clock snapshot from ready-prefix runs when
   `--audio-clock-probe` is requested. The probe now runs before decoded-image
   present and, when it finds an audio stream with clock samples, passes the
   first ready `video_master_start_clock_ns` into the decoded-image present
   timer as the muted audio-master start sample while leaving the final
   `clock_ns` as probe diagnostics. Audio snapshots now explicitly report
   `video_master_clock_ready`, `video_master_start_clock_ns`,
   `video_master_start_serial`, `video_master_start_packet_index`,
   `current_serial_start_clock_ns`, `current_serial_start_serial`, and
   `current_serial_start_packet_index`. Ready-prefix video pacing consumes the
   stable `video_master_start_*` start sample, while loop/seek evidence uses
   the `current_serial_start_*` fields so pre-present probing cannot move the
   first-frame master clock to a later loop. When playback requests more frames
   than the ready-prefix window, the audio probe enables FFmpeg EOS seek and
   expands the bounded metadata-only packet probe to expose current-serial
   reset evidence without retaining packet payloads. The H.264/H.265
   ready-prefix smokes now fail generated-source audio-clock loop probes when
   `loop_count`, `current_serial`, or `current_serial_start_serial` never
   leaves serial `0`. Next gates: replace the probe-backed start sample with a
   live audio callback/runtime clock, prove arbitrary-entry sync against real
   audio+video sources, then add audible output with daemon
   pause/mute/device lifecycle.
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
   same loop-probe gate when playback exceeds the ready-prefix window, so all
   direct ready-prefix codecs use one audio-clock evidence contract.
2. Full scene wallpaper support: the current completed work is still a native
   scene-lite subset plus explicit full-scene bridge boundaries, not full
   Wallpaper Engine scene execution. For progress accounting, full scene is
   roughly `20-25%`: package/conversion boundaries, snapshot-time propagation,
   retained sampled-image resources, solid/image mixed composition, descriptor
   heap sampling, and first-class `video` layer detection are in place; particle
   systems, full timeline animation, SceneScript, shader/material graph,
   parallax, audio response, text/path GPU rasterization, and actual
   video-as-scene composition remain open. The scene-lite subpath is much
   further along, but it is not the full-scene metric. Wallpaper Engine scene
   conversions now write a structured `full_scene` report block with
   `target_runtime=native-vulkan-full-scene`,
   `current_runtime=scene-lite-subset`, `progress_estimate_percent=22`,
   preserved source-scene metadata paths, completed boundaries, and pending
   full-scene boundaries. Static wallpapers now lower into a single-image scene
   layer before the Vulkan sampled-image runtime. Scene-lite plans already
   route through `native_vulkan/scene/` and
   `native_vulkan/vulkan/scene/` with descriptor-heap sampled-image geometry.
   The main scene-lite present entry now chooses the native fast-clear,
   solid-quad, sampled-image, or mixed solid+sampled-image Vulkan route from the
   runtime draw-pass plan, including implicit full-extent image layers that
   derive fit geometry from the swapchain extent at present time. Full-extent
   image backgrounds can now compose with solid quad overlays through the mixed
   scene route. The native spike CLI uses the same path through
   `--run-scene-lite` for image and color scene probes, and the CLI accepts
   `--scene-time-ms`/`--snapshot-time-ms` so non-zero sampled scene time reaches
   `SceneLiteWallpaperPlan` through the same entry point used by visible
   runtime smoke. Scene sampled-image uploads now use Vulkan 1.4
   `hostImageCopy`, so static scene image upload has no staging buffer, upload
   queue submit, or upload fence. Scene runtime and `SceneLiteWallpaperPlan`
   now carry `snapshot_time_ms` into the native Vulkan render item instead of
   resetting it to zero, which keeps the time-sampled scene state visible at
   the Vulkan boundary. Scene runtime and sampled-image present snapshots now
   expose `scene_input_model`,
   `scene_resource_model`, `scene_solid_quad_draw_count`,
   `scene_sampled_image_resource_count`, and
   `scene_sampled_image_descriptor_heap_required`, making the group-flattened
   core snapshot boundary and descriptor-heap-only sampled-image resource model
   directly visible in JSON evidence. Scene-lite now also accepts first-class
   `video` layers in the core document model; native runtime snapshots expose
   `draw_pass_video_op_count`, `scene_video_layer_resource_count`,
   `draw_pass_required_video_resources`, and `draw_pass_requires_video_decode`.
   The current native backend intentionally reports
   `video-layer-vulkan-video-scene-bridge-pending` with blocking reason
   `video-layer-needs-vulkan-video-scene-bridge`, which makes the next
   Vulkan-Video-as-scene composition boundary explicit instead of silently
   rasterizing or falling back.
   Current runtime smoke:
   `WAYLAND_DISPLAY=wayland-1 target/release/gilder-native-vulkan --run-scene-lite --output-name HDMI-A-1 --source artifacts/smoke/scene-lite-heap-smoke.png --fit cover --duration 1 --target-fps 30 --scene-time-ms 1234`
   presents `30` frames at `29.997164188086316` FPS and reports
   `scene_resource_model=retained-sampled-images-descriptor-heap`,
   `scene_sampled_image_resource_count=1`,
   `scene_sampled_image_descriptor_heap_required=true`,
   `uses_host_image_copy=true`, `staging_buffer_bytes=0`,
   `upload_submitted=false`, `descriptor_model=VK_EXT_descriptor_heap`,
   `uses_present_id2=true`, and `present_wait2_available=true`.
   Current regression coverage:
   `cargo test --features native-vulkan-renderer scene_lite -- --nocapture`
   passes `42` scene-lite-related tests across lib/bin/gilderd entry points.
   Next gates:
   wiring video-as-scene layer composition from this explicit bridge boundary,
   text/path rasterization,
   property updates beyond snapshot time zero, pause/resume policy, and package
   state persistence. The scene path must keep retained GPU images,
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
