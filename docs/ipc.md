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
```

返回 daemon 状态、桌面快照、输出列表、当前壁纸、暂停状态、配置/状态文件位置、性能决策信息、renderer 能力诊断和 `render_sync`。
`render_sync` 包含静态图片渲染器下一次同步需要执行的 `plans`、视频渲染器后续要消费的 `video_plans`、需要关闭的 `removals`、包加载/格式错误 `errors`，以及每个输出的 `decisions`。
视频壁纸有 poster 时，`plans` 会包含同一输出的静态 poster 占位计划，`video_plans` 仍包含实际视频 pipeline 计划。
`decisions` 会记录输出动作、当前壁纸路径和由桌面状态性能策略产生的 `mode/max_fps/reason`，视频/GStreamer 渲染器会用它执行暂停或限帧。`.gwp` 包会先解包到 `$XDG_CACHE_HOME/gilder/render-cache/`，再生成计划。
daemon 会周期刷新桌面快照，只有快照变化时才发送 `desktop.changed` 事件；只有 `render_sync` 实际变化时才投递给渲染器。
启用 `gtk-renderer` feature 的 daemon 会在 GTK 主线程消费同一份 `render_sync`，并把可用输出同步到 layer-shell background 窗口。
同时启用 `gtk-renderer` 和 `video-renderer` 时，GTK 主线程会尝试用 `gtk4paintablesink` 把视频 paintable 放入对应输出的 layer-shell 窗口；只启用 `video-renderer` 时，daemon 会启动 headless GStreamer worker 消费 `video_plans`，负责视频 pipeline 生命周期控制。
`renderer_capabilities.video.gstreamer.elements` 会报告 `playbin`、`fakesink`、`videorate`、`capsfilter` 和 `gtk4paintablesink` 是否可用；真实视频 surface 显示需要 `gtk4paintablesink` 为 `available: true`。

### outputs

```sh
gilderctl outputs
```

返回 daemon 当前知道的桌面快照和输出列表。列表会合并持久化状态中的输出和合成器适配器提供的输出。Hyprland session 下读取 `hyprctl -j monitors/clients`；niri session 下读取 `niri msg --json outputs/workspaces/windows`；不可用时回退到 generic snapshot。

### set

```sh
gilderctl set <wallpaper.gwp|wallpaper.gwpdir> [--output <name>]
```

为指定输出或所有输出设置壁纸。
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

`snapshot`、`desktop.changed` 和 `state.changed` 都包含 `desktop`、`render_sync` 和 `renderer_capabilities`，GUI 前端和 daemon 内部渲染器可以用它判断每个输出是否已有可应用的静态或视频壁纸计划，以及当前运行时是否具备视频 surface 所需插件。
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
