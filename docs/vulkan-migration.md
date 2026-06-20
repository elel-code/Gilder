# Vulkan 迁移准备路线

本文档记录 2026-06-20 之后的 renderer 方向。当前结论是：4K/240fps video
已经达到可接受的真实 Wayland 稳定基线，短期不继续围绕 active video copy/private
dirty 做底层压榨；下一阶段同时推进壁纸类型扩展和手写 Vulkan renderer spike。新增
能力必须写成可以被当前 GTK/wgpu 路径和未来纯 Vulkan 后端共同消费的形状。

## 当前决策

当前 native-wgpu/GStreamer CUDA 路线已经完成：

- `HDMI-A-1` 真实 Wayland 20s smoke 可稳定贴近 239.999Hz，`frames_skipped=0`。
- video 路径为 `gst-dmabuf` + `cuda-direct-vulkan-images-timeline`。
- CPU 和 `Private_Dirty` 仍有 driver/GStreamer/CUDA runtime floor，但 active video 已可作为
  当前高刷视频基线。
- `gpu-video` crate 路线因 codec/container 限制和维护面过窄已退休；后续 video/audio 前端保留
  GStreamer，native Vulkan 后端只消费 GStreamer 产出的 frame/texture handoff，不让
  GStreamer sink 接管显示。

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

- 当前 `wgpu` 仍作为可用的 Vulkan surface/render bridge。
- video 仍允许用 raw Vulkan/CUDA escape hatch 补足 `wgpu` 暂不暴露的能力。
- 如果后续证明 `wgpu` 的 device/surface 抽象持续阻挡 video、shader、scene 或 Web texture
  interop，就切换到 Gilder 自己拥有 Vulkan instance/device/swapchain 的 renderer 后端。

## 并行推进原则

后续工作分成两条并行线，而不是先后依赖：

- 类型线：继续补齐 `web`、`scene-lite`、`shader`、playlist、particle、audio-responsive
  等壁纸类型，让用户可见能力继续增长。
- Vulkan 线：同步建立 hand-rolled Vulkan host、device、swapchain、render graph 和
  texture/video interop spike，逐步替代当前对 `wgpu` escape hatch 的依赖。

两条线共享同一份 manifest、render plan、属性系统、动态生命周期和 telemetry。类型线不能把
新能力焊死到 GTK/WebKitGTK/wgpu；Vulkan 线也不能只服务 video，而要从一开始按完整类型矩阵设计。

## 近期优先级

类型线的近期优先级：

1. `web` runtime：独立 helper、sandbox、属性 bridge、暂停/恢复、音频/网络权限。
2. `scene-lite` runtime：从静态 snapshot 扩展到真正的 2D timeline runtime。
3. `shader` runtime：编译 WGSL/GLSL 类 shader、注入 time/resolution/mouse/property uniform。
4. `playlist` 稳定：继续补 Wallpaper Engine 复杂策略映射，并保证子项切换不泄漏 runtime 资源。
5. audio-responsive 和 particle：必须从第一天接入权限、telemetry、预算 gate。

Vulkan 线的近期优先级：

1. 最小 native Vulkan layer-shell host：clear、static image、resize、output selection。
2. 统一 renderer backend contract：让 GTK/wgpu 和 Vulkan 后端消费同一 render plan。
3. Shader-first path：fullscreen triangle、time/resolution/property uniform、surface smoke。
4. Scene-lite render target：把 deterministic scene runtime 输出接入 Vulkan pass。
5. Video interop 继续作为 spike：优先证明同场景优于当前 native-wgpu CUDA copy path，再考虑默认切换。

这些工作互不阻塞。类型 runtime 可以先用 GTK/wgpu/helper 实现，但合并前要同时写清 Vulkan-facing
contract；Vulkan spike 可以先支持少量类型，但不能引入第二套 manifest 或 daemon 语义。

## 后端边界

后续代码应维持以下边界：

- `core`、manifest、conversion report、render plan 不引用 GTK、GDK、wgpu、ash 或 GStreamer
  具体类型。
- daemon 只生成“要显示什么”的计划：entry、source、fit、time、property values、policy、target FPS。
- renderer 后端负责“怎么显示”：GTK widget、wgpu texture、Vulkan image、Web helper surface、
  shader pipeline 都留在后端内。
- status/watch telemetry 使用稳定字段描述能力和资源，不暴露某个后端独有对象生命周期作为上层契约。
- 新增类型必须先定义 headless 行为测试，再补真实 Wayland smoke；不能只靠某个 GUI 后端能显示。

推荐抽象方向：

- `SurfaceHost`：输出绑定、layer-shell surface、resize、present cadence。
- `RenderBackend`：消费 render plan，创建/更新/释放每个输出的 runtime。
- `TextureSource`：静态图、video frame、Web helper frame、scene render target、shader output。
- `DynamicRuntime`：统一 pause/resume/throttle/release/resource snapshot。
- `GpuInterop`：后端内部能力，不向 manifest 或 daemon 泄漏；当前可由 wgpu-hal/ash/CUDA 实现，未来可由纯 Vulkan 实现。

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

- 保留 native-wgpu 4K/240 smoke 作为回归基线。
- 保留 GTK/GStreamer video surface guardrail，避免主 daemon 路径回退。
- 将当前 CUDA direct import blocker 记录在文档中，避免重复走同一条失败路径。

### Phase 1: 后端无关 runtime 接口

- 清理 render plan 与 renderer runtime 的边界。
- 为 web、scene-lite、shader、playlist 子项定义共同的 dynamic lifecycle。
- status/watch 和 baseline matrix 只依赖稳定 telemetry 字段。
- 每个新增类型在 GTK/wgpu/helper 实现之外，同步定义 Vulkan 后端需要消费的资源、uniform、
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
  (`mapped_write_source=extracted-h265-access-unit`，hash=5201191167619689341)。
- `--probe-video-session --extract-bitstream` 已继续把 H.265 VPS/SPS/PPS 转成 Vulkan STD
  session parameters：2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240
  H.265 Main 源上，native parser 真实读取 profile flags、VPS/SPS DPB ordering、SPS VUI
  和 PPS 基础字段，构造 `StdVideoH265VideoParameterSet`、`StdVideoH265SequenceParameterSet`
  和 `StdVideoH265PictureParameterSet`，并成功创建 `VkVideoSessionParametersKHR`
  (`session_parameters_created=true`, VPS/SPS/PPS count 均为 1)。这一步仍未声称 decode 成功；
  剩余关键点是补 `vkCmdBeginVideoCodingKHR` / `VkVideoDecodeH265PictureInfoKHR` /
  `vkCmdDecodeVideoKHR` / `vkCmdEndVideoCodingKHR`。
- `native-vulkan-gst-video` 已补 `GstVAMemory -> vaExportSurfaceHandle(DRM PRIME) -> Vulkan`
  importer scaffold，作为 Intel/AMD VA/DMABuf 路径的基础。当前混合 GPU 机器上 VA decoder
  默认会先探测 NVIDIA DRM 设备并打印 `unsupported drm device by media driver: nvid`；
  指定 Intel render node `/dev/dri/renderD129` 时显式
  `qtdemux ! h264parse ! vah264dec ! VAMemory ! fakesink` 可谈通，但项目内
  `decodebin -> appsink` 仍会 not-negotiated。VA/DMABuf 路线后续要改成显式 codec pipeline
  或补 allocator/render-node 协商；这不是 NVIDIA direct 的主线 blocker。
- 4K/240 测试使用明确的 `3840x2160@240` H.264 源，不再用低清源判断画质。当前真实
  Wayland 证据来自 HDMI-A-1：该输出在 niri 中是 `2560x1600@239.999`、scale 1.5，
  所以这是 4K source 到 2560x1600@240 surface 的 downscale 验证，不是 4K 输出验证。
  最新 20s run：`average_render_fps=239.947`、`frames_rendered=4799`、
  `frames_imported=4778`、`eos_messages=0`、`segment_done_messages=2`、
  `last_sample_pts_delta_ms=4`、`last_import_size=3840x2160`。
- loop 使用 segment seek：启动顺序为 `Paused -> SEGMENT seek -> Playing`，收到
  `SegmentDone` 后立即 seek 回 0，避免短视频到 EOS 后硬切造成末尾抖动/卡顿。
- 建立最小 native Vulkan layer-shell renderer：clear/static/shader。
- 接入同一 render plan，不新增 manifest 分支。
- 验证单输出、多输出、resize、output selection、pause/release。
- 与类型线并行接入 shader、scene-lite 和 Web helper frame handoff。
- Video interop 可以保持独立实验，但只有证明同场景优于当前 native-wgpu/GStreamer CUDA copy
  路线后，才进入默认候选。

### Phase 4: Vulkan video/Web interop

- 在 `--run-video` lifecycle/telemetry 和 `native-vulkan-gst-video` appsink evidence 基础上，
  将 importer 明确拆成多个同级实现：NVIDIA `CUDAMemory/CUDA`、AMD/Intel
  `DMABuf/VAAPI`、可选 `GL/EGLImage`、Vulkan Video 或 libavcodec + external memory。
  GStreamer 可以继续负责 demux、硬解选择、音频和时钟，但最终 present 必须由 native
  Vulkan swapchain/render pass 完成。
- NVIDIA direct 的下一步是把已验证的 H.265 `VkVideoSessionKHR`、NV12 video resource
  image、真实 H.265 encoded AU、`VIDEO_DECODE_SRC_KHR` bitstream buffer 和
  `VkVideoSessionParametersKHR` 扩展成真正 decode：GStreamer 或等价前端只负责
  demux/parser/audio/clock，Vulkan Video 模块负责 picture info、reference slots 和
  `vkCmdDecodeVideoKHR`，再复用现有 native Vulkan NV12 shader 合成。H.264 仍可实现
  baseline/main/high，但 4K/240 H.264 level 6.1 不能作为首个 direct 成功标准。AV1 direct
  仍需补 AV1 sequence header/session parameters。10-bit H.265/AV1 已有 sampled 2-plane 420
  format evidence，后续需要单独补 P010/10-bit shader path。CUDA copy path 只保留为 fallback
  和对照基线。
- 成功标准是同场景优于当前 native-wgpu/GStreamer CUDA copy 路线，而不是理论零拷贝。
- Web helper 输出要以 texture/frame stream 形式进入后端，避免把 WebKitGTK 当作最终 renderer 架构。

### Phase 5: 后端切换

- daemon 配置允许选择 renderer backend。
- 默认后端只在真实 Wayland matrix 中证明更稳、更省、更完整后切换。
- 旧后端保留一段时间作为回退和对照。

## 当前实现约束

新增或重构其他壁纸类型时，遵守：

- 不新增 GTK-only manifest 字段。
- 不在 core/converter 中写入 wgpu/Vulkan 专用假设。
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
