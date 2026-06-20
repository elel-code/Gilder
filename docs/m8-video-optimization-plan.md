# M8 Video Optimization Plan

本文档记录 M8/T0 后续视频优化的实验顺序。目标不是先改大架构，而是先把
4K/240fps direct `gtk4paintablesink` 的当前突破变成可重复对照的底线，再逐步确认
zero-copy、YUV/NV12、队列和 fullscreen/game auto-suspend 哪些能继续压低 CPU、内存和显存。

## Current Baseline

当前本机 NVIDIA/niri 4K/240fps H.264 active GTK video surface 基线：

- 默认 direct `gtk4paintablesink` 路径选择 `nvh264dec`。
- 6s guardrail sample：平均 75.52% process CPU，按 20 逻辑 CPU 折算约 3.8% 整机 CPU。
- 峰值 `Private_Dirty` 109220 KiB，PSS/USS 387821/353660 KiB。
- NVIDIA 进程显存约 496 MiB。
- 该 4K/240 样本采自 caps-event observer 之前；当时 zero-copy 证据仍只是
  `hardware-decode`，sink/caps 仍不能证明 GPU memory/DMABuf 到达 GTK surface。

2026-06-20 证据链更新：

- `caps_reports` 现在会合并 `current_caps()`、sticky CAPS 和 runtime caps-event probe。
- 真实 niri GTK/playbin smoke 已能观察到 49 条 caps report，`caps_sources` 包含
  `caps-event|current|observer-initial|sticky`。
- 该小视频 smoke 选择 `nvh264dec`，sink-side format 为 `NV12`，sink memory features 为
  `memory:CUDAMemory|memory:SystemMemory`，zero-copy evidence 达到 `sink-gpu-memory-caps`。
- 4K/240fps 1s generated loop、6s sample 复测也达到同样证据等级：`nvh264dec`、
  `formats=NV12`、`sink_formats=NV12`、`zero_copy_evidence=sink-gpu-memory-caps`、
  `memory_path=sink-gpu-memory`。
- 这证明 direct GTK/playbin 证据链已经能穿透到 4K/240 sink 侧 GPU memory caps；仍未证明
  Wayland compositor presentation 层面的 full zero-copy。
- 2026-06-20 queue 梯度后，默认 GStreamer 内部 queue/multiqueue 调优从 8 buffers /
  50ms 收紧到 4 buffers / 25ms；2 buffers / 12ms 不作为默认，因为 CPU 和 QoS/drop 回退。

最新 4K/240fps direct sink 短样本：

- 平均 45.45% process CPU；该源被 60fps sink limiter 丢帧，QoS dropped max 876 buffers。
- 峰值 PSS/USS 418115/403768 KiB，`Private_Dirty` 115156 KiB。
- NVIDIA 进程显存 472 MiB。
- 仍在 guardrail 内：PSS <= 460800 KiB、USS/private <= 430080 KiB、
  `Private_Dirty` <= 163840 KiB、NVIDIA 显存 <= 550 MiB。
- 默认 4/25ms 落地后的 2026-06-20 4s smoke 确认样本：
  `queue_max_size_buffers=multiqueue0:4|vqueue:4`、
  `queue_max_size_time_ns=multiqueue0:25000000|vqueue:25000000`，平均 CPU 46.65%，
  峰值 PSS/USS 418423/404080 KiB，`Private_Dirty` 114408 KiB，NVIDIA 进程显存 472 MiB。

GL wrapper 对照：

- `glsinkbin+gtk4paintablesink` 约 125.30% process CPU。
- PSS/USS/GPU memory 约 661/627/689 MiB。
- 结论：默认路径不应回到 `glsinkbin`；后续只有在验证 GLMemory/DMABuf 证据时才强制使用它。

## 2026-06-20 Progress Record

本轮 M8/T0 重点不是继续做表层小改，而是把 video runtime 的底层证据和默认占用策略固化下来：

- runtime caps observer 已能捕获 `caps-event`，并与 `current_caps()` / sticky CAPS 合并到
  status、`gilderctl --video-runtime-csv`、performance summary 和 baseline CSV。
- runtime queue observer 已能捕获 playbin 动态创建的 `queue` / `queue2` / `multiqueue`，
  解决旧样本 `queue_report_count=0` 导致无法判断中间队列保留的问题。
- 4K/240fps direct `gtk4paintablesink` 已确认保持 `NV12` 到 GTK sink 侧，并达到
  `zero_copy_evidence=sink-gpu-memory-caps`、`memory_path=sink-gpu-memory`。
- 默认 queue 从 8 buffers / 50ms 收紧到 4 buffers / 25ms。2 buffers / 12ms 不采用，
  因为短样本中 CPU 上升到 49.10%，QoS dropped max 升到 813，而 PSS/USS 和显存收益很小。
- 最新默认 4/25ms smoke 目录：`/tmp/gilder-wayland-video.BMetYm`。该样本 9 passed、0 failed，
  平均 CPU 46.65%，峰值 PSS/USS 418423/404080 KiB，`Private_Dirty` 114408 KiB，
  NVIDIA 进程显存 472 MiB，仍在 M8 guardrail 内。

验证已完成：

- `cargo fmt`
- `bash -n scripts/wayland-video-surface-smoke.sh`
- `bash -n scripts/wayland-baseline-matrix.sh`
- `bash -n scripts/performance-snapshot.sh`
- `cargo build --features gtk-renderer,video-renderer`
- `cargo test --features gtk-renderer,video-renderer`
- `cargo test`
- `git diff --check`
- 4K/240fps Wayland smoke：
  `scripts/wayland-video-surface-smoke.sh --no-build --sample-performance --sample-duration 4 --sample-interval 1 --video-size 3840x2160 --video-rate 240 --video-duration 1 --require-video-runtime-row --expect-zero-copy-evidence-at-least sink-gpu-memory-caps --expect-max-private-dirty-kib-at-most 163840 --expect-max-nvidia-process-gpu-memory-mib-at-most 550 --keep`

边界条件：

- 当前证据证明的是 GStreamer/GTK runtime sink-side GPU memory caps 和 NV12 sink-side 保持。
- 当前证据还不能证明 Wayland compositor presentation full zero-copy。
- 这轮 queue tuning 没有降低 NVIDIA 进程显存高水位；三个 queue 梯度短样本均为 472 MiB。
  因此下一轮显存优化不应继续只压 queue，而应转向 fullscreen/game auto-suspend 或
  sink/compositor buffer pool 证据。

Private_Dirty 进一步突破判断：

- 当前 `Private_Dirty` 约 110-115 MiB，已经接近轻量桌面监控器的应用内存口径，但还不是极限。
- `/tmp/gilder-wayland-video.gD5IVk/performance-active/memory-mapping-summary.txt` 是新增
  `category_summary_by_private_dirty` 后的确认样本：4K/240fps direct sink、4s sample 通过
  9/9 checks，`Private_Dirty` max 113716 KiB，zero-copy evidence 仍为
  `sink-gpu-memory-caps`，queue 仍为 4/25ms。
- 该样本主要 dirty 私有页来自 `anonymous` 48696 KiB、`heap` 20148 KiB、
  `nvidia-device` 18864 KiB、`nvidia-library` 9544 KiB、`shared-memory` 8192 KiB、
  `system-library` 7012 KiB。
- 这说明后续确实还有可挖空间；优先级应是先确认 active -> fullscreen/paused/hidden 后
  `anonymous`、`heap`、driver device/library dirty pages 是否释放，再决定是否动 GTK/GStreamer
  buffer pool 或 poster/static fallback 生命周期。
- `performance-snapshot.sh` 已增加 `top_mappings_by_private_dirty`、
  `category_summary_by_private_dirty`、`memory-mapping-categories.csv` 和
  `memory_category_<category>_private_dirty_kib` summary/baseline 字段。下一轮采样要直接按
  `Private_Dirty` 分类比较 active、paused/fullscreen 和 resumed，而不是只看总 PSS/USS。
- `wayland-baseline-matrix.sh` 会额外输出 `memory-category-deltas.csv`，把每个
  scenario/phase 的分类 `Private_Dirty` 与 `active,active` baseline 相减。预算 CSV 的
  `min_release_from_active_kib` 列可以把该 CSV 中 anonymous、heap、nvidia/dri-device 和
  nvidia-library 的 `release_from_active_kib` 变成 gate，用来判断 fullscreen/game
  auto-suspend 是否真的释放 dirty pages。

## Guardrails Before Experiments

每个实验都必须保留 active direct sink 的已知底线：

- CPU <= 120% process CPU，stretch goal <= 80%。
- PSS <= 460800 KiB。
- USS/private <= 430080 KiB。
- `Private_Dirty` <= 163840 KiB。
- NVIDIA 进程显存 <= 550 MiB。
- active 必须保持 video pipeline/source footprint 存在；paused/hidden/fullscreen/session removal 必须释放 pipeline/source footprint。

建议先跑短矩阵确认本机状态：

```sh
scripts/wayland-baseline-matrix.sh \
  --report-dir /tmp/gilder-m8-video-baseline \
  --sample-duration 3 \
  --sample-interval 1 \
  --no-build \
  --budget-csv examples/wayland-memory-budget.example.csv
```

NVIDIA 主机上可以额外对 active smoke 加显存门槛：

```sh
scripts/wayland-video-surface-smoke.sh \
  --sample-performance \
  --expect-renderer-video-pipeline-lifecycle \
  --expect-max-private-dirty-kib-at-most 163840 \
  --expect-max-nvidia-process-gpu-memory-mib-at-most 550 \
  --keep
```

4K/240fps 复测可以直接复用同一 smoke，并把生成源调大：

```sh
scripts/wayland-video-surface-smoke.sh \
  --no-build \
  --sample-performance \
  --sample-duration 6 \
  --sample-interval 1 \
  --video-size 3840x2160 \
  --video-rate 240 \
  --video-duration 1 \
  --require-video-runtime-row \
  --expect-zero-copy-evidence-at-least sink-gpu-memory-caps \
  --expect-max-private-dirty-kib-at-most 163840 \
  --expect-max-nvidia-process-gpu-memory-mib-at-most 550 \
  --keep
```

## Experiment 1: Zero-copy Evidence

目的：把证据从已达到的 sink-side GPU memory caps 继续推进到 DMABuf 或 compositor
presentation 证据，或者明确记录 GTK/GStreamer/NVIDIA/niri 当前组合为什么做不到。

先做对照，不先换架构：

- direct `gtk4paintablesink`：默认 `GILDER_GTK_VIDEO_SINK_CHAIN=auto`。
- forced GTK direct：`GILDER_GTK_VIDEO_SINK_CHAIN=gtk4`。
- forced GL wrapper：`GILDER_GTK_VIDEO_SINK_CHAIN=glsinkbin`，只用于诊断。

必须采集：

- `video-runtime.csv` 和 `video-runtime-summary.txt`。
- caps/sink caps memory features。
- `memory_path`、allocation reports、pool/allocator、queue reports。
- GTK frame clock/GDK frame timings，需要时启用 `GILDER_GTK_VIDEO_FRAME_STATS=full`。
- `video-hardware-report.txt`、`memory-mapping-summary.txt`、`nvidia-smi` process row。

成功条件：

- 最低可接受：`zero_copy_evidence >= sink-gpu-memory-caps`。
- 更强目标：`zero_copy_evidence >= sink-dmabuf-caps`。
- 同时 CPU、PSS/USS、`Private_Dirty`、NVIDIA 显存不得回退到 `glsinkbin` high-memory profile。
- 已完成证据链 smoke：小视频和 4K/240fps generated loop 的 direct GTK/playbin 路径达到
  `sink-gpu-memory-caps`；下一步要用同样 observer 对比 4K/240fps direct/gtk4/glsinkbin
  三条路径。

停止条件：

- 如果硬解稳定但 sink-side caps 长期只有 `memory:SystemMemory`，且 allocation query 也没有
  DMABuf/GLMemory allocator 或 pool，则记录为 GTK path blocker。
- 只有在 blocker 明确后，才评估 custom GStreamer sink、libmpv render API 或直接
  Wayland linux-dmabuf/GL/Vulkan surface。

## Experiment 2: Keep YUV/NV12 Until Presentation

目的：确认 4K frame 是否在到达 presentation 前过早变成 RGBA/RGBx texture。4K NV12
大约 12 MiB/frame，RGBA 大约 33 MiB/frame；过早转换会显著放大显存和带宽。

需要观察：

- decoder 输出 caps 中的 format，例如 NV12、I420、P010、RGBA、RGBx。
- sink-side caps 的 media type、format 和 memory features。
- `video-runtime.csv` / `video-runtime-summary.txt` 中的 `formats`、`sink_formats`、
  `format_paths`、`frame_sizes` 和 `caps_sources`；`video-hardware-report.txt` 会把这些摘要并到同一份
  evidence，便于 direct/gtk4/glsinkbin 横向比较。
- allocation pool 是否按 raw RGBA/RGBx 容量增长。
- `memory-mapping-summary.txt` 中 NVIDIA device/library 映射是否随 queue 或 format 改动明显变化。

适合的实现方向：

- 如果 `gtk4paintablesink` 能接收 GPU memory/DMABuf，但内部仍转 RGBA，需要进一步查 GTK/GDK/GSK texture path。
- 如果 GTK paintable path 无法保留 YUV/NV12，可把该限制写成 blocker，不在 GTK path 上做高风险 hack。
- 低层替代方案只有在可以用 shader 做 YUV->RGB at presentation，并且实际显存/CPU 更低时才值得推进。

不适合的做法：

- 不要为了看到 GLMemory/DMABuf 就默认切回 `glsinkbin`；现有证据显示它显著增加 CPU、PSS/USS 和显存。
- 不要只看 decoder 是 `nvh264dec` 就把路径标记为 zero-copy。

## Experiment 3: Queue Gradient

目的：确认 240fps 下 GStreamer 中间队列是否仍保留过多 frame。旧默认已压到 8 buffers /
50ms，但 240fps 下 50ms 仍可能覆盖约 12 帧窗口；本轮已继续收紧到 4 buffers / 25ms。

已落地的诊断和调优：

- playbin dynamic `queue` / `queue2` / `multiqueue` 会被 deep-element-added observer 记录。
- `observed_video_queue_reports()` 会重新下发当前 queue 调优，防止动态子元素在创建后覆盖默认值。
- `GILDER_VIDEO_QUEUE_MAX_SIZE_BUFFERS`、`GILDER_VIDEO_QUEUE_MAX_SIZE_TIME_MS`、
  `GILDER_VIDEO_QUEUE_MAX_SIZE_BYTES` 可用于 smoke/matrix 梯度实验。
- queue max-size-buffers 已验证：8 -> 4 -> 2。
- queue max-size-time 已验证：50ms -> 25ms -> 12ms。
- 保持 max-size-bytes 关闭，避免 4K frame 触发错误的小 byte limit。

4K/240fps generated loop、direct `gtk4paintablesink`、4s sample 结果：

| Queue tuning | Reported queue limits | Current level max | CPU avg | PSS/USS max | Private_Dirty max | NVIDIA GPU memory | QoS dropped max |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: |
| 8 buffers / 50ms | `multiqueue0:8|vqueue:8`, `50000000|50000000ns` | 7 buffers, 1358557 bytes, 29166667ns | 45.67% | 393263/359216 KiB | 114716 KiB | 472 MiB | 697 |
| 4 buffers / 25ms | `multiqueue0:4|vqueue:4`, `25000000|25000000ns` | 5 buffers, 970187 bytes, 20833334ns | 44.70% | 392745/358708 KiB | 114232 KiB | 472 MiB | 754 |
| 2 buffers / 12ms | `multiqueue0:2|vqueue:2`, `12000000|12000000ns` | 5 buffers, 969451 bytes, 20833333ns | 49.10% | 392338/358256 KiB | 113732 KiB | 472 MiB | 813 |

结论：

- observer/reporting 已能看到 playbin 动态 `multiqueue0` 和 `vqueue`，旧的 `queue_report_count=0`
  blind spot 已解决。
- 4/25ms 能降低实际 queue bytes/time 窗口，PSS/USS/`Private_Dirty` 没有回退，CPU 基本不升。
- 2/12ms 只带来边际内存下降，但 CPU 和 QoS dropped 都更差，停止继续压默认值。
- NVIDIA 进程显存在这组三个短样本中固定为 472 MiB，说明该显存高水位主要不是中间 queue
  深度决定，后续应从 sink/compositor buffer pool 或 fullscreen auto-suspend 继续深挖。
- `current_level_buffers_max` 在 4/25 和 2/12 下仍短暂到 5，可能来自 multiqueue pad 级瞬时状态或
  采样时序；因此文档只声明配置上限已下发和 bytes/time 窗口降低，不把它解释成硬实时帧数上限。

后续长样本仍需记录：

- `video_queue_current_level_buffers_max`。
- `video_queue_current_level_bytes_max`。
- `video_qos_messages_max` 和 `video_qos_dropped_max`。
- CPU、PSS/USS、`Private_Dirty`、NVIDIA 进程显存。
- 是否仍能稳定显示，尤其是 240fps source 的 dropped/QoS 变化。

长样本成功条件：

- 显存或 PSS/USS 可测下降。
- QoS/drop 没有超过当前 direct sink 基线。
- CPU 不明显上升。

停止条件：

- queue 继续降低只增加 dropped 或 jitter，却不再降低显存/USS。
- 低 queue 导致 resume、loop 或 compositor presentation 不稳定。

## Experiment 4: Fullscreen/Game Auto-suspend

目的：用户进入 fullscreen/game 场景时，优先释放动态视频资源，把后台占用压到尽量接近静态或空闲。

当前已有基础：

- fullscreen/hidden/session removal 可以释放 video pipeline/source footprint。
- Wayland smoke 和 baseline matrix 已有 lifecycle gate。
- resume 会等待 active video runtime 恢复再采证。

下一步要验证的是“显存是否真的回落”，而不是只看 render plan：

- fullscreen/hidden/user-paused 后 `renderer_video_pipelines_latest == 0`。
- `renderer_video_pipeline_source_references_latest == 0`。
- `max_nvidia_process_gpu_memory_mib` 和 paused/hidden/fullscreen 的 last GPU memory 是否明显低于 active。
- `memory-mapping-summary.txt` 中 `nvidia-device`、`nvidia-library`、anonymous、heap 分类是否回落。

可选增强：

- 对 fullscreen/game 模式增加更激进的 destroy-pipeline 策略。
- 只保留低成本 poster/static fallback，或者在 game mode 下直接清空动态 surface。
- 把 resume latency 作为预算，避免释放过猛导致回到桌面时明显黑屏或卡顿。

成功条件：

- fullscreen/game 场景下 CPU 接近 idle。
- video pipeline/source footprint 为 0。
- NVIDIA 进程显存和 `Private_Dirty` 明显低于 active。
- resume latency 可接受，并且 resumed active 恢复 video runtime row。

## Recommended Order

1. 已完成：runtime caps-event 证据链和 4K/240 direct sink 复测，证明 sink-side GPU memory caps 与
   NV12 保持到 GTK sink 侧。
2. 已完成：queue observer、queue 调参开关和 8/50、4/25、2/12 梯度；默认收紧到 4/25。
3. 下一步优先做 fullscreen/game auto-suspend 显存释放验证，因为 active 路径已进入 guardrail，
   而用户进入游戏时显存释放收益最大。
4. 并行保留 compositor presentation 证据研究；只有拿到 presentation/frame callback 或
   linux-dmabuf/GL/Vulkan 更底层证据后，才把当前 `sink-gpu-memory-caps` 升级成 full zero-copy。

如果 zero-copy 在 GTK path 上被证明受限，也不应立刻推翻当前 direct sink 默认路径。当前 direct
`gtk4paintablesink` 已经是实用顶级基线，替换底层渲染栈必须用同场景证据证明 CPU、显存、内存、
frame pacing 和 lifecycle 都更好。

## Native Wayland Video Host Evidence

2026-06-20 native Wayland video helper 已能在真实 niri/Wayland 会话中创建 Gilder-owned
wlr-layer-shell background surface，并把同一 `wl_display`/`wl_surface` 通过
GStreamer Wayland context 与 `GstVideoOverlay` 交给 `waylandsink`。这条路径不经过 GTK/GDK/GSK
和 `gtk4paintablesink`，用于验证底层替代方向是否真的降低内存/显存。

实现边界：

- `native-wayland-renderer + video-renderer` 构建提供 `gilder-native-video` helper。
- `NativeWaylandVideoSession` 拥有 host 和 player，确保 GStreamer pipeline 在 Wayland surface
  销毁前先进入 `Null`；raw handle 只在该 session 内传给 `waylandsink`。
- 使用 `gst_wl_display_handle_context_new(wl_display*)` 和
  `GstVideoOverlay::set_window_handle(wl_surface*)`，随后设置 render rectangle，避免
  `waylandsink` 自建 fullscreen surface。
- native helper 默认不再把 `target_max_fps` 映射到 `waylandsink throttle-time`；真实 4K/240
  样本证明 `throttle-time` 会显著抬高 CPU。`target_max_fps` 仍用于 `max-lateness` hint，
  `--sink-throttle` 只保留作诊断/回归对比。
- 当前 helper 已可播放 4K/240 H.264 source；还没有接入 daemon render_sync、多输出输出名绑定、
  status video-runtime telemetry 或 compositor presentation feedback。

真实 4K/240 source evidence：

| Path | Report dir | FPS policy | CPU avg | PSS max | USS/private max | Private_Dirty max | NVIDIA process GPU |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: |
| GTK `gtk4paintablesink` baseline | `/tmp/gilder-wayland-video.6Uiwo2/performance-active` | 60fps sink limiter on 240fps source | 46.20% | 453617 KiB | 427420 KiB | 113580 KiB | 448 MiB |
| native layer-shell `waylandsink` | `/tmp/gilder-native-wayland-video.4k240-60fps/performance` | 60fps sink limiter on 240fps source | 75.61% | 259791 KiB | 259152 KiB | 114980 KiB | 144 MiB |
| native layer-shell `waylandsink` | `/tmp/gilder-native-wayland-video.4k240/performance` | 240fps sink limiter | 120.75% | 261545 KiB | 260904 KiB | 116912 KiB | 144 MiB |
| native layer-shell `waylandsink` | `/tmp/gilder-native-wayland-video.4k240-nofps/performance` | no throttle-time | 110.88% | 261695 KiB | 261024 KiB | 118000 KiB | 144 MiB |
| native layer-shell `waylandsink` | `/tmp/gilder-native-wayland-video.240fps-nojson/performance` | pre-fix 240fps `throttle-time` | 122.38% | 259680 KiB | 259008 KiB | 114804 KiB | 144 MiB |
| native layer-shell `waylandsink` | `/tmp/gilder-native-wayland-video.240fps-top-visible-8s/performance` | 240fps max-lateness hint, no sink throttle, visible top layer | 46.26% | 248240 KiB | 247624 KiB | 103708 KiB | 144 MiB |
| native layer-shell `waylandsink` | `/tmp/gilder-native-wayland-video.playbin3-240fps-top-no-throttle/performance` | playbin3, no sink throttle, visible top layer | 48.61% | 263717 KiB | 263104 KiB | 118436 KiB | 144 MiB |

结论：

- native Wayland video host 已经完成“能播放 + 可采样 + 内存/显存显著更低”的第一条闭环。
- `waylandsink throttle-time` 是 4K/240 native CPU 飙升的主要原因；禁用后 visible top-layer
  CPU 回到 GTK baseline 同级，且低于 standalone `waylandsink` spike 的 48.33%。
- 相比 GTK baseline，当前 visible native 240fps 样本 PSS 下降约 205 MiB，USS/private 下降约
  180 MiB，`Private_Dirty` 下降约 10 MiB，NVIDIA 进程显存下降约 304 MiB。
- `playbin3` 在当前机器上没有收益，CPU、PSS 和 `Private_Dirty` 都高于 playbin；默认继续使用
  playbin。
- `Private_Dirty` 的主要 floor 仍是 anonymous/shared-memory + NVIDIA device/library 映射；
  runtime caps 继续显示 `playsink` 后段会把 NV12 转成 4K `RGBx` system-memory pool。下一步如要继续
  大幅下降，需要替换 playsink 后段或引入可验证的 CUDA/GL/DMABuf handoff，而不是继续调
  `throttle-time` 或简单放宽 queue。
