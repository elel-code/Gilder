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
`DesktopSnapshot.power`。会话活跃状态会在 systemd-logind 可用时通过
`XDG_SESSION_ID` 和 `loginctl show-session ... Active` 读取；不可用时保守视为
active。

## 渲染路径

静态图片：

- 加载 PNG、JPEG、WebP、AVIF，后续按系统库能力扩展。
- 在包加载时读取尺寸和色彩信息，按输出选择最接近的 variant。
- 渲染层只处理 fit/crop/tile/solid background，不在热路径做重复解码。
- `gtk-renderer` feature 使用 GTK 4 与 gtk4-layer-shell 创建 background layer
  窗口，并通过 CSS background 映射 `cover`、`contain`、`stretch`、`tile`、
  `center`。启用该 feature 时，daemon 在主线程运行 GTK application，IPC accept
  loop 在后台线程运行，状态变更会通过同步队列投递到 GTK 主线程。
- daemon 会为当前 desktop snapshot 和持久化状态生成 `render_sync`，列出每个
  输出的静态渲染计划、需要移除的输出和加载错误。`.gwp` 会在计划阶段解包到
  `$XDG_CACHE_HOME/gilder/render-cache/`，GTK 主循环会消费这些计划，为匹配到的
  GDK monitor 创建或更新 background layer 窗口，并关闭 removals、加载错误和
  当前快照中已经消失的输出窗口。

视频壁纸：

- 首选 GStreamer pipeline，利用系统硬件解码能力。
- daemon 会为 video entry 生成 `render_sync.video_plans`，包含 source、poster、
  loop、muted、fit、start offset 和性能策略合成后的目标 FPS。
- 如果 video entry 提供 poster，或 manifest 的 `preview.poster` 可用，daemon 会同时
  生成一条静态 poster plan；`gtk-renderer` 可以先把它显示在 background layer，
  作为视频 sink 接入前以及加载/暂停时的占位画面。
- 同时启用 `gtk-renderer` 和 `video-renderer` 时，GTK 主线程会尝试为每个
  video plan 构建 `playbin + gtk4paintablesink`，把 GStreamer 提供的
  `GdkPaintable` 放入对应输出的 layer-shell background window；poster 仍作为加载
  前和插件缺失时的 fallback。
- 只启用 `video-renderer` 时，daemon 会启动独立 GStreamer worker，消费同一份
  `render_sync`，并用 headless sink 固化 playbin 生命周期、loop、muted、
  pause/resume/stop 和 bus polling 控制面。
- 性能策略合成出的 `target_max_fps` 会通过 `videorate ! capsfilter` 应用到
  playbin 的 `video-filter`。
- 支持 MP4/H.264、WebM/VP9/AV1，实际支持由系统插件决定。
- 循环、静音、音频丢弃、最大 FPS、poster、空闲暂停必须是 manifest 中的显式策略。
- 解码和播放控制不阻塞 GTK 主线程。

轻量动态壁纸：

- v1 不引入复杂脚本运行时。
- 优先支持视频、帧序列、简单 slideshow、参数化颜色/速度/缩放。
- Web wallpaper 作为受限运行时处理，默认关闭本地文件越界访问和网络权限。

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

## 性能原则

- 图片和视频本身已经压缩时，打包阶段默认不做二次压缩。
- 大文件按需读取，不把整个视频载入内存。
- 同一个包被多个输出使用时，共享不可变元数据和可复用纹理/解码资源。
- 输出不可见、显示器断开、用户暂停、合成器 fullscreen 时暂停动画。
- 对 HiDPI 输出优先选高分辨率 variant，避免运行时放大模糊。

## 桌面状态性能策略

性能策略独立于 GTK 渲染器和具体合成器适配器：

- 合成器适配器提供 `DesktopSnapshot`，包含输出可见性、focused、fullscreen、工作区和电源状态。
- 电源状态由 Linux `power_supply` sysfs 提供；系统电池放电时触发 battery
  策略，外接电源在线或电池正在充电/已满时视为 AC。
- 会话状态由 logind 提供；当用户切换到非活跃 session/VT 时，
  `SessionInactive` 决策会暂停渲染。
- daemon 持久化 `AppState`，记录每个输出的壁纸、暂停状态和用户属性。
- `PerformanceConfig` 从 `$XDG_CONFIG_HOME/gilder/config.toml` 读取，控制 fullscreen、unfocused、battery 时继续、限帧或暂停。
- `decide_performance` 将配置、桌面状态和输出状态合成为渲染决策：active、throttled 或 paused。

这让后续 niri/Hyprland 适配器只需要负责提供准确桌面状态，渲染器只需要执行策略结果。
`status`、`outputs`、状态变更事件和 daemon 周期刷新都会刷新桌面快照并返回每个
输出的性能决策，`render_sync.decisions` 也会随同步计划携带同一份输出级决策。
周期刷新只在桌面快照变化时发送 `desktop.changed` watch 事件，并且只在
`render_sync` 实际变化时投递给渲染器，避免固定频率重建 pipeline。GTK 静态渲染器会
在 paused 时关闭对应 background 窗口；GStreamer 渲染器根据 `mode` 和 `max_fps`
执行暂停或限帧。刷新周期由 `performance.desktop_refresh_interval_ms` 配置，默认
2000ms，实际运行会钳制到不低于 250ms。

## 安全原则

- `.gwp` 和 `.gwpdir` 不允许 manifest 路径逃逸包根目录。
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
6. Web wallpaper 受限运行时。
7. 部分 Scene wallpaper 转换为 Gilder scene-lite。
