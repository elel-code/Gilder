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
- [x] adaptive monitor 支持用户可选的 `pause-dynamic` 动作，在系统压力下暂停 video/slideshow/web/scene-lite 动态壁纸。
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
- [x] Playlist/collection 项目转换到一等 `playlist`，支持 image/video 子项和 item weight。

## M7: 打包与发布

- [x] 实现 `.gwp` 打包。
- [x] 实现 `.gwp` 解包或只读读取。
- [x] 添加 man page。
- [x] 添加 systemd user service 示例。
- [x] 添加 shell completions。
- [x] 准备发行包脚本。

## M8: 性能、内存与 zero-copy 优化

- [ ] 为真实 Wayland active、paused、fullscreen、hidden、battery、unfocused 场景建立 CPU/GPU/RSS/PSS/USS/private/shared 基线表。
- [ ] 为常见场景定义可执行的内存预算和回归阈值，优先使用 PSS、USS 和 private 占用作为判断依据。
- [x] 提供真实 Wayland baseline matrix 采集脚本，批量运行 active/user-paused/battery/unfocused/fullscreen/hidden/session 场景并汇总 CPU/GPU/RSS/PSS/USS/private/shared、renderer resource、decoder/caps 和 timing 证据。
- [x] Wayland baseline matrix 支持 `scenario,phase,metric,max` 预算 CSV，将 PSS/USS/private/retained delta 等 baseline 字段变成可执行回归阈值。
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
- [x] performance snapshot 和 headless desktop policy smoke 支持断言 renderer video pipeline source footprint，便于验证 paused/hidden/fullscreen 后运行时视频 source 是否释放。
- [x] Wayland video surface lifecycle gate 自动断言 runtime video pipeline source footprint：active/resumed 按输出数设上限，paused/hidden/fullscreen/session removal 必须为 0。
- [x] GTK/video renderer 在无 FPS 上限时不创建 `videorate`/`capsfilter` frame limiter，减少默认 active 视频 pipeline 的常驻 GStreamer element。
- [x] GTK/headless video renderer 使用最小 `playbin` flags，muted 路径只开 video，audible 路径只开 video+audio，避免 active 视频常驻 deinterlace、soft color balance 或 soft volume 分支。
- [x] 将 GTK/headless 视频限帧改为 sink `throttle-time`，不再把 `videorate ! capsfilter` 插入 decoder 到 sink 的协商路径，并关闭 sink `last-sample` 保留。
- [x] GTK renderer tick 按负载动态调度：视频 runtime 保持 50ms，纯静态空闲或长间隔 slideshow 退到最长 250ms，并且只在 render sync 变化、slideshow 实际换帧或存在 video runtime 时刷新 renderer runtime snapshot。
- [x] 静态图运行时缓存按 fit 估算降采样收益，覆盖 `contain` 极端比例大图和 `stretch` 大面积源图，减少直接让 GTK/GDK 解码原图的场景。
- [x] battery 性能策略支持用户可选 `pause-dynamic`，电池供电时释放 video/slideshow/web/scene-lite 资源但保留静态壁纸，并在 headless desktop policy smoke 中覆盖。
- [x] fullscreen、unfocused、hidden 和 session 性能策略支持用户可选 `pause-dynamic`，只释放 video/slideshow/web/scene-lite 动态壁纸并保留静态壁纸，headless smoke 覆盖静态透传和 slideshow 移除。
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
- [ ] 深入 GTK/GDK/GSK texture lifecycle、`gtk4paintablesink`、GStreamer buffer pool 和 allocator 机制，确认哪些路径会保留 CPU-side frame、poster texture 或 last-sample 引用。
- [ ] 研究并验证 GTK/GStreamer 可行的低内存 zero-copy surface 路径：DMABuf/GLMemory 保持、共享 GL context、避免隐式 readback，同时保持 frame clock 和 presentation 性能不下降。
- [x] 重构 GTK 视频 runtime：按兼容 source/loop/audio/decoder/start-offset/FPS key 共享 GStreamer pipeline 和 `GdkPaintable`，每个输出只保留独立 `gtk::Picture`、fit 和 frame-clock 统计，并在 status/CSV telemetry 中报告 `video_shared_runtimes`。
- [x] 为视频 runtime 增加 allocator/buffer-pool/caps 路径诊断，区分硬解后仍落到 CPU raw frame、decoder 侧 GPU memory、sink-side GPU memory 和 DMABuf/GLMemory runtime surface 线索。
- [x] 将视频 runtime 的 decoder/caps/allocation/memory path 诊断改为每 runtime 低频缓存刷新，避免 GTK 50ms tick 或状态轮询持续遍历 GStreamer pipeline 和发 allocation query。
- [x] headless/GTK video sink 默认启用低内存 BaseSink 调优：关闭 last-sample、开启 QoS、按目标 FPS 收紧 max-lateness，并在 runtime snapshot 中报告 sink tuning。
- [x] GTK 静态图普通 fit 从 CSS background-image 改为显式 `gtk::Picture` surface，切到视频、移除输出或换帧时释放 Picture 引用；`tile` 保留 CSS fallback。
- [x] GTK renderer telemetry 拆分 static Picture/CSS/color surface，并按 Picture paintable intrinsic size 报告估算 decoded footprint，作为 retained texture 风险线索。
- [x] desktop policy smoke、Wayland baseline matrix 和 Wayland video smoke 报告 static Picture/CSS/color surface 与估算 decoded footprint，并支持 headless 场景预算转发。
- [ ] 基于 `memory_path` 和 `allocation_reports` 定向研究 `gtk4paintablesink`、GDK/GSK texture、GStreamer allocator/buffer-pool 的低内存路径，优先减少 CPU-side frame 和重复 texture 保留。
- [ ] 继续审计 GTK 静态图 surface：确认 `gtk::Picture`/GDK/GSK decoded texture 生命周期，并把估算 decoded footprint 与真实 PSS/USS/private delta 对齐。
- [x] 扩展 adaptive monitor，让用户可选按 CPU/GPU/内存压力、电池、温度、session/output 状态自动降 FPS、暂停动态壁纸或释放资源。
- [x] 为 adaptive 行为加入保守默认值、冷却时间、恢复条件和 status/watch 可解释报告，避免自动化策略不可预期。

## M9: 壁纸类型对齐 Wallpaper Engine

- [x] 梳理 Wallpaper Engine 类型矩阵：image、video、web、scene、application、audio visualizer、shader/particle、playlist，并标注 Gilder 支持等级。
- [x] 让 `web` entry 在 runtime 未完成前使用 fallback render plan，缺少 fallback 时给出明确 unsupported 错误。
- [x] 为 `scene-lite` 定义 2D image/color/group layer、transform、opacity、keyframe timeline、动画曲线和属性 binding schema，并提供 headless snapshot evaluator 与资源校验。
- [x] 为 `scene-lite` 生成一等 render sync plan，GTK 先显示 fallback、首个 image layer 或首个 color layer，并把 fallback/layer 图片资源计入计划层与 package cache footprint。
- [x] 为 `scene-lite` 的 time=0 image/color snapshot 生成受控缓存 SVG surface，支持复用/淘汰 telemetry，避免简单 Scene 长期只显示静态 poster。
- [x] 将 `scene-lite` 属性 binding 接入 render sync 和 snapshot cache，使 IPC 数值/布尔属性可以影响 opacity、position、scale 和 rotation，并只让当前 plan 声明的绑定属性触发 daemon cache 失效。
- [x] 添加一等 `playlist` entry，支持 first-match 条件按输出、电源、focused/visible/fullscreen 和 session 状态选择 static/video/slideshow/web/scene-lite 子 entry，并让 `pause-dynamic` 按实际选中类型决策。
- [x] 扩展 `playlist` 条件：支持本地时间窗口 `{ start = "HH:MM", end = "HH:MM" }`，含跨午夜区间，供工作时段/夜间/电池等组合策略选择壁纸。
- [x] 扩展 `scene-lite` 2D layer：支持 rectangle/ellipse shape、fill、stroke、corner radius 和本地尺寸，并合成到受控 SVG snapshot。
- [x] 扩展 `scene-lite` 2D layer：支持 text layer、font size/family/weight、text align 和安全 SVG text 转义，并合成到受控 SVG snapshot。
- [x] 扩展 `scene-lite` 2D layer：支持 SVG path data、fill/stroke 和安全 SVG attribute 转义，并合成到受控 SVG snapshot。
- [ ] 扩展 `scene-lite`：补齐常见 2D scene 图层、transform、opacity、动画曲线、时间轴和属性映射。
- [ ] 设计完整 `scene` runtime：保留可高效渲染的 scene graph，不把复杂场景长期降级为静态 fallback。
- [ ] 增强 `web` 壁纸 runtime：WebKitGTK sandbox、输入策略、音频策略、资源权限、暂停/恢复和低功耗模式。
- [ ] 添加 shader 壁纸类型，优先支持 GLSL/WGSL 风格的时间、分辨率、鼠标和用户属性 uniform。
- [ ] 添加粒子/特效壁纸类型，优先覆盖 Wallpaper Engine 常见粒子发射器、纹理、速度场和 blend 模式。
- [ ] 添加音频响应壁纸能力，定义可选 PipeWire 音频采样输入和隐私/权限开关。
- [ ] 添加时钟、系统监控、媒体信息等 Linux 桌面常见信息型壁纸组件，但默认不采集敏感信息。
- [x] 扩展 playlist 选择策略：支持稳定 `weighted-random` 和 item `weight`，避免状态栏轮询导致随机壁纸抖动。
- [x] 扩展 playlist 日历条件：支持本地星期 `weekdays` 条件，并让 playlist 本地 clock 按依赖维度参与 render sync cache key。
- [x] 补 Wallpaper Engine playlist 转换：支持 image/video 子项和 item weight，web/scene 子项暂记 unsupported。
- [ ] 继续扩展 playlist/轮播策略：按媒体/系统信息和更复杂日历条件选择壁纸，并补更完整 Wallpaper Engine playlist 策略映射。
- [x] 扩展 Wallpaper Engine 转换器，为 web/scene/shader/particle/audio 响应能力输出更细的 conversion report 和缺失能力提示。
- [ ] 为每类新壁纸定义 manifest schema、示例包、转换测试、headless 计划测试和真实 Wayland smoke 验证入口。
