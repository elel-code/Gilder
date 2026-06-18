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
- [x] 在真实 niri Wayland 会话验证视频 surface 显示。
- [x] 在真实 niri Wayland 会话验证多输出同源视频 surface 显示。
- [ ] 在真实 Hyprland Wayland 会话验证视频 surface 显示。
- [x] 添加 CPU/RSS/PSS/USS/private/status 性能采样脚本。
- [x] 将 active-video 性能采样接入 Wayland 视频 surface smoke。
- [x] 将 paused-video 性能采样接入 Wayland 视频 surface smoke。
- [x] 在性能采样证据中输出 render decision CSV 和摘要。
- [x] 在性能采样证据中输出 PSS、USS/private 和 shared 内存摘要。
- [x] 在 decision CSV 中记录计划类型、资源、fit、视频限帧和静音状态。
- [x] 使用 CSV-aware 汇总器统计性能采样中的决策、计划类型、fit、静音和限帧范围。
- [x] 支持性能采样断言期望的 mode、reason、action 和计划类型。
- [x] 单次 render sync 内复用同一路径壁纸包的加载结果。
- [x] 实现 poster 显示。
- [x] 实现 max_fps 或 pipeline throttling。
- [x] 实现 slideshow 普通动态壁纸运行计划和 GTK 定时切换。
- [x] 避免重复 render sync 对视频 pipeline 反复设置未变化的 state、mute、fit、限帧和 start offset。
- [x] 避免重复 render sync 对 GTK 静态窗口反复重建 CSS provider。
- [x] 避免 GTK 初始同步和 IPC 状态变更把未变化的 render sync 重复投递给渲染器。
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
- [ ] 继续采样 GPU 和帧行为，并输出到 daemon telemetry。
- [x] 在 adaptive telemetry 中采样 DRM `gpu_busy_percent` 的 avg/max/source，并接入 telemetry CSV 和性能采样汇总。
- [x] 在视频 runtime 中报告 playback position/duration 和实际 frame limiter 状态，并接入 performance 采样汇总与断言。
- [x] 在视频 runtime 中累计 GStreamer QoS processed/dropped 统计，并接入 performance 采样汇总与断言。
- [x] 在 GTK 视频 surface runtime 中累计 frame clock tick/interval/FPS 统计，并接入 performance 采样汇总与断言。
- [ ] 继续采集 compositor presentation/frame callback 统计。
- [x] 将 adaptive monitor 结果作为只会降载的性能策略输入，支持阈值、冷却时间和恢复条件。
- [x] adaptive monitor 支持用户可选的 `pause-unfocused` 动作，在系统压力下暂停非焦点输出。
- [x] adaptive monitor 支持用户可选的 `pause-dynamic` 动作，在系统压力下暂停 video/slideshow 动态壁纸。
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
- [ ] 验证 GTK video surface 是否能保持 GPU/DMABuf 路径，区分“硬解但发生 CPU copy”和真正 zero-copy。
- [x] 本机 smoke/performance 采样记录 decoder、decoder 策略状态、caps memory features 和 sink memory features。
- [x] 本机 performance 采样支持断言 decoder 策略状态、decoder class、caps memory feature 和 sink memory feature。
- [x] 为硬解和 zero-copy 添加本机 smoke：记录 decoder、sink caps/memory features、CPU/GPU/USS/PSS 对比。

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

## M7: 打包与发布

- [x] 实现 `.gwp` 打包。
- [x] 实现 `.gwp` 解包或只读读取。
- [x] 添加 man page。
- [x] 添加 systemd user service 示例。
- [x] 添加 shell completions。
- [x] 准备发行包脚本。
