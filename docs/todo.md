# TODO

## M0: 项目骨架

- [x] 使用单个 Cargo package。
- [x] 提供 `gilderd`、`gilderctl`、`gilder-convert` 三个入口。
- [x] 定义基础 IPC socket 路径和命令。
- [x] 写入设计文档、格式文档、转换文档和 TODO。
- [x] 采用 `src/foo.rs` + `src/foo/` 的 Rust 模块组织方式。
- [x] 初始化 Git 仓库。
- [x] 添加 CI：`cargo fmt`、`cargo check`、`cargo test`。

## M1: 格式与加载器

- [x] 为 `manifest.gilder.json` 定义 Rust 数据结构。
- [x] 支持 `.gwpdir` 使用 `manifest.gilder.toml` 作为作者友好 manifest。
- [x] `.gwp` 打包时将 manifest 规范化为 `manifest.gilder.json`。
- [x] 引入 `serde`、`serde_json`、`camino` 或等价路径处理。
- [x] 实现 `.gwpdir` 加载。
- [x] 实现路径逃逸校验。
- [x] 实现 preview、entry、variant 校验。
- [x] 添加 manifest schema 测试。
- [x] 添加示例静态壁纸包。

## M2: IPC 与状态

- [x] 用真实 JSON parser 替换当前占位字符串匹配。
- [x] 实现 JSON-RPC 错误响应。
- [x] 添加 `outputs`。
- [x] 添加 `properties set/get`。
- [x] 添加 `watch`。
- [x] 状态写入 `$XDG_STATE_HOME/gilder/state.json`。
- [x] 配置读取 `$XDG_CONFIG_HOME/gilder/config.toml`。
- [x] 配置默认壁纸和按输出壁纸参与渲染计划。
- [x] 支持 IPC `set` 指定 manifest variant。
- [x] socket 权限和 stale socket 处理。
- [x] daemon 单实例检测。

## M3: GTK/Wayland 静态壁纸

- [x] 引入 GTK-rs。
- [x] 选择并接入 layer-shell 支持。
- [x] 为每个输出创建 background layer 窗口。
- [x] 实现静态图片解码和显示。
- [x] 实现 fit mode：cover、contain、stretch、tile、center。
- [x] 支持配置按输出覆盖 fit mode。
- [x] 为 daemon 状态生成静态渲染同步计划。
- [x] 没有显式 variant 时按输出尺寸自动选择资源变体。
- [x] 支持输出热插拔。
- [x] 支持按 output 设置不同壁纸。

## M4: 视频壁纸

- [x] 引入 GStreamer。
- [x] 实现视频 entry 加载。
- [x] 将 GStreamer worker 接入 daemon 渲染同步队列。
- [x] 在 GStreamer runtime 实现 loop、muted。
- [x] 将 manifest `runtime.allow_audio` 接入视频静音/音频 sink 策略。
- [x] 实现 pause/resume/stop 的 pipeline 控制。
- [x] 将视频 sink 通过 GTK paintable 接入每个输出的 layer-shell window。
- [x] 在 IPC status/watch 中报告视频 surface 运行时能力。
- [x] 添加 Wayland 视频 surface smoke 验证脚本。
- [x] 为 Wayland 视频 surface smoke 添加低干扰 preflight 和结构化检查报告。
- [x] 为 Wayland 视频 surface smoke 输出单页 validation report，汇总 compositor、输出、active video plan 和 runtime/performance 证据入口。
- [x] 为 Wayland 视频 surface smoke 添加 compositor kind 断言，避免 Hyprland/niri/generic 证据混淆。
- [x] 在真实 niri Wayland 会话验证视频 surface 显示。
- [x] 在真实 niri Wayland 会话验证多输出同源视频 surface 显示。
- [ ] 在真实 Hyprland Wayland 会话验证视频 surface 显示。
- [x] 添加 CPU/RSS/PSS/USS/private/status 性能采样脚本。
- [x] 将 active-video 性能采样接入 Wayland 视频 surface smoke。
- [x] 将 paused-video 性能采样接入 Wayland 视频 surface smoke。
- [x] 在性能采样证据中输出 render decision CSV 和摘要。
- [x] 在性能采样证据中输出 PSS、USS/private 和 shared 内存摘要。
- [x] 支持性能采样断言 RSS/PSS/USS/private/shared 最大内存预算，便于后续建立回归门槛。
- [x] 支持性能采样断言 retained/peak-over-first PSS、USS/private 和 shared 内存 delta，便于验证暂停/隐藏/恢复后的保留占用。
- [x] performance 采样接入 GTK 静态 Picture/CSS/color surface 和估算 decoded footprint 的 summary 与预算断言。
- [x] headless 桌面状态性能策略 smoke 支持向每个场景转发 RSS/PSS/USS/private/shared 内存预算断言。
- [x] headless 桌面状态性能策略 smoke 输出每场景 CPU/GPU/RSS/PSS/USS/private/shared 资源基线表。
- [x] 在 decision CSV 中记录计划类型、资源、fit、视频限帧和静音状态。
- [x] 使用 CSV-aware 汇总器统计性能采样中的决策、计划类型、fit、静音和限帧范围。
- [x] 支持性能采样断言期望的 mode、reason、action 和计划类型。
- [x] 单次 render sync 内复用同一路径壁纸包的加载结果。
- [x] 实现 poster 显示。
- [x] 实现 max_fps 或 pipeline throttling。
- [x] 实现 slideshow 普通动态壁纸运行计划和 GTK 定时切换。
- [x] slideshow `crossfade` 在 GTK renderer 中使用短生命周期 `gtk::Stack` 转场，并在转场结束后释放上一帧 Picture。
- [x] 避免重复 render sync 对视频 pipeline 反复设置未变化的 state、mute、fit、限帧和 start offset。
- [x] 避免重复 render sync 对 GTK 静态窗口反复重建 Picture/CSS fallback surface。
- [x] 避免 GTK 初始同步和 IPC 状态变更把未变化的 render sync 重复投递给渲染器。
- [x] GTK 主线程消费 renderer queue 时 drain 积压并只应用最新 render sync，避免快速状态切换反复创建中间态 GTK/GStreamer surface、pipeline 和 runtime snapshot。
- [x] 为 daemon status/watch 路径缓存未变化的 render sync，减少性能采样时的重复 manifest IO。
- [x] render sync 缓存只跟踪渲染相关 state，避免 properties set/get 造成无意义重算。
- [x] render sync 缓存只跟踪渲染相关 config，避免 adapter 开关和刷新周期造成无意义重算。
- [x] status/outputs 读请求按桌面刷新周期复用桌面快照，避免轮询时频繁调用 compositor 适配器。
- [x] 在 status 和性能采样中输出 daemon telemetry，用于审计桌面刷新节流和 render sync 缓存命中。
- [x] 支持性能采样断言 render sync 缓存命中和桌面刷新节流生效。
- [x] 在 daemon telemetry 中输出渲染器同步投递 queued/skipped 计数，用于审计投递去重。
- [x] 支持性能采样断言渲染器同步投递 queued/skipped 计数。
- [x] 设计可选 adaptive system monitor：默认关闭，支持全局/按输出启用和 kill switch。
- [x] 采样 Linux PSI CPU/内存压力、thermal zone 最高温度和 power_supply 电源细节，并输出到 daemon telemetry。
- [x] 继续采样 GPU 和视频帧行为，并输出到 daemon telemetry。
- [x] 在 adaptive telemetry 中采样 DRM `gpu_busy_percent` 的 avg/max/source，并接入 telemetry CSV 和性能采样汇总。
- [x] 在视频 runtime 中报告 playback position/duration 和实际 frame limiter 状态，并接入 performance 采样汇总与断言。
- [x] 在视频 runtime 中累计 GStreamer QoS processed/dropped 统计，并接入 performance 采样汇总与断言。
- [x] 在 GTK 视频 surface runtime 中累计 frame clock tick/interval/FPS 统计，并接入 performance 采样汇总与断言。
- [x] 在 GTK 视频 surface runtime 中累计 frame clock before-paint/update/layout/paint/after-paint phase 统计，并接入 telemetry/runtime CSV 和 performance 汇总。
- [x] performance/Wayland smoke 支持断言 GTK frame clock 指定 phase 或 all phases，作为真实 surface 验证 gate。
- [x] 在 GTK 视频 surface runtime 中累计 GDK frame timings observed/complete/presentation 线索，并接入 performance 采样汇总与断言。
- [ ] 继续采集 compositor presentation/frame callback 统计。
- [x] 将 adaptive monitor 结果作为只会降载的性能策略输入，支持阈值、冷却时间和恢复条件。
- [x] adaptive monitor 支持用户可选的 `pause-unfocused` 动作，在系统压力下暂停非焦点输出。
- [x] adaptive monitor 支持用户可选的 `pause-dynamic` 动作，在系统压力下暂停 video/slideshow/web/scene-lite/shader 动态壁纸。
- [x] adaptive monitor 支持 GPU busy 和低电量阈值触发，并在 headless smoke 中覆盖 throttle 与 `pause-dynamic` 释放资源。
- [x] 在 status/watch 中报告 adaptive monitor 的当前采样、触发原因和实际降载动作。
- [x] 在真实 niri Wayland 会话采集 battery/unfocused/fullscreen 视频 surface 策略和内存证据。
- [x] 添加 fullscreen -> active 恢复延迟采样入口和结构化证据输出。
- [x] 在真实 niri Wayland 会话采集验证覆盖下的 fullscreen -> active 恢复延迟证据。
- [x] muted 视频使用 `playbin` flags 禁用 audio stream，避免解码未使用音频。
- [x] 添加 headless 桌面状态性能策略 smoke，覆盖 active/battery/unfocused/fullscreen/hidden/session 场景。
- [x] 将 headless 桌面状态性能策略 smoke 接入 CI 并上传证据。
- [x] 为 headless 桌面状态性能策略 smoke 输出场景矩阵和顶层摘要。
- [x] headless 桌面状态性能策略 smoke 覆盖按输出 FPS/电池策略覆盖。
- [x] headless 桌面状态性能策略 smoke 覆盖 adaptive throttle、`pause-unfocused` 和焦点输出回退。
- [x] headless 桌面状态性能策略 smoke 覆盖 adaptive `pause-dynamic` 静态透传和 slideshow 暂停。
- [x] performance 采样支持断言 adaptive action telemetry，避免只验证最终 mode/reason。
- [x] 添加 MP4/WebM codec smoke 验证脚本。
- [x] 为 MP4/WebM codec smoke 输出结构化报告并在 CI 上传。
- [x] 为 MP4/WebM codec smoke 输出 GStreamer demuxer/decoder element 诊断。
- [x] 为 MP4/WebM codec smoke 添加快速 preflight 诊断模式。
- [x] 提供 Arch-like MP4/WebM codec smoke 依赖安装 helper。
- [x] 提供 Arch-like Wayland 视频 surface 验证依赖安装 helper。
- [x] 验证 MP4/H.264。
- [x] 验证 WebM/VP9。
- [x] 验证 WebM/AV1。
- [x] 添加 fullscreen 暂停策略接口。
- [x] 在 codec smoke 中记录实际 GStreamer decoder element，区分软解和硬解 codec 基线。
- [x] 在 daemon status/watch 中报告运行中视频 pipeline 的实际 decoder element。
- [x] 提供视频 decoder 策略配置：`auto`、`hardware-preferred`、`hardware-required`、`software`。
- [x] 在 GStreamer autoplug 选择中通过 decoder feature rank 优先/强制 VAAPI、VDPAU、NVDEC 等硬解 decoder，并保留明确软解回退。
- [x] 在 status/watch 中报告 decoder 策略、实际 decoder 类型和硬解/软解分类。
- [x] 在 status/watch 中报告 decoder 策略是否被实际选中的 decoder 满足。
- [x] 在 status/watch 中报告运行中视频 pad 的 negotiated caps 和 memory features。
- [x] 在 status/watch、CSV 和性能采样中报告 zero-copy 证据分级，区分软解、硬解、GPU memory caps、DMABuf caps 和 sink-side 线索。
- [ ] 验证 GTK video surface 是否能保持 GPU/DMABuf 路径，区分“硬解但发生 CPU copy”和真正 zero-copy。
- [x] 本机 smoke/performance 采样记录 decoder、decoder 策略状态、caps memory features 和 sink memory features。
- [x] 本机 performance 采样支持断言 decoder 策略状态、decoder class、caps memory feature 和 sink memory feature。
- [x] Wayland 视频 surface smoke 支持直接断言 decoder、caps、zero-copy evidence 和 GTK/GDK frame timing 证据。
- [x] Wayland 视频 surface smoke 支持要求 active phase 必须记录 live video runtime row，避免只有 render plan、没有 pipeline 证据。
- [x] Wayland 视频 surface smoke 支持 renderer video pipeline lifecycle gate，验证 active/resumed 创建 pipeline、paused/hidden/fullscreen 释放 pipeline。
- [x] 为硬解和 zero-copy 添加本机 smoke：记录 decoder、sink caps/memory features、CPU/GPU/USS/PSS 对比。
- [x] `web` 壁纸在 WebKit runtime 完成前先支持 fallback 静态 render plan，并按动态壁纸参与 `pause-dynamic` 释放策略。
- [ ] 研究 GTK/GSK paintable/texture 生命周期和 GStreamer allocator/buffer-pool 协商，减少静态图 decoded texture、视频 CPU copy 与 buffer pool 保留内存。
- [x] 为 native Vulkan 添加 `--probe-video`，枚举 Vulkan Video decode 扩展、codec operations 和 `VIDEO_DECODE` queue family；本机 NVIDIA 4060 Laptop GPU 已确认 H.264/H.265/AV1/VP9 decode ready。
- [x] 扩展 native Vulkan `--probe-video` 的 H.264 profile/format capability：NVIDIA 4060 已确认 baseline/main/high、NV12 DPB/output/sampled image 可用，但 H.264 max level 5.2 低于当前 4K/240 测试源 level 6.1。
- [x] 补 native Vulkan Video H.265/AV1 profile/format probe：NVIDIA 4060 的 H.265 main-8 为 level 6.1、AV1 main-8/main-10 为 level 7.3，二者均具备 4K/240 direct 首版优先级。
- [x] 为 native Vulkan 添加 `--probe-video-session`，真实创建并绑定 H.265 main-8 / AV1 main-8 `VkVideoSessionKHR`：2026-06-21 在 NVIDIA 4060、3840x2160 参数下验证 `vkCreateVideoSessionKHR`、session memory requirements、allocation 和 `vkBindVideoSessionMemoryKHR` 均成功。
- [x] 扩展 `--probe-video-session --allocate-video-images`：真实创建并绑定 3840x2160、8 layers、NV12 `video-decode-dst|video-decode-dpb|sampled` Vulkan image 和 2D array image view；H.265/AV1 main-8 均验证通过，image memory requirement 为 100139008 bytes device-local。
- [x] 扩展 `--probe-video-session --allocate-bitstream-buffer`：真实创建并绑定 8MiB `VIDEO_DECODE_SRC_KHR` bitstream buffer，按 256-byte alignment 对齐，绑定 host-visible/coherent memory，并映射写入 256 bytes；H.265/AV1 main-8 均验证通过。
- [x] 扩展 `--probe-video-session --extract-bitstream --source <h265.mp4>`：真实运行 `qtdemux+h265parse+appsink` 抽取 3840x2160@240 H.265 byte-stream AU，验证 VPS/SPS/PPS/IDR NAL 存在，并把 173754 bytes 的 selected AU 写入 `VIDEO_DECODE_SRC_KHR` buffer；2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060 上验证通过。
- [x] 为 H.265 direct decode 实现 Vulkan STD session parameters：解析真实 VPS/SPS/PPS 的 profile flags、DPB ordering、SPS VUI 和 PPS 基础字段，转换为 `StdVideoH265VideoParameterSet`、`StdVideoH265SequenceParameterSet`、`StdVideoH265PictureParameterSet`，并真实创建 `VkVideoSessionParametersKHR`；2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main 源上验证 `session_parameters_created=true`。
- [x] 为 AV1 direct 路线补 encoded frontend 和 Vulkan STD session parameters：新增 `scripts/native-vulkan-av1-bitstream-smoke.sh`，使用 `matroskademux/qtdemux ! av1parse ! appsink` 输出 `stream-format=obu-stream, alignment=tu`，native parser 扫描 OBU、解析 sequence header 的 profile/level/extent/color config，并转换为 `StdVideoAV1SequenceHeader` 创建 `VkVideoSessionParametersKHR`。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1` 验证 AV1 Main 8-bit 640x368 源通过，`session_parameters_created=true`、`source=native-rust-av1-sequence-header-to-vulkan-std`；随后补 frame/tile readiness telemetry，证据 `/tmp/gilder-vulkan-av1-bitstream.ivMR9n` 中 selected TU 为 sequence-header + frame OBU，`av1_decode_candidate=true`、`av1_frame_payload_bytes=2697`、`av1_first_frame_header_obu_offset=13`。
- [x] 将 AV1 direct 前端推进到 first-frame header/tile layout telemetry：native AV1 parser 现在会解析 selected frame OBU 的 key-frame header、frame size/render size、order hint、tile columns/rows、`tile_size_bytes_minus_1` 等提交前字段，`scripts/native-vulkan-av1-bitstream-smoke.sh` 会 gate `av1_first_frame_submit_present=true`、`found_frame_header=true`、`frame_type=key` 和 `tile_count>=1`。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据 `/tmp/gilder-vulkan-av1-bitstream.6QtfGN`：`av1_first_frame_tile_count=16`、`tile_columns=4`、`tile_rows=4`、`tile_size_bytes=4`。当前仍未标记为 direct submit candidate，因为 tile group table 起点/覆盖范围还未解析到可直接传给 `VkVideoDecodeAV1PictureInfoKHR` 的完整 tile offset/size 数组。
- [x] 推进 H.265/AV1 Main10 到 native Vulkan session/resource/bitstream gate：`--video-codec h265-main-10|av1-main-10` 会选择 10-bit profile 和 `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`，H.265 Main10 smoke 新增 `scripts/native-vulkan-h265-main10-bitstream-smoke.sh` 并验证 P010-like resource image、bitstream upload、VPS/SPS/PPS -> Vulkan STD session parameters；AV1 smoke 支持 `--bit-depth 10` 并验证 sequence header -> Vulkan STD session parameters。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据：H.265 `/tmp/gilder-vulkan-h265-main10-bitstream.Y0bB5M`，`session_parameters_codec=h265-main-10`、`video_image_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`；AV1 `/tmp/gilder-vulkan-av1-bitstream.86Mw24`，`requested_codec=av1-main-10`、`av1_sequence_bit_depth=10`、`session_parameters_codec=av1-main-10`。
- [x] 推进 direct H.264 到 Vulkan STD session parameters：`--video-codec h264` 映射到 H.264 High 8-bit progressive profile，`scripts/native-vulkan-h264-bitstream-smoke.sh` 通过 `qtdemux ! h264parse ! appsink` 输出 `stream-format=byte-stream, alignment=au`，native parser 解析 SPS/PPS 并转换为 `StdVideoH264SequenceParameterSet`、`StdVideoH264PictureParameterSet`，真实创建 `VkVideoSessionParametersKHR`。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60 `/tmp/gilder-vulkan-h264-bitstream.iVMCh1`，4K/240 level 5.2 `/tmp/gilder-vulkan-h264-bitstream.fs7CCw`，后者 `session_parameters_created=true`、`profile_idc=100`、`level_idc=52`、`mapped_write_bytes=217455`、`session_memory_bytes=16945152`。
- [x] 为 H.264 direct decode 实现首帧 command buffer/readback：新增 `scripts/native-vulkan-h264-first-frame-smoke.sh`，解析首个 IDR picture 的所有 slice offsets，填充 `VkVideoDecodeH264PictureInfoKHR`、`StdVideoDecodeH264PictureInfo`、`VkVideoDecodeH264DpbSlotInfoKHR` 和 setup DPB slot，录制 `vkCmdBeginVideoCodingKHR`、`vkCmdControlVideoCodingKHR(RESET)`、`vkCmdDecodeVideoKHR`、`vkCmdEndVideoCodingKHR`，并把 NV12 output plane 0/1 copy 到 host-visible readback buffer。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60 `/tmp/gilder-vulkan-h264-first-frame.AYMakX`，`first_frame_decode.completed=true`、`slice_count=11`、Y/UV 非零 `921600/460800`；4K/240 level 5.2 `/tmp/gilder-vulkan-h264-first-frame.lQiwMa`，`slice_count=20`、`src_buffer_range=217600`、Y/UV 非零 `8294400/4147200`；采样 gate `/tmp/gilder-vulkan-h264-first-frame.GJildG`，`sample_copied=true`。H.264 direct 下一步是连续 AU decode、DPB/reference tracking、visible surface presentation 和 frame pacing。
- [x] 为 H.264 direct decode 补 all-IDR 多帧 gate：新增 `--decode-h264-idr-prefix N`、`h264_access_units[]`/`h264_idr_decode_ready_prefix_count` telemetry 和 `scripts/native-vulkan-h264-idr-prefix-smoke.sh`。该 gate 生成 `keyint=1` High 8-bit 源，把多个 IDR AU 按 Vulkan bitstream offset/size alignment 拼到同一个 `VIDEO_DECODE_SRC_KHR` buffer，顺序提交多次 `vkCmdDecodeVideoKHR`，最后 readback NV12。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60 `/tmp/gilder-vulkan-h264-idr-prefix.kKR6lh`，`decoded_frame_count=8`、frame offsets `[0,35072,57088,79104,101376,123648,145920,168192]`；4K/240 level 5.2 `/tmp/gilder-vulkan-h264-idr-prefix.7H4DV3`，`decoded_frame_count=8`、frame offsets `[0,217600,329216,441600,553984,666624,779264,892160]`、Y/UV 非零 `8294400/4147183`。这证明 H.264 direct 不再只是首帧；下一步仍需 P/B reference tracking、无 per-frame reset 的 DPB 维护、visible surface presentation 和 frame pacing。
- [x] 将 H.264 direct 从 all-IDR 推进到普通 IDR+P ready-prefix：新增 `--decode-h264-ready-prefix N`、`h264_decode_reference_plan[]`、`h264_decode_ready_prefix_count` 和 `scripts/native-vulkan-h264-ready-prefix-smoke.sh`。native parser 现在能解析非 IDR P slice 的 active L0 reference count、reference-list-modification flag 和 reference marking，并为第一版连续 direct gate 维护 2-slot DPB/reference plan；decode 提交时 P 帧携带真实 `reference_slots`，不再每帧 reset。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60 `/tmp/gilder-vulkan-h264-ready-prefix.U6E7hC`，`decoded_frame_count=8`、`non_idr_frames=7`、`reference_frames=7`、`reset_control_count=1`、`reference_plan_dpb_slots=2`；4K/240 level 5.2 `/tmp/gilder-vulkan-h264-ready-prefix.e1aTOo`，frame offsets `[0,217600,305408,399360,491776,585472,677376,769536]`、reference counts `[0,1,1,1,1,1,1,1]`、planned slots `[0,1,0,1,0,1,0,1]`、Y/UV 非零 `8294400/4147194`。该阶段边界是 B slice、ref list modification、adaptive MMCO、long-term reference 和任意入口点 DPB 重建；后续 streaming queue 路径已继续推进其中一部分。
- [x] 将 H.264 ready-prefix direct decode 从单参考 P 帧推进到多参考 IPPP：`scripts/native-vulkan-h264-ready-prefix-smoke.sh --refs N` 可生成 `bframes=0:ref=N:weightp=0` 源，reference plan 不再限制 `num_ref_idx_l0_active_minus1 == 0`，而是按默认短期参考列表为 P 帧提交多个 `reference_slots`。2026-06-21 真实 `WAYLAND_DISPLAY=wayland-1` 证据：720p/60 ref=2 `/tmp/gilder-vulkan-h264-ready-prefix.PWCkbZ`，`frame_reference_counts=[0,1,2,2,2,2,2,2]`、`reference_plan_dpb_slots=3`；4K/240 level 5.2 ref=2 `/tmp/gilder-vulkan-h264-ready-prefix.ka5g3l`，`decoded_frame_count=8`、`max_reference_count=2`、`requested_reference_counts=[0,1,2,2,2,2,2,2,2]`、Y/UV 非零 `8294400/4147193`。该阶段边界是 B slice、显式 reference list modification、adaptive MMCO、long-term reference 和任意入口点 DPB 重建；后续 streaming queue 路径已继续推进其中一部分。
- [x] 将 H.264 direct ready-prefix 接到真实 Wayland 可见 swapchain：新增 `--run-h264-ready-prefix-video` 和 `scripts/native-vulkan-h264-ready-prefix-video-smoke.sh`，使用 GStreamer 只做 `qtdemux+h264parse+appsink` AU 抽取，实际解码走 `vkCmdDecodeVideoKHR`，decoded NV12 array layer 由 native Vulkan shader 采样到 Wayland background surface。2026-06-21 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 证据：720p/60 ref=2 `/tmp/gilder-vulkan-h264-ready-prefix-video.faL4eZ`，`decoded_frame_count=8`、`presented_frame_count=8`、`max_reference_count=2`、`stream_dpb_slots=3`、`bitstream_buffer_strategy=fixed-capacity-persistent-mapped-ring`；4K/240 ref=2 `/tmp/gilder-vulkan-h264-ready-prefix-video.Jy9iXF`，`decoded_frame_count=240`、`presented_frame_count=240`、`source_extent=[3840,2160]`、`frame_reference_counts` 达到 2、`video_resource_memory_bytes=37552128`、`bitstream_buffer_bytes=435200`；480-frame loop `/tmp/gilder-vulkan-h264-ready-prefix-video.S305L5` 验证 `playback_loop_count=2`、`loop_boundary_reset_count=1`。当前已达到 H.265 visible ready-prefix 的功能形态，但 4K/240 average present 仍约 `212fps`，后续要继续优化 present pacing/同步到稳定 240。
- [x] 将 H.264 visible encoded input 从长 ready-prefix payload 推进到低内存输入层：临时 spool + 固定容量 persistent mapped bitstream ring 曾作为过渡阶段证明低内存上传可行；当前维护目标已切到 `--h264-input streaming-queue`，脚本 `--streaming-queue` 仅为兼容 no-op。GStreamer appsink AU 进入 bounded packet queue，播放时按需拉取、上传压缩 AU 后丢弃 payload，runtime JSON 报告 `h264_input_mode`、queue capacity/pulled/eos/loop/max payload/retained payload。2026-06-21 真实 Wayland `HDMI-A-1` 证据：720p/60 `/tmp/gilder-vulkan-h264-ready-prefix-video.at5uDt`，`h264_input_mode=streaming-queue`、`decoded=8`、`p_frames=7`、`max_reference_count=2`、`h264_packet_queue_retained_payload_bytes=0`；4K/240 `/tmp/gilder-vulkan-h264-ready-prefix-video.ZFXzKH`，`decoded=240`、`presented=240`、`queue_capacity=32`、`queue_pulled=240`、`queue_retained=0`、`bitstream_buffer_bytes=1036800`。H.264 parser/reference plan 也已支持短期 L0 ref-list modification idc 0/1，并把 streaming mode 的 reference plan 改为增量 planner；`plans_h264_short_term_ref_list_modification_p_slice` 覆盖显式引用更早短期帧。20s 采样 `/tmp/gilder-vulkan-h264-streaming-smaps.oULUUh` 为 `decoded=4800`、`queue_eos/loops=4/4`、`queue_retained=0`、`average_present_fps=212.375`、90 个 smaps_rollup 样本 `RSS/PSS/USS/Private_Dirty max=112908/68437/49192/31272 KiB`、平均 CPU `15.13%`。当前输入层已不再保留长 AU payload，剩余 H.264 4K/240 缺口主要落在 H.264 level/capability、present pacing/同步或驱动 codec 路径，不是 packet retention；B slice、短期 B-slice L1 ref-list modification、MMCO op=1 和 long-term/MMCO planner-submit 状态已在后续 streaming queue evidence/单测中推进，剩余边界主要是真实 long-term coded stream smoke、field picture 和任意入口点 DPB 重建。
- [x] H.264 streaming planner 支持非参考 picture scratch 输出：非参考 AU 不再进入 active DPB，decode submit 不传 setup reference slot，planner 优先选择空闲 output layer，覆盖旧 reference 时会清理 reference map/order。单测 `plans_h264_non_reference_pictures_as_scratch_outputs` 覆盖 `IDR -> ref P -> non-ref P -> ref P`，确认后续帧仍引用上一张 reference picture；真实 Wayland 4K/240 回归 `/tmp/gilder-vulkan-h264-nonref-regression` 通过 `decoded/presented=2400/2400`、`queue_retained=0`。
- [x] 将 H.264/H.265 visible direct smoke 和 CLI 默认输入切到 streaming queue，并停止维护 ready-prefix spool 入口：`--h264-input ready-prefix-spool` / `--h265-input ready-prefix-spool` 现在会报错，visible video runtime 已移除可选 spool 分支，脚本中的 `--streaming-queue` 仅保留为兼容 no-op。H.264 默认 streaming 4K/240 回归 `/tmp/gilder-vulkan-h264-streaming-default-regression` 为 `decoded/presented=240/240`、`h264_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=213.179`；H.265 默认 streaming 回归 `/tmp/gilder-vulkan-h265-streaming-default-regression` 为 `decoded/presented=240/240`、`h265_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=240.915`。删除 visible spool runtime 后复测真实 Wayland `HDMI-A-1` 4K/240：`/tmp/gilder-vulkan-no-spool-h264-4k240-smoke` 为 `decoded/presented=240/240`、`h264_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=213.770`；`/tmp/gilder-vulkan-no-spool-h265-4k240-smoke` 为 `decoded/presented=240/240`、`h265_input_mode=streaming-queue`、`queue_retained=0`、`average_present_fps=240.546`。
- [x] 将 H.264 streaming queue 推进到真实 B-frame visible smoke：parser 记录 B-slice L1 active ref count、短期 L0/L1 ref-list modification 和 MMCO 列表，planner 支持默认 B L0/L1 短期引用、显式短期 L1 modification、参考 B picture、非参考 B scratch 输出，以及 MMCO op=1 short-term unused-for-reference 的 DPB drop；visible submit 为每个 decode target 提供 setup slot，但只有参考图进入 active DPB。真实 Wayland `HDMI-A-1` 证据 `/tmp/gilder-vulkan-h264-b1-streaming-smoke` 为 `decoded/presented=120/120`、`b_frames=59`、`max_reference_count=2`、`queue_retained=0`；`/tmp/gilder-vulkan-h264-bslice-streaming-smoke-final` 为 `decoded/presented=180/180`、`b_frames=119`、`max_reference_count=3`、`h264_input_mode=streaming-queue`、`queue_retained=0`。新增单测 `parses_h264_b_slice_l1_ref_list_modification_for_streaming_queue` 和 `plans_h264_b_slice_l1_short_term_ref_list_modification` 覆盖显式 L1 短期修改；long-term 增量新增 IDR long-term、MMCO6 当前图 long-term、MMCO4 上限裁剪、MMCO5 全清、long-term index replacement 和 long-term L0 modification 单测，并把 visible submit 的 active DPB long-term flag 同步到 `StdVideoDecodeH264ReferenceInfo`。2026-06-21 真实 Wayland `HDMI-A-1` 回归 `/tmp/gilder-vulkan-h264-longterm-planner-regression` 为 `decoded/presented=60/60`、`h264_input_mode=streaming-queue`、`queue_retained=0`；4K/240 `/tmp/gilder-vulkan-h264-longterm-planner-4k240-regression` 为 `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、`average_present_fps=194.8709`。短期 reference list 默认排序和 ref-list modification idc 0/1 已改为按 PicNum 处理，覆盖 `frame_num` wrap 后的参考查找；剩余 H.264 码流边界主要是真实 long-term coded stream smoke、field picture、gaps-in-frame-num/non-existing refs 和任意入口点 DPB 重建。
- [x] 记录 2026-06-21 direct video handoff snapshot：`bff077b Support H264 PicNum reference ordering` 已推送；H.264 direct Vulkan Video 默认 `streaming-queue`，P/B 帧、多参考、短期 L0/L1 modification、MMCO、long-term planner 和 `frame_num` wrap 后 PicNum ordering 已有覆盖。真实 Wayland `HDMI-A-1` 最新回归：H.264 720p/60 `/tmp/gilder-vulkan-h264-picnum-wrap-regression` 为 `decoded/presented=60/60`、`queue_retained=0`、`average_present_fps=252.939`；H.264 4K/240 B-frame `/tmp/gilder-vulkan-h264-picnum-wrap-4k240-regression` 为 `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、`average_present_fps=198.431`；H.265 4K/240 对照 `/tmp/gilder-vulkan-h265-picnum-wrap-4k240-regression` 为 `decoded/presented=240/240`、`queue_retained=0`、`average_present_fps=240.522`。该 handoff 当时不是“任意连续”完成证明；当时剩余边界仍是真实 long-term coded stream、gaps-in-frame-num/non-existing refs、field/interlaced、任意入口 DPB 重建和 H.264 4K/240 稳定帧率。
- [x] 将 H.264 `gaps_in_frame_num` / non-existing short-term refs 接入 direct planner 和 visible submit：SPS 禁止 gaps 时明确 unready；允许 gaps 时按 `max_frame_num` wrap 推断 non-existing refs、维护 DPB slot/sliding window，并把 `non_existing=true` 写入 `StdVideoDecodeH264ReferenceInfoFlags.is_non_existing`。runtime JSON 记录 reference `non_existing`、inferred refs 和 inference drop slots。新增单测覆盖 gap disallowed/allowed、`max_frame_num=65536` wrap、sliding window 和 PicNum wrap；真实 Wayland `HDMI-A-1` 回归：H.264 720p/60 `/tmp/gilder-vulkan-h264-nonexisting-regression` 为 `decoded/presented=60/60`、`queue_retained=0`、`average_present_fps=247.596`；H.264 4K/240 `/tmp/gilder-vulkan-h264-nonexisting-4k240-regression` 为 `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、`average_present_fps=202.138`；H.265 4K/240 对照 `/tmp/gilder-vulkan-h265-nonexisting-4k240-regression` 为 `decoded/presented=240/240`、`queue_retained=0`、`average_present_fps=240.622`。H.264 4K/240 长跑 `/tmp/gilder-vulkan-h264-nonexisting-4k240-memory/combined-keep/performance` 为 `decoded/presented=7200/7200`、`average_present_fps=202.047`、`queue_retained=0`，8 个 smaps samples 中 `RSS/PSS/USS/Private_Dirty max=105048/73925/56404/26756 KiB`、平均 CPU `12.10%`、NVIDIA 进程 GPU memory `104 MiB`。剩余 H.264 direct 边界是真实 long-term coded stream、field/interlaced、任意入口 DPB 重建和 4K/240 pacing/同步。
- [x] 为 H.264 streaming queue 补 weighted P-slice 码流边界：parser 不再因 PPS `weighted_pred_flag` 直接拒绝，而是按 H.264 `pred_weight_table` 语法跳读 L0/L1 luma/chroma weight/offset，随后继续解析 reference marking。单测 `parses_h264_weighted_p_slice_header_for_streaming_queue` 覆盖显式 luma/chroma weight table；脚本 `scripts/native-vulkan-h264-ready-prefix-video-smoke.sh` 新增 `--weightp/--weightb` 生成参数。真实 Wayland `HDMI-A-1` 证据：720p/60 `/tmp/gilder-vulkan-h264-weightp-streaming-smoke` 为 `decoded/presented=60/60`、`h264_input_mode=streaming-queue`、`queue_retained=0`；4K/240 `/tmp/gilder-vulkan-h264-weightp-4k240-smoke` 为 `decoded/presented=240/240`、`p_frames=239`、`max_reference_count=2`、`h264_input_mode=streaming-queue`、`queue_retained=0`、`bitstream_buffer_strategy=fixed-capacity-persistent-mapped-ring`、`average_present_fps=214.9888566483139`。
- [x] 为 H.264、AV1 和 H.265 Main10 补真实 Wayland 可见 codec smoke：新增 `scripts/native-vulkan-*-visible-video-smoke.sh`，使用 GStreamer demux/decode/appsink 作为前端，native Vulkan importer/shader/swapchain 负责可见输出，不使用 GTK/playbin/waylandsink。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 验证：H.264 720p/240 `/tmp/gilder-vulkan-visible-h264.dqQnsN`、4K/240 `/tmp/gilder-vulkan-visible-h264.K0XXrj`；AV1 640x368/60 `/tmp/gilder-vulkan-visible-av1.fBQmOz`、4K/60 `/tmp/gilder-vulkan-visible-av1.yAKhDg`；H.265 Main10 640x368/60 `/tmp/gilder-vulkan-visible-h265-main10.GxYmkr`、4K/60 `/tmp/gilder-vulkan-visible-h265-main10.0nZH7D`。这些是第二条路线的 visible importer/present gates，不代表 H.264/AV1/Main10 direct Vulkan Video `vkCmdDecodeVideoKHR` 已完成。
- [x] 为 H.265 direct decode 实现首帧 command buffer：新增 `--decode-first-frame`，解析 IDR slice offset，填充 `VkVideoDecodeH265PictureInfoKHR`、coincident DPB/output picture resource、setup reference slot，录制 `vkCmdBeginVideoCodingKHR`、`vkCmdControlVideoCodingKHR(RESET)`、`vkCmdDecodeVideoKHR` 和 `vkCmdEndVideoCodingKHR`；2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main 源上验证 queue submit/wait 完成，`first_frame_decode.completed=true`。
- [x] 验证 H.265 direct decode output image 内容：`--decode-first-frame` 在 decode 后把 NV12 array layer 0 的 plane 0/1 copy 到 host-visible readback buffer 并记录 hash、非零数、min/max/unique；2026-06-21 真实 Wayland/NVIDIA 4060/3840x2160@240 H.265 Main 源验证 `output_readback.copied=true`，Y plane 8294400 bytes、hash=8710880026335779165、unique=256，UV plane 4147200 bytes、hash=8699452048464794797、unique=169。
- [x] 把 H.265 direct decode output image 接到现有 native Vulkan NV12 shader sampling：新增 `--sample-decoded-first-frame`，在 video decode queue 与 graphics queue 分离时创建双 queue-family device 和 concurrent NV12 resource image，用 semaphore 同步 decode -> graphics sampling，离屏渲染到 3840x2160 `R8G8B8A8_UNORM` 并 readback；2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main 源上验证 `output_sampling.rendered=true`、hash=7109389899594476375。
- [x] 为 H.265 continuous decode 准备真实 AU 窗口 telemetry：`--extract-bitstream --bitstream-samples 8` 不再只停在首个 VPS/SPS/PPS AU，而是输出 `h265_access_units[]`，包含每个 AU 的 bytes/hash/PTS/duration、参数集计数、IDR/IRAP、slice_type 和 POC LSB；2026-06-21 在同一 3840x2160@240 H.265 Main 源上验证 8 个 AU，PTS 为 0/4/8/12/16/20/25/29ms，首帧 IDR，后续 7 帧 `trail-r`、slice_type=1、POC LSB=1..7。
- [x] 为 H.265 continuous decode 补短期参考集 telemetry：解析每个非 IDR AU 的 short-term RPS，输出 negative/positive delta POC、used flags 和 used delta POC 列表；2026-06-21 真实 4K/240 H.265 源验证 AU1 used negative refs 为 `[-1, -16]`，AU2 为 `[-1, -2, -19, -292]`，证明连续 decode 不能按“只引用上一帧”的简化路径实现，下一步必须做 DPB slot/reference list 映射。
- [x] 为 H.265 continuous decode 补 DPB/POC reference plan telemetry：把 AU/RPS 转成 planned output slot、current POC、reference POC、available/missing refs 和 `ready_for_decode_submit`；2026-06-21 真实 4K/240 H.265 源验证 8-slot plan 中 AU0 ready，AU1 缺 POC `-15`，后续 AU 因 AU1 及窗口外 refs 缺失均不能直接提交。连续帧 decode 下一步需要 closed-GOP/可自包含测试源或显式处理缺失 refs 的策略，而不是盲提第二帧。
- [x] 为 H.265 continuous decode 补 reference-ready 4K/240 smoke gate：新增 `h265_decode_ready_count`、`h265_decode_ready_prefix_count`、first-unready telemetry、CLI `--require-h265-ready-prefix N` 和 `scripts/native-vulkan-h265-ready-prefix-smoke.sh`；同时支持 SPS short-term RPS 写入 Vulkan STD session parameters，并让 extract-only probe 记录 `session_parameters_error` 而不是吞掉 AU telemetry。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1` 生成 H.265 Main 3840x2160@240 short-GOP 源，验证 8 个 AU ready-prefix=8、`session_parameters_created=true`。
- [x] 将 H.265 direct decode 从首帧扩展到首个真实多帧 direct smoke：新增 `--decode-h265-ready-prefix N` 和 `h265_ready_prefix_decode` telemetry，把 ready-prefix AU 按 Vulkan bitstream offset/size alignment 顺序写入 `VIDEO_DECODE_SRC_KHR` buffer，录制连续 `vkCmdDecodeVideoKHR`，并 readback 最后一帧 NV12。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、H.265 Main 3840x2160@240 short-GOP 源上先验证 2-frame IDR+P decode；随后修正 IDR 后 DPB slot allocator，使 repeated-IDR 回到 slot0/slot1，并在非首帧 IDR 前记录 `vkCmdControlVideoCodingKHR(RESET)`。8-frame ready-prefix decode 已真实通过：ready-prefix=8、decoded=8、reset_count=4、AU7 readback layer1，Y/UV unique=205/256、hash=11542476098458954487/10292639723071029932。
- [x] 将 H.265 ready-prefix direct decode 输出接入 NV12 shader sampling：新增 `--sample-h265-ready-prefix` 和脚本 `--sample-prefix`，decoded texture 的 plane image view/layout barrier 改为按实际 `base_array_layer` 创建，ready-prefix 末帧可直接作为 Vulkan texture 渲染到离屏 RGBA target 并 readback。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main short-GOP 源上验证 8-frame direct decode + sampling 通过：result=`h265-ready-prefix-decode-output-sampled-and-readback-completed`、decoded=8、reset_count=4、AU7 readback/sample layer1、RGBA hash=14093713610652448641、RGBA unique=256、nonzero=24967096。
- [x] 将 H.265 ready-prefix sampled output 从“末帧验证”扩展到“逐帧验证”：新增 `--sample-h265-ready-prefix-sequence` 和脚本 `--sample-prefix-sequence`，每个 AU decode 后立即 readback + shader sample，再把被采样 layer 显式转回 `VIDEO_DECODE_DPB_KHR` 供后续 reference/slot 复用，避免等 8 帧结束后只能看到最后两个 DPB slot。2026-06-21 在 `WAYLAND_DISPLAY=wayland-1`、NVIDIA 4060、3840x2160@240 H.265 Main short-GOP 源上验证 8-frame sequence decode + sampling 通过：result=`h265-ready-prefix-decode-output-sequence-sampled-and-readback-completed`、decoded=8、reset_count=4、sampled_sequence_count=8、sampled layers=`[0,1,0,1,0,1,0,1]`、distinct RGBA hashes=4、每帧 RGBA unique=256。
- [x] 为 H.265 ready-prefix sequence smoke 补逐帧 timing/pacing telemetry：`output_sampling_sequence[]` 记录每帧 PTS delta、decode submit/wait、NV12 readback、RGBA sampling/readback 和 total frame elapsed，`output_sampling_sequence_timing` 记录 max/avg 与 PTS delta min/max；脚本 `--sample-prefix-sequence` 将 timing 变成 gate。2026-06-21 真实 4K/240 H.265 Main short-GOP 验证：sequence_count=8、PTS delta min/max=4/5ms、max decode submit/wait=5951us、max readback=38720us、max sampling=65580us、avg debug frame=92997us。这里的 avg 包含 host readback 验证成本，不作为可见 swapchain 240fps 性能结论。
- [x] 为 H.265 ready-prefix sequence 补 render-only telemetry：在每帧 readback 验证后，复用同一个 offscreen color target 和 `NativeVulkanVideoRenderer` 再做一次 NV12 shader render，但不做 CPU copy/readback，记录 `output_render_sequence[]` 和 `output_render_sequence_timing`，用来逼近后续 swapchain/present 的渲染成本。2026-06-21 真实 4K/240 H.265 Main short-GOP 验证：render_sequence_count=8、layers=`[0,1,0,1,0,1,0,1]`、PTS delta min/max=4/5ms、average render-only=934us、max render-only=1559us，证明当前 90ms 级 debug frame 成本主要来自验证 readback，不是 NV12 shader render。
- [x] 为可见 H.265 ready-prefix path 补 DPB 最小化和 H.264 GPU-memory 对照结论：DPB 选择不再简单用 `max_active_refs + 1`，而是从 1 到 SPS 上限寻找最小可完整解码的 slot 数，并把“当前输出将覆盖的 slot”视为不可继续作为参考帧，避免过小 DPB 造成重复帧/跳变。2-ref 4K/240 H.265 evidence `/tmp/gilder-vulkan-h265-ready-prefix-video.XoHK5C` 仍需 3 层 NV12，`video_resource_memory_bytes=37552128`、`session_memory_bytes=33775616`；1-ref short-GOP evidence `/tmp/gilder-vulkan-h265-ready-prefix-video.q8NPT5` 降到 2 层，`video_resource_memory_bytes=25034752`，resource/session/bitstream 合计约 59.1MB。native-wgpu H.264 GPU-memory continuous 对照 `/tmp/gilder-native-wgpu.SWqa42` 为 `gst-dmabuf`/`cuda-direct`，`Private_Dirty max=68928 KiB`、CPU avg `26.80%`、`average_render_fps=240.09`，并且不会出现 ready-prefix window 的 `AU239 -> AU0` 强制 reset 跳变。
- [x] 历史中间阶段曾将可见 H.265 ready-prefix 的 AU payload 保留从 `Vec<Vec<u8>>` 改为临时 spool 文件加单个复用上传 buffer，避免 4K/240 长 ready-prefix 把约 499MB encoded AU payload 计入进程私有内存；当前维护面已切到 streaming queue，不再维护 spool。2026-06-21 真实 Wayland `HDMI-A-1`、3840x2160@240、4800 AU/4800 playback frames 证据 `/tmp/gilder-vulkan-h265-memory-spooled.d8pybb`：`decoded_frame_count=4800`、`presented_frame_count=4800`、`average_present_fps=239.977`、`bitstream_window_payload_bytes=499056595`、`bitstream_buffer_slot_bytes=249344`，86 个 250ms smaps samples 中 `RSS/PSS/USS/Private_Dirty max=117732/85864/68248/37664 KiB`。旧 in-memory payload 证据 `/tmp/gilder-vulkan-h265-memory.GIYC3r` 为 `RSS/PSS/USS/Private_Dirty max=1089060/1069592/1061636/1008992 KiB`，确认此前高 `Private_Dirty` 主因是 retained AU payload，不是 Vulkan Video resource/session/bitstream 显式资源。
- [x] 历史中间阶段优化 H.265 spool -> mapped bitstream upload：播放循环不再先读入临时 `Vec<u8>` 再拷贝到 Vulkan mapped buffer，而是从 spool file 直接 `read_exact` 到持久映射的 `VIDEO_DECODE_SRC_KHR` buffer；同时跟踪顺序读取位置，顺序帧避免重复 seek，并清理当前 AU aligned range 内的 padding，避免 stale bitstream tail 进入 decoder。当前维护面已切到 streaming queue，不再维护 spool。
- [x] 将 visible H.265 encoded input 改为 Bitstream Ring Buffer：默认 2-slot、固定容量 `VIDEO_DECODE_SRC_KHR` buffer，按 driver offset/size alignment 追加写入，runtime JSON 记录 `src_buffer_offset`、payload/range、allocation index、wrap count 和 ring capacity；当前可见路径每帧等待 present fence 后再分配，后续 decode-ahead 再把 timeline/fence serial 接入 range 回收。2026-06-21 真实 Wayland `HDMI-A-1`、3840x2160@240、4800 frames 证据 `/tmp/gilder-vulkan-h265-ready-prefix-video.Ldh5wL`：`average_present_fps=240.041`、`bitstream_buffer_strategy=fixed-capacity-persistent-mapped-ring`、`bitstream_buffer_bytes=498688`、`bitstream_ring_wrap_count=1200`；同配置 smaps 证据 `/tmp/gilder-vulkan-h265-ring-memory.9RFFoa`：`RSS/PSS/USS/Private_Dirty max=117836/86018/68380/37932 KiB`、`average_present_fps=240.063`。
- [x] 将 visible H.265 encoded input 接到 bounded streaming packet queue：新增 `NativeVulkanH265VideoInputMode`、CLI `--h265-input streaming-queue` 和脚本 `--streaming-queue`，H.265 也按需从 GStreamer parser/appsink 拉 AU、上传到 bitstream ring 后释放 payload，并在 runtime JSON 中报告 `h265_packet_queue_*`。2026-06-21 真实 Wayland `HDMI-A-1` 4K/240 smoke `/tmp/gilder-vulkan-h265-ready-prefix-video.uMgUWp` 为 `decoded/presented=240/240`、`average_present_fps=238.316`、`queue_capacity=32`、`queue_pulled=240`、`queue_retained=0`；20s smaps 证据 `/tmp/gilder-vulkan-h265-streaming-smaps.wTY7vB` 为 `decoded/presented=4800/4800`、`average_present_fps=240.027`、`queue_eos/loops=19/19`、`RSS/PSS/USS/Private_Dirty max=115480/71078/51800/33892 KiB`、平均 CPU `20.11%`。
- [x] 将 H.264/H.265 visible direct streaming packet queue 合并为共用输入层：新增 codec hook/泛型队列，统一 GStreamer pipeline/appsink/bus、bootstrap 参数集选择、EOS loop、payload retained accounting 和 bitstream ring sizing；H.264/H.265 只保留参数集解析、snapshot 与 pipeline hook。2026-06-21 真实 Wayland `HDMI-A-1` 回归：H.264 720p/60 `/tmp/gilder-vulkan-common-queue-h264-720p60` 为 `decoded/presented=60/60`、`queue_retained=0`、`average_present_fps=247.329`；H.264 4K/240 `/tmp/gilder-vulkan-common-queue-h264-4k240` 为 `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、`average_present_fps=199.922`；H.265 4K/240 `/tmp/gilder-vulkan-common-queue-h265-4k240` 为 `decoded/presented=240/240`、`queue_retained=0`、`average_present_fps=238.368`。H.264 4K/240 长跑 `/tmp/gilder-vulkan-common-queue-h264-4k240-memory/performance` 为 `decoded/presented=2400/2400`、`average_present_fps=204.900`、`queue_eos/loops=9/9`、`queue_retained=0`，8 个 samples 中 `RSS/PSS/USS/Private_Dirty max=105404/63269/45272/26844 KiB`、平均 CPU `12.49%`、NVIDIA 进程 GPU memory `104 MiB`。
- [x] 将 H.264 visible streaming queue 推进到任意非 IDR 入口重对齐：bootstrap 会 bounded scan 到 SPS/PPS/IDR，只保留固定 capacity 窗口，丢弃不可解 P/B 前缀，并把 EOS loop skip 同步到同一个可解入口；默认扫描上限提高到 4096/`capacity*128`，仍不保留被丢弃 payload。`scripts/native-vulkan-h264-ready-prefix-video-smoke.sh --arbitrary-entry-offset` 会用 `ffmpeg -copyinkf` 生成非关键帧入口源并 gate bootstrap discard、loop skip 和首帧 IDR。2026-06-21 真实 Wayland `HDMI-A-1` 证据：720p/60 B/P 源 `/tmp/gilder-vulkan-h264-arbitrary-entry-script-gate` 从 `0.35s` 非关键帧入口启动，`decoded/presented=60/60`、`bootstrap_discarded=39`、`loop_skip=39`、`first_frame_idr=true`、`p_frames=30`、`b_frames=29`、`max_reference_count=2`、`queue_retained=0`；手工 copyinkf 回归 `/tmp/gilder-vulkan-entry-realign-h264-copyinkf-v3` 丢弃 99 个坏前缀 AU 后可见播放 60 帧。4K/240 回归：H.264 `/tmp/gilder-vulkan-h264-arbitrary-entry-4k240-regression-seq` 为 `decoded/presented=240/240`、`b_frames=119`、`queue_retained=0`、`average_present_fps=198.012`；H.265 对照 `/tmp/gilder-vulkan-h265-bootstrap-scan-4k240-regression` 为 `decoded/presented=240/240`、`queue_retained=0`、`average_present_fps=240.927`；`cargo test --features native-vulkan-gst-video` 297 个测试通过。
- [x] 将 H.264 direct visible 扩展到 interlaced/MBAFF frame picture：High 8-bit SPS/PPS readiness 不再把 `frame_mbs_only_flag=false` 直接拒绝，layout 选择对非 frame-only 源优先 `INTERLACED_INTERLEAVED_LINES`/`INTERLACED_SEPARATE_PLANES`；planner 对 `field_pic_flag=true` 不再硬 gate，并按 top/bottom field 选择 `PicOrderCntVal`、允许同一 `frame_num` 的互补场不触发 gap 推断。2026-06-22 真实 Wayland `HDMI-A-1` evidence `/tmp/gilder-vulkan-h264-interlaced-mbaff-visible` 使用 x264 interlaced/MBAFF H.264 源，`decoded/presented=60/60`、`h264_picture_layout=interlaced-interleaved-lines`、`h264_input_mode=streaming-queue`、`b_frames=38`、`max_reference_count=3`、`video_resource_memory_bytes=7536640`、`session_memory_bytes=2215936`、`bitstream_buffer_bytes=524288`。真实 `field_pic_flag=true` field-coded 码流仍缺 smoke 源，当前覆盖为 planner/submit 单测。
- [x] 将 H.265 visible streaming queue 也纳入任意非 IDR 入口 gate：`scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --arbitrary-entry-offset` 生成 `ffmpeg -copyinkf` 非关键帧入口源，要求 bootstrap discard、loop skip 和首帧 IDR；同时把 H.265 bitstream ring gate 从“窗口 payload 必须大于 ring capacity”改成“固定 ring slot 数必须小于 decode window”，避免低码率 720p H.265 合法 streaming 被误判。2026-06-21 真实 Wayland `HDMI-A-1` 证据：H.265 720p/60 `/tmp/gilder-vulkan-h265-arbitrary-entry-script-gate-v2` 从 `0.35s` 非关键帧入口启动，`decoded/presented=60/60`、`bootstrap_discarded=39`、`loop_skip=39`、`first_frame_idr=true`、`frame_access_units_head` 从 39 开始、`queue_retained=0`；H.265 4K/240 `/tmp/gilder-vulkan-h265-arbitrary-entry-4k240-regression` 为 `decoded/presented=240/240`、`average_present_fps=239.919`、`queue_retained=0`、`video_resource_memory_bytes=37552128`、`session_memory_bytes=33775616`、`bitstream_buffer_bytes=1036800`；同一当前工作树下 H.264 arbitrary-entry `/tmp/gilder-vulkan-h264-arbitrary-entry-current-regression` 为 `decoded/presented=60/60`、`bootstrap_discarded=39`、`loop_skip=39`、`first_frame_idr=true`、`max_reference_count=2`、`queue_retained=0`；`cargo test --features native-vulkan-gst-video` 297 个测试通过。
- [ ] 将 AV1 encoded frontend 也从 ready-prefix/first-frame 文件采样推进到连续 demux/parser ring producer，并接入共用 streaming input layer：metadata ring 记录 PTS/duration/slice或tile offset/reference info，decode fence/timeline 释放 range；目标是从任意合法入口点逐步建立 DPB/reference 状态并支持连续播放、音频 clock，而不是长期依赖 reference-ready ready-prefix window。2026-06-22 已补 AV1 segmentation-enabled frame OBU header parsing，使真实 Main10 gate `/tmp/gilder-vulkan-av1-bitstream-present-worker-10-fixed3` 从“header not found”推进到 `av1_first_frame_header_found=true`、`frame_type=key`、`tile_count=16`、`session_parameters_codec=av1-main-10`。随后补 AV1 `disable_frame_end_update_cdf` 与 uniform tile spacing 解析，真实 4K Main10 gate `/tmp/gilder-vulkan-av1-disable-frame-end-cdf-gate` 已推进到 `av1_first_frame_submit_candidate=true`、`tile_offsets=[27]`、`tile_sizes=[33552]`、`session_parameters_codec=av1-main-10`。当前代码已把 AV1 temporal unit 接入 H.264/H.265 共用 `NativeVulkanStreamingPacketQueue`，bootstrap sequence header 作为 queue parameter set 复用，frame-only temporal unit 也能借 active sequence header 生成 first-frame submit snapshot；packet timeline 统一保留 access-unit index、source-loop index、PTS 和 duration，后续音频接入可把 GStreamer audio/clock 对齐到同一 metadata ring。真实 4K Main10 direct gate 已推进到首帧 `vkCmdDecodeVideoKHR`、P010 readback 和 shader sampling：`/tmp/gilder-vulkan-av1-p010-sampling-script` 为 `first-frame-decode-output-sampled-and-readback-completed`、`first_frame_decode.codec=av1-main-10`、`output_sampling.rendered=true`。本轮继续补 AV1 inter frame header reference telemetry：`NativeVulkanAv1FrameSubmitSnapshot` 现在输出 `reference_order_hints`、`frame_refs_short_signaling`、`last_frame_idx`、`gold_frame_idx` 和 7 个 `ref_frame_indices`；真实 Main10 smoke `/tmp/gilder-vulkan-av1-inter-ref-telemetry-main10` 保持首帧 P010 direct decode/sampling 通过，并在后续 TU 中看到 inter `order_hint` 与 `ref_frame_indices`，例如 `[3,0,0,0,2,0,1]`、`[5,6,2,0,4,3,1]`。这说明 AV1 已越过“inter/reference header 未解析”的旧断点，但仍未 submit inter frame；下一步是实现 reference-name slot planning、show-existing-frame 处理、continuous visible runtime 和真实 FPS/内存采集。
  2026-06-22 继续修正真实 AV1 show-existing TU 形态：frame-header-only `show_existing_frame` 不再被误判为“缺 tile-group”，而是保留 `frame_to_show_map_idx` 并明确停在 display handoff/reference slot planning。真实 Main10 smoke `/tmp/gilder-vulkan-av1-show-existing-split-fix-main10` 为 `first-frame-decode-output-sampled-and-readback-completed`，后续 TU 中 `show_existing_frame=true`、`frame_to_show_map_idx=2/5`，unsupported reason 变为 `display handoff needs reference slot planning`。这是 2026-06-22 的历史状态；后续条目已把 AV1 推进到任意入口连续可见 correctness gate，剩余转为 4K/240 性能、真实码流覆盖、audio/clock 和内存压缩。
- [x] 将 AV1 Main8/Main10 任意入口连续可见 correctness 推进到真实 Wayland gate：`scripts/native-vulkan-av1-ready-prefix-video-smoke.sh` 支持 `--arbitrary-entry-offset`、`--require-loop-skip-replay`、`--require-readback-diversity` 和 performance snapshot。AV1/WebM 的坏前缀可能在 `av1parse`/demux 阶段先被丢弃，因此 gate 明确记录 `arbitrary_entry_demux_dropped_prefix=yes` 或 runtime queue skip 作为前缀处理证据，再要求首帧 key、loop replay、zero retained payload、8-slot bitstream ring、DPB/session 一致和可选 readback diversity。2026-06-23 真实 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 证据：Main8 小窗口 `/tmp/gilder-av1-arbitrary-main8-script-gate` 与 Main10/P010 `/tmp/gilder-av1-arbitrary-main10-script-gate` 均为 `presented=120`、`playback_loop_count=2`、`loop_boundary_reset_count=1`、`readback_y_distinct=5`、`readback_uv_distinct=5`；4K/240 correctness gate Main8 `/tmp/gilder-av1-main8-arbitrary-4k240-script-gate` 为 `presented=480`、`decoded=260`、`hidden_decoded=238`、`displayed_handoff=220`、`average_present_fps=214.309`，Main10/P010 `/tmp/gilder-av1-main10-arbitrary-4k240-script-gate` 为同样帧结构、`average_present_fps=195.084`。4K/240 no-readback performance：Main8 `/tmp/gilder-av1-main8-arbitrary-4k240-performance` 为 `presented=2400`、`playback_loop_count=8`、`average_present_fps=212.343`、`RSS/PSS/USS/Private_Dirty max=105732/72353/58468/29380 KiB`、CPU `23.41%`；Main10/P010 `/tmp/gilder-av1-main10-arbitrary-4k240-performance` 为 `presented=2400`、`average_present_fps=211.028`、`RSS/PSS/USS/Private_Dirty max=108404/74981/61104/30172 KiB`、CPU `10.09%`。结论：AV1 任意入口连续 correctness 完成；剩余是 4K/240 稳定性能、真实壁纸码流矩阵、audio/clock 和 9-slot DPB/output 内存压缩。
- [x] 将任意入口 reset recovery 语义收紧到 FFmpeg 式随机访问点：H.264/H.265 streaming bootstrap 即使 reference planner 认为窗口全 ready，也必须让 reset 后的首个 AU 从 recovery point 开始；当前 H.264/H.265 recovery point 为 IDR，AV1 为 shown key frame。H.264 runtime 的 direct-DPB、display-ring 和 decode-ahead 分支增加 reset recovery guard，H.265 runtime 在 loop/reset 前重置 planner 并扫描到 recovery AU；H.264 smoke 的 `first_frame_recovery`/loop recovery gate 不再把“reset 后 DPB 为空的非 IDR”算作恢复。2026-06-24 真实 Main + AAC MP4 `/tmp/gilder-h264-real-kamen-2-arbitrary-entry-loop-1440p60-900-b` 从 `0.35s` 非 IDR 入口启动，`decoded/presented=900/900`、`bootstrap_discarded=9`、`loop_skip=9`、首帧 AU 9 为 IDR、第二轮首帧 AU 850 为 IDR、`loop_first_non_idr_count=0`、`average_present_result_drop_first_60_fps=59.979`。同轮 4K/240 回归：H.265 Main8 `/tmp/gilder-h265-main8-arbitrary-recovery-bootstrap-4k240-480-b` 为 `decoded/presented=480/480`、`bootstrap_discarded=156`、`loop_skip=156`、`first_frame_idr=true`、`average_present_fps=239.417`；H.265 Main10 长跑 `/tmp/gilder-h265-main10-arbitrary-recovery-bootstrap-4k240-2400-b` 为 `decoded/presented=2400/2400`、`loop_first_non_idr_count=0`、`average_present_fps=238.912`，说明 correctness 稳定但 Main10 4K/240 pacing 仍需调度层继续推进；AV1 Main8/Main10 `/tmp/gilder-av1-main8-arbitrary-recovery-bootstrap-4k240-480-b`、`/tmp/gilder-av1-main10-arbitrary-recovery-bootstrap-4k240-480-b` 均为 `presented=480`、`first_frame_key=true`、`loop_first_non_key_count=0`、`arbitrary_entry_demux_dropped_prefix=yes`、warmup-dropped present-result FPS `239.938/239.968`。
- [x] 接入 audio/clock 第一阶段证据脚本：新增 `scripts/native-vulkan-audio-clock-probe.sh`，先用 ffprobe 固定音频 stream/packet PTS，再用显式 GStreamer AAC 音频链 `qtdemux.audio_0 ! aacparse ! avdec_aac ! fakesink` 跑 clocked playback probe，避免 `playbin` 同时实例化视频 decoder。2026-06-24 真实 Main + AAC MP4 `/tmp/gilder-audio-clock-probe-kamen-2-aac-10s-c` 为 AAC LC、48kHz、stereo、`audio_packet_count=469`、`audio_pts_delta_min/max=0.021333/0.021334`、`gst_new_clock_count=1`、`gst_stream_start_count=1`、`gst_state_playing_count=9`，说明音频前端/时钟 telemetry 已可测；下一步再把该 clock 接入 native Vulkan pacer，而不是继续只靠 target-fps。
- [x] 将 audio clock probe 接入 H.264 visible runtime gate，并按 FFmpeg/ffplay 的 clock serial 思路修正 loop/reset：`gilder-native-vulkan --run-h264-ready-prefix-video --audio-clock-probe` 现在启动独立 AAC appsink clock pipeline，但不再让音频在视频 setup 期间 free-run；首个视频采样启动音频 clock，loop reset 推进 `audio_clock_serial`，旧 serial 的 position/sample 会被丢弃，runtime JSON 输出 monotonic audio-master estimate 和 master drift。2026-06-24 真实 Main + AAC 任意入口 loop replay `/tmp/gilder-h264-real-kamen-2-audio-clock-ffplay-serial-1440p60-900-f` 为 `decoded/presented=900/900`、`first_frame_recovery=true`、`loop_first_unrecovered_count=0`、`audio_reached_clocked_playback=true`、`audio_decoders=["avdec_aac"]`、`audio_video_decoders=[]`、`audio_clock_serial=2`、`audio_loop_restart_count=1`、`audio_position_query_hit_count=897/900`、`audio_position_stale_count=0`、`audio_sample_stale_count=0`、`audio_video_master_clock_drift_latest_ns=-61777`、`audio_video_master_clock_drift_abs_max_ns=856739`。这把上一轮约 0.78s/0.94s 的 loop drift 收敛到 sub-ms 级；下一步才是把 audio clock 升为 pacer master。
- [ ] 继续推进 audio/clock 主线路：H.264/H.265/AV1 ready-prefix runtime 已共用
  FFmpeg/ffplay-style audio serial/master-clock probe，并修正 initial start 与 loop reset 都按 video
  PTS seek 到同一 audio segment，避免任意入口/非零 PTS 时 audio pipeline 实际从 0 开始而 segment
  标成新位置。H.264 真实短源 loop gate `/tmp/gilder-h264-audio-seek-loop-240` 为
  `decoded/presented=240/240`、`playback_loop_count=2`、`loop_boundary_reset_count=1`、
  `audio_clock_serial=2`、`audio_loop_seek/restart/error=1/1/0`、`audio_position_stale_count=0`、
  `audio_sample_stale_count=0`、`audio_video_master_clock_drift_abs_max_ns=853202`。同轮
  H.265/AV1 AAC loop gate 已覆盖 Main8/Main10：H.265 Main8
  `/tmp/gilder-h265-main8-audio-loop-620` 与 Main10
  `/tmp/gilder-h265-main10-audio-loop-620` 均为 `decoded/presented=620/620`、
  `playback_loop_count=2`、`loop_boundary_reset_count=1`、`audio_clock_serial=2`、
  `audio_loop_seek/restart/error=1/1/0`、stale sample/position 均为 0，master drift abs max
  分别为 `91888ns`/`133513ns`；AV1 Main8 `/tmp/gilder-av1-main8-audio-loop-120`
  与 Main10 `/tmp/gilder-av1-main10-audio-loop-120` 均为 `presented=120`、
  `playback_loop_count=2`、`loop_boundary_reset_count=1`、`audio_clock_serial=2`、
  `audio_loop_seek/restart/error=1/1/0`、stale sample/position 均为 0，master drift abs max
  为 `272170ns`/`354300ns`。2026-06-24 已把 `GILDER_VIDEO_PACING_MASTER=audio` 和 smoke
  `--pacing-master audio` 接入 H.264/H.265/AV1 ready-prefix runtime：默认仍保持 target-fps
  master，opt-in 时用 audio master clock 计算下一帧 sleep，缺 clock sample 时回退 target-fps。
  真实 gate：H.264 `/tmp/gilder-h264-audio-master-pacing-segment-clock-240` 为
  `decoded/presented=240/240`、`pacing_strategy=audio-clock-master-with-target-fps-fallback-and-fifo-present`、
  `frame_sleep_count=238`、`missed_frame_pacing_count=1`、warmup 后
  `average_present_result_drop_first_60_fps=60.007690371045314`、
  `audio_loop_seek/restart/error=1/1/0`、master drift abs max `16606575ns`；H.265 Main8
  `/tmp/gilder-h265-main8-audio-master-pacing-segment-clock-620` 为 `decoded/presented=620/620`、
  `runtime_elapsed_ms=10317`、master drift abs max `16347675ns`；AV1 Main8 先暴露出
  PTS 空洞 fallback 使用全局 playback frame index 的错误，loop reset 后会把 video clock 推到
  旧 segment 之外，导致 120 帧跑到约 30s。修成 FFmpeg/ffplay 式 segment clock 后，
  `/tmp/gilder-av1-main8-audio-master-pacing-segment-clock-120` 为 `presented=120`、
  `runtime_elapsed_ms=2004`、warmup 后
  `average_present_result_drop_first_60_fps=59.93388987198527`、`audio_loop_seek/restart/error=1/1/0`、
  master drift abs max `16780374ns`。同轮补实际音频输出第一阶段：`--audio-output auto`
  会把独立 AAC audio clock pipeline 变为 `qtdemux-aacparse-avdec_aac-tee-appsink-autoaudiosink`，
  默认策略已从固定 `clock-only` 推进到 `--audio-output plan`：沿用上层
  `entry.muted || !runtime.allow_audio` 合成后的有效 muted 状态，muted 解析为
  `clock-only`，unmuted 解析为 `auto`，仍可用显式 `clock-only`/`auto` 覆盖。
  H.264/H.265/AV1 ready-prefix smoke 已接入 `--muted/--unmuted`、`--audio-output`
  和 sink-count gate；短 H.264 脚本 gate
  `/tmp/gilder-h264-audio-output-auto-script-60` 为 `decoded/presented=60/60`、
  `audio_output=auto`、`audio_output_mode=auto`、`audio_output_sink_count=2`、
  `audio_output_sinks=["autoaudiosink","jackaudiosink"]`、`audio_decoders=["avdec_aac"]`、
  `video_decoders=[]`、`audio_reached_clocked_playback=true`。runtime snapshot 已按
  manifest `muted` 规划输出策略。2026-06-24 继续补 plan-following gate：
  `/tmp/gilder-h264-audio-output-policy-module-plan-unmuted-60` 使用 `--audio-output plan --unmuted`
  通过，`decoded/presented=60/60`、`audio_output_expected_mode=auto`、
  `audio_plan_muted=false`、`audio_output_mode=auto`、`audio_output_sink_count=2`。同轮把
  `NativeVulkanAudioOutputMode/Policy` 拆到无 GStreamer 依赖的 `audio_policy.rs`，并让
  manifest-backed `VideoWallpaperPlan` runtime snapshot 输出 `audio_output_policy=plan` 后再
  resolve muted -> `clock-only`、unmuted -> `auto`。2026-06-24 已把 native Vulkan renderer 的
  实际 audio runtime 启停接到该 plan-following 输出路径，并继续推进成 worker/channel
  边界：`--run-video --unmuted` 按 plan resolve 到 `auto` 后启动独立 AAC runtime worker，
  video 主循环只发送 video clock sample，GStreamer audio probe 由 worker 持有并合并积压
  sample。
  真实 Wayland 1s 检查 `artifacts/video-sources/h264/audio-loop/kamen-h264-aac-2s-loop.mp4`
  为 `frames_rendered=60`、`average_render_fps=59.99649182513351`、
  `audio_runtime_status=clocked-playback-active`、`audio_runtime_buffer_count=42`、
  `audio_runtime_output_sink_count=2`、`audio_runtime_position_query_hit_count=60`、
  `audio_runtime_last_error=null`。snapshot 已把 `audio_output_*` 策略层和
  `audio_runtime_*` 运行层分开；下一步沿这个边界拆 video demux/decode/render/present。
- [x] 将 H.265 direct decode + sampled texture 从离屏 smoke 接到连续 display/swapchain，并补 frame pacing/queue 同步/释放 telemetry 和安全可见 smoke。2026-06-22 真实 Wayland `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1`、3840x2160@240 任意非 IDR 入口回归 `/tmp/gilder-vulkan-h265-after-h264-barrier-tightened` 为 `decoded/presented=2400/2400`、`playback_loop_count=9`、`loop_boundary_reset_count=8`、`h265_packet_queue_retained_payload_bytes=0`、`average_present_fps=239.82864245894595`；8 个性能 samples 为 `RSS/PSS/USS/Private_Dirty max=102456/88200/83636/24684 KiB`、平均 CPU `11.35%`、NVIDIA 进程 GPU memory `152 MiB`。
- [ ] 实现完整 native Vulkan Video decode path：H.265 main-8/H.264 high-8 已有 demux/parser streaming queue、codec parameters、Vulkan Video `vkCmdDecodeVideoKHR`、visible swapchain present 和任意入口 loop replay gate；H.264 复杂 4K/240 仍未稳定到 240fps，剩余重点是更深 decode/present decoupling、固定帧槽/descriptor/present ring、timeline/fence range 回收、audio/clock、AV1 连续 decode，以及 Main10 direct path。2026-06-22 H.264 present-overlap + persistent present worker 真实 `HDMI-A-1` evidence `/tmp/gilder-vulkan-h264-present-worker` 为 `decoded/presented=2400/2400`、`average_present_fps=234.53720838404902`、average `queue_present_us=3975`、`RSS/PSS/USS/Private_Dirty max=104972/76817/60048/28580 KiB`、平均 CPU `14.77%`、NVIDIA GPU memory `128 MiB`；相比 `/tmp/gilder-vulkan-h264-barrier-tightened-final` 的 `207.34187751641383fps` 是实质提升，但仍未达到 H.265 的 240fps 稳定形态。同日继续把 H.264 display ring 改为每槽位预绑定 descriptor set，热循环不再每帧 `vkUpdateDescriptorSets`/构造临时 display texture；真实 `HDMI-A-1` 4K/240 ref=1 evidence `/tmp/gilder-vulkan-h264-prebound-descriptor-4k240-ref1` 为 `decoded/presented=480/480`、`average_present_fps=232.68396113217636`、`avg_descriptor_update_us=0`，5s performance `/tmp/gilder-vulkan-h264-prebound-descriptor-4k240-perf` 为 `decoded/presented=1200/1200`、`average_present_fps=233.90643962520952`、`RSS/PSS/USS/Private_Dirty max=106000/91369/86404/27424 KiB`、平均 CPU `15.60%`。这是 CPU 侧提交开销优化，不是 H.264 240fps 完成。后续参考 Sunshine/FFmpeg/GStreamer/mpv 的顶层思路时，只借调度模型：固定 hardware frame/surface pool、slot ownership、timeline semaphore、descriptor 只在 source/target 变化时更新、命令 ring 和延迟销毁；不照搬具体编码器/捕获实现。2026-06-22 又补 H.264 resource layout 实验：`GILDER_H264_RESOURCE_LAYOUT=general` 让 decode resource/display-copy 保持 `GENERAL`，runtime/summary 增加 `h264_resource_image_layout`；真实 `HDMI-A-1` 4K/240 ref=1 evidence `/tmp/gilder-vulkan-h264-resource-general-4k240-ref1` 为 `average_present_fps=233.11475907497862`、`decode avg=11.79us`，但重跑 `/tmp/gilder-vulkan-h264-resource-general-layout-field-4k240-ref1` 为 `232.52402677308388fps`，说明它是可用的小幅同步优化，不能作为 H.264 稳 240 完成项。
- [ ] 将 H.264 display-ring 同步从 binary semaphore + 全局 fence 推进到完整 frame-pool ownership：2026-06-22 已把 H.264 display-ring 路径改为 per-frame acquire semaphore/fence，并为 display slot 复用增加 fence guard，避免 GPU 仍在采样旧 slot 时提前 copy 新帧。真实 Wayland `HDMI-A-1` evidence `/tmp/gilder-vulkan-h264-display-slot-fence-4k240-ref1` 为 `decoded/presented=480/480`、`average_present_fps=230.31172461134605`、`h264_present_result_wait_count=479`、`h264_present_result_wait_elapsed_us=1929885`、`avg_fence_wait_us=0.89`、`avg_present_us=310.68`；1200 帧 performance evidence `/tmp/gilder-vulkan-h264-display-slot-fence-4k240-perf` 为 `decoded/presented=1200/1200`、`average_present_fps=232.89863472099296`、`RSS/PSS/USS/Private_Dirty max=106000/90291/84616/27544 KiB`、NVIDIA GPU memory `116 MiB`，CPU raw avg `36.84%` 受第一个 0s sample=`100%` 影响，后续 1-4s samples 为 `28.6/20.9/18.0/16.7%`。该改动偏稳定性/所有权正确性，单独不是 240fps 或内存突破。负面 evidence：`GILDER_H264_ASYNC_PRESENT_DEPTH=2` 在 `/tmp/gilder-vulkan-h264-per-frame-fence-depth2-4k240-short-seq` 可跑完但降到 `219.4879316010344fps`，因为单 present queue 的 mutex 把 FIFO 等待转移到 `avg_submit_us=4175.98`；`GILDER_H264_PRESENT_QUEUE_COUNT=2` 在 `/tmp/gilder-vulkan-h264-per-frame-fence-dual-present-4k240-short-seq` 20s 超时且 runtime 为空，不能作为默认路径。下一步应迁 timeline semaphore/range 回收和更明确的 bounded decode/display/present 队列，而不是继续加深当前 present worker。
- [ ] 把 H.264 近期同步优化整理成跨 codec 同步层并迁到 H.265/Main10/AV1：抽出 persistent present worker、decode/present overlap、固定 display/descriptor ring、timeline/fence range 回收和 per-frame telemetry，H.265 先以不回退 4K/240 稳定性为约束迁 present worker/display handoff，H.264 继续用该层压 240fps；所有变更必须用真实 Wayland `HDMI-A-1` 采集 FPS、CPU、RSS/PSS/USS/Private_Dirty，对比旧同步路径。
- [ ] 拆分 `src/renderer/native_vulkan.rs`：当前文件已承载 Wayland host glue、device/session、swapchain/present、video input queue、bitstream ring、H.264/H.265/AV1 parser/submit、YUV sampling、DMABuf/CUDA/VA import 和 telemetry，后续继续大幅重构前应拆成 `native_vulkan/{host,device,session,present,video_input,video_sync,video_resource,codecs/{h264,h265,av1},sampling,telemetry}.rs` 这类边界。拆分时保持现有 public snapshot/CLI JSON 字段稳定，先移动纯 helper 和 codec parser，再移动 runtime；不要在同步/解码正确性未验证时同时改行为。
- [ ] 将 AV1 hidden decode 同步从 command-buffer ring 继续推进到 timeline semaphore + present decouple：2026-06-23 真实 `HDMI-A-1` 4K/240 arbitrary-entry Main8 evidence `/tmp/gilder-av1-main8-hidden-handoff-readback` 显示上一轮“immediate show-existing semaphore handoff”没有命中，`av1_hidden_decode_async_handoff_count=0`、`av1_hidden_decode_queue_wait_count=238`、`average_present_fps=201.318`，说明该源 hidden decode 与 show-existing 不是稳定相邻形态。随后把 hidden decode 默认同步从 `vkQueueWaitIdle` 缩到 per-submit fence wait，Main8 长跑 `/tmp/gilder-av1-main8-hidden-fence-4k240-performance` 为 `average_present_fps=209.958`、CPU `15.03%`、`RSS/PSS/USS/Private_Dirty max=108732/95546/90860/30652 KiB`。本轮进一步实现 8-slot AV1 decode command ring、per-slot fence、pending decode submission、bitstream range overlap wait、show-existing/readback/final wait，并把 AV1 decode prepare barrier 从全 DPB layers 缩到 selected layers，依赖同一 video queue 的提交顺序而不是每个 reference/output slot 都 CPU wait；runtime/summary 新增 `av1_decode_command_ring_depth`、`av1_decode_pending_max_count`、`av1_decode_deferred_hidden_count` 和 `av1_decode_slot_wait_*`。真实 evidence：Main8 readback `/tmp/gilder-av1-main8-decode-ring-queue-ordered-readback` 仍 `readback_y/uv_distinct=5`，`av1_decode_pending_max_count=8`、`hidden fence elapsed=15434us`；Main8 10s performance `/tmp/gilder-av1-main8-final-fifo-4k240-performance` 为 `presented=2400`、`average_present_fps=211.157`、CPU `9.12%`、`RSS/PSS/USS/Private_Dirty max=108760/95956/91272/30744 KiB`、`av1_hidden_decode_fence_wait_elapsed_us=25`、`av1_decode_slot_wait_elapsed_us=68`。剩余瓶颈转为 FIFO/Wayland present：该证据中 `queue_present_elapsed_us avg=4627.7`、`present_elapsed_us avg=4642.7`，约 927/2400 帧 queue-present 超过 4.166ms；`GILDER_VULKAN_PRESENT_MODE=mailbox` 可用但 480 帧仍约 203fps，`immediate` 不被 surface 支持并回落 FIFO。下一步应做 timeline semaphore + bounded decode/display/present 队列、固定 display/descriptor ring 和 compositor/present 侧诊断，而不是再恢复 hidden decode queue wait。
- [ ] 按 FFmpeg/GStreamer 的硬件视频调度模型继续收敛 AV1/H.264/H.265 同步层：固定 frame/surface pool、bounded queue backpressure、明确 buffer/frame ownership、延迟 retire、audio/clock pacing 和单 owner display/WSI queue。2026-06-23 AV1 frame-context ring 已替代散装 `*_by_frame` slot 试验：每个 context 持有 acquire/decode/render semaphore、present fence、pending present result 和 sampled DPB/output resource；WSI `pump_events`/`vkAcquireNextImageKHR` 与 present worker `vkQueuePresentKHR` 用同一 mutex 串行，修复真实 Wayland 下的 `wl_display_dispatch_queue` assertion。真实 evidence：默认 2 context + decode/bitstream ring 16 的 `/tmp/gilder-av1-main8-frame-context-default-ring16-readback` 为 `presented=480`、`readback_y/uv_distinct=9/9`；10s `/tmp/gilder-av1-main8-frame-context-default-ring16-performance` 为 `average_present_fps=222.273`、CPU `16.74%`、`RSS/PSS/USS/Private_Dirty max=113808/101169/96600/35468 KiB`；手动 2 context + ring16 `/tmp/gilder-av1-frame-context-ring16.json` 为 `average_present_fps=227.287`，3 context、ring32 和 decode16+bitstream8 都回退。当前默认固定为 AV1 frame contexts 2、decode command ring 16、bitstream ring 16；后续不要再盲目加深队列，改做 timeline semaphore、surface pool retire、display queue pacing 和 audio clock。
- [ ] 评估 `VK_KHR_sampler_ycbcr_conversion` 作为 native Vulkan video sampling 后端：当前路径是 plane image view (`R8/RG8` 或 `R16/R16G16`) + 两个普通 sampler + shader 手写 YUV->RGB；理论上 `VkSamplerYcbcrConversion` 可以把 YCbCr 采样/色彩转换交给驱动 sampler 路径，减少 shader/descriptor 维护面，并可能改善 filtered chroma、P010 和不同色彩矩阵处理。该项必须先做 capability probe，验证 Vulkan Video decoded image、DMABuf modifier import、NV12/P010、NVIDIA/AMD 驱动组合是否支持，再用真实 Wayland `HDMI-A-1` 4K/240 smoke 对比当前 plane-view shader 路径的 FPS、CPU、RSS/PSS/USS/Private_Dirty 和画面稳定性；若驱动路径不稳，则保留当前 shader 路径为默认。
- [ ] 为 H.265/AV1 main-10 direct path 补 10-bit 2-plane 420 连续 decode/present path；当前 session/resource/bitstream gate 已确认 `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16` 可用于 Vulkan Video resource image，GStreamer appsink visible importer 已支持 P010 sampling 和 `P010_10LE` 可见 smoke，direct Vulkan Video Main10 已参数化 P010 plane view/readback/sampling。2026-06-22 真实 Wayland 证据：H.265 Main10 `/tmp/gilder-vulkan-h265-main10-p010-sampling.CGax7L` 与 AV1 Main10 4K `/tmp/gilder-vulkan-av1-p010-sampling-script` 均为 `first-frame-decode-output-sampled-and-readback-completed`，readback format 为 `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`，RGBA sampling 非零且 unique=256。本轮 H.264 layout 改动后重新回归：H.265 Main10 visible 4K/240 `/tmp/gilder-vulkan-h265-main10-after-h264-general-layout-4k240` 为 `decoded/presented=480/480`、`average_present_fps=239.76366459616204`；AV1 Main10 4K direct first-frame `/tmp/gilder-vulkan-av1-main10-after-h264-general-layout-4k` 为 `first-frame-decode-output-sampled-and-readback-completed`、`first_frame_decode_codec=av1-main-10`、P010 readback `24883200` bytes、RGBA sampling unique `256`。2026-06-23 AV1 Main10 已推进到任意入口连续可见 correctness：`/tmp/gilder-av1-main10-arbitrary-4k240-script-gate` 为 `presented=480`、`decoded=260`、`hidden_decoded=238`、`displayed_handoff=220`、`readback_y_distinct=5`；长一点的 `/tmp/gilder-av1-main10-arbitrary-4k240-performance` 为 `presented=2400`、`average_present_fps=211.028`、`RSS/PSS/USS/Private_Dirty max=108404/74981/61104/30172 KiB`。当前判断：H.265 Main10 连续 correctness 和 4K/240 性能都基本稳定，剩余是真实码流/长时覆盖；AV1 Main10 连续 correctness 已完成，但 4K/240 稳定性能、真实码流覆盖、audio/clock 和 DPB/output 内存压缩仍未完成。H.264 High10 不作为主线目标：ash/Vulkan STD 绑定当前没有 High10 profile 常量，且主流硬件/驱动通常不提供 H.264 10-bit 硬解；后续最多保留 capability probe/日志，不投入完整 direct decode/present 实现。
  2026-06-22 AV1 show-existing 修正后复测 H.265 Main10 visible 4K/240：`/tmp/gilder-vulkan-h265-main10-after-av1-show-existing-fix-4k240` 为 `decoded/presented=480/480`、`average_present_fps=240.157162809936`、P010、`h265_packet_queue_retained_payload_bytes=0`，说明 AV1 parser/show-existing 改动没有打退 Main10 4K/240 基线。该段为历史基线；2026-06-23 后 AV1 Main10 的主缺口已转为性能和覆盖，而不是 reference planner/show-existing handoff 未接通。
- [ ] 稳定 Intel/AMD VA/DMABuf path：显式选择 render node，解决 `decodebin -> appsink` 的 VAMemory/DMABuf negotiation，并在同 GPU Vulkan device 上验证 `vaExportSurfaceHandle` importer。

## M5: 合成器适配

- [x] 定义合成器输出/桌面状态快照模型。
- [x] 定义 fullscreen、unfocused、battery 等性能策略决策层。
- [x] 提供验证用输出状态覆盖，便于稳定采集 unfocused/fullscreen/hidden 性能策略证据。
- [x] 提供验证用输出列表覆盖，便于无 compositor 环境采集桌面状态策略证据。
- [x] 通用 GDK monitor 后端。
- [x] Hyprland IPC 后端。
- [x] niri IPC 后端。
- [x] 输出名称稳定映射。
- [x] 工作区/fullscreen 状态感知。
- [x] 从 Linux power_supply 读取电源状态并驱动 battery 性能策略。
- [x] 提供验证用电源状态覆盖，便于稳定采集 battery 性能策略证据。
- [x] 从 logind 读取 session active 状态并驱动 inactive 暂停策略。
- [x] 从 logind 读取 session locked 状态并驱动 locked 暂停策略。
- [x] 提供验证用 session 状态覆盖，便于稳定采集 inactive/locked 性能策略证据。
- [x] 将 manifest runtime fullscreen/unfocused 暂停策略接入桌面状态决策。
- [x] daemon 周期刷新桌面状态并只在变化时投递性能策略更新。
- [x] 允许配置桌面状态刷新周期。
- [x] 组合多种桌面状态时选择最省资源的性能策略。
- [x] 支持按输出覆盖性能策略和 FPS 上限。
- [x] 配置中允许禁用特定适配器。

## M6: Wallpaper Engine 转换器

- [x] 解析 `project.json`。
- [x] 识别 image/video/web/scene/application 类型。
- [x] 静态图片转换到 `static-image`。
- [x] 视频转换到 `video`。
- [x] 复制 preview 为 poster 和 thumbnail。
- [x] 缺失 preview 时从图片生成 poster 和 thumbnail。
- [x] 缺失 preview 时为视频生成 fallback poster 和 thumbnail。
- [x] 缺失 preview 时从视频首帧生成 poster 和 thumbnail。
- [x] Web 项目复制与 bridge 注入。
- [x] 用户属性映射。
- [x] 检测 Wallpaper Engine 音频意图并映射到视频 `runtime.allow_audio`。
- [x] 生成 `metadata/conversion-report.json`。
- [x] 拒绝 executable/application 类型并输出清晰错误。
- [x] Scene 子集转换到 `scene-lite`。
- [x] Playlist/collection 项目转换到一等 `playlist`，支持 image/video/web/scene 子项和 item weight。

## M7: 打包与发布

- [x] 实现 `.gwp` 打包。
- [x] 实现 `.gwp` 解包或只读读取。
- [x] 添加 man page。
- [x] 添加 systemd user service 示例。
- [x] 添加 shell completions。
- [x] 准备发行包脚本。

## M8: 性能、内存与 zero-copy 优化

> 归档说明：本节早期 GTK/native-wgpu/native `playbin/waylandsink` 条目只作为历史
> baseline 记录。当前可执行路线已经收敛为 native Wayland host + native Vulkan
> backend + GStreamer appsink/DMA 前端；新的可见视频工作在 M10 继续推进。

- [x] T0: 建立并达到 4K/240fps 硬解视频壁纸的实用顶级 CPU 基线：一输出 active
  NVIDIA/H.264 历史 direct GTK sink 路径实际选择 `nvh264dec`，20s
  样本平均约 75% 进程 CPU；按 20 逻辑 CPU 折算约 3.8% 整机 CPU，已经低于
  <= 120%/<= 6% 目标并接近 <= 80%/<= 4% stretch goal。所有后续回归门槛仍需附带逻辑
  CPU 数、采样时长、sink path 和同一场景的 QoS/drop 证据。
- [x] T0: 建立并达到 4K/240fps active GTK video surface 的历史内存/显存基线：
  默认 direct sink 20s 峰值约 `ps` RSS 455MiB、PSS 390MiB、private/USS 356MiB、
  `Private_Dirty` 约 109MiB、NVIDIA 进程显存约 496MiB；用户侧监控器观察到的应用内存约
  100MiB 时应优先对齐 `Private_Dirty` 口径，而不是 PSS/USS。GL wrapper 对照样本为
  PSS/USS/GPU memory 约 661/627MiB/689MiB，证明 high memory 主因是
  `glsinkbin` 路径额外 driver buffer/texture/pool 保留。
- [x] T0: 把 4K/240fps active direct GTK sink 突破固化为历史 guardrail：
  `performance-snapshot.sh` 和 Wayland video smoke 均可断言 max `Private_Dirty` 与
  NVIDIA 进程显存，baseline matrix 可把 private-clean/dirty 和
  `max_nvidia_process_gpu_memory_mib` 写入 `baseline.csv` 并由预算 CSV 校验。
- [x] T0: 强化多输出、paused 和 fullscreen-resume 验证链路：Wayland video smoke 的
  `--all-outputs`、`--sample-paused`、`--measure-fullscreen-resume` 可复用同一套
  process memory、`Private_Dirty`、renderer video lifecycle、source footprint 与 NVIDIA
  显存门槛；验证报告会链接 active/paused 的 hardware report 和 smaps mapping summary。
- [x] T0: 写入下一阶段视频底层优化路线文档 `docs/m8-video-optimization-plan.md`，把
  zero-copy evidence、YUV/NV12 保持、queue 梯度和 fullscreen/game auto-suspend 拆成
  可逐步验证的实验，并定义成功/停止条件。
- [x] T0: 为 YUV/NV12 保持实验补齐运行时证据字段：`caps_reports` 记录 negotiated
  raw video `format`/width/height，并在 `current_caps()` 不可见时回退读取 pad sticky CAPS
  event；`--video-runtime-csv`、performance summary、baseline CSV 和 hardware report 输出
  `formats`、`sink_formats`、`format_paths`、`frame_sizes`、`caps_sources`，用于判断 sink
  前是否已过早转 RGBA/RGBx。
- [x] T0: 在真实 GTK/playbin Wayland video runtime 中安装 caps-event observer，并把
  observer 证据合并进 status/runtime CSV；2026-06-20 niri 小视频 smoke 已观察到
  `caps-event|current|observer-initial|sticky`、sink-side `NV12`、`memory:CUDAMemory`，
  zero-copy evidence 达到 `sink-gpu-memory-caps`。该结果证明证据链已打通，但 4K/240fps
  仍需同场景复测。
- [x] T0: Wayland video smoke 和 baseline matrix 支持 `--video-size`、`--video-rate`、
  `--video-duration`，便于用同一套 guardrail 直接生成并采样 4K/240fps 压力源。
- [x] T0: 用 2026-06-20 niri 4K/240fps generated loop 复测 direct
  历史 direct GTK sink：runtime CSV 已达到 `zero_copy_evidence=sink-gpu-memory-caps`，
  `formats=NV12`、`sink_formats=NV12`、`memory_path=sink-gpu-memory`；6s sample 峰值
  `Private_Dirty` 115156 KiB、PSS/USS 418115/403768 KiB、NVIDIA 进程显存 472MiB，仍在
  M8 guardrail 内。该项只证明 GStreamer/GTK runtime sink-side GPU memory caps，不证明
  compositor presentation 层 full zero-copy。
- [x] T0: 把 4K/240fps 的 runtime zero-copy 证据从“硬解已满足”推进到
  `sink-gpu-memory-caps`：runtime CSV 记录 `memory_path=sink-gpu-memory`、
  sink-side `NV12`、caps sources、allocation pool、sink tuning 和 GDK/GTK timing 线索。
  该项仍不等同 compositor presentation full zero-copy；后续 GTK 4.14+/可用 dmabuf 构建上目标为
  `sink-dmabuf-caps`，并补 compositor presentation/frame callback 证据。
- [x] T0: 结束 direct GTK sink、forced GTK 和 forced GL wrapper 的同场景对照路线；
  GTK/native-wgpu/native waylandsink 路线已退休，后续 zero-copy 对照转入 native
  Vulkan importer/present 证据。
- [ ] T0: 验证 native Vulkan 4K/240fps video path 是否能保持 YUV/NV12 到
  shader/present 阶段，避免过早维护 RGBA/RGBx 大纹理；若 GStreamer appsink/importer
  只能提供 CPU raw frame，记录为明确 copy-path blocker。
- [x] T0: 添加 queue 调优诊断开关并做 8/4/2 buffers、50/25/12ms 梯度实验，对比
  queue current level、QoS/drop、CPU、PSS/USS、`Private_Dirty` 和 NVIDIA 显存；4/25ms
  作为新默认，2/12ms 因 CPU 与 QoS/drop 回退不采用。短样本中 NVIDIA 显存固定 472MiB，
  说明显存高水位后续应从 sink/compositor buffer pool 或 auto-suspend 方向继续挖。
- [x] T0: 将 2026-06-20 video 重大突破同步到 `docs/m8-video-optimization-plan.md`：
  runtime caps-event、queue/multiqueue observer、4K/240 `sink-gpu-memory-caps` + `NV12`、
  默认 4/25ms queue、最新 smoke 目录和 guardrail 结果均已记录。
- [x] T0: 将 `memory-mapping-summary.txt` 增强为同时输出
  `top_mappings_by_private_dirty` 和 `category_summary_by_private_dirty`，把
  `Private_Dirty` 优化从总量观察推进到 anonymous/heap/driver/shared-memory 分类定位。
- [x] T0: 将 smaps 分类 `Private_Dirty` 从人工阅读推进到可执行数据链路：
  `performance-snapshot.sh` 输出 `memory-mapping-categories.csv` 和
  `memory_category_<category>_private_dirty_kib` summary keys，Wayland smoke validation report
  和 baseline matrix 可直接把 anonymous、heap、nvidia-device、nvidia-library 等类别写入预算。
- [x] T0: baseline matrix 输出 `memory-category-deltas.csv`，用 `active,active` baseline
  自动计算 paused/fullscreen/hidden/resumed 等 phase 的分类 `Private_Dirty` delta/release，
  让 fullscreen/game auto-suspend 是否释放 driver/heap/anonymous dirty pages 变成可直接审计的表。
- [x] T0: baseline matrix 预算 CSV 支持 `min_release_from_active_kib`，可把
  `memory-category-deltas.csv` 中的 anonymous、heap、driver device/library 分类 release
  变成 paused/fullscreen/hidden/resumed 生命周期回归 gate。
- [ ] T0: 专项压低 video active/paused/fullscreen 的 `Private_Dirty`：用
  `category_summary_by_private_dirty` 对比 active -> paused/fullscreen -> resumed，
  优先确认 `anonymous`、`heap`、driver device/library dirty pages 和 shared-memory 是否按生命周期释放。
- [ ] T0: 用 20s/30s 长样本复验 4/25ms queue 默认值，确认 active、loop、resume 和多输出场景不回退。
- [ ] T0: 强化 fullscreen/game auto-suspend 显存释放验证，要求 removal 场景不仅
  pipeline/source footprint 为 0，还要观察 NVIDIA 显存、smaps `nvidia-device`/anonymous/heap
  分类和 resume latency。
- [ ] T0: 保持 8K 静态图路径为接近顶级基线：当前交互观察为 CPU 基本 0、应用内存约
  93MiB；下一阶段把该场景纳入 native Vulkan/performance snapshot 验证，要求 CPU 接近 0、
  PSS/private 与用户可见内存口径对齐，并确认 native Vulkan texture/cache 生命周期不会在
  切换、隐藏或暂停后保留超大纹理。
- [ ] 为真实 Wayland active、paused、fullscreen、hidden、battery、unfocused 场景建立 CPU/GPU/RSS/PSS/USS/private/shared 基线表。
- [ ] 为常见场景定义可执行的内存预算和回归阈值，优先使用 PSS、USS、private 占用和分类 `Private_Dirty` release 作为判断依据。
- [x] 提供真实 Wayland baseline matrix 采集脚本，批量运行 active/user-paused/battery/unfocused/fullscreen/hidden/session 场景并汇总 CPU/GPU/RSS/PSS/USS/private/shared、renderer resource、decoder/caps 和 timing 证据。
- [x] Wayland baseline matrix 支持 `scenario,phase,metric,max[,min_release_from_active_kib]` 预算 CSV，将 PSS/USS/private/retained delta 和分类 `Private_Dirty` release 等字段变成可执行回归阈值。
- [x] 提供 `examples/wayland-memory-budget.example.csv`，作为一输出 active 视频和生命周期场景的可执行内存/资源预算起点。
- [x] 在验证文档中记录当前 release active 视频采样基线，区分 idle、headless video 和 GTK/Wayland video surface 的 RSS/PSS/USS/private 现状。
- [x] 为 active -> paused/hidden/fullscreen -> active 场景输出内存 delta，区分瞬时峰值、恢复后 retained USS/private 和共享库 RSS。
- [x] 对 paused、hidden、fullscreen 移除渲染计划后的 pipeline/window/resource 释放行为建立验证门槛。
- [x] 在 renderer telemetry 中报告 output window、static/slideshow/video surface 和 video pipeline 计数，并让性能采样/Wayland smoke 可断言 output window lifecycle。
- [x] 让 performance/Wayland smoke 可断言 static/slideshow/video surface lifecycle，补齐 output window 之外的 renderer resource gate。
- [ ] 评估视频 pipeline 共享：同源多输出时避免重复解码，优先复用解码或 texture 产物。
- [x] 在 render sync telemetry、CSV、性能采样和 baseline matrix 中报告计划层视频 source 引用数、去重数、重复引用数、最大同源 fanout 和 source 字节 footprint，作为同源多输出 pipeline 共享候选评估依据。
- [x] performance snapshot、desktop policy smoke 和 Wayland video smoke 支持断言计划层视频 source fanout、去重、重复引用和 source 字节 footprint，便于把同源多输出共享候选纳入回归门槛。
- [ ] 限制 poster、thumbnail、manifest/package cache 的内存增长，并为缓存淘汰添加 telemetry。
- [x] 为单次 render sync 的 manifest/package 临时缓存添加可配置条目上限、FIFO 淘汰、status/watch telemetry、CSV 和性能采样汇总。
- [x] 为单次 render sync 的 manifest/package 临时缓存添加去重源资源 byte 上限，超限按 FIFO 淘汰，并在 status/watch 中报告当前上限与 retained footprint。
- [x] 将 render sync package cache 命中路径改为共享已加载 package，避免多输出同一壁纸时反复深拷贝 manifest/package 数据造成额外瞬时内存分配。
- [x] 为 `.gwp` 解包 render cache 添加可配置条目上限、当前使用条目保护、最旧优先淘汰和 telemetry/CSV/性能采样汇总。
- [x] 在 render sync telemetry、CSV 和性能采样汇总中报告计划层静态图、视频 poster、slideshow 图片资源 footprint，并支持 planned image resource 上限断言。
- [x] 在 render sync telemetry、CSV、性能采样和 smoke 报告中追加计划层图片源文件字节 footprint，并支持按引用字节/去重字节设置预算门槛。
- [x] 在 status/watch、telemetry CSV、性能采样、desktop smoke 和 Wayland baseline 中报告 package cache retained manifest 资源引用数/去重数与源文件字节 footprint，并支持预算门槛。
- [x] 在 status/watch、telemetry CSV、性能采样、desktop smoke 和 Wayland baseline 中拆分 package cache retained preview thumbnail/poster 资源引用数、去重数和源文件字节 footprint，并支持预算门槛。
- [x] 在 telemetry CSV、性能采样 summary、desktop smoke 和 Wayland baseline 中报告 package cache 去重源资源 byte 上限，便于 retained footprint 与预算同表对比。
- [x] 在 GTK renderer telemetry、status/CSV、性能采样、desktop smoke 和 Wayland baseline 中报告当前 static surface/slideshow surface 源资源引用数与字节 footprint，并支持预算门槛。
- [x] 在 GTK renderer telemetry、status/CSV、性能采样、desktop smoke 和 Wayland baseline 中报告当前 static/slideshow surface 去重源资源数与去重字节 footprint，并支持预算门槛。
- [x] 在 GTK/headless renderer telemetry、status/CSV、性能采样、desktop smoke 和 Wayland baseline 中报告当前 video pipeline 源文件引用数、去重数与字节 footprint，作为运行时视频资源释放和同源 pipeline 共享优化的证据。
- [x] 为 GTK renderer static/slideshow source footprint 计算添加 headless 单元测试，避免后续内存预算依赖的 renderer 残留资源指标回归。
- [x] 在 headless desktop policy smoke 中按场景断言 planned image resource footprint：renderable 静态场景最多 1，fullscreen/hidden/session/adaptive removal 场景必须为 0。
- [x] 在真实 Wayland 视频 smoke 中把 planned image resource footprint 纳入 lifecycle gate：active/resumed 视频最多每输出 1 个 poster 引用，paused/hidden/fullscreen/session removal 必须为 0。
- [x] GTK video surface 成功接管输出后释放 poster/static surface，并在 Wayland video lifecycle gate 中要求 active/resumed 最新 static/slideshow surface 为 0。
- [x] GTK video renderer 改为视频优先同步：active 视频不再预创建 poster 静态 surface，只有 pipeline 构建或运行失败时才懒加载 poster fallback，降低启动峰值 decoded texture/私有内存。
- [x] GTK renderer 在移除 static/video Picture surface 前显式清空 file/paintable 引用，并修复 slideshow 非动画 crossfade 切换保留旧 Picture 的问题，减少 decoded texture/frame 引用滞留。
- [x] performance snapshot 和 headless desktop policy smoke 支持断言 renderer video pipeline source footprint，便于验证 paused/hidden/fullscreen 后运行时视频 source 是否释放。
- [x] Wayland video surface lifecycle gate 自动断言 runtime video pipeline source footprint：active/resumed 按输出数设上限，paused/hidden/fullscreen/session removal 必须为 0。
- [x] GTK/video renderer 在无 FPS 上限时不创建 `videorate`/`capsfilter` frame limiter，减少默认 active 视频 pipeline 的常驻 GStreamer element。
- [x] GTK/headless video renderer 使用最小 `playbin` flags，muted 路径只开 video，audible 路径只开 video+audio，避免 active 视频常驻 deinterlace、soft color balance 或 soft volume 分支。
- [x] 将 GTK/headless 视频限帧改为 sink `throttle-time`，不再把 `videorate ! capsfilter` 插入 decoder 到 sink 的协商路径，并关闭 sink `last-sample` 保留。
- [x] 历史 GTK video surface 默认使用 direct sink，并关闭 async preroll、preroll frame 和 render delay；该路线已退休，结论只作为 native Vulkan importer 的对照基线。
- [x] GTK renderer tick 按负载动态调度：video runtime 单独存在时使用 250ms 常规 polling，frame stats 按 500ms 写回最近的 runtime snapshot；slideshow 过渡仍可使用更短 tick；纯静态无动态工作不安装 renderer runtime timeout，render sync 由 GLib idle wakeup 立即消费，减少 8K static idle wakeup。
- [x] GTK video polling 先检查 video runtime 是否存在，并让 frame stats 到期判断直接读取 runtime 计数，避免无视频空 poll 或完整 resource footprint/source size 重算。
- [x] GTK 共享 video runtime 的 renderer snapshot 复用同一份 decoder/caps/allocation、position 和 duration 查询，再展开为逐输出 telemetry，减少同源多屏视频的 GStreamer 查询成本。
- [x] GTK 组合 renderer snapshot 序列化 video pipeline telemetry 时复用已有 video source footprint，避免重复读取源文件 metadata。
- [x] GTK renderer resource footprint 按路径缓存 source size，重复静态图、幻灯片帧或同源视频不再反复 `metadata()`。
- [x] 历史 GTK video frame-clock 诊断默认改为轻量 after-paint tick/counter/time/interval 统计；当前 native Vulkan 路线改用 Vulkan present、Wayland frame callback 或 `wp_presentation` 证据。
- [x] GTK/headless GStreamer video pipeline 默认压低内部 queue/queue2/multiqueue 深度到 4 buffers/25ms，并在 runtime CSV/summary 中报告 queue max/current level，减少 4K/高刷视频中间队列保留窗口并为 PSS/USS/GPU memory 深挖提供证据。
- [x] 增加历史 GTK sink-chain 底层验证入口，用同一 4K/240 场景对比 direct GTK sink 与 GL wrapper 的 sink caps、queue、PSS/USS 和 GPU memory；该入口已随 GTK 路线退休。
- [x] 用真实 4K/240 NVIDIA/niri 样本确认 high memory 主要来自 `glsinkbin` 路径：direct sink 20s 峰值 PSS/USS/GPU memory 约 390/356 MiB/496 MiB，GL wrapper 约 661/627 MiB/689 MiB；默认 `auto` 因此切到 direct sink。
- [x] 静态图运行时缓存按 fit 估算降采样收益，覆盖 `contain` 极端比例大图和 `stretch` 大面积源图，减少直接让 GTK/GDK 解码原图的场景。
- [x] battery 性能策略支持用户可选 `pause-dynamic`，电池供电时释放 video/slideshow/web/scene-lite/shader 资源但保留静态壁纸，并在 headless desktop policy smoke 中覆盖。
- [x] fullscreen、unfocused、hidden 和 session 性能策略支持用户可选 `pause-dynamic`，只释放 video/slideshow/web/scene-lite/shader 动态壁纸并保留静态壁纸，headless smoke 覆盖静态透传和 slideshow 移除。
- [ ] 为 poster、thumbnail、manifest/package、视频 pipeline 和 GTK surface 缓存定义上限、淘汰策略和 status/watch 可见的 retained memory 线索。
- [x] 静态图片 Wallpaper Engine 转换时为足够大的光栅源图生成 16:9、21:9/ultrawide 和 9:16 portrait PNG variants，供 render plan 按输出尺寸选择以减少常见场景原始超大图解码。
- [x] 优化静态大图解码路径：转换器记录静态 raster entry 源图尺寸，render plan 在没有合适 manifest variant 且源图明显大于输出时生成受上限和淘汰管理的输出尺寸级静态缓存，避免无意义加载原始超大图。
- [x] 为运行时静态图缓存添加 byte 上限、最旧优先淘汰、status/CSV/performance/baseline telemetry，避免输出尺寸级缓存长期增长不可见。
- [x] performance snapshot、headless desktop smoke 和 Wayland video smoke 支持断言运行时静态图缓存 byte footprint，把静态缓存预算变成可执行回归门槛。
- [x] 接上更强硬解路径验证：按 codec/GPU/driver 记录实际 decoder、caps、sink caps、CPU/GPU/USS/PSS 对比。
- [ ] 验证 GTK video surface 是否能保持 GPU/DMABuf 路径，区分“硬解但发生 CPU copy”和真正 zero-copy。
- [x] performance snapshot 和 Wayland video smoke 支持 `--expect-zero-copy-evidence-at-least`，按证据强度做最低等级断言，避免更强 DMABuf/GLMemory 证据无法满足较低门槛。
- [x] performance snapshot 和 Wayland video smoke 支持 `--expect-zero-copy-profile`，将硬解、sink GPU/DMABuf caps、播放推进和 GTK frame-clock 证据组合成可执行 runtime/GTK profile。
- [ ] 继续采集 compositor presentation/frame callback 统计，补足 GTK/GDK timing 之外的 compositor 侧证据。
- [ ] 将硬解、DMABuf/GLMemory、sink-side caps 和 compositor presentation 组合成更严格的 zero-copy validation profile。
- [ ] 深入 native Vulkan texture/importer lifecycle、GStreamer appsink buffer pool 和 allocator 机制，确认哪些路径会保留 CPU-side frame、poster texture 或 last-sample 引用。
- [ ] 研究并验证 GStreamer appsink/DMA 到 native Vulkan 的低内存 zero-copy/import 路径：DMABuf/CUDAMemory/VA surface 保持、避免隐式 readback，同时保持 present pacing 性能不下降。
- [x] 历史 GTK 视频 runtime 曾按兼容 source/loop/audio/decoder/start-offset/FPS key 共享 GStreamer pipeline，并在 status/CSV telemetry 中报告 `video_shared_runtimes`；字段保留为 native Vulkan 共享 runtime 对照。
- [x] 为视频 runtime 增加 allocator/buffer-pool/caps 路径诊断，区分硬解后仍落到 CPU raw frame、decoder 侧 GPU memory、sink-side GPU memory 和 DMABuf/GLMemory runtime surface 线索。
- [x] 将视频 runtime 的 decoder/caps/allocation/memory path 诊断改为每 runtime 低频缓存刷新，避免 GTK video polling 或状态轮询持续遍历 GStreamer pipeline 和发 allocation query。
- [x] headless/GTK video sink 默认启用低内存 BaseSink 调优：关闭 last-sample、开启 QoS、按目标 FPS 收紧 max-lateness，并在 runtime snapshot 中报告 sink tuning。
- [x] runtime CSV、performance summary 和 video hardware report 报告 sink element、async、last-sample、render-delay、processing-deadline 和 preroll-frame 状态，便于验证 GTK 是否进入 GL sink 低内存路径。
- [x] GTK renderer 在 pause/remove sync 时实际释放 output window、video surface 和 GStreamer pipeline，并用 Wayland smoke 实测 active/paused RSS/PSS/USS/private 下降与 paused renderer lifecycle 归零。
- [x] 历史 GTK 静态图普通 fit 曾从 CSS background-image 改为显式 Picture surface；当前 native Vulkan 静态图路径继续沿用“切换/移除时释放 texture/resource”的生命周期目标。
- [x] GTK renderer telemetry 拆分 static Picture/CSS/color surface，并按 Picture paintable intrinsic size 报告估算 decoded footprint，作为 retained texture 风险线索。
- [x] desktop policy smoke、Wayland baseline matrix 和 Wayland video smoke 报告 static Picture/CSS/color surface 与估算 decoded footprint，并支持 headless 场景预算转发。
- [x] 基于 `memory_path`、`allocation_reports` 和 sink tuning 输出 `retention_report`/CSV/summary/baseline 线索，定向识别 CPU-side frame、buffer pool 和 last-sample/preroll frame 保留风险。
- [x] performance snapshot 和 Wayland video smoke 支持断言 video memory retention level、system-memory pool 数、pool byte 上限和 sink frame retention 状态，把 retained-frame/buffer-pool 风险纳入回归门槛。
- [ ] 继续审计 native Vulkan 静态图 surface：确认 decoded texture/cache 生命周期，并把估算 decoded footprint 与真实 PSS/USS/private delta 对齐。
- [x] 扩展 adaptive monitor，让用户可选按 CPU/GPU/内存压力、电池、温度、session/output 状态自动降 FPS、暂停动态壁纸或释放资源。
- [x] 为 adaptive 行为加入保守默认值、冷却时间、恢复条件和 status/watch 可解释报告，避免自动化策略不可预期。

## M9: 壁纸类型对齐 Wallpaper Engine

- [x] 梳理 Wallpaper Engine 类型矩阵：image、video、web、scene、application、audio visualizer、shader/particle、playlist，并标注 Gilder 支持等级。
- [x] 记录后续纯 Vulkan renderer 迁移准备路线：当前不继续压 active video copy/private dirty，
  优先扩展 web/scene-lite/shader/playlist，同时要求新增 runtime 保持后端无关。
- [x] 将路线调整为壁纸类型扩展与 hand-rolled Vulkan renderer 并行推进：类型 runtime 可以先落在
  helper/headless fallback，但必须同步定义 Vulkan-facing contract。
- [x] 让 `web` entry 在 runtime 未完成前使用 fallback render plan，缺少 fallback 时给出明确 unsupported 错误。
- [x] 为 `scene-lite` 定义 2D image/color/group layer、transform、opacity、keyframe timeline、动画曲线和属性 binding schema，并提供 headless snapshot evaluator 与资源校验。
- [x] 为 `scene-lite` 生成一等 render sync plan，GTK 先显示 fallback、首个 image layer 或首个 color layer，并把 fallback/layer 图片资源计入计划层与 package cache footprint。
- [x] 为 `scene-lite` 的 time=0 image/color snapshot 生成受控缓存 SVG surface，支持复用/淘汰 telemetry，避免简单 Scene 长期只显示静态 poster。
- [x] 将 `scene-lite` 属性 binding 接入 render sync 和 snapshot cache，使 IPC 数值/布尔属性可以影响 opacity、position、scale 和 rotation，并只让当前 plan 声明的绑定属性触发 daemon cache 失效。
- [x] 添加一等 `playlist` entry，支持 first-match 条件按输出、电源、focused/visible/fullscreen 和 session 状态选择 static/video/slideshow/web/scene-lite/shader 子 entry，并让 `pause-dynamic` 按实际选中类型决策。
- [x] 扩展 `playlist` 条件：支持本地时间窗口 `{ start = "HH:MM", end = "HH:MM" }`，含跨午夜区间，供工作时段/夜间/电池等组合策略选择壁纸。
- [x] 扩展 `scene-lite` 2D layer：支持 rectangle/ellipse shape、fill、stroke、corner radius 和本地尺寸，并合成到受控 SVG snapshot。
- [x] 扩展 `scene-lite` 2D layer：支持 text layer、font size/family/weight、text align 和安全 SVG text 转义，并合成到受控 SVG snapshot。
- [x] 扩展 `scene-lite` 2D layer：支持 SVG path data、fill/stroke 和安全 SVG attribute 转义，并合成到受控 SVG snapshot。
- [ ] 扩展 `scene-lite`：补齐常见 2D scene 图层、transform、opacity、动画曲线、时间轴和属性映射。
- [ ] 设计完整 `scene` runtime：保留可高效渲染的 scene graph，不把复杂场景长期降级为静态 fallback。
- [ ] 增强 `web` 壁纸 runtime：采用可替换 provider/helper 边界，补输入策略、音频策略、资源权限、暂停/恢复和低功耗模式；`gtk-rs`/WebKitGTK 不能成为 daemon 或 native Vulkan core 依赖。
- [ ] 将 Web runtime 设计为独立 helper：WPE/CEF/WebKitGTK/浏览器进程只作为 helper 内部实现，
  daemon/core 只接收属性、权限、生命周期和 DMABuf/EGLImage/Vulkan external image/Wayland surface handoff；CPU RGBA screenshot 只能作为降级 fallback，避免阻碍未来 Vulkan 后端。
- [x] 添加一等 `shader` manifest entry，记录 GLSL/WGSL 风格的时间、分辨率、鼠标和用户属性 uniform schema；runtime 完成前使用 fallback render plan，并按动态壁纸参与 `pause-dynamic` 释放策略。
- [x] Wallpaper Engine 转换器支持明确 Shader 项目和 playlist shader 子项，生成 `shader` fallback manifest、标准 time/resolution/mouse uniform 和用户属性 uniform。
- [ ] 实现原生 shader runtime：编译/执行 GLSL/WGSL、注入 uniform、接入 GPU memory telemetry 和 Wayland surface smoke。
- [ ] 为 native scene/shader/web runtime 建立后端无关 renderer 接口，helper/headless fallback 和
  native Vulkan 后端必须消费同一 render plan、property 输入和 lifecycle telemetry。
- [ ] 添加粒子/特效壁纸类型，优先覆盖 Wallpaper Engine 常见粒子发射器、纹理、速度场和 blend 模式。
- [ ] 添加音频响应壁纸能力，定义可选 PipeWire 音频采样输入和隐私/权限开关。
- [ ] 添加时钟、系统监控、媒体信息等 Linux 桌面常见信息型壁纸组件，但默认不采集敏感信息。
- [x] 扩展 playlist 选择策略：支持稳定 `weighted-random` 和 item `weight`，避免状态栏轮询导致随机壁纸抖动。
- [x] 扩展 playlist 日历条件：支持本地星期 `weekdays` 条件，并让 playlist 本地 clock 按依赖维度参与 render sync cache key。
- [x] 补 Wallpaper Engine playlist 转换：支持 image/video/web/scene 子项和 item weight；web 子项注入 bridge，scene 子项生成独立 `scene-lite` fallback graph。
- [ ] 继续扩展 playlist/轮播策略：按媒体/系统信息和更复杂日历条件选择壁纸，并补更完整 Wallpaper Engine playlist 策略映射。
- [x] 扩展 Wallpaper Engine 转换器，为 web/scene/shader/particle/audio 响应能力输出更细的 conversion report 和缺失能力提示。
- [ ] 为每类新壁纸定义 manifest schema、示例包、转换测试、headless 计划测试和真实 Wayland smoke 验证入口。

## M10: Hand-rolled Vulkan renderer spike

- [x] 新增 `native-vulkan-renderer` feature、`native_vulkan` capability/contract 模块和
  `gilder-native-vulkan` JSON 入口，明确当前尚未接管渲染、目标是 Wayland Vulkan surface +
  DMABuf/direct texture handoff。
- [x] 添加最小 Vulkan Wayland surface probe：复用 native Wayland layer-shell host，创建
  Vulkan instance、`VK_KHR_wayland_surface`，枚举可 present 的 GPU/queue，仍不进入默认路径。
- [x] 扩展 Vulkan Wayland surface probe 的 video queue telemetry：记录 selected present queue
  的 queue flags、Vulkan Video codec operations、同设备 H.265 decode queue，以及是否必须走
  same-device cross-queue sync；新增真实 Wayland smoke 固化该判断，避免后续误把 present queue
  当成 decode queue。
- [x] 添加 native Vulkan device/swapchain/clear present loop：`gilder-native-vulkan --run-clear`
  可在真实 Wayland 输出上按目标 FPS present，并输出 runtime JSON。
- [x] 添加 Vulkan-facing 壁纸类型矩阵和 render item 映射：当前 `StaticRenderSyncPlan`
  中的 static/video/slideshow/scene-lite 可转换为 Vulkan item，web/shader/playlist 记录 helper/
  fallback/selection contract。
- [ ] 定义 renderer backend contract：helper/headless fallback 和 native Vulkan 后端消费同一
  render plan、property 输入、dynamic lifecycle 和 resource telemetry。
- [ ] 建立最小 native Vulkan layer-shell host：Wayland surface、Vulkan instance/device/swapchain、
  resize、output selection、frame pacing 和 release。
- [x] 接入 static image 最小渲染路径：`--run-static` 复用现有 static render plan 的 source/fit/
  background，CPU decode/fit 后通过 Vulkan staging buffer copy 到 swapchain image；后续替换为
  sampled texture + shader pass，并补静态 idle 策略。
- [x] 开始接入 video wallpaper type：`--run-video` 消费 `VideoWallpaperPlan` 字段，复用 native
  Vulkan surface/swapchain 生命周期，当前渲染 poster/clear placeholder 并输出 GStreamer
  handoff telemetry，不让 GStreamer sink 接管显示。
- [x] 添加 `native-vulkan-gst-video` appsink 前端：`--run-video` 启动 GStreamer decodebin 到
  appsink，记录 decoder、caps、memory feature、sample format/size 和 handoff 计数；真实 Wayland
  已验证 `nvh264dec` + `memory:CUDAMemory` + `NV12` sample 到达 appsink。
- [x] 将 native Vulkan appsink sample 导入 Vulkan texture 的第一条路径：当前已实现
  `CUDAMemory -> CUDA copy -> Vulkan external image planes -> NV12 shader sampling`，
  由 native Vulkan render pass present，不让 GStreamer sink 接管显示。
- [x] 修复 4K/240 短视频在 loop 末尾卡顿：native Vulkan GStreamer frontend 改为
  `Paused -> segment seek -> Playing`，收到 `SegmentDone` 后回到 0，避免 EOS 后硬 seek。
  真实 Wayland 20s 验证：`3840x2160@240` 源、`nvh264dec`、`CUDAMemory/NV12`、
  `frames_rendered=4800`、`frames_imported=4790`、`eos_messages=0`、
  `last_sample_pts_delta_ms=4`。当前 HDMI-A-1 mode 是 `2560x1600@239.999`，不是 4K 输出。
- [ ] 补 AMD/Intel 同级 importer：`DMABuf/VAAPI -> Vulkan external memory/image` 到同一套
  NV12/YUV sampling；CUDA 不能成为 video 后端的核心抽象，只能是 NVIDIA importer。
- [ ] 将 H.265 direct Vulkan Video visible path 改为同 device 多 queue 架构：video queue 负责
  `vkCmdDecodeVideoKHR`，graphics/present queue 负责 decoded NV12 sampled render 到 swapchain，
  中间用 semaphore 和 image ownership/sharing 保证不经 CPU NV12 copy。
- [x] 接通首个可见 direct H.265 Vulkan Video path：`--run-h265-first-frame-video` 在真实
  Wayland background surface 上创建同一 logical device 的 video decode queue + present queue，
  将首个 H.265 IDR 解码到 NV12 video resource image，再由 native Vulkan Y/UV shader 直接采样到
  swapchain；新增 4K/240 source smoke 固化 decode/present 证据。
- [ ] 将 visible direct H.265 从首帧推进到 ready-prefix sequence：复用已有 DPB/POC 计划，按 AU
  连续 decode，使用 semaphore/timeline 替代 per-run `queue_wait_idle`，并按 PTS/target FPS present
  到 swapchain。
- [x] 接通可见 direct H.265 ready-prefix sequence：`--run-h265-ready-prefix-video` 复用
  DPB/POC reference plan，将多个 H.265 AU 逐帧 `vkCmdDecodeVideoKHR` 到 NV12 array layer，并由
  present queue 的 native Vulkan NV12 shader 采样到 Wayland swapchain；新增 4K/240 source smoke
  验证 8 帧 decode/present、PTS delta 和 layer 序列。
- [x] 将 visible direct H.265 ready-prefix 的跨队列同步从 CPU wait-idle 改为 GPU binary
  semaphore：video decode queue 每帧 signal `decode_finished`，graphics/present queue 同时等待
  `image_available` 和 `decode_finished` 后采样 decoded NV12 layer；真实 Wayland 4K/240 smoke
  验证 24 个 AU 连续 decode/present，策略为 `per-frame-binary-semaphore-decode-signal-present-wait`。
- [x] 为 visible direct H.265 ready-prefix 增加受控播放循环：CLI/smoke 新增
  `--playback-frames N`，用已抽取的 ready-prefix AU window 循环提交
  `vkCmdDecodeVideoKHR` + present，并在 loop boundary 强制 reset video coding；真实 Wayland
  4K/240 背景层 20s 样本验证 24 AU window -> 4800 decode/present、200 loops、
  199 次 loop-boundary reset、平均 `240.006fps`。
- [x] 将第一条 H.265 direct smoke 的默认测试源改成连续 4K/240 口径：未显式传
  `--decode-prefix` 时，`--playback-frames 4800` 会生成并解码同长度 ready prefix，避免默认测试
  结果被短窗口 `AU239 -> AU0` loop boundary 污染；短 ready-prefix loop 只保留为显式诊断模式。
- [x] 接通可见 direct H.264 ready-prefix sequence：`--run-h264-ready-prefix-video` 复用 H.264
  DPB/frame_num reference plan，将 H.264 High IPPP AU 逐帧 `vkCmdDecodeVideoKHR` 到 NV12 array
  layer，并由 present queue 采样到 Wayland swapchain。真实 Wayland `HDMI-A-1` 4K/240 ref=2
  evidence `/tmp/gilder-vulkan-h264-ready-prefix-video.Jy9iXF` 为 240 decode/present；
  `/tmp/gilder-vulkan-h264-ready-prefix-video.S305L5` 为 480 decode/present、2 loops、1 次
  loop-boundary reset。当前功能形态已追平 H.265 ready-prefix visible path；性能上 average present
  约 212fps，下一步继续压 present pacing/同步。
- [x] 将 H.265 Main10/P010 接到 visible direct ready-prefix：`scripts/native-vulkan-h265-ready-prefix-video-smoke.sh --bit-depth 10`
  生成/验证 Main10 源，runtime 输出 `requested_codec=h265-main-10` 和
  `picture_format=G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`。2026-06-22 真实 Wayland
  `HDMI-A-1` 证据 `/tmp/gilder-vulkan-h265-main10-visible-p010-4k240` 为
  `decoded/presented=480/480`、`average_present_fps=240.32978160780624`、P010
  `video_resource_memory_bytes=75104256`、`session_memory_bytes=46309376`。renderer
  descriptor-set 扩展后回归 `/tmp/gilder-vulkan-h265-main10-renderer-regression-4k240`
  仍为 `decoded/presented=480/480`、`average_present_fps=240.2474194054933`，确认
  H.264 display-ring 预绑定 descriptor 改动没有打退 Main10/P010。
- [x] 将 H.265 Main10 任意入口连续回归固定为 4K/240 direct visible gate：2026-06-22
  `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 证据
  `/tmp/gilder-vulkan-h265-main10-final-regression-4k240` 使用 `--bit-depth 10`
  `--arbitrary-entry-offset 0.35 --require-loop-skip-replay`，输出
  `decoded/presented=480/480`、`average_present_fps=240.71777490911953`、
  `h265_packet_queue_loop_skip_access_units=156`、
  `h265_packet_queue_bootstrap_discarded_access_units=156`、
  `h265_packet_queue_retained_payload_bytes=0`。
- [ ] 将 AV1 Main8/Main10 推进到 direct Vulkan Video 任意入口连续可见：新增
  `--run-av1-ready-prefix-video` 和
  `scripts/native-vulkan-av1-ready-prefix-video-smoke.sh`，以 GStreamer 只负责
  demux/parser/appsink TU 输入，native Vulkan Video 负责 AV1 picture info、inter decode、
  show-existing handoff、bitstream ring 和 Wayland swapchain present。2026-06-22 修正
  AV1 单 DPB slot 假通过问题：inter/show-existing 流不能把 transient output 解到正在作为
  reference 的同一 layer，runtime 现在至少使用 9 个 DPB/output slots，并且 smoke 要求多
  displayed layer。真实 Wayland `HDMI-A-1` 4K/240 证据
  `/tmp/gilder-vulkan-av1-main10-dpb9-regression-4k240` 为
  `requested_codec=av1-main-10`、P010
  `G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`、`decoded_frame_count=259`、
  `displayed_handoff_frame_count=221`、`presented_frame_count=480`、
  `average_present_fps=239.94913990040843`、`stream_dpb_slots=9`、
  `video_resource_memory_bytes=225312768`、`av1_packet_queue_retained_payload_bytes=0`。
  Main8 10s 观察证据 `/tmp/gilder-vulkan-av1-main8-observe-10s-dpb9-v3` 为
  `decoded=1305`、`handoff=1095`、`presented=2400`、`average_present_fps=239.6313194270436`、
  `stream_dpb_slots=9`、displayed layers `0..8`、`video_resource_memory_bytes=112656384`。
  后续更严格的 readback diversity gate 曾证明上述 present/FPS 证据仍可能是假阳性：
  `/tmp/gilder-av1-frameid-begin-test` 和 `/tmp/gilder-av1-tile-order-test` 均为
  `decoded/presented=12/12` 且 `average_present_fps=264-280`，但
  `readback_y_distinct=1`、`readback_uv_distinct=1`。2026-06-22 已定位根因：native
  AV1 frame-header parser 把 `allow_warped_motion` 放在 `reduced_tx_set` 前的错误位置推断，
  但没有在 `skip_mode_present` 后实际消费该 bit，导致后续 inter fields 错位。修正后
  对齐 GStreamer/FFmpeg 的解析顺序，真实 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1`
  Main8 10s 证据 `/tmp/gilder-av1-10s-warped-regression` 为 `decoded_frame_count=2400`、
  `presented_frame_count=2400`、`average_present_fps=240.20825729224006`、
  `readback_y_distinct=5`、`readback_uv_distinct=5`、`loop_count=79`。同日进一步修复
  libaom hidden alt-ref reference chain：AV1 `StdVideoDecodeAV1PictureInfo.OrderHints` 和
  saved reference `SavedOrderHints` 必须按 AV1 reference-name order 填充，而不是按 DPB
  map slot order。真实 `WAYLAND_DISPLAY=wayland-1`、`HDMI-A-1` 回归
  `/tmp/gilder-av1-main8-reference-name-order-hints-rerun` 为
  `decoded=40`、`hidden_decoded=26`、`presented=64`、
  `average_present_fps=240.55662367081612`、`readback_y_distinct=5`、
  `readback_uv_distinct=5`；Main10/P010
  `/tmp/gilder-av1-main10-reference-name-order-hints-rerun` 为 `decoded=40`、
  `hidden_decoded=26`、`presented=64`、`average_present_fps=244.68053337771838`、
  `readback_y_distinct=5`、`readback_uv_distinct=5`。随后分别做 10s 观察测试：
  Main8 `/tmp/gilder-av1-main8-observe-reference-name-order-10s` 为
  `presented=2400`、`average_present_fps=239.9047972118651`、
  `readback_y_distinct=10`、`readback_uv_distinct=10`；Main10/P010
  `/tmp/gilder-av1-main10-observe-reference-name-order-10s` 为 `presented=2400`、
  `average_present_fps=239.99269927809237`、`readback_y_distinct=10`、
  `readback_uv_distinct=10`。当前 AV1 Main8/Main10 已跨过旧的 repeated-frame blocker；
  对低分辨率糊的问题，原生输出分辨率源已经验证可明显改善：libaom low-delay
  2560x1600@240 Main8 `/tmp/gilder-av1-main8-native-res-libaom-lowdelay-observe-10s`
  为 `presented=2400`、`average_present_fps=235.13213456630402`、
  `readback_y_distinct=16`、`readback_uv_distinct=16`；Main10/P010
  `/tmp/gilder-av1-main10-native-res-libaom-lowdelay-observe-10s` 为
  `presented=2400`、`average_present_fps=230.54892214299622`、
  `readback_y_distinct=16`、`readback_uv_distinct=16`。2026-06-23 继续修复
  SVT-AV1 random-access 2560x1600@240：根因是 inter single-tile OBU 的 tile payload
  边界比 FFmpeg/Vulkan compact slice 多吃了 1 个 leading zero byte，导致提交 payload
  size 始终比 FFmpeg 大 1，解码后 readback 重复。`native_vulkan_av1_tile_group_offsets_from_payload`
  现在只对 inter、single-tile、1x1 tile layout 且首字节为 0 的 payload 跳过 1 byte，
  并补 `trims_av1_single_tile_inter_leading_zero_for_tile_payload_window`。真实 Wayland
  `HDMI-A-1` 证据：`/tmp/gilder-av1-svt-leading-zero-default-ring-readback`
  为 `presented=64`、`readback_y_distinct=9`、`readback_uv_distinct=9`；
  `/tmp/gilder-av1-svt-leading-zero-default-ring-20s` 为 `presented=4800`、
  `decoded_frame_count=2420`、`hidden_decoded_frame_count=2380`、
  `displayed_handoff_frame_count=2380`、`average_present_fps=238.2264888256383`。
  同轮把 AV1 streaming bitstream ring 默认从 2 slots 提到 8 slots，H.264/H.265
  仍保持 2 slots，`GILDER_VULKAN_BITSTREAM_RING_SLOTS` 可覆盖；当前 AV1 random-access
  正确性已过。2026-06-23 随后把 AV1 默认 handoff 推进到 displayed-frame
  direct-DPB：默认使用 `GENERAL` layout 直采 decoded DPB resource，frame context 持有 sampled
  resource 到 present retire，后续 decode 只在 output 覆盖同一 DPB slot 时等待；
  `GILDER_VULKAN_AV1_DISPLAYED_DIRECT_DPB=0` 保留旧 display-ring fallback。真实 Wayland
  4K/240 证据：Main8 `/tmp/gilder-av1-main8-displayed-direct-dpb-general-4k240-2400-b`
  为 `presented=2400`、`average_present_fps=240.0128447194083`、
  `av1_display_copy_count=0`、`av1_displayed_direct_dpb_count=2400`、
  `av1_display_ring_memory_bytes=0`；Main10
  `/tmp/gilder-av1-main10-displayed-direct-dpb-general-4k240-2400-a` 为
  `presented=2400`、`average_present_fps=239.8255409247888`、P010、
  `av1_display_copy_count=0`、`av1_displayed_direct_dpb_count=2400`、
  `av1_display_ring_memory_bytes=0`。短 readback gate 也已覆盖：
  `/tmp/gilder-av1-main8-displayed-direct-dpb-general-readback-4k240-480-a` 和
  `/tmp/gilder-av1-main10-displayed-direct-dpb-general-readback-4k240-480-a` 都为
  `readback_y_distinct=9`、`readback_uv_distinct=9`、`av1_display_copy_count=0`。
  本轮还验证了 AV1 present-frame clear preroll 对齐 H.264/H.265 的 2 帧策略，但因
  Main8/Main10 收益不一致，保持 opt-in：设置
  `GILDER_VULKAN_AV1_PRESENT_FRAME_CLEAR_PREROLL=1` 才启用，默认仍为
  `av1_present_frame_preroll_count=0`。
  `/tmp/gilder-av1-main8-direct-dpb-clear-preroll-4k240-2400-a` 和
  `/tmp/gilder-av1-main10-direct-dpb-clear-preroll-4k240-2400-a` 均为
  `av1_present_frame_preroll_count=2`、`av1_display_copy_count=0`、
  `av1_displayed_direct_dpb_count=2400`。process sampling 证据：
  Main8 `/tmp/gilder-av1-main8-direct-dpb-clear-preroll-performance-4k240-2400-a`
  为 `average_present_fps=239.9059074396761`、CPU `12.95%`、
  RSS/PSS/USS/Private_Dirty max `117112/80418/67100/36912 KiB`、NVIDIA GPU memory
  `181 MiB`；Main10 `/tmp/gilder-av1-main10-direct-dpb-clear-preroll-performance-4k240-2400-a`
  为 `average_present_fps=239.9695573179769`、CPU `12.46%`、
  RSS/PSS/USS/Private_Dirty max `116776/80249/66892/36688 KiB`、NVIDIA GPU memory
  `289 MiB`。负证据：`GILDER_VULKAN_AV1_READY_CONTEXT_SELECTION=1`
  只命中 143/2400 个 ready context 且 warmup 后 missed-vblank 到 7，不默认；
  `GILDER_VULKAN_AV1_PRESENT_FRAME_QUEUE_DEPTH=2` 把 decode wait 转移为
  `av1_present_result_wait_elapsed_us=9692746` 且 `average_present_fps=239.80400159488644`，
  不默认。当前 AV1 synthetic arbitrary-entry 4K/240 correctness/performance 已可用，剩余重点是
  更多真实码流矩阵、低内存 DPB/output compaction、audio/clock 接入，以及替换/扩展 synthetic
  libaom smoke 源。2026-06-24 已把 ready-prefix smoke 的 PTS delta gate 推进为通用
  timeline range 契约：Rust 侧拆出 `src/renderer/native_vulkan/timeline.rs`，输出
  `pts_delta_expected_min_ms`、`pts_delta_expected_max_ms` 和
  `pts_delta_in_expected_range`；shell 侧拆出
  `scripts/native-vulkan-ready-prefix-video-common.sh`，H.264/H.265/AV1 三条 smoke 复用
  source-cache 和目标帧率 PTS range 校验。缺失 PTS delta 或超出 target FPS 对应帧周期范围会失败。
  生成源默认缓存到仓库内 `artifacts/video-sources/<codec>/`，避免重启后丢失，目录由
  `.gitignore` 排除。缓存复用 gate：Main8
  `/tmp/gilder-av1-main8-pts-cache-reuse-640x368-240-480-a`、Main10
  `/tmp/gilder-av1-main10-pts-cache-reuse-640x368-240-480-a` 均为
  `pts_delta_min_ms=4`、`pts_delta_max_ms=4`、`av1_present_frame_preroll_count=0`、
  `av1_display_copy_count=0`、`av1_displayed_direct_dpb_count=480`。同轮已生成并缓存
  4K Main8/Main10 源：
  `artifacts/video-sources/av1/av1-main8-3840x2160-240fps-242frames-g240.webm` 和
  `artifacts/video-sources/av1/av1-main10-3840x2160-240fps-242frames-g240.webm`；
  4K 480-frame gate `/tmp/gilder-av1-main8-pts-cache-4k240-480-a` 与
  `/tmp/gilder-av1-main10-pts-cache-4k240-480-a` 均为
  `pts_delta_min_ms=4`、`pts_delta_max_ms=4`、`av1_present_frame_preroll_count=0`、
  `av1_display_copy_count=0`、`av1_displayed_direct_dpb_count=480`。这是时间线/默认路径
  验收；2400-frame 矩阵仍是更强的长跑性能证据。
- [x] 将 PTS range gate 和持久源缓存扩到 H.264/H.265：2026-06-24 真实 Wayland
  `HDMI-A-1`、4K/240、480-frame 串行 gate 均通过。H.264
  `/tmp/gilder-h264-pts-range-cache-4k240-480-a` 使用
  `artifacts/video-sources/h264/h264-high-b0-ref2-weightp0-weightb0-3840x2160-240fps-242frames-g241-d240.mp4`，
  `presented_frame_count=480`、`pts_delta_min_ms=4`、`pts_delta_max_ms=5`、
  `pts_delta_expected_min_ms=4`、`pts_delta_expected_max_ms=5`、
  `pts_delta_in_expected_range=true`。H.265 Main8
  `/tmp/gilder-h265-main8-pts-range-cache-4k240-480-a` 与 Main10
  `/tmp/gilder-h265-main10-pts-range-cache-4k240-480-a` 分别使用
  `artifacts/video-sources/h265/h265-main-8-b0-ref1-3840x2160-240fps-242frames-g240-d240.mp4`
  和
  `artifacts/video-sources/h265/h265-main-10-b0-ref1-3840x2160-240fps-242frames-g240-d240.mp4`，
  二者均为 `presented_frame_count=480`、`pts_delta_min_ms=4`、`pts_delta_max_ms=5`、
  `pts_delta_in_expected_range=true`。AV1 Main8/Main10 也用新 runtime 字段复测通过：
  `/tmp/gilder-av1-main8-pts-range-cache-4k240-480-a`、
  `/tmp/gilder-av1-main10-pts-range-cache-4k240-480-a` 均为 `pts_delta_in_expected_range=true`、
  `av1_display_copy_count=0`、`av1_displayed_direct_dpb_count=480`。
- [x] 真实 MP4 回归推进 H.264 Main profile、采样方向和 60fps pacing：2026-06-24
  `/home/yk/Documents/mpv/动态视频MP4-假面骑士/` 下样本均为 H.264 Main/yuv420p/2560x1440/60fps。
  真实源先暴露 direct H.264 仍按 High profile 建 session/profile query 的根因；现已拆出
  `src/renderer/native_vulkan/h264.rs`，按 SPS `profile_idc` 66/77/100 映射
  Baseline/Main/High Vulkan STD profile，并接受 8-bit 4:2:0 Main。样本 1 已通过
  `/tmp/gilder-h264-real-kamen-1-orientation-pacing-10s-1440p60-600`：
  `h264_stream_profile=main`、`h264_stream_profile_idc=77`、
  `h264_vulkan_std_profile_idc=77`、`decoded/presented=600/600`。用户指出画面上下颠倒后，
  已拆出 `src/renderer/native_vulkan/sampling.rs`，把视频 top-left frame origin 的 Y 翻转折进
  fit push constants，并用单测覆盖 cover crop 不变。随后修正 FIFO pacing：240Hz FIFO 不能代替
  60fps source/target pacing；`src/renderer/native_vulkan/pacing.rs` 让 FIFO+target-fps 走
  `target-fps-cpu-sleep-with-fifo-present`。10 秒真实源证据为 `runtime_elapsed_ms=9988`、
  `average_present_result_drop_first_60_fps=59.98248822941043`、`frame_sleep_count=599`、
  `missed_frame_pacing_count=0`、`pts_delta_min/max=16/17`。
- [x] pacing 修正后回归 4K/240 生成源：2026-06-24 H.264 长跑
  `/tmp/gilder-h264-pacing-plan-4k240-2400-a` 为 `decoded/presented=2400/2400`、
  `runtime_elapsed_ms=10012`、`pacing_strategy=target-fps-cpu-sleep-with-fifo-present`、
  `average_present_fps=239.69567237903803`、
  `average_present_result_drop_first_60_fps=239.9397391191329`、`pts_delta_min/max=4/5`。
  H.265 Main8 `/tmp/gilder-h265-main8-pacing-plan-4k240-480-a` 为
  `average_present_fps=240.10211819204144`，Main10
  `/tmp/gilder-h265-main10-pacing-plan-4k240-480-a` 为
  `average_present_fps=239.89495455740342`。AV1 Main8
  `/tmp/gilder-av1-main8-pacing-plan-4k240-480-a` 为
  `average_present_result_drop_first_60_fps=239.96389660133235`、`av1_display_copy_count=0`；
  Main10 `/tmp/gilder-av1-main10-pacing-plan-4k240-480-a` 为
  `average_present_result_drop_first_60_fps=239.9814887787176`、`av1_display_copy_count=0`。
- [x] 参考 FFmpeg/ffplay frame timer，把重复的固定 `next_frame += interval` sleep
  推进成 `src/renderer/native_vulkan/pacing.rs` 里的 `NativeVulkanVideoClockPacer`：
  target deadline 用整数纳秒累计，短 late 先追赶，超过 resync threshold 才重锚定 timer；
  当前仍以 target-fps 为 master clock，后续可切到 audio clock。2026-06-24 真实源 10 秒
  `/tmp/gilder-h264-real-kamen-1-ffmpeg-clock-pacer-10s-1440p60-600` 为
  `runtime_elapsed_ms=9988`、`average_present_result_drop_first_60_fps=59.98499818598243`、
  `missed_frame_pacing_count=0`。H.264 4K/240 长跑
  `/tmp/gilder-h264-ffmpeg-clock-pacer-4k240-2400-a` 为
  `average_present_fps=239.8605765305128`、
  `average_present_result_drop_first_60_fps=240.0756582168383`、`missed_frame_pacing_count=9`。
  H.265 Main8/Main10 为 `240.1629163155045`/`240.10727993267392fps`；AV1 Main8/Main10
  warmup 后为 `239.95496406116047`/`240.01269375010384fps` 且 `av1_display_copy_count=0`。
- [x] 参考 FFmpeg/ffplay audio master 思路补 video pacing master：`pacing.rs` 增加
  `NativeVulkanVideoPacingMaster::{TargetFps, AudioClock}`。`--audio-clock-probe`
  启用时默认使用 audio master，视频帧 sleep 按下一帧 video clock 与 audio master clock
  的 delta 计算；`GILDER_VIDEO_PACING_MASTER=target`/`GILDER_PACING_MASTER=target`
  可强制回到 target-fps。三条 smoke 脚本显式设置 target/audio env 并 gate pacing label。
  2026-06-24
  H.264 `/tmp/gilder-h264-audio-master-pacing-segment-clock-240`、H.265 Main8
  `/tmp/gilder-h265-main8-audio-master-pacing-segment-clock-620` 和 AV1 Main8
  `/tmp/gilder-av1-main8-audio-master-pacing-segment-clock-120` 分别为
  `decoded/presented=240/240`、`620/620`、AV1 `presented=120`，pacing label 均为
  `audio-clock-master-with-target-fps-fallback-and-fifo-present`，audio serial/loop gate 均通过；
  AV1 的 PTS 空洞已改为 segment-frame-index fallback，120 帧 runtime 从旧约 30s 收敛到
  `2004ms`、warmup 后 `59.93388987198527fps`。下一步接真实 audio sink/静音策略。
- [x] 继续攻克 AV1 display-copy 成本和 present 直采路径：2026-06-23 已从
  show-existing-only direct-DPB 推进到 displayed-frame direct-DPB 默认路径。默认不再创建 AV1
  display ring，4K Main8 display resource 从旧 display-ring+DPB 降为 DPB-only
  `112656384` bytes，Main10/P010 为 `225312768` bytes；2400 帧 Main8/Main10 gate 均为
  `av1_display_copy_count=0`、`av1_displayed_direct_dpb_count=2400`。
- [x] 继续攻克 H.264 4K/240 稳帧和 display-copy 成本：2026-06-23
  `GILDER_H264_DISPLAY_HANDOFF=direct-sampled-dpb-output` 已切到 direct-DPB
  persistent present worker，默认 2 帧 present preroll，并在硬件允许时默认请求 2 条
  present queue。真实 Wayland `/tmp/gilder-h264-direct-sampled-dpb-default-q2-preroll-4k240-2400-a`
  为 `decoded/presented=2400/2400`、`average_present_fps=239.12677751832481`、
  `average_present_result_drop_first_60_fps=239.39705921336318`、
  `h264_display_copy_count=0`、`h264_display_ring_memory_bytes=0`、
  `descriptor_update_sum=0`、`h264_packet_queue_retained_payload_bytes=0`。
  手动双 present queue 对照
  `/tmp/gilder-h264-direct-sampled-dpb-present-worker-preroll-presentq2-4k240-2400-a`
  达到 `average_present_fps=239.7187247637892` 且仍为零 display-copy。旧的
  display-ring/per-frame-fence 负证据保留为历史，不再代表 H.264 direct-DPB 默认路径。
- [ ] 将 visible direct H.265 ready-prefix 从受控窗口循环推进到完整播放循环：补持续 AU
  demux/parser、timeline semaphore 或更完整的 pacing/scheduling、loop/seek、音频/时钟接入和更长时间
  240Hz frame pacing telemetry。2026-06-23 已把 H.264 direct-DPB present
  worker/preroll/descriptor-prebind 模式迁到 H.265 Main8/Main10：Main8
  `/tmp/gilder-h265-main8-present-worker-preroll-q2-4k240-2400-a` 为
  `2400/2400`、`average_present_fps=240.1206840555046`、`descriptor_update_sum=0`、
  `h265_present_queue_count=2`、`h265_packet_queue_retained_payload_bytes=0`；Main10
  `/tmp/gilder-h265-main10-present-worker-preroll-q2-4k240-2400-a` 为
  `2400/2400`、`average_present_fps=240.07289114727573`、P010、
  `descriptor_update_sum=0`、`h265_present_queue_count=2`。剩余 H.265 工作转为更多真实码流、
  更长 process sampling 和 audio/clock，而不是当前 ready-prefix present critical path。
- [ ] 接入 shader-first 路径：fullscreen triangle、time/resolution/property uniform、Wayland smoke
  和 GPU/resource telemetry。
- [x] 接入 scene-lite runtime 输出到 Vulkan render item 边界：native Vulkan item 消费同一
  deterministic scene graph/timeline snapshot layer 结果，不新增 scene 专用 manifest 分支。
- [x] 接入 scene-lite image/color display 到 native Vulkan session：image display 复用 static upload
  path，color display 覆盖 Vulkan clear color；这保证当前 snapshot/fallback display 不再退回默认清屏。
- [x] 纯色 scene-lite 直达 color display：单个 color layer 或无 stroke/corner/transform 的全屏
  rectangle 不再生成 SVG snapshot，也不再走 static image decode/upload，native Vulkan 可直接
  clear swapchain，减少一次文件、解码和上传成本。
- [x] 单 image layer scene-lite 直达 image display：不透明、无 transform 的单图层场景不再生成
  中间 SVG snapshot，display source 与 layer source 资源计数去重，减少 cache 文件和重复资源统计。
- [x] 拆分 scene-lite display 规划边界：`renderer/scene_lite_display.rs` 负责 direct color clear
  eligibility、snapshot renderability 和 fallback background 选择，主 render sync 不再内联这些策略。
- [x] 接入 scene-lite 原生 draw-plan/runtime telemetry：render_plan 将 image/color/shape/text/path
  layers 分类为 native draw ops，scene_lite_runtime 输出 native_draw_ready、fallback 可用性和
  unsupported layer 原因。
- [x] 固化 video route 类型和 zero-copy 边界：`pipeline.rs` 将 direct 路线定义为
  `BitstreamNativeDecode`，将 `gst-dma`/provider 解码路线定义为
  `DecodedFrameFrontend`，并要求 zero-copy 声明必须标注 bitstream upload、decoded-frame
  handoff、import、render 或 compositor present 的具体作用域。
- [x] 固化 `ash` 绑定策略：`ash` 主分支的价值是更快获得 Vulkan Video/external-memory
  绑定并减少 raw FFI/生成代码漂移；它不是 zero-copy 证据本身，zero-copy 仍必须由同设备
  extension/capability/import telemetry 证明。
- [x] 拆分外部 interop 策略边界：`native_vulkan/interop.rs` 负责 video decoded-frame
  memory handoff、`ash` 绑定策略和 Web/helper texture handoff contract，主
  `native_vulkan.rs` 不再内联这些可替换接入层策略。
- [x] 推进通用 audio runtime loop 同步：decoded video frontend 的 segment-done 现在会
  触发 audio runtime `seek_for_video_loop(loop_start_position_ms)`，worker coalescing 保证
  loop seek 优先于普通 video-clock sample，向 FFmpeg/ffplay 的 clock serial 语义靠拢。
- [x] 统一 direct 路线 decoded-frame zero-copy evidence：H.264/H.265/AV1 ready-prefix
  runtime 输出 `decoded_frame_zero_copy_scope/status`，把 direct-DPB/display-copy 证据和
  bitstream-ring upload copy 作用域明确分开。
- [x] 强化 H.265 direct-DPB evidence：H.265 ready-prefix runtime 现在与 H.264/AV1
  同级报告 `h265_display_copy_count=0`、`h265_display_ring_memory_bytes=0` 和
  `h265_displayed_direct_dpb_count=presented_frame_count`，zero-copy status 可直接归类为
  confirmed direct-DPB no-display-copy。
- [x] 拆分 direct runtime summary：`native_vulkan/direct_runtime.rs` 统一 H.264/H.265/AV1
  ready-prefix runtime 的 elapsed、average present FPS 和 decoded-frame zero-copy
  classification，codec adapter 继续只负责 parser/reference/DPB 差异。
- [x] 收敛 ffplay-style timeline serial 判定：`native_vulkan/timeline.rs` 现在统一
  loop-boundary 和 stale frame serial helper，H.264/H.265/AV1 direct loop 不再散落
  ad hoc `source_loop_index` 比较。
- [x] 将 ffplay-style audio serial/stale 证据上提到通用 video runtime：`audio_runtime_*`
  现在报告 loop seek/restart/error、last loop seek position、segment start/elapsed、
  stale position/sample、sampled video frame、position query/hit 和 clock serial，使
  audio/video 同步状态不再只藏在 audio clock 专用 snapshot 内。
- [x] 补齐 H.265 present-result 性能证据：H.265 direct runtime 现在输出
  `average_present_result_fps`、drop-first、over-budget/missed-vblank 计数，并通过
  `GILDER_H265_ASYNC_PRESENT_DEPTH` 控制 bounded present-worker backpressure。
- [x] 统一 direct present-result summary：H.264 direct-DPB、H.264 display-ring、H.265
  和 AV1 现在都走 `native_vulkan_direct_present_result_summary`，跨 codec 性能指标不再
  各自复制实现。
- [x] 收敛 GStreamer bitstream frontend builder：H.264/H.265/AV1 仍保留各自入口，
  但 `demux_gst.rs` 现在通过 codec spec 统一 filesrc/demux/queue/parser/caps/appsink
  pipeline 创建，codec adapter 只声明 parser、caps、sink name 和 pad media type。
- [x] 将 video/audio clock fallback 计算迁入 `native_vulkan/pacing.rs`：loop segment
  frame index、audio probe video clock 和 next pacing clock 现在由 codec-neutral pacing
  模块提供，H.264/H.265/AV1 direct loop 不再依赖 `native_vulkan.rs` 内部私有时钟函数。
- [x] 补齐 scene-lite native draw payload：`NativeVulkanSceneLiteDrawOp` 现在携带
  source/color/stroke/尺寸/text/path/fit/transform 等绘制输入，runtime snapshot 可验证
  image、shape、text 的 native draw payload；缺 text color 或 path paint 不再误报 native-ready。
- [x] 收敛 direct present timing 写回：`direct_runtime.rs` 现在提供
  `NativeVulkanDirectPresentTiming` 和统一 apply helper，H.264 direct-DPB、H.264 display-ring
  与 H.265 present worker 不再各自复制 frame timing 写回和 acquire-not-ready 累计逻辑。
- [x] 将 AV1 present worker 也接入 direct runtime timing helper 的 optional 变体：
  预取 acquire/record timing 可以保持可选，frame-index 校验和 present timing 写回仍走共享路径。
- [x] 收敛 direct present wait/backpressure 统计：H.264 direct-DPB、H.264 display-ring、
  H.265 和 AV1 的 present-result wait count/elapsed/max 现在共用
  `NativeVulkanDirectPresentWaitStats`，保持现有 runtime 字段但让后续 pacer/backpressure
  调整基于同一套 codec-neutral 证据。
- [x] 收敛 direct present worker 阻塞接收：H.264 direct-DPB、H.264 display-ring 和
  H.265 的 present-result recv/断线错误映射/等待耗时记录现在走
  `native_vulkan_direct_recv_present_result`，codec 路径只保留 result 应用和私有状态。
- [x] 收敛 direct present worker 非阻塞 drain：H.264 direct-DPB、H.264 display-ring 和
  H.265 的 `try_recv`/pending 递减/断线-with-pending 判定现在走
  `native_vulkan_direct_try_recv_pending_present_result`，AV1 保留自己的 frame-context pending
  状态机。
- [x] 收敛 direct present backpressure 状态：H.264 direct-DPB、H.264 display-ring 和
  H.265 不再散落裸 `pending_present_results` 加减和 `>= depth` 判断，改用
  `NativeVulkanDirectPresentBackpressure` 表达 max depth、pending、饱和等待和尾部 drain。
- [x] 将 AV1 frame-context present pending 判定接入 direct runtime：async-depth wait、
  acquire-before-present wait、acquire NOT_READY helper drain、final drain 和 disconnected
  pending 判定现在共用 `native_vulkan_direct_*pending_flags*` helper，AV1 只保留 context
  置位/清位和选择策略。
- [x] 将 AV1 present result apply/清 pending context 接入 direct runtime：
  `NativeVulkanDirectPresentPendingContext` 和 indexed pending-context apply helper 统一
  context index 校验、result apply 后清 pending、apply 失败不清 pending 的语义。
- [x] 重新采集拆分后的 4K/240 direct 10 秒性能证据：2026-06-24 当前 `ad5b647`
  release build 在真实 Wayland `HDMI-A-1` 上复测，三类 codec 仍为 239+。H.264
  `/tmp/gilder-current-h264-postsplit-4k240-2400` 为 `decoded/presented=2400/2400`、
  `average_present_fps=239.71420993822684`、drop-first-60
  `239.9106740453986`、`h264_display_copy_count=0`。H.265 Main8
  `/tmp/gilder-current-h265-main8-postsplit-4k240-2400` 为
  `average_present_fps=239.975940348187`、drop-first-60 `239.95519742034313`、
  `h265_displayed_direct_dpb_count=2400`、`h265_display_copy_count=0`；Main10/P010
  `/tmp/gilder-current-h265-main10-postsplit-4k240-2400` 为
  `average_present_fps=240.00243067261715`、drop-first-60 `239.9808507798249`、
  `h265_displayed_direct_dpb_count=2400`、`h265_display_copy_count=0`。AV1 Main8
  `/tmp/gilder-current-av1-main8-postsplit-4k240-2400` 为
  `average_present_fps=239.8182153226578`、drop-first-60 `240.01192774112042`、
  `av1_displayed_direct_dpb_count=2400`、`av1_display_copy_count=0`；Main10/P010
  `/tmp/gilder-current-av1-main10-postsplit-4k240-2400` 为
  `average_present_fps=239.763593576261`、drop-first-60 `239.90602330278284`、
  `av1_displayed_direct_dpb_count=2400`、`av1_display_copy_count=0`。
- [x] 接入 scene-lite Vulkan draw-pass 规划层：`scene_lite_draw_pass.rs` 消费
  draw-plan ops，输出 pass ready/backend ready/status/blocker、image/shape/text/path
  资源 bucket、text atlas/path tessellation 需求，并识别单 color op 的 fast-clear backend path。
- [x] 为 scene-lite draw-pass 补 command-recording 前的 quad payload：color-quad 和无
  stroke/corner radius 的 filled rectangle 现在输出 layer id、kind、RGBA、尺寸和 transform，
  runtime snapshot 报告 `draw_pass_recordable_quads`，为后续 Vulkan quad recording 提供稳定输入。
- [ ] 接入 scene-lite 原生 Vulkan draw pass：消费 draw-plan 中的 image/color/shape/text/path ops，
  建立 GPU/resource telemetry 和 Wayland smoke。
- [ ] 设计 Web helper frame/texture handoff：WebKitGTK/浏览器 helper 只作为隔离实现，native Vulkan
  后端通过稳定 helper 协议接收 frame stream 或可导入 texture。
- [ ] 继续 video interop：删除 `gpu-video` 与 native-wgpu 依赖路线后，以 GStreamer 作为 video/audio
  前端验证 GL/EGLImage/DMABuf/CUDAMemory handoff、Vulkan Video、libavcodec + external
  memory 等方案；GStreamer 不接管显示 sink，native Vulkan 后端负责最终 present，只有同场景
  优于 retired native-wgpu CUDA copy evidence 才进入默认候选。
- [ ] 将 native Vulkan 后端接入 baseline matrix，覆盖 static/video/web/scene-lite/shader/playlist
  的 active、paused、hidden、fullscreen、session release 和恢复延迟。
