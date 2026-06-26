# IPC 协议

Gilder IPC 是用户会话内的 Unix socket 协议：

```text
$XDG_RUNTIME_DIR/gilder/gilder.sock
```

`GILDER_SOCKET=/path/to/socket` 可以覆盖 daemon 和 `gilderctl` 使用的 socket
路径，适合测试脚本或多实例诊断；生产会话通常使用默认路径。

请求和响应使用 JSON Lines，每行一个 JSON-RPC 风格对象。

## 请求

```json
{"jsonrpc":"2.0","id":1,"method":"set","params":{"wallpaper":"~/Pictures/a.gwpdir","output":"eDP-1"}}
```

字段：

- `jsonrpc`: 固定 `2.0`。
- `id`: 客户端请求 ID。
- `method`: 命令名。
- `params`: 命令参数。

错误响应：

```json
{"jsonrpc":"2.0","id":1,"error":{"code":"not_found","message":"output eDP-1 not found"}}
```

## status

```sh
gilderctl status
gilderctl status --decisions-csv
gilderctl status --telemetry-csv
gilderctl status --from-file status-001.json
```

`status` 返回 daemon 状态、桌面快照、输出列表、当前壁纸、暂停状态、
配置/状态文件路径、renderer 能力、daemon telemetry 和 `render_sync`。

`render_sync` 是 renderer 的当前计划：

- `plans`: 静态图片 surface 计划。
- `video_plans`: native Vulkan video 计划。
- `slideshow_plans`: slideshow 计划。
- `scene_lite_plans`: scene-lite 计划。
- `removals`: 需要释放的输出。
- `errors`: 包加载或计划生成错误。
- `decisions`: 每个输出的 `action/mode/reason/max_fps/wallpaper`。

视频计划字段包含 `source`、`poster`、`loop_playback`、`muted`、`fit`、
`start_offset_ms`、`target_max_fps` 和 `decoder_policy`。`poster` 只作为
native Vulkan video 元数据保留；daemon 不再把 video poster 展开成静态
surface fallback。

`decoder_policy` 来自 `[video].decoder`，当前取值为 `auto`、
`hardware-preferred`、`hardware-required` 或 `software`。当前视频路线是
FFmpeg demux/bitstream filter 生成 encoded access-unit，native Vulkan Video
负责 H.264/H.265/AV1 decode/render/present。

## telemetry

`telemetry.render_sync` 报告 render plan/cache footprint，包括 package/archive
cache 命中和淘汰、静态图缓存、计划层图片资源、计划层视频 source 引用/去重、
package retained resource 和 preview retained resource。这里的字节是源文件大小
footprint，不是解码纹理内存、GPU memory 或 USS。

`telemetry.renderer` 报告 renderer runtime 的 output/static/slideshow/video surface
计数、静态/slideshow/video source footprint、视频 pipeline 数、QoS/timing 兼容字段
和当前 renderer 持有资源。native Vulkan video 的详细性能证据以专用 smoke summary
为准。

`telemetry.adaptive` 报告 CPU/memory pressure、thermal、power_supply、DRM
`gpu_busy_percent`、active triggers 和当前 adaptive action。adaptive 决策只会在
不被用户暂停、fullscreen pause、battery pause 等更强策略覆盖时生效。

`--telemetry-csv` 输出同一批字段，供 `scripts/performance-snapshot.sh` 采样和
阈值判断。该脚本支持 RSS/PSS/private/Private_Dirty/USS/shared/NVIDIA process GPU
memory 上限，也支持 render sync cache、planned resource、renderer source footprint
和 adaptive action 断言。

## commands

### ping

```sh
gilderctl ping
```

探测 daemon 是否可用。

### outputs

```sh
gilderctl outputs
```

返回当前桌面输出快照，以及每个输出匹配到的壁纸、pause 状态和性能决策。

### set

```sh
gilderctl set ./wallpaper.gwpdir --output HDMI-A-1
gilderctl set ./wallpaper.gwpdir
```

设置指定输出或默认壁纸。`.gwp` 包会先解包到
`$XDG_CACHE_HOME/gilder/render-cache/`，再进入 render plan。

### pause / resume

```sh
gilderctl pause --output HDMI-A-1
gilderctl resume --output HDMI-A-1
gilderctl pause
gilderctl resume
```

设置输出级或全局 pause 状态。pause 会让对应输出进入 `removals`。

### properties

```sh
gilderctl properties get --output HDMI-A-1
gilderctl properties set speed 0.5 --output HDMI-A-1
gilderctl properties unset speed --output HDMI-A-1
```

读写 scene-lite 等动态壁纸的运行时属性。

### watch

```sh
gilderctl watch
```

订阅 daemon 事件。桌面快照、adaptive 状态或持久状态变化时会推送对应事件；
内部 renderer 同步只在 `render_sync` 实际变化时投递。
