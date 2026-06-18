# IPC 协议

Gilder IPC 是用户会话内的 Unix socket 协议：

```text
$XDG_RUNTIME_DIR/gilder/gilder.sock
```

`GILDER_SOCKET=/path/to/socket` 可以覆盖 daemon 和 `gilderctl` 使用的 socket
路径，适合测试脚本或多实例诊断；生产会话通常使用默认路径。

早期使用 JSON Lines，每行一个请求或响应。后续可保留 JSON-RPC 2.0 形态，并在大流量事件订阅中切换到更紧凑的编码。

## 设计目标

- 能被 shell 脚本、快捷键、状态栏和 GUI 前端直接调用。
- 不暴露网络端口。
- 命令可组合，可明确指定 output。
- 错误可读，机器也可解析。

## 请求格式

```json
{"jsonrpc":"2.0","id":1,"method":"set","params":{"wallpaper":"~/Pictures/a.gwpdir","output":"eDP-1"}}
```

字段：

- `jsonrpc`: 固定 `2.0`。
- `id`: 客户端请求 ID。
- `method`: 命令名。
- `params`: 命令参数。

## 响应格式

```json
{"jsonrpc":"2.0","id":1,"result":{"accepted":true}}
```

错误：

```json
{"jsonrpc":"2.0","id":1,"error":{"code":"not_found","message":"output eDP-1 not found"}}
```

## 命令

### ping

```sh
gilderctl ping
```

用于探测 daemon 是否可用。

### status

```sh
gilderctl status
gilderctl status --decisions-csv
gilderctl status --telemetry-csv
gilderctl status --decisions-csv --from-file status-001.json
gilderctl status --telemetry-csv --from-file status-001.json
```

返回 daemon 状态、桌面快照、输出列表、当前壁纸、暂停状态、配置/状态文件位置、性能决策信息、renderer 能力诊断、daemon telemetry 和 `render_sync`。
`render_sync` 包含静态图片渲染器下一次同步需要执行的 `plans`、视频渲染器后续要消费的 `video_plans`、slideshow 渲染器要消费的 `slideshow_plans`、需要关闭的 `removals`、包加载/格式错误 `errors`，以及每个输出的 `decisions`。
视频壁纸有 poster 时，`plans` 会包含同一输出的静态 poster 占位计划，`video_plans` 仍包含实际视频 pipeline 计划。
`video_plans[].decoder_policy` 来自 `[video].decoder` 配置，当前可取 `auto`、`hardware-preferred`、`hardware-required` 或 `software`；它用于表达用户意图并进入状态/采样证据，后续硬解 autoplug 策略会消费同一字段。
`decisions` 会记录输出动作、当前壁纸路径和由桌面状态性能策略产生的 `mode/max_fps/reason`，视频/GStreamer 渲染器会用它执行暂停或限帧。`.gwp` 包会先解包到 `$XDG_CACHE_HOME/gilder/render-cache/`，再生成计划。
`slideshow_plans` 包含源图列表、切换间隔、transition、fit 和桌面状态策略合成后的 `target_max_fps`；当前 GTK renderer 会按间隔切换图片，`crossfade` 先作为格式字段保留。
`--decisions-csv` 会把 `render_sync.decisions` 与同输出的静态/视频/slideshow 计划合并为 `output_name,action,mode,reason,max_fps,wallpaper,plan_kind,source,fit,target_max_fps,muted` CSV，便于性能采样脚本和人工对比 active/paused/fullscreen/battery 场景中的实际资源、fit、视频限帧和静音策略；`--from-file` 可以重放已经保存的 `gilderctl status` JSON-RPC 响应。
`telemetry` 会报告桌面快照刷新次数、read 请求复用快照次数、桌面变化次数、`render_sync` 缓存 hit/miss，以及渲染器同步更新 queued/skipped 计数；`--telemetry-csv` 会输出 `desktop_refreshes,desktop_refresh_skips,desktop_changes,last_desktop_refresh_age_ms,render_sync_cache_hits,render_sync_cache_misses,render_sync_updates_queued,render_sync_updates_skipped`，便于确认状态栏轮询和性能采样没有持续触发 compositor 适配器、重复生成渲染计划或重复投递未变化的渲染同步。
`scripts/performance-snapshot.sh` 可以用 `--expect-render-sync-cache-hit`、`--expect-desktop-refresh-skip`、`--expect-render-sync-update-queued` 和 `--expect-render-sync-update-skipped` 把这些 telemetry 变成失败条件，适合 CI 或真实会话 smoke 证明缓存、刷新节流和渲染器同步投递去重仍然生效。
启用 `[adaptive]` 或单个 `[outputs.<name>.adaptive]` 时，`telemetry.adaptive.snapshot` 会报告 Linux PSI CPU/内存压力采样、`active_triggers` 和 kill switch 状态，`telemetry.adaptive.action` 会列出实际被降载的输出及目标 FPS。adaptive 当前只会把输出降到 throttled，决策原因是 `adaptive`；用户暂停、fullscreen pause、battery pause 等更强策略不会被覆盖。
daemon 会周期刷新桌面快照，只有快照变化时才发送 `desktop.changed` 事件；adaptive 触发状态改变但桌面快照未变化时会发送 `adaptive.changed` 事件；`status` 和 `outputs` 会按 `performance.desktop_refresh_interval_ms` 复用最近的桌面快照，并按 `adaptive.refresh_interval_ms` 复用最近的 adaptive 采样，避免轮询客户端过于频繁地调用 compositor 适配器或读取系统压力文件；`desktop.changed`、`adaptive.changed` 和 `state.changed` 会继续携带当前 `render_sync` 供客户端观察，但只有 `render_sync` 实际变化时才投递给内部渲染器。
daemon 会缓存最近一次 `render_sync`；当渲染相关配置、渲染相关状态、桌面快照和已引用壁纸包元数据未变化时，重复 `status` 会复用缓存，减少性能采样和状态栏轮询中的 manifest IO。当前 properties、adapter 开关和桌面刷新周期不参与静态/视频渲染计划，因此不会单独让缓存失效。
启用 `gtk-renderer` feature 的 daemon 会在 GTK 主线程消费同一份 `render_sync`，并把可用输出同步到 layer-shell background 窗口。
同时启用 `gtk-renderer` 和 `video-renderer` 时，GTK 主线程会尝试用 `gtk4paintablesink` 把视频 paintable 放入对应输出的 layer-shell 窗口；只启用 `video-renderer` 时，daemon 会启动 headless GStreamer worker 消费 `video_plans`，负责视频 pipeline 生命周期控制。
`renderer_capabilities.video.gstreamer.elements` 会报告 `playbin`、`fakesink`、`videorate`、`capsfilter` 和 `gtk4paintablesink` 是否可用；真实视频 surface 显示需要 `gtk4paintablesink` 为 `available: true`。
`renderer_runtime.video_pipelines` 会报告当前 daemon 内部渲染器实际持有的视频 pipeline 快照，包括输出、源文件、GStreamer state、运行模式、限帧、静音状态、`decoder_policy` 和 `actual_decoders`。`actual_decoders` 来自运行中 pipeline 的 GStreamer element factory 名称，例如 `avdec_h264`、`dav1ddec`、`vp9dec`、`vah264dec` 或 `vaav1dec`；`actual_decoder_reports` 会给每个实际 decoder 标记 `hardware`、`software` 或 `unknown`，用于区分当前播放路径实际是软解、硬解还是尚未完成 autoplug。空数组表示当前没有识别到 decoder element，常见于 pipeline 尚未 preroll、暂停在早期状态、没有视频 pipeline 或 GStreamer 选择了未列入诊断白名单的 decoder。

### outputs

```sh
gilderctl outputs
```

返回 daemon 当前知道的桌面快照和输出列表。列表会合并持久化状态中的输出和合成器适配器提供的输出。Hyprland session 下读取 `hyprctl -j monitors/clients`；niri session 下读取 `niri msg --json outputs/workspaces/windows`；不可用时回退到 generic snapshot。
验证时可以用 `GILDER_DESKTOP_OUTPUTS=eDP-1,HDMI-A-1:1920x1080@1.5` 构造虚拟输出列表，再叠加 `GILDER_OUTPUT_STATE`、`GILDER_POWER_STATE` 和 `GILDER_SESSION_STATE` 采集 headless 性能策略证据。

### set

```sh
gilderctl set <wallpaper.gwp|wallpaper.gwpdir> [--output <name>] [--variant <id>]
```

为指定输出或所有输出设置壁纸。
`--variant` 会把 manifest 中的资源变体 ID 写入当前壁纸绑定；静态图片和视频渲染计划会使用该 variant 的 `source` 替代 entry 默认 `source`。如果请求的 variant 不存在，`render_sync.errors` 会报告错误并跳过该输出的计划。
不指定 `--variant` 时，daemon 会按输出尺寸自动选择可覆盖目标尺寸的最小 variant；
没有合适 variant 时继续使用 entry 默认资源。
成功响应会返回当前 `renderer` 名称和本次投递给渲染器的 `render_sync`。

### pause / resume / stop

```sh
gilderctl pause [--output <name>]
gilderctl resume [--output <name>]
gilderctl stop [--output <name>]
```

控制动画或移除壁纸。

### properties

```sh
gilderctl properties get [key] [--output <name>]
gilderctl properties set <key> <value-json> [--output <name>]
gilderctl properties unset <key> [--output <name>]
```

读取、设置或清除壁纸用户属性。不带 `--output` 时操作全局/default 属性；带 `--output` 时操作指定输出的覆盖属性。`set` 的值按 JSON 解析，无法解析为 JSON 时作为字符串保存，因此 `true`、`0.5`、`{"x":1}` 和 `#ffaa00` 都能通过 CLI 传入。

### watch

```sh
gilderctl watch
```

订阅 daemon 事件流。连接建立后先返回一次 JSON-RPC success response，然后持续输出
JSON-RPC notification，每行一个事件：

```json
{"jsonrpc":"2.0","method":"event","params":{"sequence":1,"type":"snapshot","payload":{"outputs":[],"persisted_state":{"default_wallpaper":null,"outputs":{},"properties":{}},"render_sync":{"plans":[],"video_plans":[],"removals":[],"errors":[],"decisions":[]},"renderer":"not-implemented"}}}
```

当前事件类型：

- `snapshot`：订阅建立时发送的当前状态快照。
- `desktop.changed`：daemon 周期刷新发现桌面快照变化时发送，例如输出、fullscreen、focus、电源或 session 状态变化。
- `state.changed`：`set`、`pause`、`resume`、`stop`、`properties set/unset` 成功持久化后发送。

`snapshot`、`desktop.changed` 和 `state.changed` 都包含 `desktop`、`render_sync`、`renderer_capabilities` 和 `renderer_runtime`，GUI 前端和 daemon 内部渲染器可以用它判断每个输出是否已有可应用的静态或视频壁纸计划、当前运行时是否具备视频 surface 所需插件，以及运行中视频 pipeline 实际选中的 decoder。
`renderer` 会随 feature 组合返回 `gtk-layer-shell-static`、`gstreamer-video`、`gtk-layer-shell-static+gtk-gstreamer-video` 或 `not-implemented`；未启用渲染 feature 时只保留 IPC、状态和转换能力。

## 计划命令

- `load`：预加载壁纸包。
- `config get/set`：读取或修改配置。
- `import`：导入 `.gwpdir` 或 `.gwp` 到用户数据目录。

## 错误码

- `bad_request`
- `unsupported_protocol`
- `not_found`
- `invalid_package`
- `permission_denied`
- `renderer_error`
- `internal_error`

## 稳定性

协议版本从 `1` 开始。破坏性变更必须提升 `protocol`，并保持 `ping` 可用于版本协商。
