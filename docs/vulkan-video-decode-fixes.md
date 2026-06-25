# Vulkanalia 视频解码:绿屏 / 卡死 / 内存 调查与修复记录

本文件记录 native Vulkan(vulkanalia)视频路径在排查"video 类型严重绿屏然后卡死"过程中
定位到的根因、已落地的修复、以及尚未解决的问题。**第一参考实现是 FFmpeg**
(`references/ffmpeg/libavcodec/vulkan_decode.c`、`references/ffmpeg/libavutil/vulkan.c`)。

## 背景架构（调查中确认的事实）

- 真实壁纸守护进程 `gilderd` 目前走 **GStreamer** 路径(`src/renderer/video.rs`),
  把解码帧拷到主机内存(appsink)。这是**非 zero-copy** 路径。
- native Vulkan 解码路径(`vulkanalia_backend/`)是**正在迁移的替代实现**,目标是
  zero-copy(解码 + 采样都在 VRAM)。目前**只能通过 CLI 测试**:
  `gilder-native-vulkan --run-vulkanalia-ready-prefix-video ...`。
- `--run-vulkanalia-ready-prefix-video` 是"就绪前缀"演示模式:解码固定 N 帧 →
  逐帧呈现一遍 → 结束。它**不是连续播放器**。
- 本机环境:NVIDIA 独显;video 与 present **同一队列族**;解码图像 **CONCURRENT** 共享
  (跨队列族时返回 `[video, present]`,因此无需所有权转移)。
- `VK_EXT_descriptor_heap` 是 vulkanalia 0.35 已支持的真实(较新)扩展;本路径用它做
  resource heap / sampler heap 绑定与 embedded immutable sampler 映射。

## 如何复现 / 测试

```fish
cargo build --features native-vulkan-gst-video --bin gilder-native-vulkan

# H.264 640x368(已验证修复)
./target/debug/gilder-native-vulkan --run-vulkanalia-ready-prefix-video \
  --source artifacts/video-sources/h264/h264-high-b0-ref2-weightp0-weightb0-640x368-24fps-10frames-g24-d8.mp4 \
  --video-codec h264 --decode-h264-ready-prefix 8 --duration 6
```

- `--video-codec`:`h264` / `h265` / `h265-main-10` / `av1` / `av1-main-10`(须与源匹配)
- `--decode-XXX-ready-prefix N`:解码 N 帧(N 越大,突发呈现的运动越长)
- `--width/--height`:**不再需要**,已自动按视频分辨率创建(见下文修复)
- 开 validation(输出在 stdout):命令前加
  `env VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation`
  - 需安装 `vulkan-validation-layers`(Arch/CachyOS:`sudo pacman -S vulkan-validation-layers`)
  - 注意:`VK_LOADER_LAYERS_ENABLE='*validation*'` 在本机未生效,用 `VK_INSTANCE_LAYERS` 才强制启用

## 已修复

### 1. 卡死 —— decode→present 缺跨队列同步(已提交 `95d7451`)

**根因**:present(graphics/present 队列)在 video 队列解码写完图像之前就采样,
present submit 只等 `image_available`,不等任何 decode 信号的 semaphore。

**修复**(FFmpeg 的 `vp->sem` / `sem_value` 模型):
- 新增持久 **timeline semaphore** `decode_complete` 于
  `VulkanaliaDecodedImagePresentFrameResources`。
- 解码每帧 `submit_decode_command_buffer2` **signal** 一个单调递增值;
  present submit **wait** 该值(`ALL_COMMANDS`,以同时 gate 布局转换 barrier)。
- 值经 `after_frame_submitted` 回调从 decode 透传到 present;计数器用
  `vkGetSemaphoreCounterValue` 从持久 semaphore 取基值,跨序列严格递增
  (timeline 天然容忍"解码但跳过呈现",这是没用 binary semaphore 的原因)。
- present barrier 里非法的 `srcStageMask = VIDEO_DECODE_KHR`(graphics 队列不支持)
  改为 `NONE`,跨队列依赖交给 semaphore。

涉及:`render_present.rs`、`video_decode_commands.rs`、`video_session_bind.rs`、
`video_present_runtime.rs`。

### 2. 绿屏 —— 解码图像 extent 不匹配(未提交)

**根因(可见绿屏的真正主因)**:解码图像按 CLI 默认 **3840×2160** 创建,而视频实际可能
是 640×368。Vulkan video 按 coded 尺寸写入,解码只填了图像左上角一小块,present 按
UV 0..1 采样整张图 → 绝大部分是**未解码区域**,YCbCr 转换后呈**绿色**;只有角落一小块
是真实视频(在动)。这与 descriptor-heap / YCbCr 无关,改动前后都会绿。

**证据**:手动 `--width 640 --height 368` 后画面正常;运行报告里
`requested_extent = [3840,2160]` 而源是 640×368。

**修复**:`run_vulkanalia_ready_prefix_video`(`vulkanalia_direct.rs`)新增
`native_vulkan_vulkanalia_ready_prefix_source_extent`,从已解析的参数集取视频 coded
分辨率(H264/H265 `parameter_sets.sps.width/height`,AV1
`sequence_header.max_frame_width/height`,对齐到 16),覆盖 CLI 默认。
对齐 FFmpeg:DPB 按码流尺寸而非显示面创建。

### 3. Validation 错误(借 validation 定位,部分已修)

| VUID | 含义 | 处理 |
|---|---|---|
| `VkImageViewCreateInfo-format-06415` (28×) | 多平面 NV12 + SAMPLED 图像上的解码视图缺 YCbCr conversion | **已修**:解码 DPB/DST 视图剔除 SAMPLED usage(只留 video-decode usage),即不再需要 conversion。对齐 FFmpeg(解码视图用 `VK_IMAGE_USAGE_VIDEO_DECODE_*`)。`video_session_images.rs` |
| `VkGraphicsPipelineCreateInfo-flags-11311` (2×) | descriptor-heap pipeline 的 `layout` 必须为 `VK_NULL_HANDLE` | **已修**:`render_present.rs` pipeline 创建传 `vk::PipelineLayout::null()`(绑定来自 pushed mapping,而非 layout) |
| `vkCmdBindResourceHeapEXT-pBindInfo-11233` (16×) | 资源堆绑定的 `reservedRangeSize=0` < `minResourceHeapReservedRange(96768)` | **已修**:捕获 `min_resource_heap_reserved_range`,把 reserved range 置于描述符区之后并扩大堆,bind 时设置 `reserved_range_offset/size`。`descriptor_heap.rs`、`features.rs` |
| `vkWriteSamplerDescriptorsEXT-pSamplers-11204` (16×) | 带 YCbCr conversion 的 sampler 不允许写进 sampler heap | **未修**:embedded 路径其实不绑定 sampler heap,该写入是冗余非法但不影响画面;待清理 |
| `vkQueueSubmit2-semaphore-03868` (4×) | swapchain semaphore 复用(应每个 swapchain image 一个) | **未修**:既有问题,与绿屏无关 |
| `vkCreateDevice-ppEnabledExtensionNames-01387` (4×) | 启用 `VK_KHR_swapchain_maintenance1` 但缺实例扩展 `VK_KHR_surface_maintenance1` | **未修**:既有问题 |

### 4. present sampler 方向纠正(已回退实验)

用户曾尝试把 present 的 YCbCr sampler 从 embedded immutable 改为独立 sampler heap
(现象:绿 → 红)。该方向**错误**:YCbCr conversion sampler 在 descriptor-heap 模型下
**必须**作为 embedded/immutable 提供(对齐 FFmpeg `pImmutableSamplers`,
`libavutil/vulkan.c`)。已回退到 embedded 路径。

## 内存调查结论(private_dirty / zero-copy)

实测(NVIDIA 独显,`/proc/PID/smaps_rollup` + `nvidia-smi`):

- native Vulkan 路径 **Private_Dirty ≈ 23–27MB**,在 640×368 与真 4K、8 帧与 16 帧之间
  **几乎不变** —— 主机侧只有 bitstream/解析结构,与分辨率无关。**这条路径本身已是 zero-copy**,
  解码图像在 VRAM(不计入 RSS/private_dirty)。
- RSS ≈ 119MB,大头是不可避免的共享驱动库(`libnvidia-gpucomp.so` ~42MB、
  `libnvidia-glcore.so` ~19MB)+ `[heap]` ~12–16MB + `/dev/nvidiactl` 等。
- `nvidia-smi --query-compute-apps used_gpu_memory` 报 58MB 且不随分辨率变化,
  对 graphics/video 显存**不可靠**,不能作为依据。

**推断**:用户观察到的"private_dirty 将近 100MB"极可能来自 **GStreamer 路径**
(gilderd,解码帧拷到主机,非 zero-copy,随分辨率增长),而非这条 Vulkan 路径。
迁移到 Vulkan 路径本身即可达成 ~20MB 的 zero-copy 目标。**待用户确认其测量来源**
(工具 / 二进制 / 分辨率)。

## 未解决 / 下一步

### A. "正常画面很短然后全黑"(连续播放)

**现状**:ready-prefix 模式解码固定 N 帧 → 突发逐帧呈现(**真实运动**,按 PTS 节奏)→
放完即结束 → 函数返回 → runtime drop → 壁纸表面销毁 → **黑**。

**已排除的错误方向**:曾加"重放已解码 DPB 层"的循环来填满播放时长 —— 但 H.264 `ref2`
流只有约 **3 个 DPB 槽**(layers 0,1,2 轮流复用),解码完只保留最后 ~3 帧,循环重放只能
看到 3 帧 → **几乎不动**;且把数百个 draw 推进 sequence_builder 导致 private_dirty 涨到
40MB+。该循环**已回退**。

**正解(FFmpeg 对齐,尚未实现)**:连续循环解码 —— 持续从源解码访问单元、逐帧呈现、
DPB 槽环形复用,到源尾部从头重解,永不停止。这才能既有完整运动又保持低内存。
临时手段:`--decode-h264-ready-prefix N` 调大 N,突发即可呈现 N 帧运动(N 帧后仍黑)。

### B. 解码性能偏慢

实测解码 200 帧耗时约 34s(~170ms/帧),硬件解码不应如此(60fps 需 ~16ms/帧)。
疑似与逐帧 fence 等待 / bitstream 处理有关,需单独排查。

### C. 清理剩余 validation(11204 / 03868 / 01387)

## 提交状态

- 已提交:`95d7451 Add timeline semaphore for decode-to-present cross-queue sync`
- 未提交(working tree):extent 自动化(`vulkanalia_direct.rs`)、06415/11311/11233 修复
  (`video_session_images.rs`、`render_present.rs`、`descriptor_heap.rs`、`features.rs`)、
  对应测试 mock 字段补全。`cargo test --lib vulkanalia_backend` 通过(132)。
