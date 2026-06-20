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

暂时不继续深挖的点：

- 不继续尝试把 `gst_cuda_memory_export` 的 fd 直接导入为 Vulkan image。当前 NVIDIA/GStreamer
  栈下 direct import 失败：`OPAQUE_FD=ERROR_UNKNOWN`，`DMA_BUF_EXT` 虽可查询但
  `memory_type_bits=0x0`。这说明 copy 不是简单漏接了零拷贝路径，而是当前 CUDA exported
  fd 不能被 Vulkan image import 直接消费。
- 不再回到 `playbin+waylandsink` 作为主线；它已经证明不是后续默认方向。
- 不为了减少十几 MiB active video dirty memory 牺牲稳定性、frame pacing 或其他壁纸类型推进。

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
- 建立最小 native Vulkan layer-shell renderer：clear/static/shader。
- 接入同一 render plan，不新增 manifest 分支。
- 验证单输出、多输出、resize、output selection、pause/release。
- 与类型线并行接入 shader、scene-lite 和 Web helper frame handoff。
- Video interop 可以保持独立实验，但只有证明同场景优于当前 native-wgpu/GStreamer CUDA copy
  路线后，才进入默认候选。

### Phase 4: Vulkan video/Web interop

- 尝试 Vulkan Video、GStreamer GL/EGLImage/DMABuf、libavcodec + external memory 等方案。
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
