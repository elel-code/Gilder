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
not codec heap pressure.

Real-source and arbitrary-entry runs use the same reporting rule. If the run is
intended to prove performance, keep playback long enough for the sampler window
and pass `--performance-snapshot --performance-duration <sec>
--performance-interval <sec>`. The result summary must retain CPU, GPU memory,
RSS/PSS/USS, `Private_Dirty`, FPS, frame counts, descriptor heap, and zero-copy
fields together with the report directory.

## Next Plan

1. Audio integration: follow FFmpeg's demux, packet queue, clock serial, loop,
   and frame-timer semantics. The first accepted path is muted clock-only audio
   for synchronization; audible output comes after clock behavior is stable.
2. Full scene wallpaper support: route native Vulkan video through the normal
   daemon wallpaper lifecycle, including output selection, scene transforms,
   static-image/video composition, properties, pause/resume policy, and package
   state persistence.
3. Bitstream coverage: expand H.264, H.265, and AV1 matrices across real
   sources and generated sources, including Main/Main10, reference counts,
   B-frame patterns, arbitrary entry points, loop boundaries, long-run resource
   stability, and validation-layer correctness runs.
4. Script hygiene: keep the active codec smokes, real-source matrix,
   performance sampler, CI dependency/policy scripts, packaging scripts, and
   workshop downloader. Delete migration/spike/compatibility scripts instead of
   preserving wrappers.
