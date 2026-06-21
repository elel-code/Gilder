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
`render_sync` 包含静态图片渲染器下一次同步需要执行的 `plans`、视频渲染器后续要消费的 `video_plans`、slideshow 渲染器要消费的 `slideshow_plans`、scene-lite 渲染器后续要消费的 `scene_lite_plans`、需要关闭的 `removals`、包加载/格式错误 `errors`，以及每个输出的 `decisions`。
视频壁纸有 poster 时，`video_plans[].poster` 会保留 fallback 路径；未启用 `video-renderer` 的构建会在 `plans` 里生成同一输出的静态 poster 占位计划。启用 native Vulkan video 路径时，poster 是 Vulkan importer/decode 失败或尚未接管输出时的 fallback，不再依赖 GTK surface。
`video_plans[].decoder_policy` 来自 `[video].decoder` 配置，当前可取 `auto`、`hardware-preferred`、`hardware-required` 或 `software`；视频 renderer 会在构建 GStreamer pipeline 前用这一字段调整已知 H.264/VP9/AV1 硬解/软解 decoder 的 feature rank，影响 decodebin 的 autoplug 选择。
`decisions` 会记录输出动作、当前壁纸路径和由桌面状态性能策略产生的 `mode/max_fps/reason`，native Vulkan renderer 和 GStreamer 前端会用它执行暂停、限帧或释放。`.gwp` 包会先解包到 `$XDG_CACHE_HOME/gilder/render-cache/`，再生成计划。
`[performance].battery` 和 `[outputs.<name>.performance].battery` 支持 `continue`、`throttle`、`pause` 和 `pause-dynamic`；`fullscreen` 和 `unfocused` 支持同一组值；`hidden` 和 `session` 支持 `continue`、`pause` 和 `pause-dynamic`。其中 `pause-dynamic` 会等待 manifest 加载完成，只对 video/slideshow/web/scene-lite/shader 生成 `paused`/`remove` 决策，静态壁纸保持原有桌面状态决策。
`slideshow_plans` 包含源图列表、切换间隔、transition、fit 和桌面状态策略合成后的 `target_max_fps`；当前计划层保留 slideshow/crossfade 语义，native Vulkan runtime 后续负责可见过渡。
`scene_lite_plans` 包含 scene-lite 文档路径、fallback、time=0 snapshot layer、当前可显示 surface 和桌面状态策略合成后的 `target_max_fps`；当前计划层会优先生成可显示 snapshot/fallback，真正的动画 scene surface 后续接入 native Vulkan runtime。
`--decisions-csv` 会把 `render_sync.decisions` 与同输出的静态/视频/slideshow/scene-lite 计划合并为 `output_name,action,mode,reason,max_fps,wallpaper,plan_kind,source,fit,target_max_fps,muted` CSV，便于性能采样脚本和人工对比 active/paused/fullscreen/battery 场景中的实际资源、fit、视频限帧和静音策略；`--from-file` 可以重放已经保存的 `gilderctl status` JSON-RPC 响应。
`telemetry` 会报告桌面快照刷新次数、read 请求复用快照次数、桌面变化次数、`render_sync` 缓存 hit/miss、渲染器同步更新 queued/skipped 计数、单次 package/archive/static-image cache 状态、静态缓存 byte footprint、计划层图片资源数量和文件字节 footprint、计划层视频 source 引用/去重/重复候选、package cache retained manifest 与 scene-lite 内部资源数量和文件字节 footprint，以及 `.gwp` 解包缓存累计/本轮淘汰计数。这里的计划层字节是源文件大小合计，用来定位“大图/大 poster/scene layer 是否仍被计划引用”；视频 source 重复字段用于评估同源多输出时可共享的 decoder/texture 候选，renderer 侧 `video_shared_runtimes` 会报告当前实际共享的 GStreamer/native Vulkan video runtime 数；package cache retained 字节是当前缓存住的包 manifest 所引用源文件/目录大小，用来定位“缓存还持有哪些大资源线索”，其中 retained preview 字段单独拆出 manifest preview thumbnail/poster。这些都不是解码后的纹理内存、renderer 内部缓存或 USS。`--telemetry-csv` 会输出 `desktop_refreshes,desktop_refresh_skips,desktop_changes,last_desktop_refresh_age_ms,render_sync_cache_hits,render_sync_cache_misses,render_sync_updates_queued,render_sync_updates_skipped,render_sync_package_cache_entries,render_sync_package_cache_max_entries,render_sync_package_cache_hits,render_sync_package_cache_misses,render_sync_package_cache_evictions,render_sync_archive_cache_entries,render_sync_archive_cache_max_entries,render_sync_archive_cache_reuses,render_sync_archive_cache_extractions,render_sync_archive_cache_evictions,render_sync_archive_cache_evictions_latest,render_sync_archive_cache_eviction_errors,render_sync_archive_cache_eviction_errors_latest,render_sync_planned_static_image_resources,render_sync_planned_video_poster_resources,render_sync_planned_slideshow_image_resources,render_sync_planned_image_resource_references,render_sync_planned_unique_image_resources,adaptive_refreshes,adaptive_refresh_skips,adaptive_active_triggers,cpu_pressure_some_avg10_x100,memory_pressure_some_avg10_x100,temperature_max_millicelsius,power_external_online,power_system_battery_present,power_battery_discharging,power_battery_capacity_percent,power_battery_power_microwatts,gpu_busy_percent_avg,gpu_busy_percent_max,gpu_busy_sources,adaptive_action_types,adaptive_action_scopes,adaptive_action_configured_actions,adaptive_action_max_fps,renderer_output_windows,renderer_static_surfaces,renderer_static_picture_surfaces,renderer_static_css_surfaces,renderer_static_color_surfaces,renderer_slideshow_surfaces,renderer_video_surfaces,renderer_video_shared_runtimes,renderer_video_pipelines,renderer_video_qos_messages,renderer_video_qos_dropped_max,renderer_video_gtk_frame_clock_ticks,renderer_video_gtk_frame_clock_interval_us_max,renderer_video_gtk_frame_clock_fps_x1000_max,renderer_video_gtk_frame_timings_complete,renderer_video_gtk_frame_timings_presentation_interval_us_max,renderer_video_gtk_frame_timings_presentation_time_us_max,renderer_video_gtk_frame_clock_before_paint_ticks,renderer_video_gtk_frame_clock_update_ticks,renderer_video_gtk_frame_clock_layout_ticks,renderer_video_gtk_frame_clock_paint_ticks,renderer_video_gtk_frame_clock_after_paint_ticks,render_sync_planned_static_image_resource_bytes,render_sync_planned_video_poster_resource_bytes,render_sync_planned_slideshow_image_resource_bytes,render_sync_planned_image_resource_reference_bytes,render_sync_planned_unique_image_resource_bytes,render_sync_package_cache_retained_resource_references,render_sync_package_cache_retained_unique_resources,render_sync_package_cache_retained_resource_bytes,render_sync_package_cache_retained_unique_resource_bytes,renderer_static_surface_resource_references,renderer_static_surface_resource_bytes,renderer_slideshow_resource_references,renderer_slideshow_resource_bytes,renderer_static_surface_unique_resources,renderer_static_surface_unique_resource_bytes,renderer_static_surface_estimated_decoded_bytes,renderer_slideshow_unique_resources,renderer_slideshow_unique_resource_bytes,render_sync_static_image_cache_entries,render_sync_static_image_cache_max_entries,render_sync_static_image_cache_generations,render_sync_static_image_cache_reuses,render_sync_static_image_cache_generation_errors,render_sync_static_image_cache_evictions,render_sync_static_image_cache_eviction_errors,render_sync_planned_video_source_references,render_sync_planned_unique_video_sources,render_sync_planned_duplicate_video_source_references,render_sync_planned_max_video_source_outputs,render_sync_planned_video_source_reference_bytes,render_sync_planned_unique_video_source_bytes,renderer_video_pipeline_source_references,renderer_video_pipeline_source_reference_bytes,renderer_video_pipeline_unique_sources,renderer_video_pipeline_unique_source_bytes,render_sync_package_cache_max_retained_unique_resource_bytes,render_sync_static_image_cache_bytes,render_sync_static_image_cache_max_bytes,render_sync_package_cache_retained_preview_resource_references,render_sync_package_cache_retained_unique_preview_resources,render_sync_package_cache_retained_preview_resource_bytes,render_sync_package_cache_retained_unique_preview_resource_bytes`，便于确认状态栏轮询和性能采样没有持续触发 compositor 适配器、重复生成渲染计划、无限保留旧 package/archive/static-image cache、重复投递未变化的渲染同步、计划层图片资源没有在暂停/隐藏场景继续被引用、识别同源多输出视频共享候选、确认当前 renderer video pipeline source 是否释放、执行错误的 adaptive 动作或隐藏视频帧行为异常。`renderer_video_gtk_*` 仍是兼容字段名，不表示当前可见路径依赖 GTK。
JSON telemetry 额外提供 `planned_scene_lite_image_resources`、`planned_scene_lite_image_resource_bytes`、`scene_lite_snapshot_cache_entries`、`scene_lite_snapshot_cache_bytes`、`scene_lite_snapshot_cache_generations`、`scene_lite_snapshot_cache_reuses` 和淘汰/错误计数；CSV 中的总 `planned_image_*` 字段已经包含 scene-lite snapshot 和 layer 图片。
`telemetry.render_sync.package_cache_max_retained_unique_resource_bytes` 会在 JSON status/watch、`--telemetry-csv`、performance summary 和 baseline matrix 中报告当前临时 package cache 的去重源资源 footprint 上限，方便把 retained byte 结果和预算放在同一份证据里对比。
`telemetry.render_sync.static_image_cache_bytes` 和 `static_image_cache_max_bytes` 报告运行时输出尺寸级静态图缓存当前 PNG 文件总量和配置上限；这是磁盘缓存 footprint，用来约束大图降采样缓存增长，不代表解码后的图片内存或 USS。
`scripts/performance-snapshot.sh` 可以用 `--expect-render-sync-cache-hit`、`--expect-desktop-refresh-skip`、`--expect-render-sync-update-queued`、`--expect-render-sync-update-skipped`、`--expect-render-sync-package-cache-entries-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-resource-references-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-unique-resources-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-package-cache-retained-unique-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-package-cache-retained-preview-resource-references-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-unique-preview-resources-latest-at-most <count>`、`--expect-render-sync-package-cache-retained-preview-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-package-cache-retained-unique-preview-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-planned-image-resource-references-latest-at-most <count>`、`--expect-render-sync-planned-unique-image-resources-latest-at-most <count>`、`--expect-render-sync-planned-image-resource-reference-bytes-latest-at-most <bytes>`、`--expect-render-sync-planned-unique-image-resource-bytes-latest-at-most <bytes>`、`--expect-render-sync-static-image-cache-bytes-latest-at-most <bytes>`、`--expect-renderer-video-pipeline-source-references-latest-at-most <count>`、`--expect-renderer-video-pipeline-source-reference-bytes-latest-at-most <bytes>`、`--expect-renderer-video-pipeline-unique-sources-latest-at-most <count>`、`--expect-renderer-video-pipeline-unique-source-bytes-latest-at-most <bytes>` 和 `--expect-adaptive-action <type>` 把这些 telemetry 变成失败条件，适合 CI 或真实会话 smoke 证明缓存、刷新节流、渲染器同步投递去重、package cache 上限、package cache retained footprint、preview thumbnail/poster retained footprint、静态图运行时缓存 footprint、计划层图片资源释放、运行时视频 pipeline source footprint 释放和 adaptive 动作仍然生效。
静态 surface 仍可以用 `--expect-renderer-static-picture-surfaces-*-at-most`、`--expect-renderer-static-css-surfaces-*-at-most`、`--expect-renderer-static-color-surfaces-*-at-most` 和 `--expect-renderer-static-surface-estimated-decoded-bytes-*-at-most` 约束历史 telemetry 字段；`*` 当前支持 `latest` 或 `max`。这些字段后续会随 native Vulkan renderer 接入重新校准。
启用 `[adaptive]` 或单个 `[outputs.<name>.adaptive]` 时，`telemetry.adaptive.snapshot` 会报告 Linux PSI CPU/内存压力、thermal zone 最高温度、power_supply AC/电池细节、DRM `gpu_busy_percent` 统计、`active_triggers` 和 kill switch 状态。触发项覆盖 CPU pressure、memory pressure、temperature、GPU busy 和放电时低电量；阈值为 0 可关闭单项触发。`telemetry.adaptive.action` 会列出当前 adaptive 动作。默认 `action = "throttle"` 会报告 `type: "throttle"` 和 `max_fps`；`action = "pause-unfocused"` 在输出非焦点时报告 `type: "pause-unfocused"` 并暂停该输出，输出仍聚焦或缺少桌面输出状态时回退为 throttle，并在 action report 中保留 `configured_action`；`action = "pause-dynamic"` 会报告 `type: "pause-dynamic"` 和 `scope: "dynamic-wallpapers"`，渲染计划加载 manifest 后只暂停 video/slideshow/web/scene-lite/shader，静态图片保持原有桌面状态决策。adaptive 决策原因是 `adaptive`；用户暂停、fullscreen pause、battery pause 等更强策略不会被覆盖。
`telemetry.renderer` 会报告当前 renderer 持有的 output surface/window、静态 surface、slideshow surface、video surface 数和共享 video runtime 数，以及当前 static surface、slideshow surface 和 video pipeline 引用的源文件引用数、去重资源数和源文件字节 footprint；后者是 renderer 仍持有大图/幻灯片/视频源资源的线索，不是解码纹理内存、GStreamer buffer 或 USS。`static_picture_surfaces`、`static_css_surfaces`、`static_color_surfaces`、`static_surface_estimated_decoded_bytes` 是历史兼容字段，native Vulkan 接管后会重新校准。它也会把当前 `renderer_runtime.video_pipelines[].frame_stats` 聚合为视频 pipeline 数、QoS 消息总数、最大 dropped 计数和兼容 frame timing 计数。这是 daemon 级采样入口，方便状态栏、性能脚本和 smoke 快速判断输出资源是否释放、视频帧行为是否在变化。逐输出诊断仍应查看 `renderer_runtime.video_pipelines` 或 `--video-runtime-csv`。
renderer tick 会按当前负载动态调度：video runtime 单独存在时只做必要的 bus/EOS/error/QoS polling 和低频 runtime snapshot；纯静态无动态工作时不应安装高频 runtime timeout。renderer runtime snapshot 只会在 render sync 变化、slideshow 实际换帧、decoder/import 观察结果变化或 pipeline 报错时完整刷新；frame stats 到期判断只读取 runtime 计数，不会重新计算 surface/source footprint；同一次 snapshot 内，共享 video runtime 的 decoder/caps/allocation、position 和 duration 查询只做一次，再展开为逐输出 `video_pipelines`；resource footprint 内部按路径缓存 source size，重复引用仍按引用次数累计 bytes，但每个唯一路径只读取一次 metadata。
daemon 会周期刷新桌面快照，只有快照变化时才发送 `desktop.changed` 事件；adaptive 触发状态改变但桌面快照未变化时会发送 `adaptive.changed` 事件；`status` 和 `outputs` 会按 `performance.desktop_refresh_interval_ms` 复用最近的桌面快照，并按 `adaptive.refresh_interval_ms` 复用最近的 adaptive 采样，避免轮询客户端过于频繁地调用 compositor 适配器或读取系统压力文件；`desktop.changed`、`adaptive.changed` 和 `state.changed` 会继续携带当前 `render_sync` 供客户端观察，但只有 `render_sync` 实际变化时才投递给内部渲染器。
daemon 会缓存最近一次 `render_sync`；当渲染相关配置、渲染相关状态、桌面快照和已引用壁纸包元数据未变化时，重复 `status` 会复用缓存，减少性能采样和状态栏轮询中的 manifest IO。当前 properties、adapter 开关和桌面刷新周期不参与静态/视频渲染计划，因此不会单独让缓存失效。
启用 native Vulkan renderer 的 daemon 会通过 native Wayland host 管理 layer-shell surface；`native-vulkan-gst-video` 使用 GStreamer appsink 前端提供 decoder/caps/sample evidence，最终显示由 native Vulkan importer/render pass/present 完成。
`renderer_capabilities.video.gstreamer.elements` 会报告 GStreamer 基础元素和 decoder 可用性；真实视频 surface 不再要求 `gtk4paintablesink`。GStreamer sink 不接管显示。
`renderer_runtime` 顶层会报告 `output_windows`、`static_surfaces`、`slideshow_surfaces`、`video_surfaces` 和 `video_shared_runtimes`；`renderer_runtime.video_pipelines` 会报告当前 daemon 内部渲染器实际持有的视频输出快照，包括输出、源文件、共享 GStreamer state、运行模式、限帧、静音状态、播放 `position_ms`/`duration_ms`、实际 `frame_limiter_enabled`/`frame_limiter_max_fps`、`sink_tuning`、`frame_stats`、`decoder_policy`、`decoder_policy_status`、`actual_decoders`、`caps_reports`、`allocation_reports`、`queue_reports`、`zero_copy_evidence`、`memory_path` 和 `retention_report`。`sink_tuning` 与 `gtk_frame_*` 字段暂时保留为兼容 telemetry 名称；native Vulkan 路径下它们通常为空或 0。`actual_decoders` 来自运行中 pipeline 的 GStreamer element factory 名称，例如 `avdec_h264`、`dav1ddec`、`vp9dec`、`vah264dec` 或 `vaav1dec`；`actual_decoder_reports` 会给每个实际 decoder 标记 `hardware`、`software` 或 `unknown`。
这些 decoder/caps/allocation/memory path 诊断是慢变运行时证据，renderer 内部会按 video runtime 缓存并低频刷新；`frame_stats` 会按固定间隔写回最近的 runtime snapshot，播放位置和 duration 随完整 runtime snapshot 更新。这样状态栏轮询或 video frontend polling 不会持续触发 GStreamer pipeline 遍历和 allocation query。
`caps_reports` 记录运行中 pad 的 negotiated caps：优先使用 `current_caps()`，没有时回退到 pad 上的 sticky CAPS event，并额外合并 runtime caps-event observer 捕获到的 caps。`source` 会标记 `current`、`sticky`、`observer-initial` 或 `caps-event`；其中 `caps-event` 表示 Gilder 在 GStreamer 前端流动过程中观测到了 negotiated caps，比只做晚期静态 snapshot 更强。报告包括 element、pad、方向、caps 字符串、media type、所有 caps features、raw video `format`、width/height，以及聚合后的 `memory_features`。其中 `memory:DMABuf`、`memory:GLMemory`、`memory:CUDAMemory` 等值可作为后续 zero-copy/import 验证线索；`format`、`formats`、`sink_formats` 和 `format_paths` 用于判断 4K/高刷视频是否能把 NV12/I420/P010 等 YUV 格式保持到 appsink/importer，还是在 presentation 前已经变成 RGBA/RGBx。空 `caps_reports` 通常表示 pipeline 尚未协商到视频 caps，不代表已经或没有走 GPU/zero-copy 路径。`memory_path.level` 会把当前证据分为 `unknown`、`cpu-raw-caps`、`software-decode-no-caps`、`software-decode-cpu-raw`、`hardware-decode-no-caps`、`hardware-decode-cpu-raw`、`decoder-gpu-memory`、`decoder-dmabuf`、`sink-gpu-memory` 或 `sink-dmabuf`，用于直接区分硬解后仍落到 CPU raw frame、decoder 侧 GPU/DMABuf、appsink/importer 侧 GPU/DMABuf 等路径。`allocation_reports` 会从已协商视频 src pad 向 downstream peer 发起 allocation query，记录响应的 buffer pool、allocator 参数和 meta，用来继续分析 GStreamer buffer pool 是否保留 CPU-side frame 或是否协商到 DMABuf allocator。`retention_report` 会把 `memory_path`、`allocation_reports` 和兼容 sink/import tuning 线索合并成 `unknown`、`low`、`medium` 或 `high` 风险等级，并报告估算的 allocation pool 最小/最大 buffer 容量、system-memory/GPU/DMABuf pool 计数、历史 sink frame retention 字段和 notes；`high` 通常表示已观察到 CPU raw caps、system-memory pool 或 retained frame，`medium` 表示 decoder 侧 GPU/DMABuf 尚未证明到达 importer 或存在 pool 容量需要与 PSS/USS 对齐。`zero_copy_evidence.level` 会把已有 decoder/caps 线索分为 `missing`、`software-decode`、`hardware-decode`、`gpu-memory-caps`、`dmabuf-caps`、`sink-gpu-memory-caps` 或 `sink-dmabuf-caps`；这是运行时证据分级，不是 Wayland compositor presentation 或完整 zero-copy 证明。
`--video-runtime-csv` 会把这些运行时证据整理为 `output_name,mode,gst_state,decoder_policy,decoder_policy_status,actual_decoders,decoder_classes,caps_report_count,memory_features,sink_memory_features,zero_copy_evidence_level,memory_path_level,...` 等字段，便于本机 smoke 和性能采样把 decoder、caps/import memory features、zero-copy 证据分级、YUV/RGBA format 路径、caps 来源、内存路径、allocator/buffer-pool 协商、queue 深度、播放进度、实际限帧状态、retained frame/pool 风险、QoS dropped 统计、兼容 frame timing 与 CPU/PSS/USS/RSS 结果放在同一证据目录中。完整 CSV header 由 `gilderctl status --video-runtime-csv` 输出；`gtk_frame_*` 和 `sink_*` 名称是兼容字段。
`scripts/performance-snapshot.sh` 可以用 `--expect-max-rss-kib-at-most`、`--expect-max-pss-kib-at-most`、`--expect-max-private-kib-at-most`、`--expect-max-private-dirty-kib-at-most`、`--expect-max-uss-kib-at-most`、`--expect-max-shared-kib-at-most` 和 NVIDIA 主机上的 `--expect-max-nvidia-process-gpu-memory-mib-at-most` 把进程内存/显存预算变成失败条件；其中 PSS、USS/private 更适合判断 Gilder 自身的私有占用，`Private_Dirty` 往往更接近桌面监控器显示的小“应用内存”口径，RSS 只适合作为包含共享映射的补充信号。`summary.txt` 还会为 RSS/PSS/private-clean/private-dirty/private/USS/shared 输出 `first_*_kib`、`last_*_kib`、`retained_*_delta_kib` 和 `peak_over_first_*_kib`，并在可用时输出 `first/avg/last/max_nvidia_process_gpu_memory_mib`。retained delta 是采样窗口最后一个样本减第一个样本，用来观察 paused、hidden、fullscreen 或 resumed 场景结束时是否仍保留额外私有占用，peak-over-first 则单独记录短时峰值。脚本还会输出 `memory-mapping-summary.txt`，按 `/proc/<pid>/smaps` 聚合 top PSS mapping 和 `nvidia-device`、`anonymous`、`heap`、`gstreamer-library` 等粗分类，用于解释 PSS/USS 与监控器口径或 NVIDIA 显存口径之间的差异。`--expect-retained-*-delta-kib-at-most` 和 `--expect-peak-over-first-*-kib-at-most` 可以把这些相对内存变化变成失败条件，适合给 active->paused/hidden/fullscreen/resumed 场景设置回归门槛。它也可以用 `--expect-renderer-output-windows-*`、`--expect-renderer-static-surfaces-*`、`--expect-renderer-slideshow-surfaces-*`、`--expect-renderer-video-surfaces-*`、`--expect-renderer-video-pipelines-*` 和 `--expect-renderer-video-pipeline-*-latest-at-most` 系列断言验证 renderer telemetry 中的 output surface/window、static/slideshow/video surface、源文件 footprint、视频 pipeline 和运行时视频 source footprint 是否按桌面状态创建或释放。它也可以用 decoder/caps/memory/zero-copy/playback/QoS expectation 验证 GStreamer 前端和 native Vulkan importer 证据。`--expect-gtk-frame-*` 系列是兼容旧字段的 legacy gate，当前 native Vulkan presentation 应优先用 native Vulkan smoke/runtime telemetry 验证。

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

`snapshot`、`desktop.changed` 和 `state.changed` 都包含 `desktop`、`render_sync`、`renderer_capabilities` 和 `renderer_runtime`，GUI 前端和 daemon 内部渲染器可以用它判断每个输出是否已有可应用的静态或视频壁纸计划、当前运行时是否具备 native Wayland/Vulkan/GStreamer 能力，以及运行中视频 pipeline 实际选中的 decoder。
`renderer` 会随 feature 组合返回 native Vulkan/native Wayland host/GStreamer video 相关名称，或在未启用渲染 feature 时返回 `not-implemented`。

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
