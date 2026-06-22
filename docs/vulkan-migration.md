# Vulkan 迁移准备路线

本文档记录 2026-06-20 之后的 renderer 方向。当前结论是：4K/240fps video
已经达到可接受的真实 Wayland 稳定基线，短期不继续围绕 active video copy/private
dirty 做底层压榨；下一阶段同时推进壁纸类型扩展和手写 Vulkan renderer。新增
能力必须写成可以被 native Wayland/Vulkan 后端和 helper runtime 共同消费的形状。

## 当前决策

native-wgpu/GStreamer CUDA 路线已经完成了验证使命，但不再作为独立后端继续维护：

- `HDMI-A-1` 真实 Wayland 20s smoke 可稳定贴近 239.999Hz，`frames_skipped=0`。
- video 路径为 `gst-dmabuf` + `cuda-direct-vulkan-images-timeline`。
- CPU 和 `Private_Dirty` 仍有 driver/GStreamer/CUDA runtime floor，但 active video 已可作为
  历史高刷视频对照基线。
- `gpu-video` crate 路线因 codec/container 限制和维护面过窄已退休；后续 video/audio 前端保留
  GStreamer，native Vulkan 后端只消费 GStreamer 产出的 frame/texture handoff，不让
  GStreamer sink 接管显示。
- GTK renderer、native-wgpu renderer、native-wayland `playbin/waylandsink` video helper 和
  vendored `wgpu-hal` 已从可构建技术栈中移除。native Wayland 只保留 surface/output host 和
  linux-dmabuf feedback；显示、import、decode 和 present 由 native Vulkan 承担。

暂时不继续深挖的点：

- 不继续尝试把 `gst_cuda_memory_export` 的 fd 直接导入为 Vulkan image。当前 NVIDIA/GStreamer
  栈下 direct import 失败：`OPAQUE_FD=ERROR_UNKNOWN`，`DMA_BUF_EXT` 虽可查询但
  `memory_type_bits=0x0`。这说明 copy 不是简单漏接了零拷贝路径，而是当前 CUDA exported
  fd 不能被 Vulkan image import 直接消费。
- 不再回到 `playbin+waylandsink` 作为主线；它已经证明不是后续默认方向。
- 不为了减少十几 MiB active video dirty memory 牺牲稳定性、frame pacing 或其他壁纸类型推进。
- NVIDIA direct 不再押注 gst-va/DMABuf。当前本机 `nvh264dec` 只暴露 `CUDAMemory`、
  `GLMemory` 和 system memory，没有 `DMABuf` 或 `VulkanImage`；GStreamer `vulkanupload`
  也不接 `CUDAMemory`/`GLMemory`。因此 NVIDIA 的真正 zero-copy/direct 主线改为
  Vulkan Video decode 产出 Vulkan image，而不是安装 CUDA toolkit 或强行走 VAAPI。

保留的底层方向：

- Gilder 自己拥有 Vulkan instance/device/swapchain、render pass、import/decode/present。
- GStreamer 只作为 demux/parser/appsink/audio/clock 前端；DMA/VAAPI、CUDA、Vulkan Video
  等 GPU handoff 必须在 native Vulkan importer 内落地。
- native Wayland host 不再直接 attach video dmabuf 或代理 GStreamer overlay sink。

## 并行推进原则

后续工作分成两条并行线，而不是先后依赖：

- 类型线：继续补齐 `web`、`scene-lite`、`shader`、playlist、particle、audio-responsive
  等壁纸类型，让用户可见能力继续增长。
- Vulkan 线：同步建立 hand-rolled Vulkan host、device、swapchain、render graph 和
  texture/video interop，逐步把 video、shader、scene、web frame 都收敛到同一个 GPU 后端。

两条线共享同一份 manifest、render plan、属性系统、动态生命周期和 telemetry。类型线不能把
新能力焊死到 WebKitGTK/helper 或某个临时前端；Vulkan 线也不能只服务 video，而要从一开始按完整类型矩阵设计。

## 近期优先级

类型线的近期优先级：

1. `web` runtime：独立 helper、sandbox、属性 bridge、暂停/恢复、音频/网络权限。
2. `scene-lite` runtime：从静态 snapshot 扩展到真正的 2D timeline runtime。
3. `shader` runtime：编译 WGSL/GLSL 类 shader、注入 time/resolution/mouse/property uniform。
4. `playlist` 稳定：继续补 Wallpaper Engine 复杂策略映射，并保证子项切换不泄漏 runtime 资源。
5. audio-responsive 和 particle：必须从第一天接入权限、telemetry、预算 gate。

Vulkan 线的近期优先级：

1. 最小 native Vulkan layer-shell host：clear、static image、resize、output selection。
2. 统一 renderer backend contract：让 native Vulkan、Web helper 和 headless evaluator 消费同一 render plan。
3. Shader-first path：fullscreen triangle、time/resolution/property uniform、surface smoke。
4. Scene-lite render target：把 deterministic scene runtime 输出接入 Vulkan pass。
5. Video interop 继续作为主攻点：把 GStreamer appsink/DMA handoff 接到 native Vulkan importer。

这些工作互不阻塞。类型 runtime 可以先用 helper/headless fallback 实现，但合并前要同时写清 Vulkan-facing
contract；Vulkan spike 可以先支持少量类型，但不能引入第二套 manifest 或 daemon 语义。

## 后端边界

后续代码应维持以下边界：

- `core`、manifest、conversion report、render plan 不引用 GTK、GDK、wgpu、ash 或 GStreamer
  具体类型。
- daemon 只生成“要显示什么”的计划：entry、source、fit、time、property values、policy、target FPS。
- renderer 后端负责“怎么显示”：Vulkan image、Web helper surface、shader pipeline 和
  GPU importer 都留在后端内。
- status/watch telemetry 使用稳定字段描述能力和资源，不暴露某个后端独有对象生命周期作为上层契约。
- 新增类型必须先定义 headless 行为测试，再补真实 Wayland smoke；不能只靠某个 GUI 后端能显示。

推荐抽象方向：

- `SurfaceHost`：输出绑定、layer-shell surface、resize、present cadence。
- `RenderBackend`：消费 render plan，创建/更新/释放每个输出的 runtime。
- `TextureSource`：静态图、video frame、Web helper frame、scene render target、shader output。
- `DynamicRuntime`：统一 pause/resume/throttle/release/resource snapshot。
- `GpuInterop`：后端内部能力，不向 manifest 或 daemon 泄漏；当前由 ash/native Vulkan、
  Vulkan Video、GStreamer DMA/CUDA/VAAPI handoff 实现。

这些名字不是立即要落地的 API，而是后续重构时的边界检查标准。

## Vulkan 后端目标

纯 Vulkan 后端的目标不是“替换而已”，必须同时满足：

- 自己拥有 Wayland layer-shell surface、Vulkan instance/device/swapchain 和 render loop。
- 支持 static image、video、web、scene-lite、shader、playlist 选中子项的统一合成。
- video 允许 NV12/YUV texture sampling，避免默认转 RGBA 大纹理。
- shader 和 scene 使用同一套 property/time/uniform 输入。
- Web runtime 至少能通过 helper 进程输出可导入 texture 或 frame stream；WebKitGTK 可以留在 helper
  内，但不应污染 daemon/core 的后端抽象。
- Web helper 初期可以用 GTK-rs/WebKitGTK 承载页面，但 `native-vulkan-renderer` feature 不直接依赖
  GTK-rs；helper 和 renderer 之间只保留稳定 frame/texture handoff 协议，便于后续替换为 C
  WebKitGTK、WPE/WebKit 或其他 web runtime。
- 所有动态类型都支持 `pause-dynamic`、fullscreen/hidden/session release、resource telemetry 和
  baseline matrix 预算。

不接受的 Vulkan 迁移：

- 只实现 video，导致 web/scene/shader/playlist 需要另一套生命周期。
- 为了底层 interop 把 manifest/render plan 改成 Vulkan 专用结构。
- 缺少真实 Wayland smoke、frame pacing、资源释放和 fallback 验证。
- 只看 FPS，不同时看 CPU、PSS/USS/private dirty、GPU memory、skipped frames 和恢复延迟。

## 迁移阶段

### Phase 0: 固化当前基线

- 保留 native-wgpu 和 GTK/GStreamer 的 4K/240 数值证据作为历史对照，不再保留可构建后端。
- 保留 native Vulkan H.265 ready-prefix/first-frame/surface queue smoke 作为当前真实 Wayland evidence。
- 将当前 CUDA direct import blocker 记录在文档中，避免重复走同一条失败路径。

### Phase 1: 后端无关 runtime 接口

- 清理 render plan 与 renderer runtime 的边界。
- 为 web、scene-lite、shader、playlist 子项定义共同的 dynamic lifecycle。
- status/watch 和 baseline matrix 只依赖稳定 telemetry 字段。
- 每个新增类型在 helper/headless fallback 实现之外，同步定义 Vulkan 后端需要消费的资源、uniform、
  timeline、权限和 release 语义。

### Phase 2: 类型补全

- Web helper 先可用，允许 GTK/WebKitGTK 作为隔离实现，但通过 helper 协议和 daemon 交互。
- Scene-lite 先实现确定性 2D runtime，动画、属性 binding、资源释放必须可测试。
- Shader runtime 先覆盖 fullscreen triangle / image filter / time uniform 这类高频场景。
- 类型补全和 Vulkan spike 并行推进；类型合并不等待 Vulkan 后端完整实现，但不能破坏后端无关边界。

### Phase 3: Vulkan spike

- 2026-06-20 已开始落地 `native-vulkan-renderer` feature：先提供 capabilities、后端 contract
  和 `gilder-native-vulkan` JSON 入口；当前不改默认 renderer。
- 同日新增 `--probe-surface`：复用 native Wayland layer-shell host 创建 `VK_KHR_wayland_surface`
  并枚举 present-capable GPU/queue。
- 真实 Wayland probe 已在 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 通过：选中 NVIDIA GeForce
  RTX 4060 Laptop GPU 的 graphics/present queue 0，surface image count 范围为 2..=8。
- `--probe-surface` 现在同时记录 selected present queue 的 video flags/codec operations、同设备
  H.265 decode queue 和 `h265_decode_requires_cross_queue_sync`。真实 Wayland smoke
  `scripts/native-vulkan-surface-video-queue-smoke.sh --output-name HDMI-A-1` 固化当前机器拓扑：
  surface/present 选中 NVIDIA 4060 queue family 0 (`graphics|compute|transfer|sparse-binding`)，
  同设备 H.265 Vulkan Video decode 在 queue family 3 (`transfer|sparse-binding|video-decode`)；
  因此 visible direct path 不能假设同 queue，必须创建同一 logical device 的 video queue +
  graphics/present queue，并通过 semaphore/ownership 或 concurrent sharing 把 decoded NV12
  image 交给 shader render。
- `--run-h265-first-frame-video` 已接通首个可见 direct Vulkan Video path：真实 Wayland smoke
  `scripts/native-vulkan-h265-first-frame-video-smoke.sh --output-name HDMI-A-1` 生成
  3840x2160@240 H.265 Main 源，使用 NVIDIA 4060 queue family 3 执行 `vkCmdDecodeVideoKHR`
  解码首个 IDR 到 `G8_B8R8_2PLANE_420_UNORM` video resource image，再由 queue family 0 的
  graphics/present queue 通过 native Vulkan NV12 shader 采样到 Wayland swapchain。2026-06-21
  证据目录 `/tmp/gilder-vulkan-h265-first-frame-video.VBQdoH`：decode elapsed `4909us`，
  3 秒 present `720` 帧、平均 `239.709fps`，swapchain `B8G8R8A8_UNORM`、`2561x1601`、
  source extent `3840x2160`，video image memory `100139008` bytes，session memory
  `33775616` bytes。当前这是首帧静态重复 present，用 `queue_wait_idle` 做首版跨队列同步；
  下一步要把 ready-prefix sequence 接到同一可见 swapchain，并以 semaphore/timeline 替代 wait-idle。
- `--run-h265-ready-prefix-video` 已把 visible direct path 推到多 AU sequence：真实 Wayland smoke
  `scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --output-name HDMI-A-1` 同样使用
  3840x2160@240 H.265 Main 源，在 queue family 3 连续提交 8 个 ready-prefix AU，再由 queue
  family 0 将每个 decoded NV12 layer 采样到 Wayland swapchain。2026-06-21 证据目录
  `/tmp/gilder-vulkan-h265-ready-prefix-video.4fJHk9`：`decoded_frame_count=8`、
  `presented_frame_count=8`、frame layers `[0,1,0,1,0,1,0,1]`、PTS delta `4..=5ms`、
  max decode `4835us`、max present `1549us`、swapchain `B8G8R8A8_UNORM`、`2561x1601`。
  这一步证明连续 decoded NV12 array layers 可以直接进入可见 native Vulkan present；当前仍用
  per-frame `queue_wait_idle` 保守同步，后续要补 semaphore/timeline、持续 demux/parser、loop
  和长时间 240Hz pacing。
- 同一路径已移除 ready-prefix visible path 的 per-frame video queue `wait_idle`：video queue 每帧
  signal 一个 binary `decode_finished` semaphore，graphics/present queue submit 同时等待
  `image_available` 和 `decode_finished`，再执行 NV12 shader sampling 和 present。真实 Wayland
  smoke `scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --output-name HDMI-A-1
  --decode-prefix 24 --frames 26` 通过，证据目录
  `/tmp/gilder-vulkan-h265-ready-prefix-video.38sK0Y`：`decoded_frame_count=24`、
  `presented_frame_count=24`、策略
  `per-frame-binary-semaphore-decode-signal-present-wait`、layer 序列
  `[0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1]`、PTS delta `4..=5ms`、
  average present `238.128fps`、max decode submit `79us`、max present `4834us`。下一步不再是
  wait-idle removal，而是持续 parser/demux、loop/seek、timeline/pacing 和长时间 240Hz telemetry。
- 已为同一 visible direct H.265 path 增加受控 ready-prefix playback loop：CLI 和 smoke 支持
  `--playback-frames N`，`ready_prefix_frame_count` 只决定 GStreamer `qtdemux+h265parse`
  抽取和 Vulkan bitstream payload，`requested_playback_frame_count` 决定实际 decode/present
  帧数；loop boundary 会强制 reset video coding，并在 runtime JSON 中记录
  `playback_loop_count`、`loop_boundary_reset_count`、pacing sleep/miss 和每帧
  `playback_loop_index`。真实 Wayland 20s smoke
  `scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --no-build --output-name HDMI-A-1
  --source /tmp/gilder-vulkan-h265-ready-prefix-video.WeoJFj/source/h265-main-short-gop-3840x2160-240fps.mp4
  --decode-prefix 24 --playback-frames 4800` 通过，证据目录
  `/tmp/gilder-vulkan-h265-ready-prefix-video.SwDOks`：`decoded_frame_count=4800`、
  `presented_frame_count=4800`、`playback_loop_count=200`、
  `loop_boundary_reset_count=199`、average present `240.006fps`、max decode submit `732us`、
  avg decode submit `20us`、max present `4952us`、avg present `2812us`、
  `missed_frame_pacing_count=4`、max pacing late `846us`。这仍是受控 AU window 循环，不等价于完整
  continuous demux/parser/audio/seek runtime；下一步要把窗口替换为持续 AU supply 和 timeline/clock。
- 2026-06-21 复测确认可见抖动不能再用 8 帧 window 判断：8-frame ready-prefix 在 240Hz 下每
  33ms 回到 AU0，20 秒会循环 600 次，肉眼必然像抖动。`scripts/native-vulkan-h265-ready-prefix-video-smoke.sh`
  先改成 `decode_prefix=target_fps`、生成源 `gop_size=target_fps`；如果 looped visible playback
  的 ready-prefix 短于 1 秒，脚本会失败，除非显式传 `--allow-short-loop` 做诊断。该阶段真实
  Wayland 20s evidence：`/tmp/gilder-vulkan-h265-ready-prefix-video.YS2xQf`，源为 242 帧
  `hevc/Main`、3840x2160@240，`ready_prefix_frame_count=240`、
  `requested_playback_frame_count=4800`、`decoded_frame_count=4800`、
  `presented_frame_count=4800`、`playback_loop_count=20`、`average_present_fps=239.99556876981734`，
  FIFO present 下 `frame_sleep_count=0`、`missed_frame_pacing_count=0`。
- 第一条 H.265 direct smoke 的默认测试源已进一步改为接近第二条 GStreamer/appsink 路线的连续
  4K/240 口径：当调用
  `scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --output-name HDMI-A-1 --playback-frames 4800`
  且没有显式 `--decode-prefix` 时，脚本会把 `decode_prefix` 扩到 `playback_frames`，生成
  `testsrc2-continuous-closed-gop-h265-main` 源，并只在显式传入较短 `--decode-prefix` 时保留旧的
  AU window loop/reset 诊断模式。这样第一条路线不再默认把视觉平滑度和 `AU239 -> AU0` 边界混在一起。
- 同一 evidence 下的约 70MB private dirty 主要来自显式 Vulkan Video 资源，而不是残留 CPU
  copy：2-ref H.265 source 需要 `stream_dpb_slots=session_max_dpb_slots=3`，
  `video_resource_memory_bytes=37552128`，加上 NVIDIA driver 报告的
  `session_memory_bytes=33775616` 和 `bitstream_buffer_bytes=249344`，三者合计约 71.6MB。
  DPB 选择现在按 ready-prefix AU 的可解码性寻找最小 slot 数，并会把“当前输出将覆盖的 slot”
  视为不可继续作为参考帧，避免把过小 DPB 误判为 ready。
- 1-ref H.265 short-GOP 对照已经证明这条路径不是固定 70MB floor：真实 Wayland 证据
  `/tmp/gilder-vulkan-h265-ready-prefix-video.q8NPT5` 在 `HDMI-A-1` 上使用 3840x2160@240
  source，`stream_sps_dpb_slots=3`、`stream_dpb_slots=2`、
  `stream_max_active_reference_pictures=1`，`video_resource_memory_bytes=25034752`、
  `session_memory_bytes=33775616`、`bitstream_buffer_bytes=248320`，显式
  resource/session/bitstream 合计约 59.1MB。NVIDIA H.265 session memory 没随
  `maxActiveReferencePictures` 从 2-ref 降到 1-ref 明显下降，当前更像驱动对 H.265
  session/extent/profile 的固定成本。
- visible H.265 ready-prefix 的 4K/240 长窗口内存尖峰已定位并修掉：旧路径把所有 AU payload
  保存在 `Vec<Vec<u8>>`，所以即使 Vulkan bitstream buffer 已经是单个 249KiB reusable slot，
  4800 AU 的 `bitstream_window_payload_bytes=499056595` 仍会变成进程私有内存。历史中间阶段曾改为
  GStreamer demux/parse 后写入临时 spool 文件，播放时只按 AU offset 读入一个复用 upload buffer；
  runtime JSON 仍报告同样的 encoded window/upload 字节用于吞吐统计，但不再保留为 RSS/USS。
  2026-06-21 真实 Wayland `HDMI-A-1` 证据 `/tmp/gilder-vulkan-h265-memory-spooled.d8pybb`
  在 3840x2160@240、4800 frames 下达到 `average_present_fps=239.977`，
  `RSS/PSS/USS/Private_Dirty max=117732/85864/68248/37664 KiB`；旧 in-memory payload
  证据 `/tmp/gilder-vulkan-h265-memory.GIYC3r` 为
  `1089060/1069592/1061636/1008992 KiB`。
- H.265 spool upload 曾作为低内存中间阶段接入固定容量 Bitstream Ring Buffer：当时可见播放循环从
  spool file 直接写入持久映射的 `VIDEO_DECODE_SRC_KHR` buffer，不再经由临时 AU `Vec<u8>`；
  顺序读取时跟踪 file position，首帧强制 seek，当前 AU aligned range 内的 padding 每次清理，避免
  stale bitstream tail 进入 decoder。默认 ring 为 2-slot，按 driver 的 offset/size alignment 追加写入，
  runtime JSON 记录每帧 `src_buffer_offset`、payload/range、allocation index 和 wrap count。2026-06-21
  真实 Wayland `HDMI-A-1` 证据 `/tmp/gilder-vulkan-h265-ready-prefix-video.Ldh5wL` 在
  3840x2160@240、4800 frames 下达到 `average_present_fps=240.041`，
  `bitstream_buffer_strategy=fixed-capacity-persistent-mapped-ring`、`bitstream_buffer_bytes=498688`、
  `bitstream_ring_wrap_count=1200`；同配置 smaps 证据 `/tmp/gilder-vulkan-h265-ring-memory.9RFFoa`
  为 `RSS/PSS/USS/Private_Dirty max=117836/86018/68380/37932 KiB`，确认 ring 没把 RSS/USS
  拉回旧的 retained AU payload 级别。后续维护目标已经切到连续 demux/parser streaming queue；
  visible H.264/H.265 代码不再保留可选 spool 输入分支。decode fence/timeline 回收 range
  仍是下一步性能工作。
- H.264 visible direct path 已切到 streaming packet queue 作为维护目标：
  `gilder-native-vulkan --run-h264-ready-prefix-video` 和脚本默认使用
  `h264_input_mode=streaming-queue`，显式 `ready-prefix-spool` CLI 参数会报错；visible video
  runtime 已移除可选 spool 分支，不再扩展或作为验证口径。GStreamer appsink AU 进入 bounded packet
  queue，播放时按需拉 AU、上传压缩码流到 `VIDEO_DECODE_SRC_KHR` ring，随后丢弃 payload。
  H.264 parser/reference plan 已支持短期 L0 ref-list modification idc 0/1，并把 streaming mode
  的 reference plan 从每帧全量重算改成增量 planner。2026-06-21 真实 Wayland `HDMI-A-1`
  证据：720p/60 `/tmp/gilder-vulkan-h264-ready-prefix-video.at5uDt` 为 `decoded=8`、
  `p_frames=7`、`max_reference_count=2`、`h264_input_mode=streaming-queue`、queue retained
  payload `0`；4K/240 `/tmp/gilder-vulkan-h264-ready-prefix-video.ZFXzKH` 为
  `decoded/presented=240/240`、`queue_capacity=32`、`queue_pulled=240`、
  `bitstream_buffer_bytes=1036800`。2026-06-21 默认 streaming 4K/240 回归
  `/tmp/gilder-vulkan-h264-streaming-default-regression` 为 `decoded/presented=240/240`、
  `h264_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=213.179`。
  20s loop `/tmp/gilder-vulkan-h264-streaming-smaps.oULUUh`
  为 `decoded/presented=4800/4800`、`queue_eos/loops=4/4`、`queue_retained=0`，
  90 个 smaps_rollup 样本 `RSS/PSS/USS/Private_Dirty max=112908/68437/49192/31272 KiB`、
  平均 CPU `15.13%`、`average_present_fps=212.375`。当前内存问题已不在 packet retention；
  H.264 4K/240 剩余瓶颈在 H.264 level/capability 边界、present pacing/同步或驱动 codec 路径。
  P-slice weighted prediction 已补 `pred_weight_table` 解析并通过真实 Wayland 4K/240
  `/tmp/gilder-vulkan-h264-weightp-4k240-smoke`，结果为 `decoded/presented=240/240`、
  `p_frames=239`、`max_reference_count=2`、`queue_retained=0`、`average_present_fps=214.9888566483139`。
  B-slice 显式短期 L1 reference list modification 已补 parser/planner 单测覆盖。H.264
  long-term reference/MMCO 状态机已推进到 planner + visible submit 同步：planner 维护
  short-term/long-term DPB key，支持 ref-list modification idc 2、IDR long-term flag、
  MMCO op=2/3/4/5/6、long-term index replacement，并把
  `used_for_long_term_reference` 传入 `StdVideoDecodeH264ReferenceInfo`。新增单测覆盖
  IDR long-term、MMCO6 当前图 long-term、MMCO4 上限裁剪、MMCO5 全清、long-term index
  replacement 和 long-term L0 modification。2026-06-21 真实 Wayland `HDMI-A-1` 回归：
  720p/60 B-frame `/tmp/gilder-vulkan-h264-longterm-planner-regression` 为
  `decoded/presented=60/60`、`h264_input_mode=streaming-queue`、`queue_retained=0`；
  4K/240 B-frame `/tmp/gilder-vulkan-h264-longterm-planner-4k240-regression` 为
  `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、
  `average_present_fps=194.8709`。这证明普通连续 B-frame 路径未因 long-term 改动退化；
  短期 reference list 默认排序和 ref-list modification idc 0/1 已改为按 PicNum 处理，覆盖
  `frame_num` wrap 后的参考查找；真实 long-term coded stream smoke 仍待补充，剩余码流边界是
  field picture、gaps-in-frame-num/non-existing refs 和任意入口点 DPB 重建。
- 2026-06-21 CI 修复复测同一路线时增加了 per-frame present telemetry 和 3-image swapchain
  preference。真实 Wayland `HDMI-A-1`、3840x2160@240、2400-frame streaming queue evidence
  `/tmp/gilder-vulkan-h264-ci-fix-smoke` 为 `decoded/presented=2400/2400`、
  `average_present_fps=214.29452814312305`、`queue_retained=0`、
  `vkQueuePresentKHR avg/p95/max=4373/4868/6231us`；匹配 smaps 证据
  `/tmp/gilder-vulkan-h264-ci-fix-smaps-keep/performance` 为
  `RSS/PSS/USS/Private_Dirty max=112080/78517/61032/29176 KiB`、平均 CPU `13.48%`。
  同机 H.265 对照 `/tmp/gilder-vulkan-h265-ci-fix-smoke` 为
  `average_present_fps=238.60528994743973`，smaps
  `/tmp/gilder-vulkan-h265-ci-fix-smaps-keep/performance` 为
  `RSS/PSS/USS/Private_Dirty max=112800/79293/61652/29836 KiB`。结论不变：
  H.264 当前不是码流 retention 或 bitstream ring 内存问题，而是 FIFO present/codec path 的稳定
  240Hz 缺口；下一步需要 multi-in-flight/decode-ahead/present 解耦，而不是回退到
  playbin/waylandsink 或继续压 packet queue。
  `GILDER_VULKAN_H264_PRESENT_DELAY_US=400` 诊断只把 `vkQueuePresentKHR` avg 降到
  `3934us`，但总 FPS 降到 `212.855`，说明简单 CPU sleep 不是有效方向，已移除该临时开关。
- H.264 streaming planner 已继续推进常见连续码流：非参考 picture 不再进入 active DPB，
  但 visible submit 仍为每个 decode target 提供 setup slot，以满足 NVIDIA H.264 Vulkan Video
  对 coincident DPB/output target 的 STD 参数要求。planner 已支持默认 B-slice L0/L1 短期引用
  列表、参考 B picture、非参考 B scratch 输出，以及 MMCO op=1 short-term unused-for-reference
  的 DPB/reference drop。新增单测覆盖非参考 scratch、默认 B-slice 和 adaptive marking。
  真实 Wayland `HDMI-A-1` evidence：`/tmp/gilder-vulkan-h264-b1-streaming-smoke` 为
  `decoded/presented=120/120`、`b_frames=59`、`max_reference_count=2`、`queue_retained=0`；
  `/tmp/gilder-vulkan-h264-bslice-streaming-smoke-final` 为 `decoded/presented=180/180`、
  `b_frames=119`、`max_reference_count=3`、`h264_input_mode=streaming-queue`、`queue_retained=0`。
  long-term reference 增量继续沿用同一路线，单测覆盖 IDR long-term、MMCO 2/3/4/5/6
  和 long-term list modification，visible submit 同步 active DPB 的 long-term flag；短期
  reference list 默认排序和 idc 0/1 modification 按 PicNum 处理，覆盖 `frame_num` wrap。
  真实 long-term coded stream 仍缺 source/smoke；当前 4K/240 B-frame 回归
  `/tmp/gilder-vulkan-h264-longterm-planner-4k240-regression` 为
  `decoded/presented=240/240`、`queue_retained=0`、`average_present_fps=194.8709`。
- H.265 visible direct input 也已接到 bounded streaming packet queue，并同样默认使用
  `h265_input_mode=streaming-queue`；显式 spool CLI 参数会报错，visible video runtime 已移除
  可选 spool 分支。按需拉 AU、上传到 bitstream ring
  后释放 payload，runtime JSON 报告 `h265_packet_queue_*`。2026-06-21 真实 Wayland
  `HDMI-A-1` 4K/240 smoke `/tmp/gilder-vulkan-h265-ready-prefix-video.uMgUWp` 为
  `decoded/presented=240/240`、`average_present_fps=238.316`、`queue_pulled=240`、
  `queue_retained=0`；20s smaps `/tmp/gilder-vulkan-h265-streaming-smaps.wTY7vB` 为
  `decoded/presented=4800/4800`、`average_present_fps=240.027`、`queue_eos/loops=19/19`、
  `RSS/PSS/USS/Private_Dirty max=115480/71078/51800/33892 KiB`、平均 CPU `20.11%`。
  默认 streaming 回归 `/tmp/gilder-vulkan-h265-streaming-default-regression` 为
  `decoded/presented=240/240`、`average_present_fps=240.915`、`queue_retained=0`。本轮
  H.264 long-term planner 改动后复测 H.265 4K/240 direct streaming
  `/tmp/gilder-vulkan-h265-longterm-planner-4k240-regression` 为
  `decoded/presented=240/240`、`h265_input_mode=streaming-queue`、`queue_retained=0`、
  `average_present_fps=240.4747`。
- 2026-06-21 handoff snapshot：`bff077b Support H264 PicNum reference ordering` 已推送。
  H.264 direct Vulkan Video 现在默认走 bounded `streaming-queue`，不维护 spool，不使用
  playbin/waylandsink；P/B 帧、多参考、短期 L0/L1 modification、MMCO、long-term planner
  和 `frame_num` wrap 后的 PicNum reference ordering 都有单测或真实 Wayland evidence。
  最新 H.264 真实 Wayland `HDMI-A-1` 回归：720p/60
  `/tmp/gilder-vulkan-h264-picnum-wrap-regression` 为 `decoded/presented=60/60`、
  `h264_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=252.939`；
  4K/240 B-frame `/tmp/gilder-vulkan-h264-picnum-wrap-4k240-regression` 为
  `decoded/presented=240/240`、`b_frames=119`、`max_reference_count=2`、
  `queue_retained=0`、`average_present_fps=198.431`。H.265 对照
  `/tmp/gilder-vulkan-h265-picnum-wrap-4k240-regression` 为
  `decoded/presented=240/240`、`h265_input_mode=streaming-queue`、`queue_retained=0`、
  `average_present_fps=240.522`。该 snapshot 证明 streaming queue 和常见连续 H.264/H.265
  可见路径继续成立；它不证明原目标“任意连续”完全完成。当时剩余边界仍是 H.264 真实
  long-term coded stream、gaps-in-frame-num/non-existing refs、field/interlaced picture、
  任意入口点 DPB 重建，以及 H.264 4K/240 稳定帧率。
- H.264 direct planner 已继续推进 `gaps_in_frame_num` / non-existing short-term reference
  边界：当 SPS `gaps_in_frame_num_value_allowed_flag=false` 时，frame_num gap 会成为明确
  unready reason；当 flag 为 true 时，planner 会按 `max_frame_num` wrap 推断 non-existing
  short-term refs，维护 DPB slot/sliding window，并把 `non_existing=true` 贯穿到 visible
  submit 的 `StdVideoDecodeH264ReferenceInfoFlags.is_non_existing`。runtime JSON 现在能在
  H.264 reference telemetry 中记录 `non_existing`，并记录 inferred non-existing refs 与
  inference 阶段挤掉的 DPB slot。新增单测覆盖 gap disallowed、gap allowed、
  `max_frame_num=65536` wrap、sliding window、PicNum wrap default/ref-list modification
  与 B-slice 连续 frame_num 场景。2026-06-21 真实 Wayland `HDMI-A-1` 回归：H.264
  720p/60 `/tmp/gilder-vulkan-h264-nonexisting-regression` 为 `decoded/presented=60/60`、
  `queue_retained=0`、`average_present_fps=247.596`；H.264 4K/240 B-frame
  `/tmp/gilder-vulkan-h264-nonexisting-4k240-regression` 为 `decoded/presented=240/240`、
  `b_frames=119`、`queue_retained=0`、`average_present_fps=202.138`；H.265 4K/240 对照
  `/tmp/gilder-vulkan-h265-nonexisting-4k240-regression` 为 `decoded/presented=240/240`、
  `queue_retained=0`、`average_present_fps=240.622`。H.264 4K/240 direct 长跑采样
  `/tmp/gilder-vulkan-h264-nonexisting-4k240-memory/combined-keep/performance` 为
  `decoded/presented=7200/7200`、`average_present_fps=202.047`、`queue_retained=0`，
  8 个 smaps samples 中 `RSS/PSS/USS/Private_Dirty max=105048/73925/56404/26756 KiB`、
  平均 CPU `12.10%`、NVIDIA 进程 GPU memory `104 MiB`。这说明当前 H.264 direct 内存
  已主要落在 driver/Vulkan Video/session/heap floor，packet queue 没有 retained payload；
  但 H.264 4K/240 仍未稳定满 240fps。剩余 direct H.264 边界收敛为更多真实 long-term coded
  stream、field/interlaced picture 和 frame pacing/同步优化；任意非 IDR 入口重对齐已在后续
  streaming queue 证据中补上。
- H.264/H.265 visible direct input 已冒险合并为一套共用 streaming packet queue：
  `NativeVulkanStreamingAccessUnit` 提供 codec hook，泛型队列统一持有 GStreamer
  pipeline/appsink/bus、bootstrap 参数集选择、EOS seek loop、payload retained accounting 和
  bitstream ring sizing；H.264/H.265 只保留 pipeline、appsink 名称、参数集解析和 snapshot
  hook。这样后续 AV1/音频 clock/metadata ring 可以接同一个输入层，而不是继续维护两份
  H.264/H.265 队列代码。2026-06-21 真实 Wayland `HDMI-A-1` 回归：
  H.264 720p/60 `/tmp/gilder-vulkan-common-queue-h264-720p60` 为
  `decoded/presented=60/60`、`queue_retained=0`、`average_present_fps=247.329`；
  H.264 4K/240 `/tmp/gilder-vulkan-common-queue-h264-4k240` 为
  `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、
  `average_present_fps=199.922`；H.265 4K/240
  `/tmp/gilder-vulkan-common-queue-h265-4k240` 为 `decoded/presented=240/240`、
  `queue_retained=0`、`average_present_fps=238.368`。同轮 H.264 4K/240 长跑
  `/tmp/gilder-vulkan-common-queue-h264-4k240-memory/performance` 为
  `decoded/presented=2400/2400`、`average_present_fps=204.900`、`queue_eos/loops=9/9`、
  `queue_retained=0`，8 个 samples 中 `RSS/PSS/USS/Private_Dirty max=105404/63269/45272/26844 KiB`、
  平均 CPU `12.49%`、NVIDIA 进程 GPU memory `104 MiB`。这证明本次大重构没有把 H.265
  4K/240 路线打退，也再次确认 H.264 4K/240 的剩余缺口不是 packet retention。
- H.264 visible direct streaming queue 已补上任意非 IDR 入口的 bootstrap 重对齐：启动时
  队列不再要求最初 capacity 个 AU 自身就是 ready-prefix，而是 bounded scan 到 SPS/PPS/IDR，
  只保留固定容量窗口，丢弃不可解 P/B 前缀，并把 EOS loop 的 skip 位置同步到同一个可解入口；
  默认 `GILDER_VULKAN_STREAMING_BOOTSTRAP_SCAN_LIMIT` 从 256/`capacity*16` 提高到
  4096/`capacity*128`，仍然不保留被扫描掉的 payload。`scripts/native-vulkan-h264-ready-prefix-video-smoke.sh`
  新增 `--arbitrary-entry-offset`，会用 `ffmpeg -copyinkf` 生成从非关键帧开始的源，并 gate
  `h264_packet_queue_bootstrap_discarded_access_units > 0`、loop skip 和首帧 IDR。2026-06-21
  真实 Wayland `HDMI-A-1` 证据：720p/60 B/P 源
  `/tmp/gilder-vulkan-h264-arbitrary-entry-script-gate` 从 `0.35s` 入口启动，
  `decoded/presented=60/60`、`bootstrap_discarded=39`、`loop_skip=39`、
  `first_frame_idr=true`、`p_frames=30`、`b_frames=29`、`max_reference_count=2`、
  `queue_retained=0`；手工 copyinkf 回归 `/tmp/gilder-vulkan-entry-realign-h264-copyinkf-v3`
  丢弃 99 个坏前缀 AU 后可见播放 60 帧。4K/240 回归单独运行：
  H.264 `/tmp/gilder-vulkan-h264-arbitrary-entry-4k240-regression-seq` 为
  `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、
  `average_present_fps=198.012`；H.265 对照
  `/tmp/gilder-vulkan-h265-bootstrap-scan-4k240-regression` 为
  `decoded/presented=240/240`、`queue_retained=0`、`average_present_fps=240.927`。同轮
  `cargo test --features native-vulkan-gst-video` 297 个测试通过。
- H.265 visible direct streaming queue 也补上同等 arbitrary-entry smoke gate：
  `scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --arbitrary-entry-offset` 会生成
  `-copyinkf` 非关键帧入口源，并要求 `h265_packet_queue_bootstrap_discarded_access_units > 0`、
  loop skip 和首帧 IDR。脚本的 bitstream gate 同步修正为检查固定 ring slot 数小于 decode
  window，而不是要求低码率 H.265 的窗口 payload 必须大于 ring capacity。2026-06-21 真实
  Wayland `HDMI-A-1` 证据：720p/60 `/tmp/gilder-vulkan-h265-arbitrary-entry-script-gate-v2`
  从 `0.35s` 非关键帧入口启动，`decoded/presented=60/60`、`bootstrap_discarded=39`、
  `loop_skip=39`、`first_frame_idr=true`、`frame_access_units_head` 从 39 开始、
  `queue_retained=0`；H.265 4K/240 回归
  `/tmp/gilder-vulkan-h265-arbitrary-entry-4k240-regression` 为
  `decoded/presented=240/240`、`average_present_fps=239.919`、`queue_retained=0`、
  `video_resource_memory_bytes=37552128`、`session_memory_bytes=33775616`、
  `bitstream_buffer_bytes=1036800`。同一当前工作树下 H.264 arbitrary-entry 回归
  `/tmp/gilder-vulkan-h264-arbitrary-entry-current-regression` 为 `decoded/presented=60/60`、
  `bootstrap_discarded=39`、`loop_skip=39`、`first_frame_idr=true`、`max_reference_count=2`、
  `queue_retained=0`。
- 2026-06-22 将 H.264/H.265 arbitrary-entry 的失败和回环证据继续收紧：direct visible
  path 在创建 Wayland surface/swapchain 前先启动 bounded streaming packet queue、执行
  bootstrap realignment，并要求 queue 填满 ready-prefix window；如果非关键帧入口后的下一个
  IDR 到 EOS 不足以形成窗口，会在可见 surface 创建前失败，避免短源测试把桌面背景切黑。
  短源负测试 `/tmp/gilder-vulkan-h264-short-window-preflight-v3` 使用不存在的
  `WAYLAND_DISPLAY=gilder-missing-display` 仍在 0.23s 内返回
  `H.264 streaming bootstrap found a decodable entry, but the source ended after 9/30 queued AU(s)`，
  证明失败发生在 Wayland connect 之前。H.264/H.265 smoke 新增
  `--require-loop-skip-replay`，arbitrary-entry 且 playback 跨过 decode-prefix 时自动要求
  `packet_queue_eos_count > 0`、`packet_queue_loop_count > 0`、loop boundary reset、loop skip
  和每个 playback loop 的首帧 IDR。真实 Wayland `HDMI-A-1` gated 证据：
  H.264 `/tmp/gilder-vulkan-h264-arbitrary-entry-eos-loop-gated`
  `decoded/presented=140/140`、`playback_loop_count=3`、`loop_boundary_reset_count=2`、
  `h264_packet_queue_eos_count=2`、`h264_packet_queue_loop_count=2`、
  `loop_skip=38`、`bootstrap_discarded=38`、`loop_first_non_idr_count=0`、
  `queue_retained=0`；H.265 `/tmp/gilder-vulkan-h265-arbitrary-entry-eos-loop-gated`
  `decoded/presented=140/140`、`playback_loop_count=3`、`loop_boundary_reset_count=2`、
  `h265_packet_queue_eos_count=2`、`h265_packet_queue_loop_count=2`、
  `loop_skip=39`、`bootstrap_discarded=39`、`loop_first_non_idr_count=0`、
  `queue_retained=0`。同轮 `cargo test --features native-vulkan-gst-video` 通过
  297 个库测试、7 个 `gilderctl` 测试和 16 个 `gilderd` 测试。后续同日把 H.264
  complex arbitrary-entry 和 H.265 B/ref arbitrary-entry 继续推进到真实 Wayland gate：
  H.264 planner 不再把 planned output slot 里的引用从默认 ref list 中提前删除，而是保留
  telemetry 并按实际 selected references 标记 unavailable，避免复杂 x264 GOP 把“声明 active
  ref count”误判成缺引用。`/tmp/gilder-vulkan-h264-arbitrary-entry-complex-gated-v3`
  在 `HDMI-A-1` 上使用 720p/60、`refs=3`、`bframes=2`、`weightp=2`、`weightb=1`、
  非关键帧入口 `0.35s`，通过 `decoded/presented=160/160`、`playback_loop_count=3`、
  `loop_boundary_reset_count=2`、`p_frames=53`、`b_frames=102`、`max_reference_count=4`、
  `queue_retained=0`、`average_present_fps=242.031`。H.265 planner 则改为先计算当前帧
  RPS 需要保护的 POC，再选择不会覆盖这些 POC 的 DPB output slot；新增单测复现
  POC 顺序 `0,3,2,1,6,5` 中 POC5 仍需要 `3,2,0,6`，旧轮转会覆盖 POC0 的问题。
  `/tmp/gilder-vulkan-h265-arbitrary-entry-brefs-gated-v2` 在 `HDMI-A-1` 上使用 720p/60、
  `refs=2`、`bframes=2`、非关键帧入口 `0.35s`，通过 `decoded/presented=160/160`、
  `playback_loop_count=3`、`loop_boundary_reset_count=2`、`p_frames=53`、`b_frames=102`、
  `max_reference_count=4`、`queue_retained=0`、`average_present_fps=241.434`。同轮
  `cargo test --features native-vulkan-gst-video` 通过 298 个库测试、7 个 `gilderctl`
  测试和 16 个 `gilderd` 测试；真实 smoke 后 `niri msg layers` 只剩 quickshell/dms
  正常 layer，未发现残留 `gilder-native-vulkan`/`gilderd` 进程。
- 2026-06-22 H.265 parser/STD mapper 已补 predicted short-term RPS：slice-local RPS
  现在会解析 `delta_idx_minus1`、`delta_rps_sign`、`abs_delta_rps_minus1`、
  `used_by_curr_pic_flag` 和 `use_delta_flag`，按 HEVC 推导出实际 negative/positive delta
  POC 列表，并把 prediction flag、delta/use/used bitmask 映射到
  `StdVideoH265ShortTermRefPicSet`。新增单测
  `parses_predicted_h265_short_term_ref_pic_set` 覆盖 predicted RPS 解析和 Vulkan STD
  字段。真实 Wayland 回归 `/tmp/gilder-vulkan-h265-pred-rps-regression-v2` 继续覆盖 720p/60
  H.265 B/ref arbitrary-entry streaming queue，结果为 `decoded/presented=160/160`、
  `playback_loop_count=3`、`loop_boundary_reset_count=2`、`p_frames=53`、`b_frames=102`、
  `max_reference_count=4`、`queue_retained=0`、`average_present_fps=239.905`。同轮
  `cargo test --features native-vulkan-gst-video` 通过 299 个库测试、7 个 `gilderctl`
  测试和 16 个 `gilderd` 测试。限制：手工 x265 探针
  `/tmp/gilder-h265-rps-probe-640x368.mp4` 没有产生
  `inter_ref_pic_set_prediction_flag=true` 的真实码流，因此 predicted RPS 目前是
  parser/STD 单测覆盖和普通 H.265 回归覆盖，仍缺真实 predicted-RPS 可见 smoke 源。
- 同日继续补 H.265 long-term reference 基础：SPS parser 不再只跳过
  `long_term_ref_pics_present_flag`，而是保留 `lt_ref_pic_poc_lsb_sps` 和
  `used_by_curr_pic_lt_sps_flag`，并把该表映射到
  `StdVideoH265LongTermRefPicsSps`；slice parser 记录 `num_long_term_sps`、
  `num_long_term_pics`、SPS long-term index、显式 POC LSB、`used_by_curr_pic_lt_flag`
  和 `delta_poc_msb_cycle_lt`。H.265 planner 会把 long-term references 标记为
  `used_for_long_term_reference`，按 POC LSB 或 delta-MSB 推导匹配 DPB POC，submit
  则把 short-term negative/positive refs 分别写入 `RefPicSetStCurrBefore/After`，
  把 long-term refs 写入 `RefPicSetLtCurr`，并设置
  `StdVideoDecodeH265ReferenceInfoFlags.used_for_long_term_reference`。新增单测
  `maps_h265_sps_long_term_refs_to_vulkan_std` 和
  `plans_h265_long_term_reference_by_poc_lsb`；同轮
  `cargo test --features native-vulkan-gst-video` 通过 301 个库测试、7 个 `gilderctl`
  测试和 16 个 `gilderd` 测试。真实 Wayland 回归
  `/tmp/gilder-vulkan-h265-longterm-foundation-regression` 继续覆盖 H.265 B/ref
  arbitrary-entry streaming queue，结果为 `decoded/presented=160/160`、
  `playback_loop_count=3`、`loop_boundary_reset_count=2`、`queue_retained=0`、
  `average_present_fps=241.425`；该 synthetic source 没有 long-term refs，因此仍不能
  作为真实 long-term coded stream smoke 证明。
- 同日继续修 H.264 long-term/short-term 混合 DPB 窗口：planner 不再只按
  short-term reference 数量做滑动窗口，而是按 active short-term + long-term 总数约束
  `max_num_ref_frames`/driver session 计划，避免已有 long-term reference 时继续保留过多
  short-term refs。新增单测
  `slides_h264_short_term_window_with_existing_long_term_reference` 覆盖 IDR long-term
  加后续 short-term 滑动的场景；同轮 `cargo test --features native-vulkan-gst-video`
  通过 302 个库测试、7 个 `gilderctl` 测试和 16 个 `gilderd` 测试。
  release 构建 `cargo build --release --features native-vulkan-gst-video --bin gilder-native-vulkan`
  通过；真实 Wayland `HDMI-A-1` 回归
  `/tmp/gilder-vulkan-h264-total-ref-window-regression` 使用 720p/60、`refs=3`、
  `bframes=2`、`weightp=2`、`weightb=1`、非关键帧入口 `0.35s`，结果为
  `decoded/presented=160/160`、`playback_loop_count=3`、`loop_boundary_reset_count=2`、
  `p_frames=53`、`b_frames=102`、`max_reference_count=4`、`queue_retained=0`、
  `average_present_fps=243.750`。该回归证明 H.264 visible direct streaming queue 没被
  total-reference 修正打退；它仍不是 H.264 4K/240 满帧证明，也不是真实 long-term coded
  stream smoke。
- 同日继续补 H.265 long-term 任意连续的 session 参数缺口：visible streaming bootstrap 的
  `stream_max_active_reference_pictures` 不再只统计 short-term RPS，而是把
  `long_term_references` 中 `used_by_current=true` 的 long-term refs 一起计入，避免真实
  H.265 long-term + short-term 混合引用帧创建过小的
  `VkVideoSessionCreateInfoKHR::maxActiveReferencePictures`。新增单测
  `counts_h265_mixed_short_and_long_term_active_references` 覆盖同一 AU 同时引用
  short-term POC 和 long-term POC 的 planner/session 口径；同轮
  `cargo test --features native-vulkan-gst-video h265 -- --nocapture` 通过 12 个 H.265
  相关测试，完整 `cargo test --features native-vulkan-gst-video` 通过 303 个库测试、
  7 个 `gilderctl` 测试和 16 个 `gilderd` 测试。release 构建通过；真实 Wayland
  `HDMI-A-1` 回归 `/tmp/gilder-vulkan-h265-active-ref-count-regression` 使用 720p/60、
  `refs=2`、`bframes=2`、非关键帧入口 `0.35s`，结果为
  `decoded/presented=160/160`、`playback_loop_count=3`、`loop_boundary_reset_count=2`、
  `max_reference_count=4`、`session_max_active_reference_pictures=4`、`queue_retained=0`、
  `average_present_fps=243.778`。该回归仍是普通 B/ref source，不是真实 long-term coded
  source；它证明 session active-reference 统计修正没有打退现有 H.265 visible direct path。
- 同日继续补 H.264 reference POC 正确性：planner、active DPB、visible submit 和 runtime
  telemetry 不再只保存 `pic_order_cnt[0]` 再把它复制成 `[val,val]`，而是保留并传递完整
  `PicOrderCnt[2]`。这修掉 progressive frame 中 top/bottom POC 不同但 reference STD 信息
  被压成单值的隐患，也为后续 field/top/bottom DPB key 重构打基础；field picture 仍未放开。
  新增单测 `plans_h264_references_with_full_pic_order_count_pair` 覆盖 reference snapshot
  保留 `[top,bottom]` POC；同轮 `cargo test --features native-vulkan-gst-video h264 -- --nocapture`
  通过 29 个 H.264 相关测试，完整 `cargo test --features native-vulkan-gst-video`
  通过 304 个库测试、7 个 `gilderctl` 测试和 16 个 `gilderd` 测试。release 构建通过；
  真实 Wayland `HDMI-A-1` 回归 `/tmp/gilder-vulkan-h264-full-poc-regression` 使用
  720p/60、`refs=3`、`bframes=2`、`weightp=2`、`weightb=1`、非关键帧入口 `0.35s`，
  结果为 `decoded/presented=160/160`、`playback_loop_count=3`、
  `loop_boundary_reset_count=2`、`max_reference_count=4`、`queue_retained=0`、
  `average_present_fps=243.810`；runtime JSON 中 frame/reference 均输出 `pic_order_cnt`。
- 2026-06-22 开始把 Sunshine 作为 native Vulkan/video interop 参考，而不是把某条路线照搬：
  Sunshine 的 `platform/linux/vulkan_encode.cpp` 把 Vulkan device/render-node 选择、DMA-BUF
  import、DRM fourcc/modifier/plane layout、FFmpeg hardware frame 和 shader/compute conversion
  分开处理；`cbs.cpp` 使用 FFmpeg Coded Bitstream API 处理 H.264/H.265 SPS/VUI，而不是把
  bitstream patch、picture identity 和 GPU submit 混在一起。Gilder 对应结论是：direct
  Vulkan Video 侧继续拆清 demux/parser、picture/reference planner、Vulkan session/queue 和
  present loop；GStreamer/DMA 侧的 importer 也必须拿到完整 DMA-BUF contract，包括 DRM
  format、modifier、每 plane fd/offset/pitch、modifier plane count、source GPU/render node
  与目标 Vulkan physical device 是否匹配。只看到 `memory:DMABuf` caps 不能证明 zero-copy。
- 同日把 Sunshine 的 modifier-plane 经验先落到 `native-vulkan-gst-video` telemetry：DMABUF/VA
  import 进入 Vulkan 前会通过 `vkGetPhysicalDeviceFormatProperties2` 查询当前 DRM
  fourcc/modifier 在目标 Vulkan physical device 上的 driver-expected plane count，并把
  `format/fourcc/modifier/available_plane_count/drm_object_count/y_uv_same_fd/offset/stride`
  写入 runtime JSON 的 `last_dmabuf_import`。当前 importer 仍只放行单 DRM object、Y/UV
  两个 plane layout；如果 driver 对该 modifier 期待的 plane count 不是 2，会明确失败而不是把
  `memory:DMABuf` 误报为可 zero-copy。下一步才是补完整多 object/aux-plane 导入和
  render-node/physical-device 匹配。
- 同日继续把 Sunshine 的 route-gate 思路落到 runtime：`video_runtime.memory_route` 现在会把
  `direct-dmabuf-import` / `direct-va-drm-prime-import`、`dmabuf-caps-pending-import`、
  `cuda-vulkan-copy`、`gl-memory-intermediate` 和 system/unsupported path 分开报告，并给出
  `direct_candidate`、`direct_import_confirmed`、`copy_risk`。这让 GStreamer/DMA 路线后续
  可以用真实 smoke 直接区分“caps 看起来像 DMA”和“已经完成 Vulkan 外部内存直通导入”。
- 同轮本机验证：`cargo test --features native-vulkan-gst-video` 通过 309 个库测试、7 个
  `gilderctl` 测试和 16 个 `gilderd` 测试，默认 `cargo test` 也通过；`cargo check
  --features native-vulkan-gst-va --bin gilder-native-vulkan` 通过。真实 Wayland `HDMI-A-1`
  visible smoke `/tmp/gilder-vulkan-gst-dma-contract-smoke-cuda` 使用 `nvh264dec`、
  `memory:CUDAMemory` 和 `CUDAMemory->CUDA->Vulkan external image planes` 路径，
  `frames_rendered=120`、`frames_imported=118`、`average_render_fps=59.998`、
  `last_dmabuf_import=null`。这说明本机 NVIDIA 路径没有触发 DMABUF；`gst-inspect-1.0 va`
  当前显示 VA plugin 有 0 个 feature，`vah264dec` 不存在，因此真实 VA/DMABUF contract
  smoke 还需要先让系统 GStreamer VA decoder 暴露可用 feature。
- route-gate 增量同轮补充验证：`cargo fmt -- --check`、默认 `cargo test`、
  `cargo test --features native-vulkan-gst-video` 和
  `cargo build --release --features native-vulkan-gst-cuda --bin gilder-native-vulkan`
  通过。真实 Wayland `HDMI-A-1` smoke `/tmp/gilder-vulkan-sunshine-route-gate-h264`
  使用 `nvh264dec`，`frames_rendered=240`、`frames_imported=238`、
  `average_render_fps=59.999`，runtime 明确输出
  `memory_route.route=cuda-vulkan-copy`、`direct_import_confirmed=false`、
  `copy_risk=gpu-copy-or-sync-risk`、`last_dmabuf_import=null`。
- 同日继续吸收 Sunshine 对 `gst-dma` 的直接启发：DMABUF import contract 现在不只记录
  format/modifier/plane，还会在 Vulkan image 创建后记录 `image_memory_type_bits`、
  `fd_memory_type_bits`、二者交集 `compatible_memory_type_bits`、最终
  `selected_memory_type_index` 和 `memory_allocation_size`。如果交集为 0，错误信息会直接带出
  image/fd 两侧 bitmask；这对应 Sunshine `vkGetMemoryFdPropertiesKHR` gate，能把
  “GStreamer caps 看起来是 DMA” 与 “目标 Vulkan device 真的能 import 这个 fd” 分开。
  `scripts/native-vulkan-visible-codec-smoke.sh` 也把这些字段写进 summary，后续 VA/DMABUF
  真机 smoke 不再需要手翻 runtime JSON。
- 同日继续把 Sunshine 的 render-node/device identity 思路落到 `gst-dma` telemetry：
  native Vulkan runtime 现在通过 `VK_EXT_physical_device_drm` 记录 selected physical device
  的 primary/render DRM major/minor、Linux `dev_t` 和可推导的 `/dev/dri/card*` /
  `/dev/dri/renderD*` 路径，并同时写入顶层 runtime 和 `video_runtime`。这不是把 DMABUF fd
  伪装成 render-node 匹配证明；它用于把“GStreamer/VA/DMABUF 来源在哪个 DRM device 上”
  与“Gilder 目标 Vulkan physical device 是哪个 DRM device”放到同一份证据里，后续 AMD/VA
  direct smoke 可以直接 gate 跨设备/隐式 copy 风险。本轮真实 Wayland `HDMI-A-1`
  `/tmp/gilder-vulkan-visible-h264-sunshine-drm-identity-final` 使用 `nvh264dec`，
  `frames_rendered=120`、`frames_imported=118`、`memory_route=cuda-vulkan-copy`、
  `direct_import_confirmed=false`、`selected_vulkan_drm_render_node=/dev/dri/renderD128`、
  `selected_vulkan_drm_render_dev_t=57984`、`last_dmabuf_import=null`，符合当前 NVIDIA
  GStreamer 路线仍是 CUDAMemory fallback、不是 DMABUF direct 的事实。
- H.264 direct 路线的 picture-layout probe 已从单一 `PROGRESSIVE` 扩展为
  `PROGRESSIVE`、`INTERLACED_INTERLEAVED_LINES`、`INTERLACED_SEPARATE_PLANES` 三种
  layout；`--probe-video` 的 H.264 profile matrix 会分别报告 layout。真实本机 probe
  `/tmp/gilder-vulkan-h264-layout-probe.json` 显示 NVIDIA RTX 4060 Laptop GPU 的 H.264
  Main/High progressive 与两种 interlaced layout 均可查询通过，Baseline 仍只有 progressive。
  `run_h264_ready_prefix_video` 现在根据 SPS `frame_mbs_only_flag` 和 bootstrap window 是否出现
  field picture 选择 layout，并把 `h264_picture_layout` 写入 runtime/smoke summary。
- 本轮验证：`cargo fmt -- --check`、`bash -n scripts/native-vulkan-visible-codec-smoke.sh`、
  `bash -n scripts/native-vulkan-h264-ready-prefix-video-smoke.sh`、默认 `cargo test`、
  `cargo test --features native-vulkan-gst-video`、`native-vulkan-gst-video` release build
  和 `native-vulkan-gst-cuda` release build 均通过。真实 Wayland `HDMI-A-1` H.264
  direct regression `/tmp/gilder-vulkan-h264-picture-layout-progressive-regression`
  为 `decoded/presented=120/120`、`h264_picture_layout=progressive`、
  `h264_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=243.404`。
  第二条 GStreamer visible route-gate
  `/tmp/gilder-vulkan-visible-h264-sunshine-dmabuf-contract` 为 `frames_rendered=120`、
  `frames_imported=118`、`memory_route=cuda-vulkan-copy`、`direct_import_confirmed=false`、
  `dmabuf_*_memory_type_bits=none`，符合本机 NVIDIA `nvh264dec` 仍走 CUDAMemory fallback 的事实。
- 同日 H.264 planner 已把 short-term reference 的内部 key 从单独 `frame_num` 扩展为
  `frame_num + field_kind`，并把 `field_pic_flag/bottom_field_flag` 贯穿到 reference snapshots、
  active DPB 和 `StdVideoDecodeH264ReferenceInfoFlags`。新增单测覆盖同一个 `frame_num` 的
  top/bottom field key 可并存，以及 Vulkan reference flags 正确设置 top/bottom 位。随后继续
  放开 H.264 interlaced/MBAFF 路线：High 8-bit parameter-set readiness 不再要求
  `frame_mbs_only_flag=true`，`frame_mbs_only=false` 时优先选择 interlaced layout；planner
  对 `field_pic_flag=true` 不再硬拒绝，并按 top/bottom field 选择 `PicOrderCntVal`，同一个
  `frame_num` 的互补场不会再触发 gaps-in-frame-num 推断。2026-06-22 真实 Wayland
  `HDMI-A-1` evidence `/tmp/gilder-vulkan-h264-interlaced-mbaff-visible` 使用 x264
  interlaced/MBAFF H.264 源，通过 `decoded/presented=60/60`、
  `h264_picture_layout=interlaced-interleaved-lines`、`h264_input_mode=streaming-queue`、
  `b_frames=38`、`max_reference_count=3`、`video_resource_memory_bytes=7536640`、
  `session_memory_bytes=2215936`、`bitstream_buffer_bytes=524288`。这证明 interlaced frame
  picture 可见路径已走通；真实 `field_pic_flag=true` field-coded 码流 smoke 仍未拿到，
  目前只有 planner/submit 侧单测覆盖。
- 同日继续把 H.264 任意连续的 DPB 语义补齐到 B/ref-list/MMCO/long-term 边界：
  long-term reference 的内部 key 从裸 `long_term_frame_idx` 扩展为
  `long_term_frame_idx + field_kind`，ref-list modification idc 2 和 MMCO 2 都按当前
  field/frame 语境把 `LongTermPicNum` 解析成具体 top/bottom/frame key；提交到 Vulkan STD
  时 `StdVideoDecodeH264ReferenceInfo.FrameNum` 对 long-term reference 改为写
  `long_term_frame_idx`，符合 Vulkan/H.264 STD 要求。MMCO 1/3 的短参考 `PicNumX`
  也修正为 `CurrPicNum - (difference_of_pic_nums_minus1 + 1)`，不再在 frame_num wrap
  后错误 modulo 成正 PicNum；这修复了 4K/240 x264 B/ref 流在 `15 -> 0` wrap 后
  `MMCO 1 requested unavailable short-term PicNum 11` 的真实失败。新增单测覆盖
  long-term field PicNum、field MMCO 1 精确删除、MMCO 1 wrap 删除 `frame_num=11`，
  同轮 `cargo test --features native-vulkan-gst-video` 通过 318 项，release build 通过。
  真实 Wayland `HDMI-A-1` 4K/240 任意入口 evidence：
  `/tmp/gilder-vulkan-h264-arbitrary-continuous-mmco-wrap`，生成源为
  `testsrc2-continuous-closed-gop-h264-high-b2-weightp2-weightb1`、`refs=3`、
  `bframes=2`、`arbitrary_entry_offset=0.35`、`require_loop_skip_replay=yes`，
  结果为 `decoded/presented=480/480`、`bootstrap_discarded=155`、`loop_skip=155`、
  `first_frame_idr=true`、`loop_first_non_idr_count=0`、`stream_dpb_slots=5`、
  `max_reference_count=4`、`bitstream_ring_wrap_count=43`、
  `video_resource_memory_bytes=62586880`。同轮 H.265 任意入口回归
  `/tmp/gilder-vulkan-h265-arbitrary-continuous-regression` 也通过 4K/240、`refs=2`、
  `bframes=2`、`decoded/presented=480/480`、`bootstrap_discarded=153`、
  `loop_skip=153`、`stream_dpb_slots=5`、`max_reference_count=4`、
  `average_present_fps=240.976`。H.264 这次仍是 present-limited
  (`average_present_fps=195.617`)，因此后续若把“完成”定义为稳定 240fps + audio/clock，
  仍需要继续做 pacing/long-duration/perf 采样；但 H.264/H.265 任意入口连续 decode/present
  功能 gate 已经跑通。
- 同日把 arbitrary-entry smoke 的性能采样直接接进 H.264/H.265 脚本：
  `--performance-snapshot` 会在 native Vulkan 进程运行时调用
  `scripts/performance-snapshot.sh --pid ... --keep`，并把 RSS/PSS/USS/Private_Dirty、
  CPU、NVIDIA 进程显存和 smaps 分类路径写入同一个 summary。真实 Wayland `HDMI-A-1`、
  3840x2160@240、非关键帧入口 `0.35s`、2400-frame replay 复测结果：H.264
  `/tmp/gilder-vulkan-h264-arbitrary-performance-keep` 为 `decoded/presented=2400/2400`、
  `playback_loop_count=9`、`loop_boundary_reset_count=8`、`bootstrap_discarded=155`、
  `loop_skip=155`、`queue_retained=0`、`bitstream_ring_wrap_count=214`、
  `average_present_fps=197.51976491979758`，smaps
  `/tmp/gilder-vulkan-h264-arbitrary-performance-keep/performance` 为
  `RSS/PSS/USS/Private_Dirty max=105144/70095/58636/26924 KiB`、平均 CPU `13.30%`、
  NVIDIA 进程显存 `130 MiB`。H.265 `/tmp/gilder-vulkan-h265-arbitrary-performance-keep`
  为 `decoded/presented=2400/2400`、`playback_loop_count=9`、
  `loop_boundary_reset_count=8`、`bootstrap_discarded=153`、`loop_skip=153`、
  `queue_retained=0`、`bitstream_ring_wrap_count=57`、
  `average_present_fps=240.1502442126708`，smaps
  `/tmp/gilder-vulkan-h265-arbitrary-performance-keep/performance` 为
  `RSS/PSS/USS/Private_Dirty max=103088/68051/56592/24660 KiB`、平均 CPU `10.90%`、
  NVIDIA 进程显存 `152 MiB`。跑完后 `ps` 未发现残留 `gilder-native-vulkan`/`gilderd`，
  `niri msg layers` 只剩 quickshell/dms layer。结论：任意入口连续、EOS replay、
  streaming queue、bitstream ring 和资源采样证据已成立；后续进入 AV1/场景壁纸前不能把
  H.264 4K/240 满帧性能债误写成完成。
- 同日 H.264 display ring 继续压 CPU 侧热循环：`NativeVulkanVideoRenderer` 支持多个
  descriptor set，H.264 双槽 display ring 在初始化时按槽位预绑定 Y/UV view，播放循环只选择
  对应 descriptor set，不再每帧 `vkUpdateDescriptorSets` 或把 cached-view texture 推入 retire
  队列。真实 Wayland `HDMI-A-1`、3840x2160@240、ref=1 证据
  `/tmp/gilder-vulkan-h264-prebound-descriptor-4k240-ref1` 为 `decoded/presented=480/480`、
  `average_present_fps=232.68396113217636`、`avg_descriptor_update_us=0`；5s performance
  `/tmp/gilder-vulkan-h264-prebound-descriptor-4k240-perf` 为 `decoded/presented=1200/1200`、
  `average_present_fps=233.90643962520952`、`RSS/PSS/USS/Private_Dirty max=106000/91369/86404/27424 KiB`、
  平均 CPU `15.60%`、NVIDIA 进程显存 `116 MiB`。H.265 Main10/P010 回归
  `/tmp/gilder-vulkan-h265-main10-renderer-regression-4k240` 仍为 `decoded/presented=480/480`、
  `average_present_fps=240.2474194054933`。结论：descriptor 预绑定是实测成立的小幅 CPU/提交面优化，
  但 H.264 4K/240 仍未达到 H.265 的稳定 240fps，下一步要继续拆 decode/copy/present overlap。
- H.265 visible/sequence submit 侧不再把 active DPB 简化成 `POC` 数组：新增
  `NativeVulkanH265ActiveDpbReference { poc, used_for_long_term_reference }`，并在
  `vkCmdBeginVideoCodingKHR` 的 begin reference slots 中用当前 entry 的 reference usage
  覆盖 active slot 状态，避免 planner 已识别 long-term reference、但 begin slot 的
  `StdVideoDecodeH265ReferenceInfoFlags.used_for_long_term_reference` 仍被写成 0。新增单测
  `h265_begin_slots_preserve_current_long_term_reference_flags` 覆盖该路径。验证：
  `cargo test --features native-vulkan-gst-video` 通过 308 个库测试、7 个 `gilderctl` 测试和
  16 个 `gilderd` 测试，release 构建通过；真实 Wayland `HDMI-A-1` 回归
  `/tmp/gilder-vulkan-h265-active-dpb-reference-state` 使用 720p/60、`refs=2`、`bframes=2`、
  非关键帧入口 `0.35s`，结果为 `decoded/presented=160/160`、`h265_input_mode=streaming-queue`、
  `bootstrap_discarded=36`、`loop_skip=36`、`queue_retained=0`、`max_reference_count=4`、
  `missed_frame_pacing_count=0`。真实 long-term coded stream smoke 仍未补齐。
- H.264 GPU-memory/native-wgpu 对照是另一条口径：真实 Wayland 证据
  `/tmp/gilder-native-wgpu.SWqa42` 使用 `gst-dmabuf`、`pipeline_kind=cuda-direct`、
  `video_last_memory_types=gst.cuda.memory`、`video_last_export_source=cuda-direct-vulkan-staging`，
  8s 采样 `Private_Dirty max=68928 KiB`、平均 CPU `26.80%`、平均 render
  `240.09fps`。它是连续 GStreamer 解码流，不会像 ready-prefix smoke 一样每个窗口强制
  `AU239 -> AU0` reset，因此不能用 ready-prefix 的可见 loop boundary 直接对比平滑度。
- GTK H.264 direct sink 仍作为守卫基线，但不是内存最优对照：`/tmp/gilder-wayland-video.D6hbCj`
  的 active phase 为 `nvh264dec`、`NV12`、`memory:CUDAMemory`、`sink-gpu-memory-caps`，
  `Private_Dirty max=114412 KiB`、平均 CPU `35.99%`、NVIDIA 进程显存 `448 MiB`。
  因此后续判断 native Vulkan 是否值得默认切换，应优先和 native-wgpu H.264 GPU-memory
  continuous path 同场景对比，而不是和 GTK sink 或 ready-prefix loop 视觉结果混在一起。
- `--run-clear` 已接入 logical device、swapchain、command buffer、semaphore/fence 和 clear present
  loop；同场景 `--duration 3 --target-fps 240` 跑到 720 frames，平均 239.996fps，swapchain 为
  `B8G8R8A8_UNORM`、1707x1067、3 images、FIFO present。
- `--type-support` 暴露完整壁纸类型矩阵：static/video/slideshow/scene-lite 已有 Vulkan render item
  入口；web/shader/playlist 仍按 helper/fallback/selection contract 推进。
- `--run-static` 已接入静态图片最小显示路径：使用 `image 0.25.10` 解码 PNG/JPEG/WebP，按
  `cover/contain/stretch/center/tile` CPU fit 到 swapchain 尺寸，通过 host-visible staging buffer
  copy 到 swapchain image。真实 Wayland smoke：`--duration 3 --target-fps 240 --source
  /tmp/gilder-vulkan-static.png --fit contain` 跑到 719 frames，平均 239.517fps，staging bytes
  7285476。当前仍是最小 copy path，下一步要换成 sampled texture + shader pass，并让静态壁纸
  render-on-change 后 idle。
- `--run-video` 已开始接入 video wallpaper type：消费 `VideoWallpaperPlan` 的 source、poster、
  fit、loop、muted、target FPS、decoder policy 和 start offset，复用 native Vulkan surface/swapchain
  生命周期并输出 video handoff telemetry；当前只渲染 poster/clear placeholder，不启动 GStreamer
  解码，也不使用 GStreamer sink 接管显示。下一步是 GStreamer appsink/DMABuf/GPU-memory frame
  handoff。
- `native-vulkan-gst-video` feature 已接入 GStreamer appsink 前端：`--run-video` 会启动
  `decodebin -> appsink`，只拉取 sample 和记录 caps/memory/decoder evidence，不使用 GStreamer
  显示 sink。真实 Wayland smoke 已观察到 `nvh264dec`、`video/x-raw(memory:CUDAMemory)`、
  `NV12` sample 和 appsink handoff active。
- native Vulkan video 已开始实际 texture import：当前实现了 NVIDIA 机器上的
  `CUDAMemory -> CUDA copy -> Vulkan external image planes -> NV12 shader sampling` 路径，
  由 native Vulkan render pass 合成到 swapchain。CUDA 只是一个 importer 实现，不是 video
  架构边界；AMD/Intel 后续必须补同级的 `DMABuf/VAAPI -> Vulkan external memory` importer，
  复用同一套 Vulkan Y/UV sampling 和 present。
- native Vulkan video importer 已补 10-bit/P010 可见采样：GStreamer sample meta 现在区分
  `NV12` 与 `P010_10LE`，P010 使用
  `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16` image format，plane view 使用
  `R16_UNORM` / `R16G16_UNORM`，CUDA external image 导入按 8-bit/16-bit channel
  分别选择 array format。VA/DMABuf scaffold 同步保留 `DRM_FORMAT_P010`、
  `DRM_FORMAT_R16` 和 `DRM_FORMAT_GR1616`，后续 AMD/Intel importer 可复用同一格式模型。
- `--probe-video` 已加入 native Vulkan CLI，用 `vkGetPhysicalDeviceQueueFamilyProperties2`
  枚举 Vulkan Video decode 扩展和 queue family，不创建 surface、不解码。2026-06-21 在
  `WAYLAND_DISPLAY=wayland-1` 下验证：NVIDIA GeForce RTX 4060 Laptop GPU 报告
  `video_decode_ready=true`，有 `VK_KHR_video_queue`、`VK_KHR_video_decode_queue`、
  H.264/H.265/AV1/VP9 decode 扩展，并在 queue family 3 暴露独立 `VIDEO_DECODE` queue；
  Intel Iris Xe 当前 Vulkan probe 中 `video_decode_ready=false`。
- `--probe-video` 已进一步查询 H.264 Vulkan Video profile/format capabilities：NVIDIA
  4060 的 baseline/main/high 8-bit 4:2:0 progressive 都可用，`max_coded_extent=4096x4096`，
  `max_level=5.2`，decode flags 为 `dpb-and-output-coincide`，NV12
  `G8_B8R8_2PLANE_420_UNORM` 同时支持 `video-decode-dst`、`video-decode-dpb` 和
  `sampled`。这证明 H.264 direct decode 到 sampled NV12 Vulkan image 是真实候选；
  但当前 4K/240 H.264 测试源 caps 为 level 6.1，高于驱动报告的 H.264 max level 5.2，
  因此 direct Vulkan Video 首版不能假设覆盖该 H.264 源，必须同时验证 H.265/AV1 direct
  path 或保留 CUDA/NVDEC fallback。
- `--probe-video` 已补 H.265/AV1 profile/format capabilities：NVIDIA 4060 的 H.265 main-8
  报告 `max_level=6.1`、`max_coded_extent=8192x8192`，NV12 sampled output 可用；H.265
  main-10 也可用，输出格式为 `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`。AV1
  main-8/main-10 均报告 `max_level=7.3`、`max_coded_extent=8192x8192`，main-8 的 NV12
  sampled output 可用，main-10 同样返回 10-bit 2-plane 420 sampled format。结论：4K/240
  direct Vulkan Video 首版应优先验证 H.265/AV1；H.264 level 6.1 源继续由 CUDA/NVDEC
  fallback 覆盖，直到证明驱动/源参数可落入 H.264 level 5.2。
- `--probe-video-session` 已进入真实 Vulkan Video session 创建/绑定阶段：2026-06-21 在
  `WAYLAND_DISPLAY=wayland-1` 下用 3840x2160 参数验证 H.265 main-8 和 AV1 main-8，均选中
  NVIDIA GeForce RTX 4060 Laptop GPU 的 `VIDEO_DECODE` queue family 3，成功调用
  `vkCreateVideoSessionKHR`、`vkGetVideoSessionMemoryRequirementsKHR`、
  `vkAllocateMemory` 和 `vkBindVideoSessionMemoryKHR`。H.265 session memory requirements
  为 4 个 bind、总计 33775616 bytes；AV1 为 5 个 bind、总计 14143488 bytes；二者均确认
  NV12 DPB/output/sampled format 可用。
- `--probe-video-session --allocate-video-images` 已继续验证 Vulkan Video resource image：
  2026-06-21 同样在 `WAYLAND_DISPLAY=wayland-1`、3840x2160 参数下，H.265 main-8 和
  AV1 main-8 均成功创建一张 `G8_B8R8_2PLANE_420_UNORM`、8 array layers、usage 为
  `video-decode-dst|video-decode-dpb|sampled` 的 2D array image，绑定 device-local memory
  并创建 2D array image view。该 image 的 memory requirement 为 100139008 bytes、
  alignment 65536、`imageCreateFlags=mutable-format`。
- `--probe-video-session --allocate-bitstream-buffer` 已继续验证 Vulkan Video decode input
  buffer：2026-06-21 在 H.265 main-8 和 AV1 main-8 3840x2160 resource smoke 中，均成功创建
  8MiB `VIDEO_DECODE_SRC_KHR` buffer，挂载同一 `VkVideoProfileListInfoKHR`，绑定
  host-visible/coherent memory，按 driver 的 256-byte bitstream alignment 对齐，并映射写入
  256 bytes。该 buffer 的 memory requirement 为 8388864 bytes、alignment 256、
  `memory_type_bits=31`。这已经越过“只创建 session/resource image”的阶段，下一步是把
  已创建的 session parameters 接入 command buffer，并提交 `vkCmdDecodeVideoKHR`。
- `--probe-video-session --extract-bitstream --source <h265.mp4>` 已把 native Vulkan Video
  输入推进到真实 encoded front-end：2026-06-21 在 `WAYLAND_DISPLAY=wayland-1` 下，用本机
  生成的 3840x2160@240 H.265 MP4 验证 `qtdemux ! h265parse` 只负责容器 demux 和 parser，
  输出 `stream-format=byte-stream, alignment=au` 的 encoded access unit；probe 识别出
  VPS/SPS/PPS/IDR NAL，并把选中的 173754-byte AU 写入 `VIDEO_DECODE_SRC_KHR` buffer
  (`mapped_write_source=extracted-encoded-video-unit`，hash=5201191167619689341)。
- direct H.264 已从 capability probe 推进到 session/resource/bitstream gate：`--video-codec h264`
  选择 H.264 High 8-bit progressive profile，`scripts/native-vulkan-h264-bitstream-smoke.sh`
  通过 `qtdemux ! h264parse ! appsink` 输出 `stream-format=byte-stream, alignment=au`，
  真实创建 H.264 `VkVideoSessionKHR`、NV12 decode resource image、
  `VIDEO_DECODE_SRC_KHR` bitstream buffer 和 `VkVideoSessionParametersKHR`。
  native parser 会读取 SPS/PPS 并转换成 `StdVideoH264SequenceParameterSet` 与
  `StdVideoH264PictureParameterSet`；smoke 现在要求 `session_parameters_requested=true`、
  `session_parameters_created=true`、`source=native-rust-h264-sps-pps-to-vulkan-std`。
  2026-06-21 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60
  `/tmp/gilder-vulkan-h264-bitstream.iVMCh1`，4K/240 level 5.2
  `/tmp/gilder-vulkan-h264-bitstream.fs7CCw`；后者报告 `profile_idc=100`、
  `level_idc=52`、`framerate=240`、`mapped_write_bytes=217455`、
  `session_memory_bytes=16945152`、`video_resource_memory_bytes=100139008`。
- H.264 direct first-frame 已接到真实 Vulkan Video command buffer：
  `scripts/native-vulkan-h264-first-frame-smoke.sh` 解析首个 IDR picture 的所有 slice
  offset，填充 `StdVideoDecodeH264PictureInfo`、`VkVideoDecodeH264PictureInfoKHR`、
  `StdVideoDecodeH264ReferenceInfo` 和 setup DPB slot，录制
  `vkCmdBeginVideoCodingKHR`、`vkCmdControlVideoCodingKHR(RESET)`、
  `vkCmdDecodeVideoKHR`、`vkCmdEndVideoCodingKHR`，再把 NV12 decode output plane 0/1
  copy 到 host-visible readback buffer。2026-06-21 `WAYLAND_DISPLAY=wayland-1` 证据：
  720p/60 `/tmp/gilder-vulkan-h264-first-frame.AYMakX`
  (`first_frame_decode.completed=true`, `slice_count=11`, Y/UV 非零
  `921600/460800`)，4K/240 level 5.2 `/tmp/gilder-vulkan-h264-first-frame.lQiwMa`
  (`slice_count=20`, `src_buffer_range=217600`, Y/UV 非零
  `8294400/4147200`)。额外采样 gate
  `/tmp/gilder-vulkan-h264-first-frame.GJildG` 证明 H.264 decoded NV12 layer 也能进入
  native Vulkan shader sampling (`sample_copied=true`)。
- H.264 direct 已继续推进到 all-IDR multi-frame gate：
  `scripts/native-vulkan-h264-idr-prefix-smoke.sh` 生成 `keyint=1` 的 High 8-bit all-IDR
  源，并通过 `--decode-h264-idr-prefix N` 把多个 AU 按 driver bitstream offset/size
  alignment 拼入同一个 `VIDEO_DECODE_SRC_KHR` buffer，顺序录制多次
  `vkCmdDecodeVideoKHR`，最终 readback 最后一帧 NV12。2026-06-21 真实
  `WAYLAND_DISPLAY=wayland-1` 证据：720p/60
  `/tmp/gilder-vulkan-h264-idr-prefix.kKR6lh`，4K/240 level 5.2
  `/tmp/gilder-vulkan-h264-idr-prefix.7H4DV3`；后者 `decoded_frame_count=8`、
  frame offsets 为 `[0,217600,329216,441600,553984,666624,779264,892160]`，
  `reset_control_count=8`，Y/UV 非零 `8294400/4147183`。这一步证明 H.264 direct
  不是只会解首帧，但仍是 all-IDR gate；当前 H.264 direct 边界改为 P/B reference
  tracking、无 per-frame reset 的 DPB 维护、visible surface presentation 和 frame pacing。
- H.264 direct 已从 all-IDR gate 推进到普通 IDR+P ready-prefix gate：
  `scripts/native-vulkan-h264-ready-prefix-smoke.sh` 生成 `bframes=0/ref=1/keyint>prefix`
  的 High 8-bit 源，`--decode-h264-ready-prefix N` 会解析 P slice 的 active L0 reference
  count、reference-list-modification flag 和 reference marking，生成
  `h264_decode_reference_plan[]`，并在 `vkCmdDecodeVideoKHR` 中为 P 帧传入真实
  `reference_slots`。2026-06-21 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60
  `/tmp/gilder-vulkan-h264-ready-prefix.U6E7hC` 和 4K/240 level 5.2
  `/tmp/gilder-vulkan-h264-ready-prefix.e1aTOo`；后者 `decoded_frame_count=8`、
  `non_idr_frames=7`、`reference_frames=7`、`reset_control_count=1`、
  `reference_plan_dpb_slots=2`，frame reference counts 为 `[0,1,1,1,1,1,1,1]`，
  planned DPB slots 为 `[0,1,0,1,0,1,0,1]`，Y/UV 非零 `8294400/4147194`。
  该阶段边界是 B slice、reference list modification、adaptive MMCO、
  long-term reference 和任意入口点 DPB 重建；后续 streaming queue 路径已继续推进其中一部分。
- H.264 direct 已接到同一套真实 Wayland visible ready-prefix path：新增
  `--run-h264-ready-prefix-video` 和
  `scripts/native-vulkan-h264-ready-prefix-video-smoke.sh`。GStreamer 只负责
  `qtdemux+h264parse+appsink` AU 抽取，H.264 picture decode 由 Vulkan Video queue family 3
  执行 `vkCmdDecodeVideoKHR`，decoded NV12 array layer 由 present queue family 0 的 native
  Vulkan shader 采样到 Wayland swapchain。2026-06-21 `WAYLAND_DISPLAY=wayland-1`、
  `HDMI-A-1` 证据：720p/60 ref=2 `/tmp/gilder-vulkan-h264-ready-prefix-video.faL4eZ`
  为 `decoded_frame_count=8`、`presented_frame_count=8`、`max_reference_count=2`；
  4K/240 ref=2 `/tmp/gilder-vulkan-h264-ready-prefix-video.Jy9iXF` 为 240 decode/present、
  `stream_dpb_slots=3`、`video_resource_memory_bytes=37552128`、
  `bitstream_buffer_bytes=435200`；480-frame loop
  `/tmp/gilder-vulkan-h264-ready-prefix-video.S305L5` 验证 `playback_loop_count=2`、
  `loop_boundary_reset_count=1`。这一步功能形态已追平 H.265 visible ready-prefix；
  后续输入方向已收敛到 `streaming-queue`，`ready-prefix-spool` 只作为历史 CLI 值被拒绝。
  `--h264-input streaming-queue` 在 4K/240 evidence
  `/tmp/gilder-vulkan-h264-ready-prefix-video.c4GQ0l` 中验证 `h264_input_mode=streaming-queue`、
  `queue_pulled=240`、`queue_retained=0`、`p_frames=239`、`max_reference_count=2`。性能上
  average present 仍约 206-212fps，后续要继续优化 present/sync 到稳定 240Hz。
- `--probe-video-session --extract-bitstream` 已继续把 H.265 VPS/SPS/PPS 转成 Vulkan STD
  session parameters：2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240
  H.265 Main 源上，native parser 真实读取 profile flags、VPS/SPS DPB ordering、SPS VUI
  和 PPS 基础字段，构造 `StdVideoH265VideoParameterSet`、`StdVideoH265SequenceParameterSet`
  和 `StdVideoH265PictureParameterSet`，并成功创建 `VkVideoSessionParametersKHR`
  (`session_parameters_created=true`, VPS/SPS/PPS count 均为 1)。
- `--probe-video-session --extract-bitstream --video-codec av1` 已进入 AV1 Vulkan STD
  session parameters 阶段：`matroskademux/qtdemux ! av1parse ! appsink` 输出
  `stream-format=obu-stream, alignment=tu` 的 temporal unit；native parser 扫描 OBU，
  解析 sequence header 的 profile/operating point/extent/tool flags/color config，确认
  AV1 Main 8-bit 4:2:0 且无 film grain 后构造 `StdVideoAV1SequenceHeader`，并真实创建
  `VkVideoSessionParametersKHR`。2026-06-21 最新证据
  `/tmp/gilder-vulkan-av1-bitstream.ivMR9n`：`session_parameters_created=true`、
  `source=native-rust-av1-sequence-header-to-vulkan-std`、sequence extent 为 640x368；
  selected temporal unit 现在还会报告 frame/tile readiness，当前为 sequence-header + frame OBU，
  `av1_decode_candidate=true`、`av1_frame_payload_bytes=2697`、`av1_first_frame_header_obu_offset=13`。
- H.265/AV1 Main10 已推进到 session/resource/bitstream gate，而不是只停留在 capability
  probe：`--video-codec h265-main-10|av1-main-10` 现在会选择 10-bit profile 和
  `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`。2026-06-21 真实
  `WAYLAND_DISPLAY=wayland-1` 证据 `/tmp/gilder-vulkan-h265-main10-bitstream.Y0bB5M`
  显示 H.265 Main10 成功创建 P010-like resource image、上传 encoded AU、创建
  `VkVideoSessionParametersKHR`，`session_parameters_codec=h265-main-10`、
  `video_image_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`；2026-06-22 继续把
  AV1 Main10 从 session/submit-candidate 推到真实 first-frame `vkCmdDecodeVideoKHR`
  readback gate。`/tmp/gilder-vulkan-av1-smoke-first-frame-main10` 和
  `/tmp/gilder-vulkan-av1-smoke-first-frame-main10-4k` 均为
  `first-frame-decode-and-output-readback-completed`，并报告
  `first_frame_readback_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`。同日新增
  P010 plane view sampling：AV1 Main10 4K script gate
  `/tmp/gilder-vulkan-av1-p010-sampling-script`、H.265 Main10
  `/tmp/gilder-vulkan-h265-main10-p010-sampling.CGax7L` 均为
  `first-frame-decode-output-sampled-and-readback-completed`，RGBA readback 非零且
  unique=256。当前 10-bit 边界收敛到连续 DPB/display handoff 和 direct present path，
  不能复用现有 NV12/8-bit visible smoke。
- `--probe-video-session --decode-first-frame --source <h265.mp4>` 已进入真实 H.265
  Vulkan Video command buffer：2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、
  3840x2160@240 H.265 Main 源上，probe 解析 IDR slice offset 2444，使用 8-layer
  NV12 coincident DPB/output image 的 layer 0 作为 `dst_picture_resource` 和 setup reference
  slot，按 256-byte alignment 提交 173824-byte src range，并成功完成
  `vkCmdBeginVideoCodingKHR`、`vkCmdControlVideoCodingKHR(RESET)`、`vkCmdDecodeVideoKHR`、
  `vkCmdEndVideoCodingKHR`、`vkQueueSubmit` 和 `vkQueueWaitIdle`
  (`first_frame_decode.completed=true`)。
- 同一个 `--decode-first-frame` smoke 已继续验证 decode output image 内容：首帧 decode 后将
  NV12 array layer 0 追加 `TRANSFER_SRC` usage，转为 `TRANSFER_SRC_OPTIMAL`，把 plane 0/1
  copy 到 host-visible readback buffer 并记录 hash/非零数/min/max/unique。2026-06-21 的真实
  Wayland 结果为 `output_readback.copied=true`，Y plane 8294400 bytes、
  hash=8710880026335779165、unique=256，UV plane 4147200 bytes、
  hash=8699452048464794797、unique=169，combined hash=17815028520596919621。这一步证明
  driver 不只接受 command buffer，decode 后的 NV12 plane 数据也可从 Vulkan image 读出。
- `--probe-video-session --sample-decoded-first-frame --source <h265.mp4>` 已把 H.265 decode
  output 接到现有 native Vulkan NV12 shader sampling：2026-06-21 在
  `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main 源上，probe 发现
  video decode queue family 3 只有 `transfer|sparse-binding|video-decode`，因此同时创建
  graphics queue family device queue，并把 NV12 video resource image 按需设为 concurrent
  sharing。decode queue signal semaphore 后，graphics queue 等待该 semaphore，将 decoded
  NV12 layer 0 的 plane 0/1 创建为 `R8_UNORM`/`R8G8_UNORM` sampled view，复用已有 Y/UV
  shader 渲染到 3840x2160 `R8G8B8A8_UNORM` offscreen color target，再 copy 到 readback
  buffer。真实结果为 `output_sampling.rendered=true`、`copied=true`、RGBA 33177600 bytes、
  hash=7109389899594476375、nonzero=27480639、unique=256。这一步证明 Vulkan Video
  输出已经能不经 CPU NV12 copy 进入 Vulkan shader 合成；下一步是连续帧 decode/display、
  visible surface smoke 和 frame pacing。
- H.265 encoded 前端已为 continuous decode 补 AU timeline telemetry：`--extract-bitstream`
  现在会继续收集 `--bitstream-samples` 个 access unit，同时仍选择第一个带 VPS/SPS/PPS 的
  IDR AU 作为首帧 decode 输入。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、
  3840x2160@240 H.265 Main 源上，`--bitstream-samples 8` 真实输出 8 个 AU、
  total bytes=369716、selected index=0；AU0 为 173754 bytes、PTS 0ms、duration 4ms、
  IDR `idr-n-lp`、slice_type=2、hash=5201191167619689341；AU1..AU7 为 `trail-r`、
  slice_type=1，PTS 为 4/8/12/16/20/25/29ms，POC LSB 为 1..7。完整
  `--sample-decoded-first-frame --bitstream-samples 8` 同时验证首帧 decode/readback/sampling
  仍通过，`output_sampling.rgba_hash=7109389899594476375`。
- 同一 AU telemetry 已继续解析 H.265 short-term reference picture set：真实 4K/240 源的
  P-slice 不是单参考链。AU1 的 used negative delta POC 为 `[-1, -16]`，AU2 为
  `[-1, -2, -19, -292]`，AU3/AU4/AU6/AU7 为 `[-1, -2, -21]`。这意味着连续帧 direct
  decode 不能只把前一帧放进 reference slot；下一步要先建立 DPB slot/ring、POC 跟踪和
  RefPicSetStCurrBefore/After 到 Vulkan reference slot index 的映射，再提交第二帧及后续帧。
- `h265_decode_reference_plan` 已把上述 AU/RPS 转成 DPB/POC 可提交性计划：按当前 8-slot
  smoke，AU0 规划到 slot 0 且 `ready_for_decode_submit=true`；AU1 需要 POC 0 和 POC -15，
  其中 POC 0 可由 AU0/slot0 提供，但 POC -15 不在当前 decode window，故 AU1
  `ready_for_decode_submit=false`；后续 AU 因 AU1 未进入 DPB 且还引用窗口外 POC，也不能直接
  提交。结论是：当前真实 4K/240 源不能用“IDR 后直接解第二帧”的简单 smoke 验证 continuous
  decode。下一步应补 closed-GOP/自包含 H.265 4K/240 测试源，或实现明确的缺失 reference
  策略后再提交多帧 `vkCmdDecodeVideoKHR`。
- 已补 reference-ready 基准源和 gate：`--extract-bitstream` 现在输出
  `h265_decode_ready_count`、`h265_decode_ready_prefix_count`、
  first-unready AU 和 missing POC；`--require-h265-ready-prefix N` 可把该条件变成 probe
  失败条件。`scripts/native-vulkan-h265-ready-prefix-smoke.sh` 会生成 H.265 Main
  3840x2160@240 short-GOP 源（`keyint=2`、no-B、HRD off），再运行真实 native Vulkan
  session/bitstream probe。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1` 验证 8 个 AU、
  ready-prefix=8、`session_parameters_created=true`，为下一步多帧 `vkCmdDecodeVideoKHR`
  提供不受窗口外 reference 干扰的基准输入。该轮还把非预测 SPS short-term RPS 接入
  `StdVideoH265SequenceParameterSet.pShortTermRefPicSet`；extract-only probe 遇到暂不支持的
  参数集时记录 `session_parameters_error`，不再丢失 AU/RPS telemetry。
- `--decode-h265-ready-prefix N` 已开始把 ready-prefix AU 真正提交给 Vulkan Video：
  probe 会把多个 AU 按 `min_bitstream_buffer_offset_alignment` 和
  `min_bitstream_buffer_size_alignment` 排进同一个 `VIDEO_DECODE_SRC_KHR` buffer，按
  `h265_decode_reference_plan` 的 DPB slot 创建 setup/reference slots，连续录制
  `vkCmdDecodeVideoKHR`，最后把末帧 NV12 layer copy 到 host readback。2026-06-21 在
  `WAYLAND_DISPLAY=wayland-1` 和同一 3840x2160@240 H.265 Main short-GOP 源上，2-frame
  IDR+P direct decode 通过：AU0 写 slot0，AU1 写 slot1 并引用 slot0，final readback
  layer1 的 Y/UV unique=198/248，hash=6114663086929905156/6011523946105094791。
  随后修正 repeated-IDR 的 DPB slot allocator：IDR 清空软件 DPB 后 next slot 回到 0，
  repeated short-GOP 因此复用 slot0/slot1；同时在非首帧 IDR 前记录
  `vkCmdControlVideoCodingKHR(RESET)`。同日真实验证 8-frame ready-prefix decode 通过：
  ready-prefix=8、decoded=8、reset_count=4、AU7 readback layer1，Y/UV unique=205/256，
  hash=11542476098458954487/10292639723071029932。脚本 `--decode-prefix` 要求 readback
  非单值，避免把“命令完成但画面无效”的路径误判为通过。
- H.265 P 帧重复/闪烁根因已确认并固化：`VkVideoDecodeH265PictureInfoKHR::pSliceSegmentOffsets`
  对 Annex-B 输入应指向 slice segment 所在的 3-byte start code `00 00 01`，不是 NAL
  payload/header offset，也不能简单取 4-byte start code 的第一个 0。对 `00 00 00 01`
  前缀，正确值是 `payload_offset - 3`。错误偏移会让 `vkCmdDecodeVideoKHR` 和 decode query
  仍报告完成，但 AU1/P 帧的 sampled/readback hash 会重复 AU0 或上一帧，表现为 P 帧不变化、
  闪烁或抖动。实现已集中到 `native_vulkan_h265_annex_b_slice_segment_offset()` 和
  `NativeVulkanH265NalPayload::slice_segment_offset`；单测
  `uses_three_byte_h265_start_code_for_slice_segment_offset` 固化 3-byte/4-byte 两种前缀行为。
  早期通过 `GILDER_VULKAN_H265_SLICE_OFFSET_ADJUST=-3` 证明该方向，当前默认逻辑已不需要该
  手工调整。
- `--sample-h265-ready-prefix` 已把多帧 direct decode 的末帧 NV12 output 接到现有 Vulkan
  shader sampling/offscreen render path。decoded texture 的 plane image view 和 shader
  read barrier 现在按实际 `base_array_layer` 创建，避免多帧 DPB/output 复用时仍假设 layer0。
  `scripts/native-vulkan-h265-ready-prefix-smoke.sh --sample-prefix` 会要求 sampled RGBA
  readback 非单值，并要求 sampled layer 与 final readback layer 一致。2026-06-21 在
  `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main short-GOP 源上，
  8-frame direct decode + sampling 通过：result=`h265-ready-prefix-decode-output-sampled-and-readback-completed`、
  decoded=8、reset_count=4、AU7 readback/sample layer1、RGBA hash=14093713610652448641、
  RGBA unique=256、nonzero=24967096。
- `--sample-h265-ready-prefix-sequence` 已把上述 sampled output 从末帧验证扩展到逐帧验证：
  每个 AU decode 后立即 readback + shader sample，并在下一帧 decode 前把被采样 layer 从
  shader-read layout 显式转回 `VIDEO_DECODE_DPB_KHR`，覆盖 slot 复用和 reference 继续使用的
  同步路径。脚本 `--sample-prefix-sequence` 要求 sequence 数量等于 decode-prefix、每帧
  NV12/RGBA readback 非单值、sampled layer 序列与 decode frame layer 序列一致。2026-06-21 在
  `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main short-GOP 源上，
  8-frame sequence decode + sampling 通过：result=`h265-ready-prefix-decode-output-sequence-sampled-and-readback-completed`、
  sampled_sequence_count=8、layers=`[0,1,0,1,0,1,0,1]`、distinct RGBA hashes=4、
  每帧 RGBA unique=256。
- sequence smoke 现在还记录逐帧 timing/pacing telemetry：每帧包含 PTS delta、
  decode submit/wait、NV12 readback、RGBA sampling/readback 和 total frame elapsed；
  sequence 级别包含 max/avg 与 PTS delta min/max，脚本会把这些字段作为 gate。2026-06-21
  同一真实 4K/240 source 上，PTS delta min/max=4/5ms，max decode submit/wait=5951us，
  max NV12 readback=38720us，max RGBA sampling+readback=65580us，avg debug frame=92997us。
  这些 frame timings 包含 host readback 验证成本；它们用于定位 debug smoke 的瓶颈，不代表
  后续 visible swapchain path 的 240fps present 成本。
- sequence smoke 同时补了 render-only telemetry：每帧 readback 验证后，复用同一个 offscreen
  color target 和 `NativeVulkanVideoRenderer` 对对应 decoded NV12 layer 再做一次 shader render，
  但不做 CPU copy/readback。2026-06-21 同一真实 4K/240 source 上，render_sequence_count=8，
  layers=`[0,1,0,1,0,1,0,1]`，PTS delta min/max=4/5ms，average render-only=934us，
  max render-only=1559us。这条证据更接近下一步 visible swapchain/present 的渲染成本，并确认
  当前 90ms 级 debug frame 主要来自 host readback 验证链路。
- `native-vulkan-gst-video` 已补 `GstVAMemory -> vaExportSurfaceHandle(DRM PRIME) -> Vulkan`
  importer scaffold，作为 Intel/AMD VA/DMABuf 路径的基础。当前混合 GPU 机器上 VA decoder
  默认会先探测 NVIDIA DRM 设备并打印 `unsupported drm device by media driver: nvid`；
  指定 Intel render node `/dev/dri/renderD129` 时显式
  `qtdemux ! h264parse ! vah264dec ! VAMemory ! fakesink` 可谈通，但项目内
  `decodebin -> appsink` 仍会 not-negotiated。VA/DMABuf 路线后续要改成显式 codec pipeline
  或补 allocator/render-node 协商；这不是 NVIDIA direct 的主线 blocker。Sunshine 对这里的
  直接启发是：不能只看 `memory:DMABuf`/`VAMemory`，还必须把 DRM PRIME descriptor/object、
  modifier plane count 和目标 Vulkan device 的 modifier 支持一起作为 importer gate。
- 4K/240 测试使用明确的 `3840x2160@240` H.264 源，不再用低清源判断画质。当前真实
  Wayland 证据来自 HDMI-A-1：该输出在 niri 中是 `2560x1600@239.999`、scale 1.5，
  所以这是 4K source 到 2560x1600@240 surface 的 downscale 验证，不是 4K 输出验证。
  最新 20s run：`average_render_fps=239.947`、`frames_rendered=4799`、
  `frames_imported=4778`、`eos_messages=0`、`segment_done_messages=2`、
  `last_sample_pts_delta_ms=4`、`last_import_size=3840x2160`。
- visible codec smoke 已覆盖 H.264、AV1 和 H.265 Main10：新增
  `scripts/native-vulkan-h264-visible-video-smoke.sh`、
  `scripts/native-vulkan-av1-visible-video-smoke.sh` 和
  `scripts/native-vulkan-h265-main10-visible-video-smoke.sh`，均以 GStreamer
  demux/decode/appsink 为前端，native Vulkan importer/shader/swapchain 负责可见输出。
  2026-06-21 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 证据：H.264 720p/240
  `/tmp/gilder-vulkan-visible-h264.dqQnsN`，4K/240
  `/tmp/gilder-vulkan-visible-h264.K0XXrj`；AV1 640x368/60
  `/tmp/gilder-vulkan-visible-av1.fBQmOz`，4K/60
  `/tmp/gilder-vulkan-visible-av1.yAKhDg`；H.265 Main10 640x368/60
  `/tmp/gilder-vulkan-visible-h265-main10.GxYmkr`，4K/60
  `/tmp/gilder-vulkan-visible-h265-main10.0nZH7D`。这组证据验证的是第二条路线的
  native Vulkan visible importer/present，不把 H.264/AV1/Main10 误标为 direct Vulkan
  Video picture-info decode 已完成。
- loop 使用 segment seek：启动顺序为 `Paused -> SEGMENT seek -> Playing`，收到
  `SegmentDone` 后立即 seek 回 0，避免短视频到 EOS 后硬切造成末尾抖动/卡顿。
- 建立最小 native Vulkan layer-shell renderer：clear/static/shader。
- 接入同一 render plan，不新增 manifest 分支。
- 验证单输出、多输出、resize、output selection、pause/release。
- 与类型线并行接入 shader、scene-lite 和 Web helper frame handoff。
- Video interop 不再作为 wgpu 分支实验；GStreamer appsink/DMA handoff 的目标实现是 native
  Vulkan importer，并用历史 native-wgpu/GStreamer CUDA copy 数值做同场景对照。

### Phase 4: Vulkan video/Web interop

- 在 `--run-video` lifecycle/telemetry 和 `native-vulkan-gst-video` appsink evidence 基础上，
  将 importer 明确拆成多个同级实现：NVIDIA `CUDAMemory/CUDA`、AMD/Intel
  `DMABuf/VAAPI`、可选 `GL/EGLImage`、Vulkan Video 或 libavcodec + external memory。
  GStreamer 可以继续负责 demux、硬解选择、音频和时钟，但最终 present 必须由 native
  Vulkan swapchain/render pass 完成。
- NVIDIA direct 的下一步是把已验证的 H.265/H.264 `VkVideoSessionKHR`、NV12 video resource
  image、真实 encoded AU、`VIDEO_DECODE_SRC_KHR` bitstream buffer、
  `VkVideoSessionParametersKHR` 和 visible ready-prefix decode/display 扩展成完整持续播放：
  GStreamer 或等价前端只负责 demux/parser/audio/clock，Vulkan Video 模块负责 picture info、
  reference slots 和 queue 同步，再复用 native Vulkan NV12 shader 合成到 visible swapchain。
  H.264/H.265 High/Main 8-bit 已有 4K/240 任意入口 visible direct gate，覆盖 B 帧、
  多参考、ref-list/MMCO、loop skip replay 和 fixed-capacity bitstream ring；H.265
  Main10/P010 和 AV1 Main10/P010 也已推进到真实 Wayland 4K/240 任意入口 direct visible
  gate。后续 video 侧还应补长时段稳定 240fps、audio/clock 和更多真实码流采样。
  AV1 direct 当前采用逻辑 reference-name slot planner 和 show-existing handoff，优先保证
  连续可见正确性；物理 DPB slot 压缩和更真实码流覆盖留作后续优化。CUDA copy path 只保留为
  fallback 和对照基线。
- 成功标准是同场景优于历史 native-wgpu/GStreamer CUDA copy 路线，而不是理论零拷贝。
- 2026-06-22 的 H.264 display-copy handoff 证明了一个重要边界：双槽 NV12 display
  ring 可以把 H.264 decode-ahead submit 从 hazard-skip 状态推进到 `2399/2399`，并在
  `HDMI-A-1` 真实 Wayland 上保持任意入口连续可见；但复杂 4K/240 H.264 仍只有
  `207.34fps`，同时额外占用约 `25.6MB` Vulkan image memory。H.265 同源形态仍稳定在
  `239.83fps`。因此后续 H.264 性能主线应转向固定帧槽、descriptor/present ring、
  timeline/fence 管理和更深 decode/present decoupling；全量 DPB->display copy 只能作为
  hazard 规避实验，不是最终零拷贝/低内存目标。
- 同日继续把 H.265 Main10 从 first-frame/readback 推进到 visible ready-prefix：H.265 smoke
  支持 `--bit-depth 10`，CLI 用 `--video-codec h265-main-10` 创建 Main10 session、
  `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16` P010 picture/resource，并由 native Vulkan
  shader 采样到 Wayland swapchain。真实 Wayland `HDMI-A-1` 4K/240 证据
  `/tmp/gilder-vulkan-h265-main10-visible-p010-4k240` 为 `decoded/presented=480/480`、
  `average_present_fps=240.32978160780624`、`playback_loop_count=2`、
  `video_resource_memory_bytes=75104256`、`session_memory_bytes=46309376`。
  H.264 同轮实验确认了性能/内存边界：`GILDER_H264_DISPLAY_HANDOFF=direct` 可以去掉
  25.6MB display ring，但 4K/240 ref=1 只有
  `/tmp/gilder-vulkan-h264-direct-sampled-4k240-ref1` 的 `211.867fps`；双 present queue
  (`GILDER_H264_PRESENT_QUEUE_COUNT=2`) 和更深 async present 在本机也未提升，默认保守回到
  `h264_present_queue_count=1`、`h264_async_present_depth=1`。最新默认 H.264 4K/240
  `/tmp/gilder-vulkan-h264-telemetry-default-4k240-ref1` 为 `decoded/presented=480/480`、
  `average_present_fps=230.37179368303578`、`queue_retained=0`。结论仍是：
  H.264 稳定 240fps 未完成，瓶颈在 H.264 decode/display/present critical path，
  不是 streaming packet retention。
- 后续调度重构应借鉴成熟硬件视频路径的“frame pool + ownership”模型，而不是继续
  对现有 present worker 小修小补。Sunshine 本地 Vulkan/FFmpeg 路径的可借鉴点包括：
  `AVHWFramesContext` 预分配硬件帧池、DMABuf 导入时携带 explicit modifier/plane layout、
  source/target 变化时才更新 descriptor、命令 buffer ring、timeline semaphore 把 GPU
  写入完成状态交还给 FFmpeg，以及 source image 延迟销毁。Gilder 的 decode/present 仍走
  自有 Vulkan Video/swapchain，但 H.264/H.265/AV1 应共享同一套 slot state：
  `Free -> DecodeWriting -> DecodeReady -> DisplayCopyWriting -> Sampling -> Presented/Retired`，
  并由 timeline value 或 per-slot fence 回收 bitstream range、DPB layer、display ring slot
  和 swapchain image。
- 2026-06-22 H.264 已完成第一步 slot ownership 化：display-ring 路径改为 per-frame
  acquire semaphore/fence，并在复用 display slot 前等待上一次采样该 slot 的 frame fence。
  真实 Wayland `HDMI-A-1` 证据 `/tmp/gilder-vulkan-h264-display-slot-fence-4k240-ref1` 为
  `decoded/presented=480/480`、`average_present_fps=230.31172461134605`、
  `h264_present_result_wait_elapsed_us=1929885`、平均 fence wait 约 `0.89us`。这验证了
  所有权 guard 没有引入明显阻塞，但它不是 240fps 突破。更深 async present 的负面结果也已
  固化：`GILDER_H264_ASYNC_PRESENT_DEPTH=2`
  `/tmp/gilder-vulkan-h264-per-frame-fence-depth2-4k240-short-seq` 降到
  `219.4879316010344fps`，因为单队列 mutex 让 `avg_submit_us=4175.98`；双 present queue
  `/tmp/gilder-vulkan-h264-per-frame-fence-dual-present-4k240-short-seq` 20s 超时且 runtime
  为空。下一步应迁移 timeline semaphore/range scheduler，而不是继续提高当前 worker depth。
- H.265 Main10 作为对照仍稳定：同工作树真实 Wayland
  `/tmp/gilder-vulkan-h265-main10-after-h264-framepool-fence-4k240` 为
  `decoded/presented=480/480`、`average_present_fps=240.3833285970556`、
  P010 `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`。
- 继续下探 H.264 layout 成本后，新增 `GILDER_H264_RESOURCE_LAYOUT=general`
  实验入口：H.264 decode resource image 和 display-copy source/final layout 可保持
  `GENERAL`，runtime/summary 增加 `h264_resource_image_layout`。真实 Wayland
  `/tmp/gilder-vulkan-h264-resource-general-4k240-ref1` 为
  `decoded/presented=480/480`、`average_present_fps=233.11475907497862`，
  但重跑 `/tmp/gilder-vulkan-h264-resource-general-layout-field-4k240-ref1` 为
  `232.52402677308388fps`；该方向证明 layout churn 有成本，但收益波动，不足以解释
  H.264 与 H.265 稳 240 的差距。`--probe-video` 同时确认本机 NVIDIA H.264 只暴露
  `dpb-and-output-coincide` 且 max level 为 `5.2`，没有 `dpb-and-output-distinct`，
  因此 H.264 zero-copy + reference decode/present overlap 受驱动能力约束。
- 同轮保持 Main10/AV1 不退化：H.265 Main10 visible 4K/240
  `/tmp/gilder-vulkan-h265-main10-after-h264-general-layout-4k240` 为
  `decoded/presented=480/480`、`average_present_fps=239.76366459616204`；AV1 Main10
  4K first-frame direct gate
  `/tmp/gilder-vulkan-av1-main10-after-h264-general-layout-4k` 为
  `first-frame-decode-output-sampled-and-readback-completed`，P010 readback 和 RGBA
  shader sampling 均非零。
- AV1 continuous direct 的下一块已从 first-frame submit 扩到 inter reference telemetry：
  `NativeVulkanAv1FrameSubmitSnapshot` 输出 `reference_order_hints`、
  `frame_refs_short_signaling`、`last_frame_idx`、`gold_frame_idx` 和 7 个
  `ref_frame_indices`。真实 Main10 `/tmp/gilder-vulkan-av1-inter-ref-telemetry-main10`
  首帧仍完成 P010 direct decode/readback/sampling，后续 temporal units 已显示 inter
  `order_hint` 和 `ref_frame_indices`，但 `vulkan_submit_candidate=false`，因为
  reference-name slot planner、show-existing-frame 和 inter `vkCmdDecodeVideoKHR` 尚未完成。
  同轮 H.265 Main10 visible 4K/240 回归
  `/tmp/gilder-vulkan-h265-main10-after-av1-inter-ref-telemetry-4k240` 为
  `decoded/presented=480/480`、`average_present_fps=240.30745235839046`、P010、
  `h265_packet_queue_retained_payload_bytes=0`。
- AV1 show-existing 的真实 frame-header-only TU 已不再被 split OBU 路径误判为缺
  tile-group：`/tmp/gilder-vulkan-av1-show-existing-split-fix-main10` 中后续 temporal
  units 输出 `show_existing_frame=true`、`frame_to_show_map_idx=2/5`，unsupported reason
  变为 `display handoff needs reference slot planning`。这把 AV1 连续路线推进到
  reference map/display handoff 层，但仍未完成 inter `vkCmdDecodeVideoKHR` 或连续可见
  runtime。同轮 H.265 Main10 visible 4K/240 回归
  `/tmp/gilder-vulkan-h265-main10-after-av1-show-existing-fix-4k240` 为
  `decoded/presented=480/480`、`average_present_fps=240.157162809936`、P010、
  `h265_packet_queue_retained_payload_bytes=0`。
- AV1 direct 已跨过上述 show-existing/reference planner 阻塞，接入真实可见 runtime：
  新增 `--run-av1-ready-prefix-video` 和
  `scripts/native-vulkan-av1-ready-prefix-video-smoke.sh`。runtime 支持 AV1 Main8/Main10、
  inter `vkCmdDecodeVideoKHR`、show-existing display handoff、fixed-capacity persistent
  mapped bitstream ring 和 Wayland swapchain present。2026-06-22 真实 Wayland
  `HDMI-A-1` 小窗口任意入口证据
  `/tmp/gilder-vulkan-av1-main10-arbitrary-visible-direct-32tu-final` 为
  `decoded=24`、`handoff=8`、`presented=32`、`average_present_fps=264.3964901002394`；
  4K/60 证据 `/tmp/gilder-vulkan-av1-main10-arbitrary-visible-direct-4k60` 为
  `decoded=72`、`handoff=48`、`presented=120`、`average_present_fps=244.7861114137336`；
  4K/240 完整窗口的旧单 DPB slot 证据后来被真实观察判定为会灰闪：它能通过 submit/present
  counter，但 inter/show-existing 帧会复用 active reference layer。修正后 AV1 visible
  runtime 至少使用 9 个 DPB/output slots，并且 active reference state 只保留仍在 reference
  map 中的 slot。Main10 回归
  `/tmp/gilder-vulkan-av1-main10-dpb9-regression-4k240` 为
  `requested_codec=av1-main-10`、P010
  `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`、`decoded=259`、`handoff=221`、
  `presented=480`、`average_present_fps=239.94913990040843`、`stream_dpb_slots=9`、
  `video_resource_memory_bytes=225312768`、`session_memory_bytes=14143488`。Main8 10s
  observation `/tmp/gilder-vulkan-av1-main8-observe-10s-dpb9-v3` 为 `decoded=1305`、
  `handoff=1095`、`presented=2400`、`average_present_fps=239.6313194270436`、
  displayed layers `0..8`、`video_resource_memory_bytes=112656384`。H.265 Main10
  同轮 4K/240 任意入口回归
  `/tmp/gilder-vulkan-h265-main10-final-regression-4k240` 为
  `decoded/presented=480/480`、`average_present_fps=240.71777490911953`、
  `h265_packet_queue_loop_skip_access_units=156`、`h265_packet_queue_retained_payload_bytes=0`。
  当前 AV1 正确性优先使用逻辑 reference-name slot 规划；物理 DPB slot compaction、长时段
  process sampling、更多真实 AV1 码流、audio/clock 接入仍是后续工作。
- 2026-06-22 继续修正 AV1 repeated-frame 假阳性：严格 readback diversity gate 曾显示
  present/FPS counter 正常但 inter 内容重复，根因是 native parser 的 frame-header bit order
  和 GStreamer/FFmpeg 不一致。`allow_warped_motion` 需要在 `skip_mode_present` 之后、
  `reduced_tx_set` 之前读取；旧代码提前推断该字段且没有消费 bit，导致后续 inter
  picture/reference 字段错位。修正后真实 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1`、
  Main8 10s evidence `/tmp/gilder-av1-10s-warped-regression` 为
  `decoded_frame_count=2400`、`presented_frame_count=2400`、
  `average_present_fps=240.20825729224006`、`readback_y_distinct=5`、
  `readback_uv_distinct=5`、`loop_count=79`。这把 AV1 Main8 continuous direct 从旧的
  repeated-frame blocker 推进到可见/readback 有效状态；后续重点转为 Main10 长时覆盖、
  真实壁纸样本矩阵、降低 9-slot DPB/output 内存和接入 audio/clock。
- 2026-06-22 同轮继续修复 AV1 hidden alt-ref reference chain：`OrderHints` 和
  reference `SavedOrderHints` 需要按 AV1 reference-name order 提交，不能按内部 map slot
  order。旧顺序会在 libaom hidden frame 第二帧附近把 ALTREF order hint 放错位置，导致
  readback 又退化为重复关键帧。修正后真实 Wayland rerun
  `/tmp/gilder-av1-main8-reference-name-order-hints-rerun` 为
  `decoded=40`、`hidden_decoded=26`、`presented=64`、
  `average_present_fps=240.55662367081612`、`readback_y_distinct=5`、
  `readback_uv_distinct=5`；Main10/P010
  `/tmp/gilder-av1-main10-reference-name-order-hints-rerun` 为 `decoded=40`、
  `hidden_decoded=26`、`presented=64`、`average_present_fps=244.68053337771838`、
  `readback_y_distinct=5`、`readback_uv_distinct=5`。随后分别跑 10s 观察测试：
  Main8 `/tmp/gilder-av1-main8-observe-reference-name-order-10s` 为
  `presented=2400`、`average_present_fps=239.9047972118651`、`readback_y_distinct=10`、
  `readback_uv_distinct=10`；Main10/P010
  `/tmp/gilder-av1-main10-observe-reference-name-order-10s` 为 `presented=2400`、
  `average_present_fps=239.99269927809237`、`readback_y_distinct=10`、
  `readback_uv_distinct=10`。AV1 现在从“能 present 但可能重复”
  推进到 Main8/Main10 可见且 readback 有效。原生分辨率 low-delay 源进一步验证显示质量：
  Main8 `/tmp/gilder-av1-main8-native-res-libaom-lowdelay-observe-10s` 为
  `presented=2400`、`average_present_fps=235.13213456630402`、
  `readback_y_distinct=16`、`readback_uv_distinct=16`；Main10/P010
  `/tmp/gilder-av1-main10-native-res-libaom-lowdelay-observe-10s` 为
  `presented=2400`、`average_present_fps=230.54892214299622`、
  `readback_y_distinct=16`、`readback_uv_distinct=16`。当时 SVT-AV1 random-access
  `/tmp/gilder-av1-main8-native-res-svt-observe-10s` 仍失败，`readback_y_distinct=1`、
  `readback_uv_distinct=1`，说明后续必须继续验证 hidden/show-existing reference chain。
- 2026-06-23 修复 SVT-AV1 random-access repeated-frame：对比 FFmpeg Vulkan 后确认
  FFmpeg 提交的 inter single-tile payload 比本实现小 1 byte；SVT inter frame OBU
  在当前 parser tile boundary 前有一个 leading zero byte，旧代码把它喂给
  `vkCmdDecodeVideoKHR`，导致 readback 反复落在同一画面。现在
  `native_vulkan_av1_tile_group_offsets_from_payload` 只在 inter、single-tile、1x1 tile
  layout 且 tile payload 首字节为 0 时跳过该 byte，key frame 与非零 tile start
  不变，并用 `trims_av1_single_tile_inter_leading_zero_for_tile_payload_window` 锁住。
  真实 Wayland `HDMI-A-1` 证据：`/tmp/gilder-av1-svt-leading-zero-default-ring-readback`
  为 `presented=64`、`readback_y_distinct=9`、`readback_uv_distinct=9`；
  `/tmp/gilder-av1-svt-leading-zero-default-ring-20s` 为 `presented=4800`、
  `decoded=2420`、`hidden_decoded=2380`、`displayed_handoff=2380`、
  `average_present_fps=238.2264888256383`、19 次 clean source loop。AV1 streaming
  bitstream ring 默认也调到 8 slots（H.264/H.265 仍为 2），降低 wrap 频率并把
  SVT no-readback 10s 样本推进到约 238-239fps 区间；剩余差距主要在 AV1 runtime
  仍把 hidden decode/reference planning/present 串在单线程，后续应复用 H.264/H.265
  的 present-overlap/persistent present worker。
- 2026-06-23 继续把 AV1 任意入口可见 direct gate 补到 H.264/H.265 同级的
  correctness 形态：`scripts/native-vulkan-av1-ready-prefix-video-smoke.sh` 新增
  `--arbitrary-entry-offset` 证据强化、`--require-readback-diversity`、performance
  sampling、layer/fit/display 参数和 summary 中的 RSS/PSS/USS/Private_Dirty 字段。
  AV1/WebM 的坏前缀可能在 `av1parse`/demux 阶段就被丢弃，因此 gate 明确记录
  `arbitrary_entry_demux_dropped_prefix=yes`，而不是强行要求 Gilder runtime queue
  必须看到并丢弃坏 TU。真实 Wayland `HDMI-A-1` 证据：Main8 小窗口
  `/tmp/gilder-av1-arbitrary-main8-script-gate` 和 Main10/P010
  `/tmp/gilder-av1-arbitrary-main10-script-gate` 都为 `presented=120`、
  `playback_loop_count=2`、`loop_boundary_reset_count=1`、`readback_y/uv_distinct=5`。
  4K/240 correctness gate：Main8
  `/tmp/gilder-av1-main8-arbitrary-4k240-script-gate` 为 `presented=480`、
  `decoded=260`、`hidden_decoded=238`、`displayed_handoff=220`、
  `readback_y_distinct=5`、`average_present_fps=214.309`；Main10/P010
  `/tmp/gilder-av1-main10-arbitrary-4k240-script-gate` 为同样帧数结构、
  `readback_y_distinct=5`、`average_present_fps=195.084`。长一点的 no-readback
  4K/240 performance：Main8 `/tmp/gilder-av1-main8-arbitrary-4k240-performance`
  为 `presented=2400`、`playback_loop_count=8`、`average_present_fps=212.343`、
  `RSS/PSS/USS/Private_Dirty max=105732/72353/58468/29380 KiB`、CPU `23.41%`、
  NVIDIA process GPU memory `180 MiB`；Main10/P010
  `/tmp/gilder-av1-main10-arbitrary-4k240-performance` 为 `presented=2400`、
  `average_present_fps=211.028`、RSS/PSS/USS/Private_Dirty max
  `108404/74981/61104/30172 KiB`、CPU `10.09%`、NVIDIA process GPU memory
  `288 MiB`。结论：AV1 Main8/Main10 任意入口连续 correctness 已可用；未完成项是
  4K/240 稳定性能、真实壁纸码流矩阵、audio/clock 和 DPB/output 内存压缩。
- Web helper 输出要以 texture/frame stream 形式进入后端，避免把 WebKitGTK 当作最终 renderer 架构。

### Phase 5: 后端切换

- daemon 配置允许选择 renderer backend。
- 默认后端只在真实 Wayland matrix 中证明更稳、更省、更完整后切换。
- 旧后端保留一段时间作为回退和对照。

## 当前实现约束

新增或重构其他壁纸类型时，遵守：

- 不新增 GTK-only manifest 字段。
- 不在 core/converter 中写入 Vulkan 专用假设。
- 不把 WebKitGTK 直接放进 daemon 核心运行时；优先 helper 化。
- scene-lite evaluator 保持 headless deterministic，renderer 只消费 evaluator/runtime 输出。
- shader source 和 uniform schema 保持后端可编译，不绑定 WGSL-only 或 GLSL-only。
- 每个动态 runtime 都提供 release path，并能在 paused/hidden/fullscreen/session 场景被验证为资源归零或显著下降。

## 验证门槛

每个新类型至少需要：

- manifest/schema 单元测试。
- converter 测试和 conversion report 断言。
- render plan/headless policy 测试。
- pause-dynamic 生命周期测试。
- Wayland smoke 或明确记录暂不可 smoke 的 blocker。
- resource telemetry：runtime count、source footprint、cache footprint、释放后状态。

Vulkan 后端开始落地后，任何 renderer backend 都必须跑同一套类型矩阵；只有后端能力差异可以不同，
manifest 和 daemon 行为不能分裂。
