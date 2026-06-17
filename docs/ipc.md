# IPC 协议

Gilder IPC 是用户会话内的 Unix socket 协议：

```text
$XDG_RUNTIME_DIR/gilder/gilder.sock
```

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

返回 daemon 状态、桌面快照、输出列表、当前壁纸、暂停状态、配置/状态文件位置和性能决策信息。

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
{"jsonrpc":"2.0","method":"event","params":{"sequence":1,"type":"snapshot","payload":{"outputs":[],"persisted_state":{"default_wallpaper":null,"outputs":{},"properties":{}},"renderer":"not-implemented"}}}
```

当前事件类型：

- `snapshot`：订阅建立时发送的当前状态快照。
- `state.changed`：`set`、`pause`、`resume`、`stop`、`properties set/unset` 成功持久化后发送。

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
