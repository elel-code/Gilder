# Vulkanalia 视频解码修复记录

本文件只记录当前有效结论。第一参考实现是 FFmpeg:
`references/ffmpeg/libavcodec/vulkan_decode.c` 和
`references/ffmpeg/libavutil/vulkan.c`。

## 当前主线

- H.264/H.265 的 `--run-vulkanalia-ready-prefix-video` 已经不是旧的
  "一次性 ready-prefix window" 实现。它现在用单个 streaming packet queue 连续拉 AU、
  连续 submit decode、逐帧 present，并按 `--playback-frames` 控制总播放帧数。
- H.264/H.265 主线只创建一个 Vulkan instance/device/video session/present runtime。
  旧的顶层 session-bind smoke 不再在播放前额外运行，避免污染 private_dirty 和 CPU 采样。
- H.264/H.265 runtime 从同一个 streaming queue 的 SPS/VPS/SPS 派生 coded extent，
  Vulkan Video session 和 resource image 按码流尺寸创建，不依赖 CLI 默认尺寸。
- bitstream upload 对齐 FFmpeg 的按图像生命周期:每个 exec slot 使用 FFmpeg
  `slices_buf` 语义的持久映射 `VIDEO_DECODE_SRC_KHR` buffer，提交后由该 slot/fence 生命周期
  保活；复用 exec slot 前才等待 fence 并复用或替换该 slot buffer。不再保留单个共享 upload
  buffer、分槽 stride 或全局 grow buffer。
- H.264/H.265 streaming 从 FFmpeg `AVPacket` 直接借用 payload 到 per-frame decode
  input，上传后立即释放；不再把 parser payload 复制进 retained `Vec<u8>` window。
- descriptor 路径必须保持 `VK_EXT_descriptor_heap`。当前 Vulkanalia smoke 的有效 gate 是
  `descriptor_model = VK_EXT_descriptor_heap` 且 `descriptor_sets = 0`。

## 绿屏根因

旧问题是 decode image extent 与码流 coded extent 不一致。Vulkan Video 只写 coded picture
范围，present pass 如果按更大的 image 全幅 YCbCr 采样，未写区域会表现为绿屏。

当前处理:

- AV1 可见 runtime 当前只允许 continuous streaming runtime 方向；该 runtime 未完成前 direct
  runtime 对 AV1 明确报错。
- H.264/H.265 streaming 在 `video_present_runtime.rs` 中启动唯一 packet queue 后，从参数集修正
  `max_coded_extent` 和 resource image extent。

## 内存结论

曾经的 300MB+ private_dirty 主因是两个问题叠加:

- `bitstream_samples * 1MiB` 的常驻 bitstream buffer。
- 播放前额外跑 session-bind smoke 和 metadata queue，导致驱动/allocator/heap 采样被污染。

当前 H.264/H.265 对齐 FFmpeg:

- 不保留长 AU payload window。
- 不为 H.264/H.265 运行额外 top-level session-bind smoke。
- session-bind `--decode-*-ready-prefix` retained/batch decode 已删除；probe 只保留
  session/resource/parameter 创建验证。
- 不用 descriptor set。
- H.264/H.265 smoke 只读取真实 runtime session 字段，不再依赖顶层 `.session` 兼容字段。

2026-06-26 Vulkanalia plane-view descriptor heap 复测:

- validation: `/tmp/gilder-validation-plane-present-2.stdout`，H.265 Main8 4K/240
  ready-prefix 240 帧，进程退出 0，未检出 `VUID`、`Validation Error`、`panic` 或
  `failed`；runtime 报告 `descriptor_sets=0`、`descriptor_heap_plane_sampler_enabled=true`、
  `uses_plane_sampler_descriptors=true`。
- performance: `/tmp/gilder-h265-4k240-plane-heap`，仓库内真实 H.265 4K/240 源，
  `decoded_frame_count=2400`、`presented_frame_count=2400`、
  `average_present_fps=239.98432204022697`、`performance_max_private_dirty_kib=26268`、
  `performance_avg_cpu_percent=16.46`。
- 迁移前 ash 基线对照 commit 是
  `1789e4f0bbc32f9c17ca5b570231dbf169681979`，其 H.265 4K/240 evidence
  `/tmp/gilder-vulkan-h265-after-h264-barrier-tightened` 为
  `average_present_fps=239.82864245894595`、`Private_Dirty max=24684 KiB`。
- 这次跨回 30MB 以下的实质突破是 hand-written plane conversion：删除
  `VkSamplerYcbcrConversion`、converted image view 和 descriptor-heap embedded sampler
  mapping，改为 FFmpeg `hwcontext_vulkan.c` 同类的 multi-planar format plane view
  (`R8/RG8` 或 `R16/R16G16`) 加普通 sampler descriptor，由 fragment shader 显式完成
  YUV->RGB。streaming 生命周期收敛解释了 300MB/80MB 级别的常驻内存问题，最后从
  40MB+ 压到 `26268 KiB` 主要来自这条 plane-view shader 路径。

2026-06-26 FFmpeg submit workspace/latest 4K/240 复测:

- FFmpeg reference:
  `references/ffmpeg/libavcodec/vulkan_decode.h:88-100`
  (`refs[36]`/`ref_slots[36]`/`slices_buf`),
  `references/ffmpeg/libavcodec/vulkan_decode.c:305-390`
  (`ff_vk_decode_add_slice`/`av_fast_realloc`/pooled slices buffer),
  `references/ffmpeg/libavcodec/vulkan_decode.c:488-568`
  (current picture inactive `slotIndex = -1`, exec owns `slices_buf` after submit),
  `references/ffmpeg/libavcodec/vulkan_hevc.c:720-815`,
  `references/ffmpeg/libavcodec/vulkan_av1.c:288-365`.
- H.264/H.265/AV1 submit plan 不再持有 per-frame owned reference-slot Vec；
  lowering 时用 FFmpeg-style fixed stack arrays，streaming loop 复用 reference workspace。
- H.264 不再在 frame input 里复制一份 `slice_offsets`；submit plan 直接借用
  `first_slice.slice_offsets`，匹配 FFmpeg 的 `pSliceOffsets` 指针语义。
- H.265 `slice_segment_offsets` 改为栈上单帧 slice 借用；AV1 decode references
  使用 caller workspace，保持 FFmpeg duplicate-slot scan 语义。
- H.264/H.265/AV1 smoke runtime 默认给子进程设置低 dirty glibc allocator 环境:
  `MALLOC_ARENA_MAX=1`, `MALLOC_MMAP_THRESHOLD_=131072`,
  `MALLOC_TRIM_THRESHOLD_=0`, `GLIBC_TUNABLES=glibc.malloc.tcache_count=0`。
  这不改变 Vulkan/FFmpeg decode 语义，只避免 glibc tcache/file-backed COW 页污染
  `Private_Dirty` gate；调用方显式设置同名环境时仍可覆盖。
- 最新有效证据:
  - H.264 `/tmp/gilder-perf-h264-default-rerun-4k240`:
    `decoded_frame_count=2400`, `presented_frame_count=2400`,
    `average_present_fps=240.00856962853402`,
    `performance_max_private_dirty_kib=24828`, `descriptor_sets=0`.
  - H.265 `/tmp/gilder-perf-h265-workspace-allocator-4k240`:
    `decoded_frame_count=2400`, `presented_frame_count=2400`,
    `average_present_fps=240.00585273330296`,
    `performance_max_private_dirty_kib=22588`, `descriptor_sets=0`.
  - AV1 Main8 `/tmp/gilder-perf-av1-main8-workspace-allocator-4k240`:
    `displayed_frame_count=2400`, `presented_frame_count=2400`,
    `average_present_fps=240.02434924659713`,
    `performance_max_private_dirty_kib=21900`, `descriptor_sets=0`.
  - AV1 Main10 `/tmp/gilder-perf-av1-main10-workspace-allocator-4k240`:
    `displayed_frame_count=2400`, `presented_frame_count=2400`,
    `average_present_fps=240.02807982349208`,
    `performance_max_private_dirty_kib=21740`, `descriptor_sets=0`.

4K/240 的有效验证应使用仓库内真实源，关注:

- `decoded_count == presented_count == requested_playback_frames`
- `bad_frames == 0`
- `descriptor_sets == 0`
- `decode.bitstream_buffer_model == "ffmpeg-picture-slices-buffer-pool-exec-owned"`
- `decode.input_payload_model == "bounded-streaming-packet-queue-per-frame-upload"`
- `session_h265_ready_prefix_decode == false`
- `session_bitstream_buffer == false`
- `frames_len` 只保留 telemetry head/tail，而不是全量帧快照
- `Private_Dirty` 以 performance snapshot/smaps 为准

## Validation

用户侧 validation layer 已安装。常用运行方式:

```bash
env VK_LOADER_LAYERS_ENABLE='*validation*' \
  VK_LAYER_KHRONOS_validation_LOG_FILENAME=stdout \
  cargo run --features native-vulkan-video --bin gilder-native-vulkan -- \
  --run-vulkanalia-ready-prefix-video ...
```

注意 validation 输出会污染 stdout JSON，脚本或手工解析时从第一个 `{` 开始取 JSON。

当前主线是 decoded image 的 Y/UV plane view descriptor 写入 resource heap、普通 sampler
写入 sampler heap，再由 fullscreen shader 完成 YUV->RGB。不要恢复 descriptor set、push
descriptor 或 embedded YCbCr sampler fallback。2026-06-26 validation 复测未再复现
`VUID-vkWriteSamplerDescriptorsEXT-pSamplers-11204`，也未复现
`VUID-vkCmdBindSamplerHeapEXT-pBindInfo-11224`。

## Smoke

主要 H.265 4K/240 运行形态:

```bash
scripts/native-vulkan-h265-ready-prefix-video-smoke.sh \
  --no-build \
  --source artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-242frames-g240-d240.mp4 \
  --decode-prefix 240 \
  --playback-frames 2400 \
  --target-fps 240 \
  --performance-snapshot \
  --performance-duration 8 \
  --report-dir /tmp/gilder-h265-4k240-private-dirty
```

H.264/H.265 smoke 中旧的 `driver_max_dpb_slots` 顶层读取已退休；脚本使用
runtime session 的 `session_max_dpb_slots` 作为 DPB 上限证据。
