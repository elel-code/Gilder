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
- bitstream upload 对齐 FFmpeg 的按图像生命周期:保留单个持久映射
  `VIDEO_DECODE_SRC_KHR` buffer，初始为 FFmpeg 风格单 picture 下限，payload 超过当前容量时
  才 grow；不再按 decode window 或 `bitstream_samples` 常驻分配。
- H.264/H.265 streaming 从 GStreamer `Buffer` 直接持有 readable mapped payload 到 per-frame
  decode input，上传后立即释放；不再把 parser payload 复制进 retained `Vec<u8>` window。
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

4K/240 的有效验证应使用仓库内真实源，关注:

- `decoded_count == presented_count == requested_playback_frames`
- `bad_frames == 0`
- `descriptor_sets == 0`
- `decode.bitstream_buffer_model == "streaming-persistent-mapped-reused-upload-buffer"`
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
  cargo run --features native-vulkan-gst-video --bin gilder-native-vulkan -- \
  --run-vulkanalia-ready-prefix-video ...
```

注意 validation 输出会污染 stdout JSON，脚本或手工解析时从第一个 `{` 开始取 JSON。

仍需清理但不能用 descriptor set 规避的问题:

- `VUID-vkCreateDevice-ppEnabledExtensionNames-01387`
- `VUID-vkWriteSamplerDescriptorsEXT-pSamplers-11204`

`11204` 的方向是清理 descriptor-heap sampler 写入路径；YCbCr sampler 仍应走
descriptor heap / embedded immutable sampler 语义。

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
