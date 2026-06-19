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
gilderctl status --video-runtime-csv
gilderctl status --decisions-csv --from-file status-001.json
gilderctl status --telemetry-csv --from-file status-001.json
gilderctl status --video-runtime-csv --from-file status-001.json
```

返回 daemon 状态、桌面快照、输出列表、当前壁纸、暂停状态、配置/状态文件位置、性能决策信息、renderer 能力诊断、daemon telemetry 和 `render_sync`。
`render_sync` 包含静态图片渲染器下一次同步需要执行的 `plans`、视频渲染器后续要消费的 `video_plans`、slideshow 渲染器要消费的 `slideshow_plans`、需要关闭的 `removals`、包加载/格式错误 `errors`，以及每个输出的 `decisions`。
视频壁纸有 poster 时，`plans` 会包含同一输出的静态 poster 占位计划，`video_plans` 仍包含实际视频 pipeline 计划。
`video_plans[].decoder_policy` 来自 `[video].decoder` 配置，当前可取 `auto`、`hardware-preferred`、`hardware-required` 或 `software`；视频 renderer 会在构建 GStreamer pipeline 前用这一字段调整已知 H.264/VP9/AV1 硬解/软解 decoder 的 feature rank，影响 decodebin/playbin 的 autoplug 选择。
`decisions` 会记录输出动作、当前壁纸路径和由桌面状态性能策略产生的 `mode/max_fps/reason`，视频/GStreamer 渲染器会用它执行暂停或限帧。`.gwp` 包会先解包到 `$XDG_CACHE_HOME/gilder/render-cache/`，再生成计划。
`[performance].battery` 和 `[outputs.<name>.performance].battery` 支持 `continue`、`throttle`、`pause` 和 `pause-dynamic`；`fullscreen` 和 `unfocused` 支持同一组值；`hidden` 和 `session` 支持 `continue`、`pause` 和 `pause-dynamic`。其中 `pause-dynamic` 会等待 manifest 加载完成，只对 video/slideshow 生成 `paused`/`remove` 决策，静态壁纸保持原有桌面状态决策。
`slideshow_plans` 包含源图列表、切换间隔、transition、fit 和桌面状态策略合成后的 `target_max_fps`；当前 GTK renderer 会按间隔切换图片，`crossfade` 先作为格式字段保留。
`--decisions-csv` 会把 `render_sync.decisions` 与同输出的静态/视频/slideshow 计划合并为 `output_name,action,mode,reason,max_fps,wallpaper,plan_kind,source,fit,target_max_fps,muted` CSV，便于性能采样脚本和人工对比 active/paused/fullscreen/battery 场景中的实际资源、fit、视频限帧和静音策略；`--from-file` 可以重放已经保存的 `gilderctl status` JSON-RPC 响应。
`telemetry` 会报告桌面快照刷新次数、read 请求复用快照次数、桌面变化次数、`render_sync` 缓存 hit/miss、渲染器同步更新 queued/skipped 计数、单次 package/archive/static-image cache 状态、计划层图片资源数量和文件字节 footprint、计划层视频 source 引用/去重/重复候选、package cache retained manifest 资源数量和文件字节 footprint，以及 `.gwp` 解包缓存累计/本轮淘汰计数。这里的计划层字节是源文件大小合计，用来定位“大图/大 poster 是否仍被计划引用”；视频 source 重复字段用于评估同源多输出时理论上可共享的 decoder/texture 候选，不表示当前已实际共享；package cache retained 字节是当前缓存住的包 manifest 所引用源文件/目录大小，用来定位“缓存还持有哪些大资源线索”。这些都不是解码后的纹理内存、GTK 内部缓存或 USS。`--telemetry-csv` 会输出 `desktop_refreshes,desktop_refresh_skips,desktop_changes,last_desktop_refresh_age_ms,render_sync_cache_hits,render_sync_cache_misses,render_sync_updates_queued,render_sync_updates_skipped,render_sync_package_cache_entries,render_sync_package_cache_max_entries,render_sync_package_cache_hits,render_sync_package_cache_misses,render_sync_package_cache_evictions,render_sync_archive_cache_entries,render_sync_archive_cache_max_entries,render_sync_archive_cache_reuses,render_sync_archive_cache_extractions,render_sync_archive_cache_evictions,render_sync_archive_cache_evictions_latest,render_sync_archive_cache_eviction_errors,render_sync_archive_cache_eviction_errors_latest,render_sync_planned_static_image_resources,render_sync_planned_video_poster_resources,render_sync_planned_slideshow_image_resources,render_sync_planned_image_resource_references,render_sync_planned_unique_image_resources,adaptive_refreshes,adaptive_refresh_skips,adaptive_active_triggers,cpu_pressure_some_avg10_x100,memory_pressure_some_avg10_x100,temperature_max_millicelsius,power_external_online,power_system_battery_present,power_battery_discharging,power_battery_capacity_percent,power_battery_power_microwatts,gpu_busy_percent_avg,gpu_busy_percent_max,gpu_busy_sources,adaptive_action_types,adaptive_action_scopes,adaptive_action_configured_actions,adaptive_action_max_fps,renderer_output_windows,renderer_static_surfaces,renderer_slideshow_surfaces,renderer_video_surfaces,renderer_video_pipelines,renderer_video_qos_messages,renderer_video_qos_dropped_max,renderer_video_gtk_frame_clock_ticks,renderer_video_gtk_frame_clock_interval_us_max,renderer_video_gtk_frame_clock_fps_x1000_max,renderer_video_gtk_frame_timings_complete,renderer_video_gtk_frame_timings_presentation_interval_us_max,renderer_video_gtk_frame_timings_presentation_time_us_max,renderer_video_gtk_frame_clock_before_paint_ticks,renderer_video_gtk_frame_clock_update_ticks,renderer_video_gtk_frame_clock_layout_ticks,renderer_video_gtk_frame_clock_paint_ticks,renderer_video_gtk_frame_clock_after_paint_ticks,render_sync_planned_static_image_resource_bytes,render_sync_planned_video_poster_resource_bytes,render_sync_planned_slideshow_image_resource_bytes,render_sync_planned_image_resource_reference_bytes,render_sync_planned_unique_image_resource_bytes,render_sync_package_cache_retained_resource_references,render_sync_package_cache_retained_unique_resources,render_sync_package_cache_retained_resource_bytes,render_sync_package_cache_retained_unique_resource_bytes,renderer_static_surface_resource_references,renderer_static_surface_resource_bytes,renderer_slideshow_resource_references,renderer_slideshow_resource_bytes,renderer_static_surface_unique_resources,renderer_static_surface_unique_resource_bytes,renderer_slideshow_unique_resources,renderer_slideshow_unique_resource_bytes,render_sync_static_image_cache_entries,render_sync_static_image_cache_max_entries,render_sync_static_image_cache_generations,render_sync_static_image_cache_reuses,render_sync_static_image_cache_generation_errors,render_sync_static_image_cache_evictions,render_sync_static_image_cache_eviction_errors,render_sync_planned_video_source_references,render_sync_planned_unique_video_sources,render_sync_planned_duplicate_video_source_references,render_sync_planned_max_video_source_outputs,render_sync_planned_video_source_reference_bytes,render_sync_planned_unique_video_source_bytes,renderer_video_pipeline_source_references,renderer_video_pipeline_source_reference_bytes,renderer_video_pipeline_unique_sources,renderer_video_pipeline_unique_source_bytes`，便于确认状态栏轮询和性能采样没有持续触发 compositor 适配器、重复生成渲染计划、无限保留旧 package/archive/static-image cache、重复投递未变化的渲染同步、计划层图片资源没有在暂停/隐藏场景继续被引用、识别同源多输出视频共享候选、确认当前 renderer video pipeline source 是否释放、执行错误的 adaptive 动作或隐藏视频帧行为异常。
`scripts/performance-snapshot.sh` 可以用 `--expect-render-sync-cache-hit`、`--expect-desktop-refresh-skip`、`--expect-render-sync-update-queued`、`--expect-render-sync-update-skipped`、`--expect-render-sync-package-cache-entries-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-resource-references-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-unique-resources-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-package-cache-retained-unique-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-planned-image-resource-references-latest-at-most <count>`、`--expect-render-sync-planned-unique-image-resources-latest-at-most <count>`、`--expect-render-sync-planned-image-resource-reference-bytes-latest-at-most <bytes>`、`--expect-render-sync-planned-unique-image-resource-bytes-latest-at-most <bytes>`、`--expect-renderer-video-pipeline-source-references-latest-at-most <count>`、`--expect-renderer-video-pipeline-source-reference-bytes-latest-at-most <bytes>`、`--expect-renderer-video-pipeline-unique-sources-latest-at-most <count>`、`--expect-renderer-video-pipeline-unique-source-bytes-latest-at-most <bytes>` 和 `--expect-adaptive-action <type>` 把这些 telemetry 变成失败条件，适合 CI 或真实会话 smoke 证明缓存、刷新节流、渲染器同步投递去重、package cache 上限、package cache retained footprint、计划层图片资源释放、运行时视频 pipeline source footprint 释放和 adaptive 动作仍然生效。
启用 `[adaptive]` 或单个 `[outputs.<name>.adaptive]` 时，`telemetry.adaptive.snapshot` 会报告 Linux PSI CPU/内存压力、thermal zone 最高温度、power_supply AC/电池细节、DRM `gpu_busy_percent` 统计、`active_triggers` 和 kill switch 状态。触发项覆盖 CPU pressure、memory pressure、temperature、GPU busy 和放电时低电量；阈值为 0 可关闭单项触发。`telemetry.adaptive.action` 会列出当前 adaptive 动作。默认 `action = "throttle"` 会报告 `type: "throttle"` 和 `max_fps`；`action = "pause-unfocused"` 在输出非焦点时报告 `type: "pause-unfocused"` 并暂停该输出，输出仍聚焦或缺少桌面输出状态时回退为 throttle，并在 action report 中保留 `configured_action`；`action = "pause-dynamic"` 会报告 `type: "pause-dynamic"` 和 `scope: "dynamic-wallpapers"`，渲染计划加载 manifest 后只暂停 video/slideshow，静态图片保持原有桌面状态决策。adaptive 决策原因是 `adaptive`；用户暂停、fullscreen pause、battery pause 等更强策略不会被覆盖。
`telemetry.renderer` 会报告当前 GTK renderer 持有的 output window、静态 background surface、slideshow surface 和 video surface 数，以及当前 static CSS provider、slideshow surface 和 video pipeline 引用的源文件引用数、去重资源数和源文件字节 footprint；后者是 GTK renderer 仍持有大图/幻灯片/视频源资源的线索，不是解码纹理内存、GStreamer buffer 或 USS。它也会把当前 `renderer_runtime.video_pipelines[].frame_stats` 聚合为视频 pipeline 数、QoS 消息总数、最大 dropped 计数、GTK frame clock tick 总数、before-paint/update/layout/paint/after-paint phase tick 总数、最大 frame interval、最大观察 FPS、GDK frame timings 完成数和 presentation interval/time 线索。这是 daemon 级采样入口，方便状态栏、性能脚本和 smoke 快速判断输出资源是否释放、视频帧行为是否在变化；逐输出诊断仍应查看 `renderer_runtime.video_pipelines` 或 `--video-runtime-csv`。GTK video surface 成功接管输出后会释放实际 poster/static CSS surface，因此 active video 的 `render_sync` 仍可能报告 planned poster resource，但 `renderer.static_surfaces` 的最新值应回到 0；pipeline 构建失败或后续报错时会按保留的 poster plan 恢复 fallback surface。
daemon 会周期刷新桌面快照，只有快照变化时才发送 `desktop.changed` 事件；adaptive 触发状态改变但桌面快照未变化时会发送 `adaptive.changed` 事件；`status` 和 `outputs` 会按 `performance.desktop_refresh_interval_ms` 复用最近的桌面快照，并按 `adaptive.refresh_interval_ms` 复用最近的 adaptive 采样，避免轮询客户端过于频繁地调用 compositor 适配器或读取系统压力文件；`desktop.changed`、`adaptive.changed` 和 `state.changed` 会继续携带当前 `render_sync` 供客户端观察，但只有 `render_sync` 实际变化时才投递给内部渲染器。
daemon 会缓存最近一次 `render_sync`；当渲染相关配置、渲染相关状态、桌面快照和已引用壁纸包元数据未变化时，重复 `status` 会复用缓存，减少性能采样和状态栏轮询中的 manifest IO。当前 properties、adapter 开关和桌面刷新周期不参与静态/视频渲染计划，因此不会单独让缓存失效。
启用 `gtk-renderer` feature 的 daemon 会在 GTK 主线程消费同一份 `render_sync`，并把可用输出同步到 layer-shell background 窗口。
同时启用 `gtk-renderer` 和 `video-renderer` 时，GTK 主线程会尝试用 `gtk4paintablesink` 把视频 paintable 放入对应输出的 layer-shell 窗口；只启用 `video-renderer` 时，daemon 会启动 headless GStreamer worker 消费 `video_plans`，负责视频 pipeline 生命周期控制。
`renderer_capabilities.video.gstreamer.elements` 会报告 `playbin`、`fakesink`、`videorate`、`capsfilter` 和 `gtk4paintablesink` 是否可用；真实视频 surface 显示需要 `gtk4paintablesink` 为 `available: true`。
`renderer_runtime` 顶层会报告 `output_windows`、`static_surfaces`、`slideshow_surfaces` 和 `video_surfaces`；`renderer_runtime.video_pipelines` 会报告当前 daemon 内部渲染器实际持有的视频 pipeline 快照，包括输出、源文件、GStreamer state、运行模式、限帧、静音状态、播放 `position_ms`/`duration_ms`、实际 `frame_limiter_enabled`/`frame_limiter_max_fps`、`frame_stats`、`decoder_policy`、`decoder_policy_status`、`actual_decoders`、`caps_reports` 和 `zero_copy_evidence`。`frame_stats` 包含 GStreamer QoS 计数，以及 GTK surface 路径下被动观察到的 `gtk_frame_clock_*` after-paint 计数、before-paint/update/layout/paint/after-paint phase 计数、frame counter、frame time、interval、FPS 和预测 presentation time 线索；它也会记录 GDK `FrameTimings` 的 observed/complete 计数、frame time、predicted presentation time、presentation time、presentation interval 和 refresh interval。headless GStreamer worker 没有 GTK frame clock，因此这些 GTK/GDK 字段通常为 0 或空。`actual_decoders` 来自运行中 pipeline 的 GStreamer element factory 名称，例如 `avdec_h264`、`dav1ddec`、`vp9dec`、`vah264dec` 或 `vaav1dec`；`actual_decoder_reports` 会给每个实际 decoder 标记 `hardware`、`software` 或 `unknown`，用于区分当前播放路径实际是软解、硬解还是尚未完成 autoplug。空数组表示当前没有识别到 decoder element，常见于 pipeline 尚未 preroll、暂停在早期状态、没有视频 pipeline 或 GStreamer 选择了未列入诊断白名单的 decoder。`decoder_policy_status` 会报告 `not-applicable`、`not-observed`、`satisfied`、`software-fallback`、`violated` 或 `unknown-decoder`，用于判断当前实际 decoder 是否满足策略。
`caps_reports` 只记录运行中 pad 的 negotiated `current_caps()`，包括 element、pad、方向、caps 字符串、media type、所有 caps features，以及聚合后的 `memory_features`。其中 `memory:DMABuf`、`memory:GLMemory` 等值可作为后续 zero-copy 验证线索；空 `caps_reports` 通常表示 pipeline 尚未协商到视频 caps，不代表已经或没有走 GPU/zero-copy 路径。`zero_copy_evidence.level` 会把已有 decoder/caps 线索分为 `missing`、`software-decode`、`hardware-decode`、`gpu-memory-caps`、`dmabuf-caps`、`sink-gpu-memory-caps` 或 `sink-dmabuf-caps`；这是运行时证据分级，不是 Wayland compositor presentation 或完整 zero-copy 证明。
`--video-runtime-csv` 会把这些运行时证据整理为 `output_name,mode,gst_state,decoder_policy,decoder_policy_status,actual_decoders,decoder_classes,caps_report_count,memory_features,sink_memory_features,zero_copy_evidence_level,zero_copy_evidence_notes,media_types,caps_paths,position_ms,duration_ms,frame_limiter_enabled,frame_limiter_max_fps,qos_messages,qos_processed_max,qos_dropped_max,qos_stats_format,qos_jitter_ns_latest,qos_jitter_ns_abs_max,qos_proportion_x1000_latest,gtk_frame_clock_ticks,gtk_frame_clock_counter_latest,gtk_frame_clock_time_us_latest,gtk_frame_clock_interval_us_latest,gtk_frame_clock_interval_us_max,gtk_frame_clock_fps_x1000_latest,gtk_frame_clock_refresh_interval_us_latest,gtk_frame_clock_predicted_presentation_time_us_latest,gtk_frame_timings_observed,gtk_frame_timings_complete,gtk_frame_timings_counter_latest,gtk_frame_timings_complete_counter_latest,gtk_frame_timings_frame_time_us_latest,gtk_frame_timings_predicted_presentation_time_us_latest,gtk_frame_timings_presentation_time_us_latest,gtk_frame_timings_presentation_interval_us_latest,gtk_frame_timings_presentation_interval_us_max,gtk_frame_timings_refresh_interval_us_latest,source,gtk_frame_clock_before_paint_ticks,gtk_frame_clock_update_ticks,gtk_frame_clock_layout_ticks,gtk_frame_clock_paint_ticks,gtk_frame_clock_after_paint_ticks`，便于本机 smoke 和性能采样把 decoder、sink caps/memory features、zero-copy 证据分级、播放进度、实际限帧状态、QoS dropped 统计、GTK/GDK frame timing 与 CPU/PSS/USS/RSS 结果放在同一证据目录中。
`scripts/performance-snapshot.sh` 可以用 `--expect-max-rss-kib-at-most`、`--expect-max-pss-kib-at-most`、`--expect-max-private-kib-at-most`、`--expect-max-uss-kib-at-most` 和 `--expect-max-shared-kib-at-most` 把进程内存预算变成失败条件；其中 PSS、USS/private 更适合判断 Gilder 自身的私有占用，RSS 只适合作为包含共享映射的补充信号。`summary.txt` 还会为 RSS/PSS/private/USS/shared 输出 `first_*_kib`、`last_*_kib`、`retained_*_delta_kib` 和 `peak_over_first_*_kib`；retained delta 是采样窗口最后一个样本减第一个样本，用来观察 paused、hidden、fullscreen 或 resumed 场景结束时是否仍保留额外私有占用，peak-over-first 则单独记录短时峰值。`--expect-retained-*-delta-kib-at-most` 和 `--expect-peak-over-first-*-kib-at-most` 可以把这些相对内存变化变成失败条件，适合给 active->paused/hidden/fullscreen/resumed 场景设置回归门槛。它也可以用 `--expect-renderer-output-windows-*`、`--expect-renderer-static-surfaces-*`、`--expect-renderer-slideshow-surfaces-*`、`--expect-renderer-static-surface-resource-*-latest-at-most`、`--expect-renderer-static-surface-unique-*-latest-at-most`、`--expect-renderer-slideshow-resource-*-latest-at-most`、`--expect-renderer-slideshow-unique-*-latest-at-most`、`--expect-renderer-video-surfaces-*`、`--expect-renderer-video-pipelines-*` 和 `--expect-renderer-video-pipeline-*-latest-at-most` 系列断言验证 renderer telemetry 中的 GTK output window、static/slideshow/video surface、源文件 footprint、视频 pipeline 和运行时视频 source footprint 是否按桌面状态创建或释放。它也可以用 `--expect-decoder-policy-status`、`--expect-decoder-class`、`--expect-memory-feature`、`--expect-sink-memory-feature`、`--expect-zero-copy-evidence`、`--expect-video-position-progress`、`--expect-frame-limiter-enabled`、`--expect-frame-limiter-max-fps`、`--expect-video-qos`、`--expect-qos-dropped-max-at-most`、`--expect-gtk-frame-clock`、`--expect-gtk-frame-clock-phase <phase>` 和 `--expect-gtk-frame-timings` 把 `video-runtime.csv` 中的 decoder/caps/playback/QoS/GTK frame clock/GDK timing 证据变成失败条件，适合真实 Wayland 会话里区分“已观察到硬解 decoder”、“sink caps 暴露 DMABuf/GLMemory 线索”、“zero-copy 证据分级达到预期”、“播放进度确实推进”、“限帧已作用到 pipeline”、“QoS 没有报告超过阈值的 dropped 单位”、“GTK surface frame clock 确实在 tick”、“GTK frame clock 进入了指定 phase”、“GDK frame timings 确实完成”和“仍缺少 zero-copy 证据”。

### outputs

```sh
gilderctl outputs
```

返回 daemon 当前知道的桌面快照和输出列表。列表会合并持久化状态中的输出和合成器适配器提供的输出。Hyprland session 下读取 `hyprctl -j monitors/clients`；niri session 下读取 `niri msg --json outputs/workspaces/windows`；不可用时回退到 generic snapshot。
验证时可以用 `GILDER_DESKTOP_OUTPUTS=eDP-1,HDMI-A-1:1920x1080@1.5` 构造虚拟输出列表，再叠加 `GILDER_OUTPUT_STATE`、`GILDER_POWER_STATE`、`GILDER_SESSION_STATE` 和 `GILDER_ADAPTIVE_STATE=inactive|cpu-pressure|memory-pressure|temperature|gpu-busy|low-battery|all` 采集 headless 性能策略证据。

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
