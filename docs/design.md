# Gilder 设计文档

## 目标

Gilder 是一个面向 Wayland 独立合成器的壁纸引擎，首要目标是替代 Linux 上零散的静态壁纸脚本和能力不足的动态壁纸工具。

优先支持对象：

- niri、Hyprland 等 wlroots/独立合成器用户。
- 多显示器、不同缩放比例、不同壁纸配置的桌面环境。
- 静态图片、普通视频循环、轻量参数化动态壁纸。
- 从 Wallpaper Engine 项目迁移静态、视频、Web、部分场景壁纸资源。

非目标，至少在早期阶段不做：

- 运行 Windows Wallpaper Engine 的原生场景运行时。
- 完整兼容 SceneScript、专有 shader/effect 行为。
- 提供通用桌面 shell、面板、锁屏或窗口管理功能。
- 在 root 权限下运行或写入系统级配置。

## 组件

Gilder 使用单个 Cargo package，保留清晰模块边界：

- `gilderd`：会话级守护进程，负责 Wayland 窗口、渲染、包加载、状态持久化和 IPC。
- `gilderctl`：命令行客户端，面向脚本、快捷键和配置管理。
- `gilder-convert`：转换器，读取 Wallpaper Engine 项目并生成 `.gwpdir` 或 `.gwp`。
- `src/core.rs`：core 模块入口和 re-export。
- `src/core/`：Gilder 壁纸格式常量和核心类型。
- `src/ipc.rs`：IPC 模块入口和 re-export。
- `src/ipc/`：IPC 协议常量、命令解析和 socket 路径。

后续如果模块变大，再拆成 crate；当前不使用 Cargo workspace。

Rust 模块组织采用 2018+ 常见布局：`src/foo.rs` 作为模块入口，`src/foo/`
作为该模块的子模块目录。这样路径保持扁平，子模块增长时也不用把所有实现堆进
单个文件。

## 运行模型

`gilderd` 是用户会话内的长驻进程：

1. 启动后绑定 `$XDG_RUNTIME_DIR/gilder/gilder.sock`。
2. 读取 `$XDG_CONFIG_HOME/gilder/config.toml` 和上次状态。
3. 为每个需要壁纸的输出创建一个 layer-shell 窗口。
4. 加载指定 `.gwpdir` 或 `.gwp`，选择适合输出的 variant。
5. 在 GTK 主循环中管理窗口生命周期，在后台线程或 GStreamer pipeline 中处理重型 IO/解码。
6. 通过 IPC 接收切换、暂停、恢复、状态查询和配置写入。

## Wayland 集成

早期实现以 GTK 4 为主：

- 使用 GTK-rs 构建每个输出上的无装饰窗口。
- 使用 layer-shell 协议把窗口放入 background/bottom 层，锚定四边，覆盖整个输出。
- 输出枚举优先使用 GDK monitor 信息；Hyprland/niri 适配器用于增强输出名称、工作区/fullscreen 状态、热插拔语义。
- 每个输出独立持有壁纸状态，允许同一包使用不同 variant 或不同裁剪参数。

合成器适配分层：

- `generic-wayland`：只依赖 GDK/Wayland 可见信息，作为默认路径。
- `hyprland`：可选调用 Hyprland IPC 获取 monitor 名称、活动工作区和 fullscreen 状态。
- `niri`：可选调用 niri IPC 获取 output/workspace 信息。

合成器适配器不能成为核心渲染路径的硬依赖；没有适配器时仍应能显示壁纸。
当前 daemon 已经支持从 `hyprctl -j monitors/clients` 和
`niri msg --json outputs/workspaces/windows` 构建 `DesktopSnapshot`。如果对应
session 环境变量或命令不可用，会回退到 `generic-wayland` 占位快照；真实 GDK
monitor 后端由 GTK 主线程读取，后台 IPC 线程不会用空的 GDK 结果覆盖已经采集到
的输出快照。电源状态独立于合成器读取，会从 Linux
`/sys/class/power_supply` 推断 AC/Battery/Unknown，并注入同一份
`DesktopSnapshot.power`。会话活跃/锁屏状态会在 systemd-logind 可用时通过
`XDG_SESSION_ID` 和 `loginctl show-session ... Active/LockedHint` 读取；不可用时
保守视为 active/unlocked。
验证和性能采样可以用 `GILDER_POWER_STATE=ac|battery|unknown` 覆盖当前 daemon
进程看到的电源状态，便于在台式机或 CI 中复现 battery throttling；未设置时仍只读
真实 sysfs 状态。
`GILDER_OUTPUT_STATE=active|unfocused|fullscreen|hidden` 可以覆盖当前 daemon
进程看到的输出可见性、焦点和 fullscreen 状态，用于稳定复现 unfocused
throttling、fullscreen pause 和 output-hidden pause；未设置时使用合成器/GDK
采集到的真实输出状态。
`GILDER_OUTPUT_STATE_FILE=/path/to/state` 是同一覆盖的文件形式，daemon 每次刷新
桌面快照时重新读取文件内容，便于验证脚本在同一个 daemon 生命周期内模拟
fullscreen -> active 这类状态转换并测量恢复延迟。
`GILDER_SESSION_STATE=active|inactive|locked` 可以覆盖当前 daemon 进程看到的
logind session 状态，用于稳定复现 session-inactive 和 session-locked 暂停策略。
`GILDER_ADAPTIVE_STATE=inactive|cpu-pressure|memory-pressure|temperature|gpu-busy|low-battery|all`
可以覆盖 adaptive monitor 的系统压力样本，用于在 CI/headless smoke 中稳定复现
adaptive throttle、`pause-unfocused` 和 `pause-dynamic`；未设置时仍只读真实
PSI、thermal、power_supply 和 DRM 采样。

## 渲染路径

当前默认 daemon 仍以 GTK/GStreamer 路径覆盖完整壁纸类型和生命周期；native-wgpu 路径用于
验证更底层的 Wayland/Vulkan/CUDA 能力。后续新增 web、scene-lite、shader、particle 等
runtime 时，manifest、render plan、属性系统和 telemetry 必须保持 renderer 后端无关。纯
Vulkan 后端的迁移准备和停止/成功条件记录在 `docs/vulkan-migration.md`；在它成为默认后端前，
现有 GTK/wgpu 路径继续作为功能覆盖和真实 Wayland 回归基线。

静态图片：

- 加载 PNG、JPEG、WebP、AVIF，后续按系统库能力扩展。
- 在包加载时读取尺寸和色彩信息，按输出选择最接近的 variant。
- 渲染层只处理 fit/crop/tile/solid background，不在热路径做重复解码。
- `gtk-renderer` feature 使用 GTK 4 与 gtk4-layer-shell 创建 background layer
  窗口。普通静态图使用 `gtk::Picture` 映射 `cover`、`contain`、`stretch` 和
  `center`，只在 `tile` 这种 Picture 不直接支持的模式退回 CSS background。启用该
  feature 时，daemon 在主线程运行 GTK application，IPC accept loop 在后台线程运行，
  状态变更会通过同步队列投递到 GTK 主线程。
- daemon 会为当前 desktop snapshot 和持久化状态生成 `render_sync`，列出每个
  输出的静态渲染计划、需要移除的输出和加载错误。`.gwp` 会在计划阶段解包到
  `$XDG_CACHE_HOME/gilder/render-cache/`，GTK 主循环会消费这些计划，为匹配到的
  GDK monitor 创建或更新 background layer 窗口，并关闭 removals、加载错误和
  当前快照中已经消失的输出窗口。
- GTK 静态渲染器会记住每个输出上次应用的静态 plan；当 source、fit、background
  和输出名都未变化时，后续同步不会重复移除/创建 Picture 或 CSS fallback surface。

视频壁纸：

- 首选 GStreamer pipeline，利用系统硬件解码能力。
- daemon 会为 video entry 生成 `render_sync.video_plans`，包含 source、poster、
  loop、muted、fit、start offset 和性能策略合成后的目标 FPS。
- 如果 video entry 提供 poster，或 manifest 的 `preview.poster` 可用，daemon 会同时
  生成一条静态 poster plan；`gtk-renderer` 可以先把它显示在 background layer，
  作为视频 sink 接入前以及加载失败时的占位画面。视频 pipeline 成功接管输出后，
  GTK renderer 会释放实际 static surface，但保留 poster plan 作为错误 fallback。
- 同时启用 `gtk-renderer` 和 `video-renderer` 时，GTK 主线程会尝试为每个
  video plan 构建 `playbin + gtk4paintablesink`，把 GStreamer 提供的
  `GdkPaintable` 放入对应输出的 layer-shell background window；poster 仍作为加载
  前、插件缺失和 pipeline 后续错误时的 fallback，不应在 active video 期间长期保留
  为额外 static surface。
- 只启用 `video-renderer` 时，daemon 会启动独立 GStreamer worker，消费同一份
  `render_sync`，并用 headless sink 固化 playbin 生命周期、loop、muted、
  pause/resume/stop 和 bus polling 控制面。
- 默认音频被丢弃。只有 manifest `runtime.allow_audio = true` 且 video entry
  `muted = false` 时，GStreamer 才允许音频输出；否则 playbin 使用 `fakesink`
  丢弃音频。
- 性能策略合成出的 `target_max_fps` 会通过 video sink 的 `throttle-time`
  应用，避免在 decoder 和 sink 之间插入 `videorate ! capsfilter` 干扰
  DMABuf/GLMemory caps 协商。
- 渲染器在应用 video plan 时会跳过未变化的 state、mute、fit、target FPS 和
  start offset，避免周期性 render sync 造成重复 GStreamer property 更新或把视频反复
  seek 回起始偏移。
- 支持 MP4/H.264、WebM/VP9/AV1，实际支持由系统插件决定。
- 循环、静音、音频丢弃、最大 FPS、poster、空闲暂停必须是 manifest 中的显式策略。
- 解码和播放控制不阻塞 GTK 主线程。

GTK/GStreamer 低内存渲染方向：

- 不把硬解等同于 zero-copy。运行时必须同时报告实际 decoder、decoder class、decoder
  policy status、decoder/sink caps memory features、memory path、allocation query、
  QoS 和 GTK frame clock；只有出现
  sink-side DMABuf/GLMemory 等 GPU memory caps，并且后续补齐 compositor
  presentation 证据后，才把路径视为强 zero-copy 证据。
- 避免在 decoder 到 sink 之间插入会破坏 GPU memory caps 协商的通用 CPU 元件。当前
  active 视频默认不再插入 `videorate ! capsfilter`，而是使用 sink
  `throttle-time`；muted 路径只启用 video playbin flag，并关闭 sink
  `enable-last-sample`，减少无意义的 audio/deinterlace/last-sample 常驻引用。
- GTK 视频路径默认直接使用 `gtk4paintablesink`，避免 `glsinkbin` 在 NVIDIA/GL wrapper
  场景中引入额外 driver buffer、texture/pool 和 anonymous memory 保留。需要验证
  GLMemory/DMABuf 时仍可用环境变量强制 `glsinkbin+gtk4paintablesink`。sink 低内存调优还会
  在支持时关闭 async preroll、preroll frame 和 render delay，减少 paused/preroll/last-frame
  路径的额外帧保留。
- GTK renderer 已把视频运行时从单个输出对象里拆出来：对兼容的
  `(source, loop, muted/audio policy, decoder policy, start offset, target FPS)` 使用一个
  共享 GStreamer pipeline 和一个共享 `GdkPaintable`，每个输出只持有自己的
  `gtk::Picture`、fit 和 frame-clock 统计。输出暂停或移除时只 detach 对应 picture；
  最后一个输出释放时才把 pipeline 置为 `Null`。`renderer_runtime` 和 telemetry 会报告
  `video_shared_runtimes`，用于区分 video surface 数和实际共享 GStreamer runtime 数。
  这能同时降低多输出同源视频的解码、buffer pool、sink texture 和进程私有内存占用。
- 启用 video renderer 的构建中，视频计划不再同时生成 poster 静态 surface 计划；GTK
  会先尝试建立视频 surface，只有 pipeline 构建失败或运行中报错时才从
  `video_plans[].poster` 懒加载 poster fallback。这样 active/resumed 视频路径不会先解码
  一张通常很大的 poster 再立刻释放，启动峰值 decoded texture 和私有内存会更低。未启用
  video renderer 的构建仍保留 poster 静态 fallback，避免没有视频能力时空白。
- 运行时已经报告 `memory_path`、`allocation_reports` 和 `retention_report`，能区分 CPU raw
  frame、decoder 侧 GPU/DMABuf、sink 侧 GPU/DMABuf、已响应的 allocator/buffer pool，以及
  system-memory pool、last-sample/preroll frame 保留等风险。后续继续用真实 PSS/USS/private
  delta 和 compositor presentation 证据校准 `gtk4paintablesink`、GDK/GSK texture、GStreamer
  allocator 和 buffer pool 生命周期；优先通过运行时证据和小步重构减少保留，而不是只调高内存预算。
- decoder/caps/allocation/memory path 诊断按 video runtime 缓存并低频刷新；共享 video
  runtime 的 telemetry snapshot 会在一次 renderer snapshot 内复用同一份诊断、position 和
  duration 查询；常规 video polling 只处理 bus/EOS/error/QoS，不会在每个 tick 反复遍历
  pipeline 或发 allocation query。
- GTK renderer tick 会按当前负载动态调度：video runtime 单独存在时使用 250ms 常规
  polling，frame stats 按 500ms 写回最近的 runtime snapshot；如果同一进程还有 slideshow
  过渡需要更短 tick，则按 slideshow 的短间隔运行；纯静态无动态工作时不安装 renderer
  runtime timeout，render sync 由 bridge 线程投递 GLib idle wakeup 后在 GTK 主线程立即消费，
  减少 8K static idle wakeup。完整 renderer runtime snapshot 只在收到新 render sync、
  slideshow 实际换帧、decoder 观察结果变化或 pipeline 报错时写入；frame stats 到期判断只读取
  video runtime 计数，不会重新计算 surface/source footprint；GTK 组合 snapshot 复用
  `resource_snapshot()` 中已经计算出的 video source footprint，不会为了序列化 video pipeline
  telemetry 再重复读取源文件 metadata。resource footprint 内部按路径缓存 source size，
  重复静态图、幻灯片帧或同源视频仍保持 reference byte 语义，但每个唯一路径只读取一次 metadata。
- GTK video frame-clock 诊断默认走轻量模式：每个输出只连接 after-paint handler，并且每帧只
  记录 tick、frame counter/time 和 interval，避免 4K/高刷 active 视频在 GTK 主线程持续执行
  phase signal、FPS/refresh_info 和 GDK `FrameTimings` 查询。需要完整 presentation 证据时，
  启动 daemon 前设置 `GILDER_GTK_VIDEO_FRAME_STATS=full`；需要最大化性能排查时可设置为
  `off` 完全关闭这组 GTK frame-clock 诊断。
- GStreamer video pipeline 会在 `playbin` 子元素创建时把 queue/queue2/multiqueue 这类内部队列压到
  4 buffers、25ms、byte 上限关闭，用小时间/帧数窗口约束 4K/高刷视频的中间缓冲，而不是让
  默认 1-2s 队列在 raw/GL 帧路径上放大 PSS/USS/GPU memory。该调优不默认开启 leaky/drop；
  下游跟不上时先让上游背压，避免为了省内存破坏关键帧解码稳定性。runtime diagnostics 会报告
  `queue_reports`，用于把 queue current level 和 PSS/显存采样对齐。
- GTK sink chain 默认仍为 `auto`，但默认路径使用 direct `gtk4paintablesink`，把 NVIDIA/GL
  wrapper 场景中额外的 PSS/USS 和显存保留排除出常规播放路径。需要验证 GLMemory/DMABuf 或
  零拷贝行为时，可用 `GILDER_GTK_VIDEO_SINK_CHAIN=glsinkbin` 强制
  `glsinkbin+gtk4paintablesink`；`gtk4` 可显式固定 direct sink，便于对比同一视频下的
  sink caps、queue reports、PSS/USS 和 GPU memory。
- 静态图普通 fit 已从 CSS background 改为显式 `gtk::Picture` surface，切到视频、
  移除输出或换帧时会从 GTK 容器移除 Picture 引用；`tile` 仍保留 CSS background
  fallback。大图已有输出尺寸级缓存，后续还要继续确认 GDK/GSK decoded texture
  生命周期。telemetry 已拆分 static Picture/CSS/color surface 数，并按 Picture
  paintable intrinsic size 估算 RGBA decoded footprint，作为 retained texture 风险线索。

当前技术栈冲击顶级性能的判断：

- GTK + GStreamer + direct `gtk4paintablesink` 仍是下一阶段的主线。它已经能把
  视频解码、GTK layer-shell surface、生命周期释放、decoder/caps/allocation 诊断和性能
  策略放在同一套 runtime 中，工程复杂度和可维护性明显低于自写 Wayland/GL sink。
- 这条栈可以做到“壁纸软件顶级”：静态 8K 路径应保持 CPU 接近 0、低私有内存；4K/240fps
  视频路径在 NVIDIA/niri 20 逻辑 CPU 本机样本中已降到约 75% 进程 CPU、约 3.8% 整机 CPU，
  `Private_Dirty` 约 109MiB、PSS/USS/显存约 390/356MiB/496MiB。后续增强应把
  sink-side GLMemory/DMABuf、allocation pool、compositor presentation 证据纳入回归门槛，
  而不是再把优化前的 `glsinkbin` high-memory 路径当作默认目标。
- 这条栈不保证天然达到“播放器/渲染引擎极限”。如果 GTK/GDK/GSK 或
  `gtk4paintablesink` 在目标驱动/合成器上无法稳定暴露 sink-side GLMemory/DMABuf，或者
  presentation/frame pacing 仍高于预算，就需要评估更低层方案：自定义 GStreamer sink、
  libmpv/render API、或直接 Wayland linux-dmabuf/GL/Vulkan surface。替换栈的触发条件必须
  来自同场景实测：硬解已满足但 sink caps 只能到 SystemMemory、PSS/private/GPU 显存无法压到
  T0 预算、或 compositor presentation 证明长期掉帧。

Native Wayland host 方向：

- 更低层重构不应把所有壁纸类型都塞进一个 renderer。目标是把 Wayland/layer-shell
  surface 生命周期、输出匹配、缩放、viewport、frame callback 和暂停/恢复策略抽成
  `native-wayland-host`，再按内容类型接入不同 runtime。
- 该 host 可以借鉴 linux-wallpaperengine 的 wlr-layer-shell/smithay-client-toolkit
  做法：为每个输出创建 `Layer::Background` surface，锚定四边，处理 fractional scale
  与 viewporter，并把 raw Wayland display/surface 交给内容 runtime。Gilder 主 crate
  已把 `unsafe_code` 从 forbid 放宽为 warn；raw handle、GStreamer overlay 或 GPU surface
  创建这类不可避免的 `unsafe` 必须留在明确的 native Wayland/GPU 边界内，并通过实测和代码审计
  逐步证明为 safe wrapper。
- 第一阶段 `native-wayland-renderer` 只建立 host/surface 生命周期、configure/scale 状态和
  capability 暴露；raw Wayland handle 导出会在接入 wgpu/GStreamer overlay 前以单独 safe
  wrapper 落地，避免依赖 Wayland crate 私有指针布局。
- static image、slideshow 和 scene-lite 可以优先走同一套 wgpu/CPU upload renderer，
  在静态或暂停状态只按 configure/属性变化重绘；video 走 GStreamer sink/overlay 或后续
  DMABuf-aware sink；shader 走 wgpu shader runtime；这些路径共享 host 生命周期和性能策略。
- web 壁纸是单独的浏览器 runtime，不是 video sink 的延伸。最现实的第一阶段是
  `gilder-web-renderer` helper：内部可使用 WebKitGTK/GTK layer-shell 或等价 C/GObject
  绑定，主 daemon 只通过 IPC 下发 root/index、属性、mute、pause、FPS/可见性和权限策略。
  这样 GTK/WebKit 的固定内存与崩溃面被限制在 web helper 内，主 renderer 不再依赖
  gtk-rs；后续如果 WPE WebKit、CEF/Ozone Wayland 或 offscreen texture 路径能用实测证明
  更低内存，再替换 helper 内部实现。
- 是否替换 GTK 默认路径必须用同一批证据 gate：4K/240fps video 的 CPU、PSS/USS、
  `Private_Dirty`、NVIDIA/DRM 显存、QoS/drop、presentation/frame callback、active ->
  paused/fullscreen/hidden 的资源释放，以及 static/web/scene/shader 的空闲 wakeup 和恢复延迟。

轻量动态壁纸：

- v1 不引入复杂脚本运行时。
- 优先支持视频、帧序列、简单 slideshow、参数化颜色/速度/缩放。
- daemon 会为 slideshow entry 生成 `render_sync.slideshow_plans`，包含 source
  列表、切换间隔、transition、fit 和性能策略合成后的目标 FPS。GTK renderer
  当前使用主线程低开销定时器执行即时切换，后续再扩展 crossfade 等过渡。
- Web wallpaper 作为受限运行时处理，默认关闭本地文件越界访问和网络权限；在
  WebKit runtime 完成前，renderer 使用 manifest `fallback` 生成静态计划，并按
  动态壁纸参与 `pause-dynamic` 资源释放策略。
- Shader wallpaper 先作为一等 manifest/fallback 类型处理：包可以声明 GLSL/WGSL
  source、uniform schema 和静态 fallback；原生 GPU shader runtime 完成前，renderer
  显示 fallback，并按动态壁纸参与 `pause-dynamic` 资源释放策略。

## 状态与配置

建议路径：

- 配置：`$XDG_CONFIG_HOME/gilder/config.toml`
- 状态：`$XDG_STATE_HOME/gilder/state.json`
- 缓存：`$XDG_CACHE_HOME/gilder/`
- 用户安装包：`$XDG_DATA_HOME/gilder/wallpapers/`
- IPC socket：`$XDG_RUNTIME_DIR/gilder/gilder.sock`

配置关注用户偏好，状态关注当前输出绑定：

- 默认壁纸。
- 每个输出的壁纸、variant、fit mode、暂停状态。
- 性能策略，如最大 FPS、接电/电池策略、fullscreen 暂停策略。
- 转换器生成包的导入目录。

配置里的 `default_wallpaper` 和 `[outputs.<name>].wallpaper` 会作为默认绑定参与
`render_sync`，`[outputs.<name>].fit` 可以覆盖该输出上的 manifest fit mode。IPC
产生的 persisted state 是运行时覆盖层，壁纸选择优先级为：输出状态壁纸、默认状态
壁纸、输出配置壁纸、默认配置壁纸。
`gilderctl set <wallpaper> --variant <id>` 会把 manifest variant 绑定到默认状态或
指定输出状态，当前静态图片和视频计划会用该 variant 的资源路径替代 entry 默认资源。
如果没有显式 variant，计划阶段会使用合成器/GDK 输出尺寸与 scale 自动选择最小的
可覆盖 variant；没有可覆盖资源时保留 entry 默认资源。

## 性能原则

- 图片和视频本身已经压缩时，打包阶段默认不做二次压缩。
- 大文件按需读取，不把整个视频载入内存。
- 同一个包被多个输出使用时，单次 render sync 会共享包加载/校验结果；后续渲染层
  继续共享可复用纹理/解码资源。
- 输出不可见、显示器断开、用户暂停、合成器 fullscreen 时暂停动画。
- 对 HiDPI 输出优先选能覆盖物理目标尺寸的 variant，避免运行时放大模糊，同时避免
  小输出默认加载过大的 4K/8K 资源。

## 桌面状态性能策略

性能策略独立于 GTK 渲染器和具体合成器适配器：

- 合成器适配器提供 `DesktopSnapshot`，包含输出可见性、focused、fullscreen、工作区和电源状态。
- 电源状态由 Linux `power_supply` sysfs 提供；系统电池放电时触发 battery
  策略，外接电源在线或电池正在充电/已满时视为 AC。
- `GILDER_POWER_STATE=ac|battery|unknown` 是验证用覆盖入口，可以强制当前 daemon
  进程的 `DesktopSnapshot.power`，用于稳定采集 battery/AC 对比证据。
- `GILDER_DESKTOP_OUTPUTS=eDP-1,HDMI-A-1:1920x1080@1.5` 是验证用输出列表覆盖入口，
  可以在没有真实 compositor 输出的 headless smoke 中构造虚拟输出。
- `GILDER_OUTPUT_STATE=active|unfocused|fullscreen|hidden` 是验证用输出状态覆盖入口，
  用于采集 focused/unfocused/fullscreen/hidden 场景对比证据。
- `GILDER_OUTPUT_STATE_FILE` 是可动态修改的验证用输出状态覆盖入口，用于在不重启
  daemon 的情况下切换输出状态并采集恢复延迟证据。
- 会话状态由 logind 提供；当用户切换到非活跃 session/VT 时，
  `SessionInactive` 决策会暂停渲染；锁屏时 `SessionLocked` 决策也会暂停渲染。
- daemon 持久化 `AppState`，记录每个输出的壁纸、暂停状态和用户属性。
- `PerformanceConfig` 从 `$XDG_CONFIG_HOME/gilder/config.toml` 读取，控制 fullscreen、hidden、session、unfocused、battery 时继续、限帧、暂停或仅暂停动态壁纸。
- `[outputs.<name>.performance]` 可以覆盖单个输出的 FPS 和 fullscreen/hidden/session/unfocused/battery 策略，适合把副屏、投影输出或高耗电输出配置得更保守。
- `decide_performance` 将配置、桌面状态和输出状态合成为渲染决策：active、throttled 或 paused。多个条件同时命中时选择最省资源的结果：paused 优先于 throttled，同为 throttled/active 时选择更低 `max_fps`；同等强度时保留更早命中的明确原因。
- `battery = "pause-dynamic"`、`fullscreen = "pause-dynamic"`、`unfocused = "pause-dynamic"`、`hidden = "pause-dynamic"` 和 `session = "pause-dynamic"` 是可选动态壁纸释放策略：daemon 在未加载 manifest 前不提前移除输出；确认壁纸是 video/slideshow/web/scene-lite/shader 后才把该输出转为 paused/remove，静态壁纸仍按原桌面状态渲染。
- manifest `runtime.pause_when_fullscreen` 和 `runtime.pause_when_unfocused` 会在包加载后作为额外保守策略合入同一份决策；如果配置、用户暂停、输出隐藏或会话 inactive 已经要求暂停，daemon 不会为了读取 manifest 再加载包。
- manifest `runtime.allow_audio` 与 video entry 的 `muted` 合成最终视频静音状态，默认不输出音频。
- adaptive system monitor 是用户可选策略层，默认关闭。开启后会采样 Linux PSI
  CPU/内存压力、thermal zone 最高温度、power_supply 电源/电池容量细节和可用 DRM
  `gpu_busy_percent` 计数，按阈值把 CPU、GPU、内存、温度和低电量结果作为保守输入
  合入 `decide_performance` 之后的输出级决策；默认动作是降低 FPS，也可以配置为只在
  输出非焦点时暂停，焦点输出仍回退为降 FPS，或只暂停 video/slideshow/web/scene-lite/shader 这类动态壁纸。
  adaptive 决策不能覆盖用户暂停、fullscreen pause、battery pause 等更强策略；同为 throttled 时会保留更低 FPS 的策略。
  该策略支持阈值、冷却时间、每输出开关、每输出动作覆盖和全局 kill switch，并在
  `status`/telemetry 中报告当前采样、触发原因和 adaptive 动作，方便用户审计。视频
  renderer runtime 会报告播放 position、duration、实际 frame limiter 状态、GStreamer
  QoS processed/dropped 统计、GTK frame clock tick/interval 统计，以及从实际 decoder
  和 caps memory features 推导的 zero-copy 证据分级、memory path 分级和
  allocator/buffer-pool 协商线索；compositor presentation feedback 或原生 Wayland frame
  callback 统计仍是后续工作。
  `GILDER_ADAPTIVE_STATE` 仅作为验证入口，用于构造高于当前阈值的 CPU/内存压力、温度、
  GPU busy 或低电量样本，让 headless smoke 可以确定性覆盖 adaptive 动作。

这让后续 niri/Hyprland 适配器只需要负责提供准确桌面状态，渲染器只需要执行策略结果。
`status`、`outputs`、状态变更事件和 daemon 周期刷新都会返回每个输出的性能决策，
`render_sync.decisions` 也会随同步计划携带同一份输出级决策。读请求会按
`desktop_refresh_interval_ms` 复用最近的桌面快照，避免状态栏轮询或性能采样过于频繁
地调用 compositor 适配器；状态修改命令和周期刷新仍会强制采集新的桌面快照。
`status.telemetry` 会暴露桌面刷新、read 请求快照复用、桌面变化和 `render_sync`
缓存 hit/miss 计数、单次 render sync 的 package/archive cache 统计、archive cache
淘汰计数、静态大图运行时降采样缓存的生成/复用/淘汰计数、计划层静态图/poster/slideshow 图片资源数量和源文件字节 footprint、计划层视频 source 引用/去重/重复候选、GTK renderer
当前 static surface/slideshow surface/video pipeline 指向的源资源引用数、去重资源数和字节 footprint、static surface 类型计数、估算 decoded footprint，以及渲染器同步更新
queued/skipped 计数。计划层和 renderer 源文件字节不是解码后的纹理内存或 USS，但能在性能采样中暴露
大图、大 poster、slideshow 图片或视频源是否仍被计划引用或被 GTK surface/pipeline 持有，便于用性能采样证明确实没有因为轮询
反复调用 compositor 适配器、重复生成渲染计划、无限保留旧 `.gwp` 解包缓存、GTK surface 残留或重复投递未变化的同步。
视频 source 重复候选用于定位同一视频在多个输出上被计划为独立 pipeline 的场景，为后续解码/texture 共享优化提供基线。
周期刷新只在桌面快照变化时发送 `desktop.changed` watch 事件，并且只在
`render_sync` 实际变化时投递给渲染器，避免固定频率重建 pipeline。IPC 状态变更
仍会广播 `state.changed` 供客户端更新 UI，但如果生成的 `render_sync` 和上一份一致，
daemon 不会把它再次送入渲染器队列。GTK renderer bridge 线程收到新同步后会投递一个
GLib idle wakeup；GTK 主线程消费队列时会 drain 当前积压并只应用最新 render sync，因为每份
render sync 都描述完整目标状态；这样快速切换或状态风暴不会让 GTK/GStreamer 反复创建中间态
surface/pipeline 或重复生成 runtime snapshot。GTK 静态渲染器会在 paused 时关闭对应
background 窗口；GStreamer 渲染器根据 `mode` 和 `max_fps` 执行暂停或限帧。
刷新周期由 `performance.desktop_refresh_interval_ms` 配置，默认 2000ms，实际运行会
钳制到不低于 250ms。
daemon 会缓存最近一次 `render_sync`，当渲染相关 config（壁纸绑定、fit、性能策略和
FPS 上限、视频 decoder 策略、package/render/static-image cache 上限）、渲染相关 state（壁纸绑定、variant、暂停状态和输出条目）、desktop
snapshot、cache 目录和已引用壁纸包的 JSON/TOML manifest/`.gwp` 元数据都未变化时，后续
`status`、watch snapshot 和状态事件会复用缓存，避免性能采样期间反复读取
manifest、校验资源或解包。当前不参与渲染的 properties、adapter 开关和桌面状态刷新
周期不会单独让缓存失效。
单次 render sync 生成期间会用临时 package cache 复用已解析的 manifest/package，默认最多保留 16 个条目，并且这些条目引用的去重源资源 footprint 默认最多 512MiB；超过条目数或 `package_cache_max_retained_unique_resource_bytes` 后按最早插入优先淘汰。`[cache].package_cache_max_entries = 0` 或 `[cache].package_cache_max_retained_unique_resource_bytes = 0` 会禁用该临时保留，适合希望压低 plan 构建峰值内存的用户。这里的 byte 上限基于 manifest 引用的源文件/目录大小，用作大包保留线索，不是解码纹理、GTK 内部缓存或 USS；telemetry 还会把 retained preview thumbnail/poster 的引用数、去重数和源文件 byte footprint 单独拆出，便于发现超大 preview 资产。
`.gwp` 解包目录会写入 `$XDG_CACHE_HOME/gilder/render-cache/`，默认最多保留 32 个旧
archive cache 条目；生成计划时当前正在使用的 archive cache 条目会被保护，其余条目按最旧优先淘汰。
`[cache].render_cache_max_entries = 0` 表示尽量只保留当前受保护条目，适合希望 aggressive
清理旧包缓存的用户。
静态 raster entry 如果带有源图 `width`/`height`，且没有显式 variant、没有可覆盖输出
尺寸的 manifest variant、源图像素面积至少是目标输出的 2 倍，daemon 会在有 `ffmpeg`
可用时生成 `$XDG_CACHE_HOME/gilder/static-image-cache/` 下的输出尺寸级 PNG 缓存，并把
静态计划 source 指向该缓存文件。默认最多保留 32 个静态缓存文件且总量最多 512MiB；
当前 render sync 引用的文件会被保护，其余文件按最旧使用时间淘汰，直到同时满足条目数和
`static_image_cache_max_bytes`。`[cache].static_image_cache_max_entries = 0`
会禁用运行时静态降采样缓存；`[cache].static_image_cache_max_bytes = 0` 表示不按 byte
总量额外淘汰，只保留条目数上限。
scene-lite 的 time=0 静态 snapshot 会写入
`$XDG_CACHE_HOME/gilder/scene-lite-cache/` 下的 SVG 文件，并复用同一组
`static_image_cache_max_entries`/`static_image_cache_max_bytes` 上限和最旧优先淘汰策略。
这些 SVG 是 scene graph 的轻量显示 surface，不是解码后的纹理内存；status telemetry
会报告 snapshot cache 的条目数、字节数、生成、复用和淘汰计数。

示例：

```toml
[performance]
interactive_max_fps = 60
background_max_fps = 30
battery_max_fps = 24
fullscreen = "pause" # continue, throttle, pause, pause-dynamic
hidden = "pause" # continue, pause, pause-dynamic
session = "pause" # continue, pause, pause-dynamic
unfocused = "throttle" # continue, throttle, pause, pause-dynamic
battery = "throttle" # continue, throttle, pause, pause-dynamic

[adaptive]
enabled = false
kill_switch = false
refresh_interval_ms = 2000
cooldown_ms = 10000
throttle_max_fps = 15
action = "throttle" # throttle, pause-unfocused, pause-dynamic
cpu_pressure_threshold_percent = 75
memory_pressure_threshold_percent = 20
temperature_threshold_celsius = 85
gpu_busy_threshold_percent = 90
battery_capacity_threshold_percent = 20

[video]
decoder = "auto" # auto, hardware-preferred, hardware-required, software

[cache]
package_cache_max_entries = 16
package_cache_max_retained_unique_resource_bytes = 536870912
render_cache_max_entries = 32
static_image_cache_max_entries = 32
static_image_cache_max_bytes = 536870912

[outputs."HDMI-A-1"]
wallpaper = "/home/me/Wallpapers/quiet.gwpdir"
fit = "contain"

[outputs."HDMI-A-1".performance]
background_max_fps = 12
unfocused = "pause-dynamic" # continue, throttle, pause, pause-dynamic
hidden = "pause-dynamic" # continue, pause, pause-dynamic
battery = "pause-dynamic" # continue, throttle, pause, pause-dynamic

[outputs."HDMI-A-1".adaptive]
enabled = true
throttle_max_fps = 9
action = "pause-unfocused"
```

`[video].decoder` 由视频 renderer 在构建 GStreamer pipeline 前消费。第一版策略通过调整
已知 H.264/VP9/AV1 decoder 的 feature rank 影响 `playbin`/`decodebin` autoplug：
`hardware-preferred` 提高 VAAPI/VDPAU/NVDEC 等硬解 decoder rank 并保留软解 fallback；
`hardware-required` 禁用已知软解 fallback；`software` 禁用已知硬解 decoder；
`auto` 恢复宿主 GStreamer 原始 rank。运行时仍需通过 `actual_decoder_reports`、
`decoder_policy_status` 和 `caps_reports` 验证实际路径。

## 安全原则

- `.gwp` 和 `.gwpdir` 不允许 manifest 路径逃逸包根目录。
- `.gwpdir` 可用 TOML 手写 manifest，但运行时统一反序列化到同一套 manifest
  结构并执行同样校验；`.gwp` 发布包使用规范化 JSON manifest。
- 转换器不执行 Wallpaper Engine 项目的脚本，只解析项目元数据和资源。
- Web wallpaper 默认使用受限权限：禁止任意本地路径读取，网络访问需要显式启用。
- IPC 只绑定用户 runtime 目录内的 Unix socket，不提供 TCP 监听。
- 包内配置 schema 必须可验证，未知字段应保留但不能影响安全策略。

## 早期里程碑

1. 静态图片包加载和单输出显示。
2. 多输出绑定、IPC 切换和状态持久化。
3. 视频循环壁纸和暂停策略。
4. Wallpaper Engine 静态/视频项目转换。
5. Hyprland/niri 输出适配器。
6. Web wallpaper fallback 渲染计划，再扩展为受限 WebKit runtime。
7. 部分 Scene wallpaper 转换为 Gilder scene-lite graph，再扩展为原生 scene surface。
