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
  configures the process allocator before FFmpeg streaming starts so this freed
  storage returns to low-dirty behavior without requiring distribution-time env
  tuning.

## Substantial Breakthroughs

1. The practical memory breakthrough was shader-owned plane conversion.
   Removing the `VkSamplerYcbcrConversion`/embedded-sampler route and sampling
   Y/UV plane views explicitly through descriptor heaps dropped host
   `Private_Dirty` below the 25,000 KiB gate while keeping zero-copy GPU present.
2. The bitstream path was aligned to FFmpeg's picture-owned `slices_buf` model:
   two pooled 2 MiB slots, exec-owned lifetime after submit, no global growing
   AU buffer, and no retained payload window.
3. Submit/reference construction stopped allocating per-frame reference Vecs.
   H.264, H.265, and AV1 now lower into fixed/borrowed workspaces matching
   FFmpeg's fixed `refs[36]`/`ref_slots[36]` contract.
4. The packet queue stores AU metadata, PTS/timeline data, and serial state;
   payload is uploaded and released instead of being retained through present.
5. Presentation follows the FFmpeg queue shape: bounded queue depth three,
   `keep_last` semantics, serial reset handling, and frame-timer PTS-delta
   pacing.
6. Smoke runs default to the distribution allocator environment
   (`--allocator-profile system`). The previous glibc low-dirty allocator
   profile is kept only for explicit comparison with
   `--allocator-profile glibc-low-dirty`; performance gates should be judged
   with the system profile unless a run is specifically documenting allocator
   sensitivity. Native streaming also configures glibc malloc in-process before
   FFmpeg opens the source, so shipped binaries do not require external malloc
   environment variables.
7. Decode/present timeline synchronization follows FFmpeg's per-frame semaphore
   dependency shape: decode signals at `VIDEO_DECODE_KHR` completion and
   present waits on that per-frame value before touching the decoded image. Low
   GPU busy with stable 240 fps should be treated as CPU/submit/synchronization
   headroom, not as a reason to add copy paths or descriptor sets.
8. FFmpeg read-thread handoff is codec-limited rather than a hidden second
   queue. H.264 uses rendezvous handoff to keep heap dirty below 25,000 KiB; H.265
   uses two handoff slots to recover Main10 throughput; AV1 keeps the default
   single handoff slot because both Main8/Main10 already pass the memory/FPS
   gates.

## Format Evidence

New performance evidence uses `allocator_profile=system` by default. Older
report directories named `workspace-allocator` or captured before this rule are
kept as architecture/throughput evidence, but allocator-sensitive memory gates
must be judged with the system profile.

### H.264

- Source:
  `artifacts/video-sources/h264/h264-high-b0-ref2-weightp0-weightb0-3840x2160-240fps-2640frames-g2401-d2400.mp4`.
- Breakthroughs: descriptor-heap Y/UV plane shader conversion, borrowed slice
  offsets from the first slice path, fixed reference workspace, two-slot
  FFmpeg-style slices buffer pool, H.264 rendezvous FFmpeg packet handoff, and
  bounded streaming packet upload.
- Evidence directory: `/tmp/gilder-final-h264-4k240-25000`.
- Result: decoded/presented `2400/2400`, `average_present_fps=240.01407249629443`,
  `performance_max_private_dirty_kib=24580`, `performance_avg_cpu_percent=16.27`,
  `performance_max_pss_kib=66646`, `performance_max_uss_kib=41824`,
  `performance_avg_gpu_busy_percent=33`, `performance_max_gpu_busy_percent=47`,
  `performance_max_nvidia_process_gpu_memory_mib=281`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.

### H.265

- Main8 source:
  `artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-242frames-g240-d240.mp4`.
- Main10 source:
  `artifacts/video-sources/h265/h265-main-10-b0-ref1-3840x2160-240fps-566frames-g240-d240.mp4`.
- Breakthroughs: HEVC reference sets follow FFmpeg's `vp->ref_slots[idx]`
  filling, slice offsets are stack/borrowed instead of heap-retained, Main10
  uses the 10-bit two-plane Vulkan format directly, and both profiles share the
  descriptor-heap shader conversion path. H.265 keeps two FFmpeg read-thread
  handoff slots because Main10 needs the demux/BSF overlap that
  `references/ffmpeg/fftools/ffplay.c:3132-3141` provides.
- Main8 evidence directory: `/tmp/gilder-final-h265-main8-4k240-25000`.
- Main8 result: decoded/presented `2400/2400`,
  `average_present_fps=240.0048899610724`,
  `performance_max_private_dirty_kib=23820`, `performance_avg_cpu_percent=21.08`,
  `performance_max_pss_kib=66128`, `performance_max_uss_kib=41928`,
  `performance_avg_gpu_busy_percent=34`, `performance_max_gpu_busy_percent=45`,
  `performance_max_nvidia_process_gpu_memory_mib=281`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Main10 evidence directory: `/tmp/gilder-final-h265-main10-4k240-25000`.
- Main10 result: decoded/presented `2400/2400`,
  `average_present_fps=240.0069369147951`,
  `performance_max_private_dirty_kib=24016`, `performance_avg_cpu_percent=20.27`,
  `performance_max_pss_kib=66215`, `performance_max_uss_kib=42104`,
  `performance_avg_gpu_busy_percent=30`, `performance_max_gpu_busy_percent=31`,
  `performance_max_nvidia_process_gpu_memory_mib=484`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`.

### AV1

- Main8 source:
  `artifacts/video-sources/av1/av1-main8-3840x2160-240fps-566frames-g240.webm`.
- Main10 source:
  `artifacts/video-sources/av1/av1-main10-3840x2160-240fps-566frames-g240.webm`.
- Breakthroughs: AV1 reference lowering follows FFmpeg's duplicate-slot scan,
  unique `referenceNameSlotIndices`, caller-owned workspaces, and the same
  two-slot slices buffer pool; this removed retained-copy pressure while keeping
  continuous 4K240 present.
- Main8 evidence directory: `/tmp/gilder-final-av1-main8-4k240-25000`.
- Main8 result: displayed/presented `2400/2400`,
  `average_present_fps=240.00077061156145`,
  `performance_max_private_dirty_kib=21812`, `performance_avg_cpu_percent=14.60`,
  `performance_max_pss_kib=64267`, `performance_max_uss_kib=40160`,
  `performance_avg_gpu_busy_percent=27`, `performance_max_gpu_busy_percent=29`,
  `performance_max_nvidia_process_gpu_memory_mib=262`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G8_B8R8_2PLANE_420_UNORM`.
- Main10 evidence directory: `/tmp/gilder-final-av1-main10-4k240-25000`.
- Main10 result: displayed/presented `2400/2400`,
  `average_present_fps=240.0206298081496`,
  `performance_max_private_dirty_kib=21768`, `performance_avg_cpu_percent=15.35`,
  `performance_max_pss_kib=64125`, `performance_max_uss_kib=40024`,
  `performance_avg_gpu_busy_percent=33`, `performance_max_gpu_busy_percent=39`,
  `performance_max_nvidia_process_gpu_memory_mib=453`, `descriptor_sets=0`,
  `descriptor_heap_only=true`, `all_zero_copy_presented=true`,
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`.

## Allocator Profile Evidence

`system` is the distribution/default profile: scripts clear
`MALLOC_ARENA_MAX`, `MALLOC_MMAP_THRESHOLD_`, `MALLOC_TRIM_THRESHOLD_`, and
`GLIBC_TUNABLES` before launching the video process. `glibc-low-dirty` is only
for explicit allocator sensitivity comparisons.

Current 4K240 system-profile comparison is the evidence in the format sections
above. All listed runs are under `performance_max_private_dirty_kib < 25000`,
`average_present_fps >= 239.999`, `descriptor_sets=0`,
`descriptor_heap_only=true`, and `all_zero_copy_presented=true`.

## Smoke Commands

Use the codec-specific ready-prefix smoke scripts with the repository 4K240
sources, `--playback-frames 2400`, and `--performance-snapshot`. The smoke
scripts default performance snapshots to `--max-private-dirty-kib 25000`; an
explicit `--max-private-dirty-kib` only overrides that hard gate. Validation
layer runs are for correctness only; do not use them for the memory/FPS gate.

The default allocator profile is `system`, which clears
`MALLOC_ARENA_MAX`, `MALLOC_MMAP_THRESHOLD_`, `MALLOC_TRIM_THRESHOLD_`, and
`GLIBC_TUNABLES` before launching the video process. Native video configures the
process allocator internally before FFmpeg opens the source, so this is still
the no-external-tuning distribution path. Use `--allocator-profile
glibc-low-dirty` only for allocator sensitivity comparisons.

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
