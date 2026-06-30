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
- All work must be designed for the long-term native runtime. Do not add
  short-term substitutes, sample-specific fixes, hidden compatibility branches,
  magic offsets, layer hiding, resource re-export hacks, preview fallbacks, or
  temporary render paths to make one scene appear correct. If a gap is caused
  by unsupported format, effect, material, mask, blend, interaction,
  renderer-quality, or runtime behavior, implement or design that first-class
  subsystem and document any remaining boundary explicitly.

## FFmpeg References

- `references/ffmpeg/fftools/ffplay.c:114-123`: `PacketQueue` carries packet
  count, duration, and serial state.
- `references/ffmpeg/fftools/ffplay.c:125-128`: video queue size is three.
- `references/ffmpeg/fftools/ffplay.c:3132-3141`: the read thread blocks only
  when queues have enough packets; native keeps this asynchronous shape but caps
  the handoff by codec to stay under the 25,000 KiB `Private_Dirty` gate.
- `references/ffmpeg/fftools/ffplay.c:420-456`: `av_packet_move_ref` transfers
  packet payload ownership into and out of the packet queue.
- `references/ffmpeg/fftools/ffplay.c:3295`,
  `references/ffmpeg/fftools/ffplay.c:2205`, and
  `references/ffmpeg/fftools/ffplay.c:580-680`: ffplay uses a demux read
  thread, packet queues, and decoder worker loops around
  `avcodec_send_packet`/`avcodec_receive_frame`; native should move toward this
  shape by separating read, decode-submit, and present workers without adding a
  host-side compressed-payload FIFO.
- `references/ffmpeg/libavcodec/pthread.c:45-80` and
  `references/ffmpeg/libavcodec/pthread_slice.c:112-148`: FFmpeg CPU decode
  threading is frame/slice-thread driven. Vulkan hardware decode should not add
  arbitrary CPU decode threads; concurrency comes from queue handoff and Vulkan
  async execution depth.
- `references/ffmpeg/libavcodec/vulkan_decode.c:1370-1377` and
  `references/ffmpeg/libavcodec/decode.c:1088-1095`: FFmpeg sizes Vulkan decode
  async execution/hardware frame pools from the decode queue and hardware
  frames context. Native async-depth changes must follow this model and still
  pass the 25,000 KiB `Private_Dirty` gate.
- `references/ffmpeg/libavutil/buffer.h:222-303`: `AVBufferPool` reuses
  fixed-size buffers and only frees the pool after outstanding buffer refs are
  released. Scene animation resource updates should use the same lifetime
  shape: retain a bounded per-frame resource ring and update only the frame
  slot whose fence has completed, rather than forcing a global idle/wait before
  touching shared data.
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
- `references/ffmpeg/libavcodec/h264_slice.c:1295-1367`: H.264 output
  selection is a delayed-picture queue. It raises `has_b_frames` from SPS
  `num_reorder_frames` when VUI bitstream restriction is present, may increase
  the reorder buffer when observed POC order requires it, and outputs the
  lowest display-order picture only when the delayed queue exceeds that depth.
- `references/ffmpeg/libavcodec/vulkan_hevc.c:743-815` and
  `references/ffmpeg/libavcodec/vulkan_hevc.c:828-842`: HEVC fills
  `vp->ref_slots[idx]`, reference sets, and slice offsets.
- `references/ffmpeg/libavcodec/hevc/refs.c:267-305` and
  `references/ffmpeg/libavcodec/hevc/hevcdec.c:3371-3372`: HEVC output uses
  pending-output frames and the active SPS temporal layer's `num_reorder_pics`
  as the `max_output` display delay.
- `references/ffmpeg/libavcodec/vulkan_av1.c:298-358`: AV1 scans duplicate
  reference slots, fills unique refs, and writes `referenceNameSlotIndices`.
- `references/ffmpeg/libavutil/mem.c:98-165` and
  `references/ffmpeg/libavutil/mem.c:247-253`: FFmpeg allocates packet/parser
  storage through aligned malloc/realloc and releases with `av_free`; native
  must keep AV storage on FFmpeg ownership boundaries. Process allocator policy
  may limit glibc heap retention, but it is not a substitute for releasing
  packet/frame ownership at the FFmpeg lifetime boundary.

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
6. Distribution runs use the native process allocator policy. The scripts clear
   external malloc/glibc tuning variables before launch so user env cannot hide
   regressions; the `gilder-native-vulkan` binary then self-execs with the
   required glibc startup tunables and the FFmpeg shim applies process
   `mallopt` before opening AV input. Memory reductions still must come first
   from FFmpeg-aligned ownership, queue, copy, and lifetime changes.
7. Decode/present timeline synchronization follows FFmpeg's per-frame semaphore
   dependency shape: decode signals at `VIDEO_DECODE_KHR` completion and
   present waits on that per-frame value before touching the decoded image. Low
   GPU busy with stable 240 fps should be treated as CPU/submit/synchronization
   headroom, not as a reason to add copy paths or descriptor sets.
8. FFmpeg read-thread handoff is rendezvous by default rather than a hidden
   second compressed-payload FIFO. H.264/H.265 use the single-packet
   `packet_queue_get` shape; AV1 is the only path that declares packet splitting
   and it shares the FFmpeg packet backing by byte range. The worker now only
   allocates a pending access-unit queue for codecs that set
   `FFMPEG_PACKET_SPLITS_ACCESS_UNITS`; H.264/H.265 do not carry an unused
   hidden `VecDeque`.
9. Annex-B conversion keeps one reusable scratch buffer. Extra free converted
   payload buffers are not retained after upload, matching FFmpeg's
   packet-unref lifetime and keeping long-source `Private_Dirty` under the
   gate without allocator tuning.
10. The packet handoff queue can still hold three active packets, but the free
    Annex-B scratch pool retains only one buffer capped at 128 KiB. This keeps
    FFmpeg's `av_packet_move_ref` queue depth while avoiding a second hidden
    three-packet retained-payload pool.
11. Vulkan Video parameter sets now use `VK_KHR_video_maintenance2` inline
    submit. H.264/H.265/AV1 keep one validated STD payload owner per stream,
    create the video session with the inline-session-parameters flag bit, pass
    codec parameters through `VkVideoDecodeInfoKHR` pNext, and leave
    `VkVideoBeginCodingInfoKHR::videoSessionParameters` null in the streaming
    path.
12. H.264/H.265 decoded-frame handoff now follows FFmpeg display-order
    semantics instead of decode FIFO. H.264 parses SPS VUI bitstream
    restriction and uses `num_reorder_frames` as the initial `has_b_frames`,
    then adapts on observed B-picture/out-of-order keys as FFmpeg does; H.265
    uses SPS `max_num_reorder_pics` for the active temporal layer. DPB
    `array_layers` no longer masquerades as display reorder depth.
13. Ready-prefix video now has the FFmpeg execution split in runtime evidence:
    FFmpeg packet read thread -> bounded packet queue -> single video decode
    worker -> bounded decoded-frame handoff -> present worker. The default
    decode thread count remains one, while Vulkan async-depth follows FFmpeg's
    Vulkan decode formula. This is worker ownership and lifetime alignment, not
    host CPU decode parallelism or relaxed memory gates.
14. Animated full-scene spritesheet geometry now follows the same bounded
    per-frame ownership model: static sampled-image scenes keep one retained
    vertex buffer, while animated atlas draw steps allocate one host-visible
    vertex buffer per WSI frame slot. The present loop waits only the current
    slot fence, rewrites that slot's UVs from runtime elapsed time, and records
    the command buffer with that slot's vertex buffer. This removes the previous
    all-in-flight fence wait and matches the `AVBufferPool`/hardware-frame-pool
    lifetime rule above: resources are reused only after their owning frame has
    completed.

## Format Evidence

Performance evidence uses no optional allocator profile. Current memory gates
must be judged with external malloc/glibc tuning env cleared; the distribution
binary itself applies the required process allocator policy.

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
- Black-frame regression recovery after the video-session memory-type selector
  was relaxed back to the Vulkan-compatible rule:
  `/tmp/gilder-vulkan-h264-ready-prefix-video.RiIECB` from the 4K240 H.264
  ready-prefix smoke reports `present_backend=vulkanalia-decoded-image-dynamic-rendering-present`,
  decoded/presented `1440/1440`, `decoded_image_zero_copy_presented=true`,
  `present_mode=fifo-latest-ready`, `average_present_fps=240.19974298874544`,
  `performance_max_private_dirty_kib=21560`, `performance_max_pss_kib=99134`,
  `memory_category_heap_private_dirty_kib=3280`, and GPU process memory
  `84 MiB`. The selected video-session bind plan includes a driver-allowed
  host-visible bind for one requirement; this is now accepted instead of
  falling back to the clear placeholder path.

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

There is no optional allocator profile and no script-only tuning. Scripts clear
external `MALLOC_ARENA_MAX`, `MALLOC_MMAP_THRESHOLD_`,
`MALLOC_TRIM_THRESHOLD_`, `MALLOC_TOP_PAD_`, and `GLIBC_TUNABLES` before
launching the measured process so local shell state cannot make a run pass.
When built with `native-vulkan-video`, `gilder-native-vulkan` self-execs once
with the required distribution environment:
`MALLOC_ARENA_MAX=1`, `MALLOC_MMAP_THRESHOLD_=131072`,
`MALLOC_TRIM_THRESHOLD_=0`, `MALLOC_TOP_PAD_=0`, and
`GLIBC_TUNABLES=glibc.malloc.tcache_count=0` while preserving unrelated glibc
tunables. The FFmpeg shim also calls `mallopt(M_ARENA_MAX, 1)`,
`mallopt(M_TRIM_THRESHOLD, 0)`, `mallopt(M_TOP_PAD, 0)`, and
`mallopt(M_MMAP_THRESHOLD, 128 KiB)` before AV input open, and calls
`malloc_trim(0)` after video/audio FFmpeg teardown.

Current 4K240 comparison is the evidence in the format sections above. All
listed runs are under `performance_max_private_dirty_kib < 25000`,
`average_present_fps >= 239.999`, `descriptor_sets=0`,
`descriptor_heap_only=true`, and `all_zero_copy_presented=true`.

Real-source measurements show why the startup tcache tunable is required for
distribution behavior, not as a loose local profile:

- Workshop `3498008367`, H.264 Main 4K60 video-only, 360 presented frames:
  source-level `mallopt` only reported `Private_Dirty=24036 KiB`,
  heap `8820 KiB`, anon `2124 KiB`, and
  `average_present_fps=60.043573313409325`; adding the startup glibc tunables
  reported `Private_Dirty=21060 KiB`, heap `3272 KiB`, anon `4692 KiB`, and
  `average_present_fps=60.04155505460744`. After moving the tunables into the
  binary self-exec path and clearing external env before launch,
  `/tmp/gilder-real-video-3498008367-binary-allocator.1782592023` reports
  `Private_Dirty=20984 KiB`, heap `3200 KiB`, anon `4692 KiB`, and
  `average_present_fps=60.038757663791046`.
- Workshop `3454093707`, H.264 Main 4K60 video-only, 360 presented frames:
  source-level `mallopt` only reported `Private_Dirty=24496 KiB`,
  heap `8464 KiB`, anon `2776 KiB`, and
  `average_present_fps=60.04367534462413`; adding the startup glibc tunables
  reported `Private_Dirty=21872 KiB`, heap `3540 KiB`, anon `5260 KiB`, and
  `average_present_fps=60.0403326570722`.
- Workshop `3454093707`, same source with PipeWire S16LE output and
  audio-master pacing: source-level `mallopt` only reported
  `Private_Dirty=25128 KiB`; adding the startup glibc tunables reported
  `Private_Dirty=24772 KiB` with PipeWire streaming active, audio output
  frames `282`, samples `288768`, bytes `1155072`, and zero xrun/buffer/timeout
  errors. The binary self-exec path now passes the formal H.264 smoke at
  `/tmp/gilder-vulkan-h264-ready-prefix-video.n0gewv` with external allocator
  env cleared: `Private_Dirty=24688 KiB`, heap `4292 KiB`, anon `6664 KiB`,
  `presented_frame_count=360`, `average_present_fps=60.25415646627786`,
  `audio_output_backend=pipewire-s16le`, and zero xrun/buffer/timeout errors.
- Workshop `3407391149`, H.264 High 1700x1080 display / 1712x1088 coded
  extent plus AAC, 360 presented frames: this source exposed B-frame display
  reordering. The first FIFO decode-order handoff failed with non-monotonic PTS
  and display order; the FFmpeg-aligned delayed-picture handoff now passes at
  `/tmp/gilder-vulkan-h264-3407391149-audio-6s-trim0` with
  `Private_Dirty=24712 KiB`, heap `3920 KiB`, anon `5888 KiB`,
  `average_present_fps=60.24337748247325`, `audio_video_sync.ready=true`,
  source PTS deltas `16666666..16666667 ns`,
  `audio_output_xrun_count=0`, `ffmpeg_slices_buffer_pool_capacity_bytes=1651200`,
  and `max_src_buffer_range=953600`. After the PipeWire small-stack and
  FFmpeg Annex-B scratch retention update, the same coded `1712x1088` source
  passes at `/tmp/gilder-vulkan-h264-3407391149-audio-6s-smallstack-payload128`
  with `Private_Dirty=24780 KiB`, `average_present_fps=60.24070514810772`,
  `audio_video_sync.ready=true`, `audio_output_process_callbacks=282`, and
  `audio_output_xrun_count=0`.
- Workshop `3655044877` (`Mac-OS Dubai Night 4k 240 FPS`) is advertised as
  240 FPS but `ffprobe` identifies the downloaded MP4 as H.264 Main 8-bit
  `3840x2160`, `60/1` FPS, video-only. The first strict 6s smoke passed at
  `/tmp/gilder-vulkan-h264-3655044877-video-6s` with
  `Private_Dirty=21984 KiB`, heap `3064 KiB`, anon `5076 KiB`,
  `average_present_fps=60.0436230234559`, 360 submitted/presented frames,
  PTS/display order monotonic, and zero-copy present. The no-build rerun
  `/tmp/gilder-vulkan-h264-3655044877-video-6s-rerun` also passed with
  `Private_Dirty=22000 KiB`, heap `3060 KiB`, anon `5076 KiB`,
  `average_present_fps=60.04763680389402`,
  `ffmpeg_slices_buffer_pool_capacity_bytes=948992`, and
  `max_src_buffer_range=626688`.
- Workshop `2985290493` (`Reverse: 1999 Vertin`) is advertised as 4K120 but
  `ffprobe` identifies the MP4 as H.264 High 8-bit `3840x2160`, `60/1` FPS,
  plus AAC LC stereo 44.1 kHz. Full audio/video 6s strict smoke
  `/tmp/gilder-vulkan-h264-2985290493-audio-6s` passed with
  `Private_Dirty=24948 KiB`, heap `3944 KiB`, anon `7464 KiB`,
  `average_present_fps=60.25450651934367`, `audio_video_sync.ready=true`,
  source PTS deltas `16666000..16667000 ns`, zero xrun, 360
  submitted/presented frames, `ffmpeg_slices_buffer_pool_capacity_bytes=242688`,
  and `max_src_buffer_range=160768`. The no-build rerun
  `/tmp/gilder-vulkan-h264-2985290493-audio-6s-rerun` also passed at
  `Private_Dirty=24988 KiB`, `average_present_fps=60.276466283413136`, and the
  same A/V sync/zero-copy/PTS gates. The matching video-only run
  `/tmp/gilder-vulkan-h264-2985290493-video-6s` passed at
  `Private_Dirty=21788 KiB`, heap `3128 KiB`, anon `5568 KiB`, and
  `average_present_fps=60.04126508038644`, making this a current AAC/PipeWire
  memory-pressure sample rather than a video decode pressure sample. The
  current small-stack plus FFmpeg payload-retention cap binary passes
  `/tmp/gilder-vulkan-h264-2985290493-audio-6s-smallstack-payload128` at
  `Private_Dirty=24836 KiB`, `average_present_fps=60.27155100510979`,
  `audio_output_process_callbacks=259`, `audio_output_xrun_count=0`; its
  no-build rerun `/tmp/gilder-vulkan-h264-2985290493-audio-6s-smallstack-payload128-rerun`
  passes at `Private_Dirty=24908 KiB` and `performance_avg_cpu_percent=9.93`.

If a performance run starts immediately after
`target/release/gilder-native-vulkan` was rebuilt or replaced, Linux can report
the freshly executed binary mapping as private dirty memory. The sampler now
classifies `gilder-native-vulkan` under `memory_category_gilder_binary_*`
instead of the generic `file-mapping` bucket so this is visible. The native
binary also syncs its own executable before allocator self-exec on Linux, so a
freshly rebuilt binary enters the measured Vulkan process with clean executable
file mappings instead of relying on a script-side warmup. The H.264 smoke script
still records the release-binary fingerprint before and after `cargo build`,
preserves contaminated attempts as `performance.fresh-build-contaminated[.N]`,
and retries only to prove the same unadjusted memory gate; no file-mapping or
gilder-binary dirty memory is subtracted from reported totals.

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

- `src/renderer.rs`: top-level renderer planning orchestration and shared
  plan types. Scene-specific runtime property/controller code should not be
  added back here.
- `src/renderer/scene_runtime.rs`: `SceneWallpaperRuntimeSampler`
  source-backed runtime frame resampling and sampler state only.
- `src/renderer/scene_runtime/input.rs`: scene property resolution,
  manifest/render property numeric defaults, native controller active-state
  sampling, idle fade-ramp sampling, deterministic audio-response property
  values, and retained scene input-property collection.
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
- Scene present snapshots do not retain an unbounded per-frame `present_ids`
  vector. They expose `retained_frame_telemetry_limit`, `present_ids_head`, and
  `present_ids_tail`; the default retained-frame limit is `0`, so present-id2
  remains enabled for pacing while the CPU side keeps no historical frame ID
  list in steady state.
- `VK_KHR_present_mode_fifo_latest_ready` is queried and enabled through the
  device feature chain when available. Present mode selection now uses
  `FIFO_LATEST_READY` when that KHR feature and surface mode are both
  available, otherwise `FIFO_RELAXED`/`FIFO`. `MAILBOX` is not a native Vulkan
  fallback even when the surface advertises it. Ready-prefix video smoke gates
  now accept only `fifo-latest-ready`, `fifo-relaxed`, or `fifo` and fail
  `mailbox`/`immediate` evidence.
- Resource memory binding and host mapping use Vulkan 1.4-style
  `vkBind*Memory2`, `vkMapMemory2`, and `vkUnmapMemory2`.
- Scene sampled-image upload now follows the video memory model instead of a
  CPU decoded-image model: native `.gtex` BC7 payload is streamed directly
  through one 128 KiB host-visible staging buffer, recorded with
  `cmd_copy_buffer_to_image2`, submitted with `queue_submit2`, and trimmed after
  each chunk. The runtime never keeps a full PNG/JPG/RGBA payload in process
  memory. Dynamic/full scene image layers still use the retained GPU image plus
  `VK_EXT_descriptor_heap` real-time render path.
- Pure static `.gtex` wallpapers use a transfer-only first-present route:
  upload into a BC7 `TRANSFER_SRC` image, `cmd_blit_image2` into the swapchain,
  wait the submit fence, then destroy the source image. This is a fast path, not
  a capability cut; video layers, mixed scene layers, animation, and full scene
  rendering continue through their native render/decode paths.
- Dynamic scene geometry now follows the same reuse rule as the video
  bitstream ring: dynamic solid-quad vertex buffers and dynamic/atlas
  sampled-image vertex buffers are allocated per frame slot and kept mapped
  until resource teardown. Per-frame updates serialize vertices directly into
  the mapped buffer and flush only when the selected memory type is
  non-coherent, so the hot path does not build an intermediate byte `Vec`,
  repeat `vkMapMemory2`/`vkUnmapMemory2`, or overwrite vertex data still owned
  by an in-flight frame. Dynamic scene sampling now takes a lightweight runtime
  frame and lowers its layers directly into Vulkan geometry; mixed
  sampled/solid scenes share one sampled frame per elapsed timestamp, move the
  sampled/solid geometry into the upload call, and drop the short-lived cache
  after both sides are consumed. Dynamic updates validate retained topology and
  rewrite only vertex bytes; index buffers and sampled-resource lists are
  retained and compared, not rebuilt as upload payloads every frame. Static
  topology animated atlases go narrower again: each frame slot is initialized
  once, runtime animation patches only the UV bytes for the animated vertices,
  and the present resources keep compact animated-UV metadata instead of a full
  CPU-side base-vertex copy.
- Vulkan Video decode uses `VK_KHR_video_maintenance2` inline session
  parameters when the device enables it. The video session is created with
  `VK_VIDEO_SESSION_CREATE_INLINE_SESSION_PARAMETERS_BIT_KHR` (bit `0x20` in
  current vulkanalia), H.264/H.265/AV1 parameter payloads are attached to
  `VkVideoDecodeInfoKHR`, and the streaming path does not create or bind a
  `VkVideoSessionParametersKHR` object.
- Vulkan Video session memory binding now treats `memoryTypeBits` as the
  driver-owned compatibility set. The selector still prefers
  device-local/non-host-visible memory, then device-local memory, but it no
  longer rejects a driver-allowed host-visible or BAR-style type when that is
  the only legal bind target. Rejecting those bits caused the runtime to fall
  back to the clear placeholder path, which looked like a black video frame
  even though decode had never started.
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

1. Extend the scene/video resource-lifetime pattern to remaining scene payloads:
   bounded staging/ring buffers, `queue_submit2` fence ownership, and immediate
   CPU-side release after GPU handoff. `hostImageCopy` remains optional for
   paths where it gives lower retained memory than the 128 KiB staging ring, but
   it must not reintroduce CPU-retained decoded-frame copies.
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

The scripts clear externally supplied `MALLOC_ARENA_MAX`,
`MALLOC_MMAP_THRESHOLD_`, `MALLOC_TRIM_THRESHOLD_`, `MALLOC_TOP_PAD_`, and
`GLIBC_TUNABLES` before launching the video process. There is no
`--allocator-profile` option; allocator behavior is the binary's fixed
process-glibc-mallopt-tcache-off policy.

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
   retaining packet payloads. The native audio runtime now uses a fixed
   128 KiB Rust output-worker stack and a 128 KiB PipeWire thread-loop stack;
   larger PipeWire buffer overrides were rejected because they raised process
   callback count without meaningful memory benefit. Video FFmpeg Annex-B
   scratch retains only one small reusable payload buffer and releases 4K-scale
   packet storage after handoff, matching FFmpeg's `av_packet_unref` lifetime
   more closely. Full-scene audio output integration is complete on the same
   FFmpeg/PipeWire-only backend used by direct video; audio-response remains a
   separate visual input system, not an alternate PulseAudio/ALSA/GStreamer
   output path.
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
   Workshop item `3454093707` (`Mercy_Full_Audio.mp4`, H.264 Main 4K60 plus
   AAC) exposed a startup deadlock when audio output could be heard but decoded
   video stayed black and the run never exited. The cause was treating the
   fixed 3-frame FFmpeg-style handoff FIFO as a startup preroll requirement
   while this stream only needs two coincident DPB/output image layers
   (`session_max_dpb_slots=2`, `resource_image.array_layers=2`). Decode could
   block waiting for a layer release while present was still waiting for three
   queued frames. Decoded-image present now starts as soon as the first display
   frame is available; the 3-frame value remains FIFO capacity only. A rebuilt
   4-frame run exits with `presented_frame_count=4`,
   `decoded_image_zero_copy_presented=true`,
   `audio_output_backend=pipewire-s16le`, and `audio_output_xrun_count=0`.
   The current binary self-exec allocator run exits, presents `360/360` frames
   at `average_present_fps=60.25415646627786`, reports
   `audio_video_sync.ready=true`, and passes the strict 25MiB
   `Private_Dirty` gate at `24688 KiB`; the remaining work is further source
   lifetime reduction, not black-screen recovery or script-side allocator
   tuning.
   Direct video audio was revalidated against the same `3454093707`
   `Mercy_Full_Audio.mp4` source through
   `--run-video --audio-clock-probe --audio-output auto --unmuted` for 6 s:
   `presented_frame_count=360`, `decoded_image_zero_copy_presented=true`,
   `audio_output_backend=pipewire-s16le`,
   `audio_output_stream_state=streaming`, `audio_output_stream_ready=true`,
   `audio_output_frames=282`, `audio_output_bytes=1155072`,
   `audio_video_sync.ready=true`, drift `15999999 ns`, and zero xruns,
   buffer errors, or timeout errors.
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
   full Wallpaper Engine scene execution. Status is tracked by the completed
   and pending native boundary lists:
   package/conversion boundaries, `scene/gscene` format validation,
   snapshot-time propagation, render clear-color snapshot layers,
   WE `scene.pkg` direct import,
   WE parent-id graph lowering into gscene children,
   native scene graph transform/opacity execution,
   WE `shape`/`solid`/`radius` lowering into native rectangle/ellipse nodes,
   explicit WE keyframe timeline lowering into gscene timeline channels,
   WE `{script,value}` wrapper lowering without a JS engine, deterministic
   numeric SceneScript expression lowering into native property bindings,
   geometry-field timeline/property animation, parallax depth camera-property
   offsets, Hyprland compositor cursor parallax input,
   WE TEXV0005/TEXB0004 RGBA texture decoding, spritesheet atlas resources
   with time-sampled UV frame selection, retained sampled-image
   resources, solid/image mixed composition, descriptor heap sampling,
   visible scene runtime status, native present route selection,
   retained resource status, clear-background composition, native runtime
   layer-coverage accounting, rounded-rectangle tessellation, simple/concave
   path tessellation, cubic/smooth-cubic/quadratic/smooth-quadratic path
   flattening, SVG elliptical arc path flattening, compound even-odd path
   fill, stroke geometry, deterministic text glyph geometry,
   native deterministic particle emitter expansion into solid or sampled-image
   sprite geometry,
   first-class `video` layer detection, single-video-layer Vulkan Video scene
   composition, clear-background plus video scene composition, scene
   timeline animation snapshotting plus per-frame fixed-topology geometry
   updates during native present, property update binding, pause/resume
   policy, package state/property persistence, retained scene input properties
   for bound user properties and native controller aliases, renderer-resolved
   scene audio cues, native FFmpeg/PipeWire scene audio cue playback, and native
   audio-response visual geometry driven by standardized gscene audio
   property bindings, and native idle video-switch controller sampling for
   `scene.controller.<node>.active` property bindings including fade-in
   opacity ramps are in place;
   arbitrary SceneScript, shader/material graph, WE particle rope/trail
   renderers, complex particle operators, particle material shader parity,
   real PipeWire spectrum/FFT audio-response input,
   complex font
   shaping/atlas typography, explicit nonzero path fill-rule selection,
   and actual mixed
   video-as-scene composition remain open. Wallpaper Engine scene conversions
   now write `assets/*.gscene.json`
   documents with `source`, `size`, `render`, `camera`, `import`,
   `resources`, `nodes`, `systems`, `native_lowering`, and
   `unsupported_features` sections, plus a structured
   `full_scene` report block with
   `target_runtime=native-vulkan-full-scene`,
   `current_runtime=native-vulkan-scene-runtime`,
   preserved source-scene metadata paths, completed boundaries, and pending
   full-scene boundaries. Current gating uses the explicit boundary lists.
   Converted scene entries no longer inject a default
   `max_fps: 60`; scene FPS is governed by explicit manifest/user policy caps
   and present pacing, so real-time scene rendering is not artificially limited
   by video ready-prefix/playback frame policy. Gilder scene is the runtime
   format, not a
   Wallpaper Engine schema clone: WE's historical fields are treated as an
   input dialect and are isolated in converter-owned `provenance`/`import`
   metadata. Runtime-facing node roots stay clean (`type`, `transform`,
   `resource`, `effects`, `audio`, draw properties), while WE ids, parent ids,
   dependencies, original transforms, model/material chains, particles,
   animation layers, and instance overrides live under node `provenance`.
   Matched WE parent ids are lowered into real gscene `children`, so the core
   snapshot path now composes parent/child transform and opacity as a completed
   native scene graph runtime boundary instead of only preserving parent ids as
   metadata. Parent rotation is applied to child and particle positions during
   snapshot flattening, so nested rotated groups no longer leave native layers
   offset from the scene-space transform hierarchy. `native_lowering.fallback`
   was removed from the gscene runtime
   schema: package preview images may still exist for UI metadata, but they are
   not referenced as scene resources and are not substituted as runtime nodes.
   `render.clear_color` with
   `clear_enabled != false` now emits the first snapshot color layer, so
   converted WE scene clear color participates in native clear-background and
   mixed scene composition. WE `{ value: ... }` wrappers for text, point size,
   font, and horizontal alignment are lowered to gscene text node fields. Font
   values that reference package font files now copy into first-class
   `SceneResourceKind::Font` entries with text-node `font_resource` references,
   and the snapshot/render/native draw data path carries the resolved package
   font source for future atlas shaping. WE `visible: { value, user }` is
   lowered to a gscene opacity property binding so runtime property updates can
   reveal/hide the layer without a legacy visibility path. WE `shape`/`solid`
   objects now lower directly into
   gscene `rectangle`/`ellipse` nodes with color, size, and `corner_radius`,
   so ordinary vector shape layers enter the same native solid-geometry
   runtime instead of staying as source metadata. Native gscene
   `particle-emitter` nodes now expand deterministically at snapshot time from
   `properties.particle` into stable rectangle/ellipse particle layers with
   bounded `count`, seeded spawn phase, lifetime, rate/count, speed, size,
   spread, gravity, fade, and emitter area; WE independent particle objects
   lower through `SceneParticleIr`, including `particle`/`emitter`,
   `instanceoverride.count/rate/speed/size/lifetime/colorn`, `speedMin`,
   `speedMax`, `directionDeg`, `spreadDeg`, `gravityDirection`,
   `gravityStrength`, `fadeOut`, particle dimensions, and spawn area into
   native runtime fields. External WE particle definition files referenced as
   `particles/*.json` now also lower through the same IR: `maxcount`,
   `material`, `emitter` `boxrandom`/`sphererandom` runtime fields,
   `distancemax`/`directions` spawn area, `rate`, `speedmin`/`speedmax`,
   simple `sizerandom`/`lifetimerandom`/`colorrandom` initializers, movement
   gravity, renderer fade metadata, and CWE-backed defaults become gscene
   particle properties before object fields override them and
   `instanceoverride` applies WE count/rate/speed/size/lifetime multipliers.
   Particle `material` definitions now reuse the WE material texture lowering
   path, copy the material/texture resources into gscene, attach the renderable
   texture resource to the particle-emitter node, and make emitted particle
   layers enter the sampled-image scene path when a material texture is
   available; particles without a material texture remain deterministic
   solid geometry. WE built-in particle bubble texture references such as
   `particle/bubbles/bubble3` now generate a native BC7 `.gtex` sprite resource
   with role `we-builtin-particle-texture` instead of being reported as a
   missing package file, so preset particle materials can still enter the same
   sampled-image runtime.
   The current WE field references for this mapping are
   `references/linux-wallpaperengine/src/scene/loader/object.rs`, whose
   `Object` includes `particle` and whose `Instanceoverride` carries
   `alpha`, `speed`, `size`, `lifetime`, `count`, `rate`, and `colorn`, plus
   `references/linux-wallpaperengine/src/scene/loader/scene.rs` for
   `gravitydirection`/`gravitystrength`, and
   `references/cwe/src/WallpaperEngine/Data/Parsers/ObjectParser.cpp` plus
   `references/cwe/src/WallpaperEngine/Render/Objects/CParticle.cpp` for WE
   particle JSON loading, emitter/default parsing, instance override
   multiplication, box/sphere spawn ranges, initializer behavior, and movement
   gravity. Explicit source
   keyframe tracks for supported transform/opacity properties now lower into
   gscene `timelines`, including vector `origin`/`scale` split into native
   `x`/`y` and `scale-x`/`scale-y` channels, so the existing core timeline runtime
   executes converted motion instead of leaving it only in provenance.
   `SceneTimelineIr` now also extracts supported property-local keyframes from
   WE dynamic value wrappers such as `origin: { value, keyframes }` or
   `alpha: { value, frames }`, treats bare WE `time` fields as seconds,
   unwraps `{ value: ... }` frame values, and accepts compact `[time, value]`
   frame pairs before writing clean gscene timeline channels. The same
   `SceneTimelineIr` path now lowers deterministic WE `animationlayers`
   keyframes into executable gscene timelines. Animation layer
   `rate`/`speed`/`timescale` now compiles into keyframe time-scale transforms
   in IR, so faster/slower layers execute through the same native timeline
   runtime; only complex blend/weight layer semantics remain preserved as
   explicit pending metadata. The same channel now covers geometry fields
   (`width`, `height`, `corner-radius`), and WE
   `{script: ..., value: ...}` wrappers are unwrapped to deterministic gscene
   defaults without introducing a JS engine. User-bound
   scalar wrappers for transform, opacity, size, and radius lower into
   `property_bindings`; deterministic numeric SceneScript expressions over one
   user property, `value`, constants, parentheses, and `+ - * /` now compile
   into the same native `scale`/`offset` binding model. Arbitrary JS-like
   SceneScript remains explicit pending work instead of being executed by a
   compatibility VM. The intended full SceneScript direction is native
   lowering, not embedding a JS engine: common `engine.frametime`,
   `engine.setTimeout`, `visible`/`alpha`, `play`/`pause`/`stop`, user property
   changes, idle timers, mouse/cursor input, and audio-spectrum inputs should
   lower into first-class gscene controllers/state machines and a bounded
   scene input model. Scripts that cannot be reduced to direct property
   bindings should produce a restricted scene IR such as `on-idle -> reveal
   layer -> play video/audio -> fade -> hide`, with unresolved source scripts
   kept as explicit pending systems rather than compatibility runtime code.
   The converter now has an internal IR boundary for native scene lowering.
   WE utility/control scripts are first normalized into `SceneControllerIr`
   with a typed controller kind, target layer, native active property,
   default-hide policy, and copied controller settings; only then are gscene
   `properties.controller` metadata emitted. Idle/click/property controllers
   lower to opacity `property_bindings` plus completed/pending input-source
   and fade-ramp features. Deterministic timed visibility controllers using
   `targetLayerName`, `enableAutoControl`, `startDelay`, `showDuration`,
   `hideOnStart`, `fadeDuration`, `loopControl`, and `loopInterval` lower
   directly to target-node opacity timelines, so `engine.setTimeout`-style
   reveal/fade/hide scripts execute through the native timeline runtime
   without a JS VM. Built-in fullscreen utility target layers with renderable
   signals lower to native viewport-sized rectangles using the scene
   `orthogonalprojection` when they do not carry an explicit size. Deterministic
   WE clock/date text scripts based on `new Date()` now lower to native
   `properties.text_binding` entries for clock time, vertical date, and vertical
   weekday text; the runtime resolves those bindings through the scene sampler's
   text resolver instead of preserving a static `value` or executing script.
   Deterministic numeric SceneScript expressions now lower through
   `SceneNumericPropertyBindingIr`, which owns the linear expression parser and
   emits native `scale`/`offset` gscene property bindings. Runtime sampling
   consumes the lowered native controller settings, so `fadeInDuration` on idle
   controllers becomes a sampled opacity ramp rather than an instant 0/1
   switch. This IR is not a runtime compatibility layer and is not serialized
   as a public wallpaper format; it is the converter-owned normalization step
   that future SceneScript, animation-layer, and effect-graph lowering should
   target before writing clean gscene.
   Parallax now has a gscene runtime model:
   `render.parallax.amount` plus node `parallax_depth` consumes
   `scene.parallax.x/y` property values to offset snapshot transforms.
   The converter now understands WE `object.image` as a model JSON entry
   rather than a direct image path, follows `model -> material -> texture`,
   copies model/material/effect/audio/texture assets into the gscene resource
   graph, and assigns `node.resource` only to a native sampled-image resource.
   Standard WE `TEXV0005/TEXB0004` RGBA `.tex` material textures are decoded
   through their LZ4 block payload, and WE `.tex` video payloads with MP4/WebM
   container signatures are extracted as first-class gscene `video` resources
   instead of being copied as runtime `texture` compatibility assets.
   The converter split follows the same-name module root plus same-name
   directory rule (`wallpaper_engine.rs` with `wallpaper_engine/tex.rs`,
   `wallpaper_engine/gtex.rs`, `wallpaper_engine/effect.rs`,
   `wallpaper_engine/ir.rs`, `wallpaper_engine/ir/animation.rs`,
   `wallpaper_engine/ir/controller.rs`, `wallpaper_engine/ir/effect.rs`,
   `wallpaper_engine/ir/particle.rs`, and `wallpaper_engine/ir/timeline.rs`);
   `mod.rs` is not used for new scene-conversion code. `ir.rs` stays the
   SceneScript numeric/property-binding IR root, controller state-machine
   lowering lives in `ir/controller.rs`, deterministic animation layer
   expansion lives in `ir/animation.rs`, WE opacity effect normalization lives
   in `ir/effect.rs`, WE object-level particle fields and external particle
   definition normalization live in `ir/particle.rs`, and explicit
   keyframe/timeline normalization, including embedded property keyframe
   extraction, lives in `ir/timeline.rs`.
   WE built-in utility models such as `models/util/fullscreenlayer.json` and
   `models/util/composelayer.json` are now recognized as first-class native
   utility script layers in provenance instead of being treated as missing
   project files. Pure WE sound objects lower to `type: "audio"` gscene cue
   nodes and are skipped by visual Vulkan draw planning; they no longer inflate
   audio-response detection or material graph status.
   Non-spritesheet textures are cropped to the
   model's first frame when the model width/height divides the atlas; WE
   `SPRITESHEET` materials instead write the full atlas as a generated native
   BC7 `.gtex` image resource and attach `properties.spritesheet`
   (`atlas-grid`, atlas size, frame size, columns/rows, frame count, FPS, loop
   flag) to the gscene node. The original `.tex` remains as provenance, while
   the runtime-facing sampled image/video is the generated `.gtex` or extracted
   video file. Runtime `_rt_`
   textures, custom shaders,
   arbitrary SceneScript, executable effect graphs, and audio-response systems
   are preserved structurally and reported as explicit pending runtime systems
   instead of being hidden behind a legacy loader. `SceneEffect` now carries an
   explicit `runtime` classification: `native-opacity-timeline`,
   `native-text-glow`, `metadata-only`, or `wallpaper-engine-effect`.
   Deterministic no-op or invisible WE effects are preserved as metadata and no
   longer block a renderable texture material graph. WE
   `effects/opacity/effect.json` alpha constants and bounded fade SceneScript
   patterns now lower through `SceneOpacityEffectIr` into native gscene
   `opacity` timelines without embedding a JS engine, including
   `delayTime`/`fadeTime` plus `startDelay`/`fadeDuration` and explicit
   source/target alpha aliases. WE `blurprecise/effect.json` text effects lower
   to native text glow snapshot geometry with bounded sample offsets, so common
   text glow/blur does not require a WE shader graph runtime. Native-lowered or
   no-op/invisible effects are preserved as structured node metadata but no
   longer copied into runtime `we-effect` resources. Only
   `wallpaper-engine-effect` entries with real WE effect resources keep the
   shader/effect graph boundary pending.
   Current real Workshop semantic-debug sample: Steam Workshop item
   `3742497499` (`麻匪 白泽夢`) converts to
   `/tmp/gilder-we-3742497499-output-user-bindings`. The default runtime
   snapshot `/tmp/gilder-3742497499-user-bindings-default-snapshot.json`
   already contains the long-hair draw ops: shadow hair nodes
   `node-43..48`, main hair nodes `node-51..56`, and resources
   `resource-80-1-frame-0.gtex`, `resource-85-2-frame-0.gtex`,
   `resource-90-3-frame-0.gtex`, `resource-95-4-frame-0.gtex`,
   `resource-101-5-frame-0.gtex`, and
   `resource-107-6-frame-0.gtex`. The source project default
   `newproperty28=true` means the shadow/long-hair branch is enabled by
   default; `visible.value` is save-time UI state and must not be treated as a
   permanent gate when a WE `visible.user` condition exists. Therefore the
   observed missing/default-short-hair and missing transparent blue background
   are conversion-semantics blockers, not a reason to reopen `.tex`, `.gtex`,
   BC1/BC3/BC7 payload, Vulkan sampled-image upload, target-size conversion,
   or static flattened-preview investigations. The sample is a stack of many
   WE image components and effect/material passes, not one image that should be
   re-cropped, re-packed, or re-uploaded differently. Any later regression on
   this item must be debugged by comparing the source WE graph, normalized IR,
   gscene nodes, and runtime draw ops for each component's global transform,
   visibility, material, blend, and draw order. Do not route this back through
   texture decoding or Vulkan upload unless new direct evidence contradicts the
   existing draw-op/resource evidence.
   Status as of 2026-06-30: the current working conversion for this sample is
   `/tmp/gilder-we-3742497499-output-we-mesh-uv`. The following issues are
   closed and must not be reopened without new direct evidence: missing
   transparent blue background, missing/incorrect WE `colorBlendMode` routing,
   static puppet body, top hair/eyes detaching from the head, sampled-image
   texture/upload visibility, ordinary WE model image UV direction for the two
   `底发` stacks, and the leg-area residual/ghost layer. Validation evidence is
   the 6 second no-FPS-limit native run on `HDMI-A-1`, with
   `scene_present_route=sampled-image`, 10 Vulkan blend pipelines (`solid` and
   `sampled-image` alpha/additive/multiply/screen/max),
   `puppet_animation_layer_count=10`, dynamic full-scene sampling, and explicit
   WE UV meshes on the bottom-hair nodes (`node-43..48` and `node-51..56`).
   The leg residual was not an extra layer to delete: source object `1142`
   (`角色主影子` -> `主身体`) is a valid animated shadow/blur body branch with
   `alpha=0.30000001`, `color="0.00000 0.00000 0.00000"`, and the same puppet
   animation layers as the main body. The fix is sampled-image color/tint
   modulation through the retained Vulkan sampled-image vertex format and
   fragment shader, preserving the moving clothing/skirt shadow while drawing
   it as a dark translucent layer instead of a second original-color body.
   If bottom-hair visual alignment is reported again, investigate the generated
   WE model-image mesh/UV semantics and runtime mesh sampling before considering
   transform math; do not add one-off offsets or texture edits.
   Current open visual gaps on this sample are shader/effect-mask runtime gaps:
   water ripple/caustic visibility depends on WE `watercaustics`, `waterflow`,
   `waterripple`, `waterwaves`, normal/phase/mask textures, and material pass
   semantics; the existing native-effect-motion approximation is not enough to
   reproduce the missing water-surface ripple visible in the reference video.
   Closed-eye transparency is also an effect/mask issue, not a reason to hide
   the eye layer: source eyes `1336` and `1530` use the same
   `models/眼睛.json` puppet mesh, with `1336` carrying `iris` plus
   `waterripple` effects and `1530` carrying an `opacity` mask effect. The
   current symptom is that the eyelid/eyebrow close state moves down while the
   transparent iris remains visible behind it. Those `wallpaper-engine-effect`
   mask paths must be implemented as reusable material/effect modules so the
   closed-eye state occludes the transparent iris correctly; do not paper over
   it by deleting or globally hiding the eye layers.
   The same material/effect runtime backlog also covers the currently missing
   clothing-side soft blur/drift and skirt floating motion; those are WE
   blur/sway/waterwave-style effect semantics layered on top of the preserved
   image/mesh nodes, not replacement textures. Visible jagged edges remain an
   open renderer-quality gap to verify against sampler mode, alpha-mask
   execution, geometry edge coverage, and any future MSAA or post-filtering
   pass. Do not treat edge aliasing as a reason to re-export source PNGs or
   hand-edit alpha unless resource evidence directly proves a bad asset.
   Follow-up engineering constraints from this point:
   all future work must optimize for the long-term native architecture, not a
   short-term visual substitute. Do not add sample-specific compatibility
   branches, magic offsets, hidden-layer switches, resource re-export hacks,
   preview fallbacks, or temporary alternate render paths to mask missing WE
   semantics. If a visual gap comes from unsupported effect, material, mask,
   blend, interaction, scene format, or renderer-quality behavior, fix or
   design that first-class subsystem and document any remaining boundary.
   performance validation for this WE scene must use the release
   `gilder-native-vulkan` binary; debug builds are acceptable for functional
   smoke checks, but FPS/frame pacing numbers from debug builds must not be
   used as performance evidence. The current JSON gscene document is no longer
   a sufficient long-term format for the lightweight runtime target. A new
   binary scene format needs a real design pass covering versioning, schema
   evolution, resource-table indexing, compact animation/timeline data,
   random-access loading, and retained GPU resource binding; do not try to
   solve that with ad hoc JSON trimming. The native scene renderer also needs
   module boundaries before more WE effects are added: blend policy/equations,
   solid quads, sampled images, puppet/skinned geometry, effect-lowered visual
   layers, and future shader/material graph execution should live in focused
   modules instead of continuing to grow the current large draw-pass/runtime
   files. This split is a maintainability requirement for future effect
   completeness and extension work, not optional cleanup.
   Future work on this sample must stay on the WE-to-gscene semantic path:
   material passes must lower `shader`, `blending`, `combos`, depth/cull, and
   texture-pass metadata into `properties.material`; WE `translucent` and
   layer/effect `blend`/`alpha` semantics must not be rendered as an ordinary
   opaque image with opacity `1`; utility `composelayer`/`fullscreenlayer`
   output and `watercaustics`, `waterflow`, `waterripple`, `waterwaves`, and
   `shake` effects must lower to native gscene visual/motion IR; relative loop
   parent-origin timelines on the two `底发` groups must continue to drive
   their child image stacks; user color bindings such as `newproperty5` and
   `newproperty6` must resolve through the same runtime text/property resolver
   used by scene planning. A concrete example is `Water Caustic`
   (`node-57-models-workshop-2790231929-wc-test-json`): its material pass is
   `shader=genericimage2`, `blending=translucent`, depth disabled, and it sits
   immediately after the long-hair layers. Treating that material as a normal
   fully opaque sampled image can hide the already-present long-hair draw ops.
   The fix is first-class gscene material/effect semantics, not a preview
   fallback, legacy loader mapping, resource probe, compatibility branch, or
   one-off Workshop-specific patch.
   There is no internal legacy scene format, loader, preview-fallback scene
   node, or lowering bridge; old `layers` fixture data was replaced by
   `nodes/resources` gscene documents. Static wallpapers now lower into a
   single-image scene
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
   runtime smoke. The same CLI now loads package-owned `.gscene.json` sources
   directly; `--scene-root` pins the package root when the source is not under
   the standard `assets/` directory, so converted full-scene packages can enter
   the native presenter without first being flattened into an image/video-only
   CLI plan. Text layers now lower into deterministic built-in glyph
   geometry and render through the same solid dynamic-rendering pipeline as
   rectangles, rounded rectangles, ellipses, and simple paths; this gives text
   layers real native coverage without adding a legacy font-renderer
   compatibility path; package font files are now available as retained scene
   resources instead of being only string metadata. Path layers now parse
   `M/L/H/V/Z`, `C/S/Q/T` cubic,
   smooth cubic, quadratic, and smooth quadratic Bezier commands, plus `A/a`
   SVG elliptical arcs. Curves flatten into deterministic 16-segment
   polylines; arcs flatten from SVG center-parameterized geometry with 8
   segments per quadrant, scaled radii, rotation, and large-arc/sweep flags.
   The resulting native points feed the existing fill/stroke tessellation
   path. Multi-subpath paths now enter deterministic scanline fill respecting
   explicit nonzero and even-odd fill rules, so common Wallpaper Engine/SVG
   curve, arc, compound-hole, and winding-filled shapes render as solid Vulkan
   geometry instead of staying as unsupported path metadata. The CLI exposes
   this through `--path-fill-rule nonzero|evenodd`; WE/SVG `fillRule` fields
   lower into the same first-class gscene `path_fill_rule`.
   Scene sampled-image uploads now stream native `.gtex` BC7 through one 128 KiB
   staging buffer with `cmd_copy_buffer_to_image2` and `queue_submit2`, matching
   the video path's bounded-buffer handoff instead of retaining full CPU image
   payloads. Pure static `.gtex` sources additionally use
   `scene-static-transfer-visible-present`: no scene geometry buffer, no
   descriptor heap, no sampled-image graphics pipeline, and the source
   transfer image is destroyed after the first present fence. The 8K cloud-city
   static run on `2026-06-28` presented for 6s with
   `max_pss_dirty_kib=17602`, `max_private_dirty_kib=17600`, a 128 KiB staging
   buffer, and `scene_resource_model=static-transfer-first-present-source-release`.
   The corrected-orientation source `/tmp/gilder-cloud-city-8k-static.gtex`
   repeats the same route at `/tmp/gilder-dgop-static-gtex-original.UgDluL`
   with `max_pss_dirty_kib=17516`, `max_private_dirty_kib=17508`, dgop
   `memoryCalculation=pss_dirty` after startup, and 40 MiB process GPU memory.
   Mixed, animated, video, and full scene rendering are not reduced by this fast
   path; they continue through the native descriptor-heap real-time scene
   renderer. Dynamic scene geometry now uses persistently mapped per-frame
   vertex buffers for dynamic solid quads, dynamic sampled-image quads, and
   animated atlas sampled-image quads, so full-scene animation updates reuse
   bounded buffers instead of mapping/unmapping or overwriting a single
   in-flight vertex buffer. Dynamic sampled/solid geometry now shares one
   lightweight sampled frame per elapsed timestamp, reuses sampler-owned
   snapshot/render-layer scratch buffers, and bypasses full runtime snapshot
   construction in the per-frame path. Sampled-image draw planning now
   deduplicates identical source paths before assigning descriptor/image
   resource indices, so many sprite particles using the same material texture
   keep one retained sampled resource instead of one resource per emitted
   particle. Solid-only dynamic scenes now lower
   render layers directly into Vulkan solid geometry instead of building the
   diagnostic draw-plan/pass-plan payload first; dynamic sampled/mixed scenes
   now do the same direct render-layer lowering for sampled-image quads and
   solid overlays. The resulting geometry is moved into the upload side without
   clone-retained diagnostic payloads, and only vertex bytes are written
   directly into the mapped frame-slot buffer.
   Scene runtime and `SceneWallpaperPlan`
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
   `curve_path_layer_count`, `arc_path_layer_count`,
   `compound_path_layer_count`,
   `text_geometry_layer_count`, and
   `stroke_geometry_layer_count`, so scene
   progress is tied to actual layer
   coverage rather than treating scene as full scene.
   Renderer plans now also count `timeline_animation_count`,
   `timeline_animated_layer_count`, and `property_binding_count`; these values
   are carried into `NativeVulkanRenderItem::Scene` and
   `runtime.full_scene` instead of being inferred at the reporting boundary.
   The property binding path uses the persisted global/output `AppState`
   property store and the same resolver used to build visible scene snapshots.
   Scene audio cues are preserved in `SceneSnapshotLayer`, resolved into
   renderer-local `SceneRenderAudioCue` package paths, counted in
   `SceneWallpaperPlan`, surfaced as `scene_audio_cue_count` plus
   `scene_audio_cue_resource_model_ready` in `runtime.full_scene`, and played
   by the scene present runtime through the same FFmpeg audio reader and
   PipeWire-only output backend used by direct video. `start_silent=true` cues
   are not auto-started; `playback_mode=loop` enables FFmpeg EOS seek for the
   requested present duration. The stable gscene format now carries native
   `audio[].active_conditions` entries, each with a numeric `property` and
   optional `equals` value. Snapshot sampling resolves those conditions through
   the same property/controller resolver as visual bindings and exposes only
   active cues to the PipeWire scene audio worker; Wallpaper Engine script
   details stay in converter IR and are not runtime fields. Audio response
   remains a separate pending visual-response system.
   Static Wallpaper Engine image projects with real audio files are not
   converted to a no-audio static-image package: the converter writes a
   first-class `scene` manifest with one static image node plus gscene audio
   cues, converts the visual image offline to native BC7 `.gtex`, sets
   `runtime.allow_audio=true`, copies the audio into `assets/`, and marks
   `static-image-bc7-gtex-conversion`, `static-image-audio-scene` plus
   `scene-audio-cue-pipewire-present-runtime`. This keeps static visual
   wallpapers eligible for the same PipeWire audio runtime without adding a
   legacy static-audio side path.
   Current scene-audio runtime smoke converts a static image project whose
   `audio` field references `Mercy_Full_Audio.mp4`; the converted gscene uses
   `assets/wallpaper.gtex` plus `assets/audio-cue-0.mp4`. Running
   `--run-scene --audio-output auto --unmuted` for 6 s reports
   `scene_present_route=sampled-image`, `full_scene_complete=true`,
   coverage `100`, one scene audio cue, `audio_output_backend=pipewire-s16le`,
   `audio_output_stream_state=streaming`,
   `audio_output_stream_ready=true`, `audio_output_frames=282`,
   `audio_output_bytes=1155072`, `playback_target_reached=true`, and zero
   xruns, buffer errors, or timeout errors.
   Cursor parallax remains a first-class gscene property/camera model, but the
   compositor-global input source is intentionally not a full-scene completion
   gate. Core Wayland does not provide transparent background pointer events;
   linux-wallpaperengine leaves cursor tracking unavailable on wlr-layer-shell
   background surfaces, while CWE either uses focused Wayland surface events or
   a Hyprland-specific `j/cursorpos` socket path and may set a large input
   region that can intercept desktop clicks. Gilder keeps
   `cursor-parallax-input-source` as an explicit unsupported boundary unless a
   real desktop cursor source is supplied, because this only affects a minority
   of mouse-reactive wallpapers and must not compromise normal Linux desktop
   input.
   Scene runtime sampling now carries `scene_input_properties` from package
   manifest defaults, output/user property state, and native controller input
   aliases into every sampled frame. Bound user properties no longer apply only
   to the initial scene plan; `scene.input.controller.<id>.active` and
   target-node aliases resolve to the lowered `scene.controller.<id>.active`
   properties used by native controller bindings.
   Visible scene present results now include `runtime.full_scene`, with
   `target_runtime=native-vulkan-full-scene`,
   `current_runtime=native-vulkan-scene-runtime`,
   `native_present_route_ready`,
   `retained_resource_model_ready`, `timeline_snapshot_runtime_ready`,
   `timeline_animation_runtime_ready`, fixed-topology timeline geometry is
   resampled on each native present frame, `timeline_animation_count`,
   `timeline_animated_layer_count`, `property_update_runtime_ready`,
   `property_binding_count`, `cursor_parallax_input_ready`,
   `pause_resume_policy_ready`,
   `package_state_persistence_ready`, `scene_state_persistence_model`,
   `source_layer_count`, flattened draw counts, per-feature layer counts,
   completed boundaries, pending boundaries, and `runtime_display_available`
   instead of the old preview/fallback availability field. A single scene
   `video` layer
   now routes through the same Vulkanalia ready-prefix Vulkan Video presenter
   used by direct video wallpapers and reports
   `video-layer-vulkan-video-scene-bridge-ready`. A leading full-screen color
   scene layer plus one video layer now routes through the same presenter as
   `clear-background-video-layer-vulkan-video-scene-bridge-ready`; the dynamic
   rendering attachment clear color is carried in each decoded-image draw
   snapshot. Single-video scenes can also carry sampled-image and solid scene
   overlay resources into the decoded-image dynamic rendering pass, where the
   video plane draw is followed by `cmd_draw_scene_overlay_inside_video_rendering`
   before present. Scene conversion now distinguishes real mixed-video
   composition from script-controlled video switching: if only one video layer
   is initially visible, the converted scene records
   `initial-visible-video-scene-composition` and keeps the Vulkan Video scene
   path active for the initial state; multiple simultaneously visible video
   layers still remain explicitly pending under `mixed-video-scene-composition`.
   Current real Workshop scene conversion sample: Steam Workshop item
   `3726503096` (`Beneath The Seventh`) is tagged `3840 x 2160` by Workshop,
   but the package's WE scene/model frame is `2160x1440` and its material
   texture is a `6480x5760` atlas (`3x4`, 12 frames). The native conversion at
   `/tmp/gilder-we-3726503096-output-bc7` now starts from the original
   Workshop directory, parses `scene.pkg` `PKGV0023` directly, and writes
   `assets/scene-resources/scene/resource-4-img-5944-atlas.gtex` as the native
   BC7 sampled-image atlas resource. The gscene node `resource` is
   `resource-4-img-5944-atlas`; `properties.spritesheet` records atlas
   `6480x5760`, frame `2160x1440`, `columns=3`, `rows=4`,
   `frame_count=12`, `fps=12.0`, and `loop=true`; the original `.tex` remains
   as `resource-3-img-5944` provenance. The conversion report records
   `scene-we-package-import`, `scene-we-tex-bc7-gpu-texture`, and
   `scene-we-spritesheet-atlas-runtime`, so this sample now uses the completed
   atlas runtime path. The 6 second native scene smoke without fixed
   `--scene-time-ms`
   `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 target/debug/gilder-native-vulkan --run-scene --output-name HDMI-A-1 --source /tmp/gilder-we-3726503096-output-bc7/assets/scene.gscene.json --scene-root /tmp/gilder-we-3726503096-output-bc7 --fit cover --duration 6 --target-fps 60`
   presents through `scene_present_route=sampled-image`, `frames_presented=360`,
   `average_present_fps=59.9992342697725`,
   `draw_pass_backend_status=clear-background-sampled-image-recording-ready`,
   `draw_pass_background_clear_color=#b3b3b3`, `texture_region.frame_count=12`,
   `native_runtime_coverage_percent=100`,
   `scene_resource_model=retained-sampled-images-descriptor-heap`,
   `uses_host_image_copy=false`, `staging_buffer_bytes=131072`, and geometry
   `source_label=scene-runtime-sampled-image-draw-plan+scene-viewport-fit`,
   `vertex_buffer_count=2`, and
   `upload_model=persistently mapped per-frame host-visible sampled-image vertex buffers reused by frame slot`.
   Repeating the same direct gscene smoke after rebuild with
   `GILDER_CURSOR_PARALLAX=HDMI-A-1:0.4,-0.2` reports
   `frames_presented=360`, `average_present_fps=59.99940346593094`,
   `cursor_parallax_input_ready=true`, completed
   `cursor-parallax-input-source`, no pending cursor boundary, and
   `vertex_buffer_count=2`.
   The release 6s/60fps process snapshot for the same sampled-image atlas
   after the direct dynamic sampled-geometry, bounded present-id telemetry,
   and animated-atlas UV patching changes
   (`/tmp/gilder-scene-uv-patch-atlas-perf`) reports
   `frames_presented=360`, `average_present_fps=59.99916869151805`,
   `max_pss_dirty_kib=18332`, `max_private_dirty_kib=18324`,
   `avg_cpu_percent=11.35`, heap dirty `2720 KiB`, file-mapping dirty `0 KiB`,
   gilder-binary dirty `108 KiB`, `max_nvidia_process_gpu_memory_mib=78`,
   `retained_frame_telemetry_limit=0`, empty
   `present_ids_head`/`present_ids_tail`,
   `scene_resource_model=retained-sampled-images-descriptor-heap`, and
   `vertex_buffer_count=2`.
   Additional real Workshop full-scene stress sample: Steam Workshop item
   `3724575699`
   (`Arknights-萌萌香/遥-常世之幻【可交互/待机动画】`) can be downloaded
   with the cached Steam user `wykszsd0`. It is a standard packaged scene, not
   a loose source tree: the Workshop directory contains `project.json`,
   `preview.gif`, and a `302328013` byte `scene.pkg`. Current-source
   conversion to `/tmp/gilder-we-3724575699-output` writes a first-class
   `assets/scene.gscene.json` entry with no injected `max_fps`, imports
   `scene.pkg` `PKGV0023` with 34 entries, extracts the three large
   3840x2160 WE material `.tex` video payloads into native `.mp4` scene
   resources (`resource-3-2-video.mp4`, `resource-6-1-video.mp4`, and
   `resource-9-f4-video.mp4`), emits no runtime `.tex` resource files, and
   records `scene-we-opacity-effect-timeline`,
   `scene-we-tex-video-layer-runtime`, `scene-we-material-graph-runtime`,
   `wallpaper-engine-util-model-lowering`,
   `scene-we-noop-effect-preserved`, `audio-policy`, and ready native
   particles. The preset mouse-trail particle material now resolves its WE
   built-in `particle/bubbles/bubble3` texture to
   `resource-14-we-builtin-bubble3.gtex`, records
   `wallpaper-engine-builtin-particle-texture`, and completes
   `scene-we-particle-material-runtime` without a missing-resource boundary.
   The converted scene recognizes the built-in
   `models/util/*layer.json` references as native utility controllers or
   native viewport-sized utility target rectangles instead of missing
   resources, lowers pure sound objects to first-class `audio` cue nodes, and
   detects no `audio-response` system for this sample. The two WE
   utility controllers with `scriptproperties.targetLayerId` now lower to
   native `SceneControllerIr` first, then to
   native `properties.controller` metadata plus
   `scene.controller.<node>.active -> target opacity` property bindings, so
   the idle and click video targets are hidden by opacity at startup and can be
   revealed by the core property-binding runtime without a JS VM. The renderer
   now resolves native `idle-video-switch` controller properties during both
   initial scene planning and runtime sampler frames, and runtime sampler
   frames retain bound user/output properties plus
   `scene.input.controller.<id>.active` aliases for click/property controller
   state. In this sample the idle `fullscreenlayer` becomes active after its
   `mouse_inactive_sec=70` threshold, while the `composelayer` click controller
   remains inactive until an explicit native scene input property or real
   pointer-event source activates it. The `入场云雾` control layer with
   `scriptproperties.targetLayerName = "云"` now lowers as a deterministic
   timed-visibility controller: the target `云` fullscreen layer is a
   3840x2160 native rectangle, starts at opacity `0`, fades to `1` at `610ms`,
   stays visible until `37610ms`, and fades out by `38220ms` through a native
   gscene timeline. The reconversion records
   `native-scene-controller-idle-input-source` and
   `native-scene-controller-external-input-source-required` as IR-derived
   conversion features, records `scene-we-timed-visibility-controller` and
   `native-scene-controller-timed-visibility`, moves
   `scene-idle-controller-input-source` under completed boundaries, records
   `wallpaper-engine-timed-visibility-controller-lowering`, and records
   `scene-controller-fade-ramp-runtime` because the idle controller's
   `fadeInDuration=0.77999997` plus the timed visibility fade are sampled
   natively,
   and records `scene-controller-input-source` as an unsupported boundary for
   live click/property event-source wiring rather than the internal
   property-binding runtime. The
   `Clock`, `Date`, and `D a y` text layers now lower their deterministic WE
   `new Date()` scripts to native `text_binding` properties:
   `scene.clock.local.time.hm24`,
   `scene.clock.local.we-date.vertical-month-abbrev`, and
   `scene.clock.local.we-day.vertical-weekday-abbrev-upper`; runtime snapshots
   resolve live text from the sampler rather than freezing the imported
   fallback `value`. The standby voice script and music-selection script now
   lower to native scene audio controllers: standby voice cues activate only
   when the idle video controller is active and `bbrstandbyvoiceswitch` is
   truthy, while the music cues activate from the `music` choice property via
   gscene `audio[].active_conditions`. This records
   `scene-audio-controller-runtime` and
   `wallpaper-engine-detected-scenescript-native-lowering`; the sample's
   6 s native runtime smoke at
   `/tmp/gilder-we-3724575699-native-ready-clean/assets/scene.gscene.json`
   now reports `scene_present_route=video`,
   `draw_pass_backend_status=video-layer-vulkan-video-scene-bridge-ready`,
   `full_scene_complete=true`, `native_runtime_coverage_percent=100`, empty
   pending boundaries, `present_backend=vulkanalia-decoded-image-dynamic-rendering-present`,
   no H.264 present error, `presented_frame_count=360`,
   `decoded_image_zero_copy_presented=true`, and latest draw command order
   containing `cmd_draw_scene_overlay_inside_video_rendering` after the video
   plane draw. This is native Vulkan Video plus scene overlay composition, not
   a clear placeholder or CPU/video fallback.
   `systems.scenescript` status is now `ready` because every detected source
   script has a native lowering.
   Effect
   metadata is explicit: the three visible `blurprecise` text effects now lower
   to `runtime: "native-text-glow"` without copied runtime effect resources,
   the opacity fade is `runtime: "native-opacity-timeline"`, and the
   default-hidden clouds effect is `runtime: "metadata-only"`. Current-source
   reconversion reports `full_scene_complete=true`,
   `progress_estimate_percent=100`, and empty `pending_boundaries`; the only
   explicit unsupported native-lowering boundaries are
   `cursor-parallax-input-source` and `scene-controller-input-source`. The
   conversion report's `unsupported_features` list is now limited to
   `cursor-parallax-input-source`; WE `scenetexture` user properties are
   retained as manifest text metadata instead of being reported as runtime
   blockers. The sample is useful because it
   combines video-texture scene layers, multiple MP3 cue layers, mouse trail
   particle controls,
   standby/interactive SceneScript that targets other layers and video texture
   play state, native-lowered clock/date text, script-controlled audio cue
   activation, visible WE effect passes for blur/clouds, and a
   native-lowered opacity fade effect. Current explicit gaps for this sample
   are not compatibility fallbacks: live click/property event-source wiring is
   unsupported beyond the state-property input bridge, and compositor cursor
   input remains unsupported when no real desktop cursor source is supplied.
   The runtime now carries the gscene document size (`2160x1440`) into the
   sampled-image present path and applies scene-level `cover` viewport mapping
   before recording geometry for the actual swapchain extent (`2561x1601` in
   this smoke), so scene-space coordinates are centered/cropped instead of
   being interpreted as output pixels. Spritesheet draw steps retain `columns`,
   `rows`, `fps`, and `loop_playback`; animated atlas regions now use a
   per-frame host-visible vertex-buffer ring and update only the current frame
   slot's UV bytes from runtime elapsed time after that slot fence has
   completed; the present resources no longer retain a full CPU-side
   `base_vertices` copy for static-topology animated atlases.
   Atlas-only scenes with no explicit timeline channels are also marked as
   time-sampled native scene state when any layer carries an animated
   `texture_region`, so sampled-image spritesheet wallpapers do not freeze on
   the initial frame.
   Current dynamic timeline scene smoke after rebuilding the native CLI:
   `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 target/debug/gilder-native-vulkan --json --run-scene --output-name HDMI-A-1 --source /tmp/gilder-dynamic-timeline.gscene.json --fit cover --duration 2 --target-fps 60`
   presents through `scene_present_route=solid-quad`, `frames_presented=120`,
   `average_present_fps=59.99628407014983`,
   completed
   `per-frame-timeline-geometry-runtime`,
   `timeline_animation_count=2`, `timeline_animated_layer_count=1`,
   `present.geometry.upload_model=persistently mapped per-frame host-visible solid-quad vertex buffers reused by frame slot`,
   `vertex_buffer_bytes=96`,
   `index_count=6`, and `descriptor_set_count=0`. Fixed-topology timeline
   transform/opacity changes are resampled from the source gscene at native
   present elapsed time and written into the current frame slot's vertex buffer;
   layer/resource topology changes still remain explicit unsupported scene
   graph changes instead of being silently patched into an old path.
   The current release 6s/240fps process snapshot for this same dynamic
   timeline scene after the direct-mapped vertex write change
   (`/tmp/gilder-scene-direct-write-timeline-perf-4`) on `2026-06-28`
   presented `1440` frames at `239.99710187499682` fps with
   `max_pss_dirty_kib=18426`, `max_private_dirty_kib=18424`,
   `avg_cpu_percent=8.05`, and `max_nvidia_process_gpu_memory_mib=40`; mapping
   categories show heap dirty at `2656 KiB`, file-mapping dirty at `12 KiB`,
   and the remaining dirty memory in driver/system mappings.
   Current native particle scene smoke:
   `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 target/debug/gilder-native-vulkan --json --run-scene --output-name HDMI-A-1 --source /tmp/gilder-particles.gscene.json --fit cover --duration 2 --target-fps 60`
   presents through `scene_present_route=solid-quad`, `frames_presented=120`,
   `average_present_fps=59.99593587530381`,
   `runtime.full_scene.scene_particle_system_ready=true`,
   `particle_runtime_layer_count=96`, completed
   `native-particle-system-runtime`, `unsupported_layer_count=0`,
   `present.geometry.upload_model=persistently mapped per-frame host-visible solid-quad vertex buffers reused by frame slot`,
   `vertex_buffer_bytes=112896`, `index_count=13824`, and
   `descriptor_set_count=0`. Particle systems are source-backed time-sampled
   scene state, so native present enables the same dynamic geometry sampler
   used by timelines even when no explicit timeline channels are present.
   After sampler scratch reuse, direct solid runtime lowering, and executable
   sync-before-self-exec, the rebuilt-release first-run 6s/240fps process
   snapshot for this heavier particle scene
   (`/tmp/gilder-scene-particle-fsync-before-exec-perf`) on `2026-06-28`
   reported `max_pss_dirty_kib=18846`, `max_private_dirty_kib=18844`,
   `avg_cpu_percent=13.13`, heap dirty `3084 KiB`,
   `memory_category_gilder_binary_private_dirty_kib=108`, file-mapping dirty
   `12 KiB`, and `max_nvidia_process_gpu_memory_mib=40`. The same binary rerun
   (`/tmp/gilder-scene-particle-fsync-rerun-perf`) presented `1440` frames at
   `239.87647025440134` fps with `max_pss_dirty_kib=18898`,
   `max_private_dirty_kib=18896`, heap dirty `3104 KiB`, and the same `108 KiB`
   gilder-binary dirty category.
   After explicit fill-rule support, the direct dynamic geometry changes, and
   bounded present-id telemetry, the release 6s/240fps particle smoke
   (`/tmp/gilder-scene-presentid-bounded-particle-perf`) reported `1436`
   frames, `average_present_fps=239.19133556317604`,
   `max_pss_dirty_kib=18826`, `max_private_dirty_kib=18824`, heap dirty
   `3068 KiB`, gilder-binary dirty `104 KiB`, file-mapping dirty `12 KiB`,
   `retained_frame_telemetry_limit=0`, empty
   `present_ids_head`/`present_ids_tail`, and
   `max_nvidia_process_gpu_memory_mib=40`.
   Sized `scene-render-clear-color` layers are treated as render
   clear-background layers in the native draw pass, not as normal geometry, so
   the WE clear color composes with the atlas image without introducing a
   compatibility fallback. The converter must not up-label the generated frame
   to 4K; the 4K tag is a presentation/display label, not the asset frame
   dimensions.
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
   `uses_host_image_copy=false`, `staging_buffer_bytes=131072`,
   `upload_submitted=true`,
   `descriptor_heap.descriptor_model=VK_EXT_descriptor_heap`,
   `uses_present_id2=true`, `present_wait2_available=true`,
   `swapchain.present_id2_enabled=true`, `swapchain.present_wait2_enabled=true`,
   `retained_frame_telemetry_limit=0`, empty `present_ids_head`/`present_ids_tail`,
   no unbounded `present_ids` vector, and no legacy
   `uses_present_id`/`present_wait_available` fields.
   Current curve-path scene smoke after rebuilding the native CLI:
   `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 target/debug/gilder-native-vulkan --json --run-scene --output-name HDMI-A-1 --path-data 'M0 0 C25 80 75 -80 100 0 S175 80 200 0 L200 80 L0 80 Z' --color '#cc8844' --duration 1 --target-fps 30 --fit cover`
   reports `scene_present_route=solid-quad`, `frames_presented=30`,
   `average_present_fps=29.99756458772382`,
   `draw_pass_backend_status=solid-quad-recording-ready`,
   `runtime.full_scene.curve_path_layer_count=1`, completed
   `curve-path-flattening-runtime`, `vertex_count=35`, `index_count=99`,
   and `descriptor_set_count=0`.
   Current arc-path scene smoke after rebuilding the native CLI:
   `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 target/debug/gilder-native-vulkan --json --run-scene --output-name HDMI-A-1 --path-data 'M100 50 A50 50 0 1 1 0 50 A50 50 0 1 1 100 50 Z' --color '#22aa88' --duration 1 --target-fps 30 --fit cover`
   reports `scene_present_route=solid-quad`, `frames_presented=30`,
   `average_present_fps=29.998314994646755`,
   `draw_pass_backend_status=solid-quad-recording-ready`,
   `runtime.full_scene.arc_path_layer_count=1`, completed
   `arc-path-flattening-runtime`, `vertex_count=32`, `index_count=90`,
   and `descriptor_set_count=0`.
   Current compound-path scene smoke after rebuilding the native CLI:
   `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 target/debug/gilder-native-vulkan --json --run-scene --output-name HDMI-A-1 --path-data 'M0 0 L100 0 L100 100 L0 100 Z M25 25 L75 25 L75 75 L25 75 Z' --path-fill-rule evenodd --color '#22aa88' --duration 1 --target-fps 30 --fit cover`
   reports `scene_present_route=solid-quad`, `frames_presented=30`,
   `average_present_fps=29.99702765452377`,
   `draw_pass_backend_status=solid-quad-recording-ready`,
   `runtime.full_scene.compound_path_layer_count=1`, completed
   `compound-path-evenodd-fill-runtime`, `vertex_count=16`,
   `index_count=24`, and `descriptor_set_count=0`.
   The same compound shape with default nonzero winding fill reports completed
   `compound-path-nonzero-fill-runtime`, `vertex_count=12`, `index_count=18`,
   and the same descriptor-heap-free solid geometry route.
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
   Focused regression coverage asserts gscene package validation, clean
   WE scene-to-gscene conversion, WE model/material texture provenance,
   renderable material image texture resource resolution, WE parent
   graph lowering into gscene children, render clear-color snapshot layers,
   WE text wrapper/font resource conversion, visible property binding lowering, WE
   shape/solid/radius lowering into native snapshot nodes, explicit WE
   keyframe timeline lowering and deterministic WE animation-layer keyframe
   plus rate/time-scale lowering into native timeline snapshot values,
   geometry field timeline/property animation, script/value wrapper lowering
   without a JS engine, deterministic numeric SceneScript expression lowering,
   embedded WE property keyframe extraction into gscene timelines,
   bounded opacity effect lowering into native gscene timelines,
   native gscene particle emitter expansion into deterministic solid geometry
   and sampled-image sprite layers when a WE particle material resolves to a
   texture resource, sampled-image source deduplication for repeated sprite
   resources,
   WE `.tex` RGBA/LZ4 first-frame or spritesheet-atlas conversion to native
   BC7 `.gtex` sampled-image resources, authoritative `scene.pkg` direct import,
   and parallax depth property-camera
   offsets, timeline animation metadata reaches `SceneWallpaperPlan`,
   source-backed `SceneWallpaperRuntimeSampler` resamples timeline layers,
   atlas-frame texture regions are time sampled, property binding counts reach
   the native runtime, and the completed full-scene boundaries include
   `timeline-animation-runtime`, `per-frame-timeline-geometry-runtime`,
   `property-update-runtime`,
   `pause-resume-policy-runtime`, `package-state-persistence`,
   `native-particle-system-runtime`,
   `scene-we-particle-material-runtime`,
   `wallpaper-engine-scene-pkg-import`,
   `scene-we-embedded-property-timeline`,
   `scene-we-spritesheet-atlas-runtime`, `curve-path-flattening-runtime`,
   `arc-path-flattening-runtime`,
   `compound-path-evenodd-fill-runtime`, and
   `compound-path-nonzero-fill-runtime`; cursor scene coverage asserts both
   explicit `cursor-parallax-input-source` completion when a real source exists
   and unsupported-boundary reporting when it does not, and native
   idle utility controllers now assert
   `scene-idle-controller-input-source` plus
   `scene-controller-fade-ramp-runtime` completion, and deterministic timed
   visibility controller conversion asserts target-node opacity timelines plus
   runtime snapshot sampling at reveal/fade/hide time points. Deterministic WE
   clock/date text conversion asserts native `text_binding` output and runtime
   snapshot text replacement through the scene text resolver. Native audio
   controller conversion asserts gscene `audio[].active_conditions`, snapshot
   filtering of inactive cues, manifest/render property defaults for conditional
   audio, and completion of
   `wallpaper-engine-detected-scenescript-native-lowering` when all detected
   source scripts lower to native IR. Renderer coverage now also
   asserts that manifest/output property values and
   `scene.input.controller.<id>.active` aliases are retained by
   `SceneWallpaperRuntimeSampler` frames.
   `native_runtime_coverage_percent=100` means the currently completed native
   runtime boundaries have no pending native layers; current 3724575699
   conversion also reports full native-scene completion with only explicit
   Linux input-source unsupported boundaries. Focused conversion coverage for
   3724575699 confirms no `.tex`
   runtime files, no util or built-in particle missing-resource warnings, no
   audio-response system,
   initial-visible video scene composition completed, native controller
   property bindings for idle/click video switching, completed native idle
   controller input sampling, native idle fade-ramp sampling, deterministic
   timed visibility controller lowering for the `云` fullscreen target,
   deterministic clock/date text lowering for `Clock`, `Date`, and `D a y`,
   native audio cue activation for standby voice/music selection scripts, and
   native blurprecise text glow lowering. Remaining expansion surfaces are live
   click/property event sources beyond the state-property input bridge,
   font atlas shaping/rasterization beyond deterministic built-in glyph
   geometry, broader WE animation-layer
   blend/weight semantics, executable shader/effect material graphs for effects
   that cannot lower to native IR, real PipeWire spectrum/FFT audio-response
   input, and multi-video overlay composition for scenes that show more than
   one video layer at the same time.
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
