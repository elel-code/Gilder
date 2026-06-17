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

返回 daemon 状态、输出列表、当前壁纸、暂停状态、配置/状态文件位置和性能决策信息。

### outputs

```sh
gilderctl outputs
```

返回 daemon 当前知道的输出列表。早期实现会合并持久化状态中的输出和合成器适配器提供的输出；在真实合成器适配器接入前，列表可能只包含通过 IPC 设置过的输出。

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

## 计划命令

- `load`：预加载壁纸包。
- `config get/set`：读取或修改配置。
- `properties get/set`：读取或修改壁纸用户属性。
- `watch`：订阅输出变化、状态变化、错误事件。
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
