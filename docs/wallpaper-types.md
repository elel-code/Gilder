# 壁纸类型矩阵

本文档记录 Gilder 如何对齐 Wallpaper Engine 的项目类型和常见运行能力。矩阵刻意区分
“转换支持”和“运行时支持”：某类壁纸可以先转换成带 fallback 的 Gilder 包，但这并不
代表原生运行时已经完整实现。

支持等级：

- `完整`：可以作为原生 Gilder 壁纸运行，并具备表中列出的行为。
- `部分`：可以转换或表达，但保真度、运行时能力或验证链路仍有缺口。
- `fallback`：保留静态或视频 fallback，并在转换报告记录缺失能力。
- `阻塞`：当前拒绝或必须等待后续运行时。

## 类型覆盖

| Wallpaper Engine 输入 | 当前 Gilder 类型 | 转换 | 运行时 | 当前行为 | 主要缺口 |
| --- | --- | --- | --- | --- | --- |
| Image | `static-image` | 完整 | 完整 | 复制源图、preview/poster、fit 意图；足够大的光栅图可生成 16:9、21:9/ultrawide 和 9:16 portrait variants；带尺寸 metadata 的超大静态图可生成输出尺寸级运行时缓存。 | 更多编码 variant、真实 Wayland USS/PSS 基线和不同 fit 模式的质量/内存阈值还需要继续优化。 |
| Video | `video` | 完整 | 完整 | 复制可播放视频，必要时生成 poster，支持 loop、静音/音频意图、max FPS、decoder policy 和运行时证据；native-wgpu 4K/240fps CUDA/Vulkan image path 已作为当前高刷基线。 | active video copy/private dirty 暂不继续深挖；后续更大突破按 `docs/vulkan-migration.md` 评估纯 Vulkan 或更底层 interop。 |
| Web | `web` | 部分 | fallback plan | 复制 HTML/CSS/JS 资源，注入兼容 bridge，映射用户属性；renderer 可显示 fallback poster，并按动态壁纸参与 `pause-dynamic` 资源释放。 | Web runtime 需要独立浏览器 helper；WebKitGTK/GTK 可先隔离到 helper 内，sandbox、输入/audio/FPS bridge、权限模型和后端无关 frame/texture handoff 未完成。 |
| Scene | `scene-lite` | 部分 | first-class plan + static snapshot | 生成 Gilder scene-lite graph，支持 2D image/color/rectangle/ellipse/text/path/group layer、transform、opacity、keyframe/timeline 曲线和属性 binding；daemon 生成 `scene_lite_plans`，GTK 当前把 time=0 snapshot 合成为受控缓存 SVG surface，IPC 数值/布尔属性可影响 snapshot layer，并统计 snapshot/layer 图片资源。 | 原生动画 scene runtime、effect stack、particle system、shader node 和 audio response 未完成；新增 runtime 必须保持后端无关，避免绑定 GTK snapshot 路径。 |
| Shader / shader effect | `shader`（手写包或明确 WE Shader）/ `scene-lite` fallback（Scene 内 shader） | 部分 | fallback plan | Gilder 包格式可声明一等 `shader` entry，记录 GLSL/WGSL source、time/resolution/mouse/property uniform、max FPS 和 fallback poster；明确 Wallpaper Engine Shader 项目和 playlist shader 子项可转为 `shader` fallback entry；当前 renderer 显示 fallback，并按动态壁纸参与 `pause-dynamic` 资源释放。Scene 内 custom shader/effect graph 仍记录为缺失能力。 | 原生 GPU shader compile/render、uniform 注入、GPU memory telemetry 和 Wayland shader surface smoke 未完成；uniform/schema 不能绑定单一后端。 |
| Application / executable | 无 | 阻塞 | 阻塞 | 拒绝转换并生成 conversion report。 | 为安全和可移植性，原生可执行壁纸不作为目标能力。 |
| Playlist / collection | `playlist` / `slideshow` | 部分 | 部分 | 静态图片序列可转为 `slideshow`；Wallpaper Engine playlist/collection 中的 image/video/web/scene 子项可转为一等 `playlist` item 并保留 weight，web 子项注入 bridge，scene 子项降级为独立 `scene-lite` fallback graph；GTK renderer 支持定时切换和非 `tile` fit 的 crossfade；一等 `playlist` entry 可按 first-match 条件或稳定 weighted-random 在 static/video/slideshow/web/scene-lite/shader 子 entry 间选择，支持 item weight、输出、电源、本地时间窗口、本地星期、focused/visible/fullscreen 和 session 条件。 | 媒体/系统信息、更复杂日历选择和更完整 Wallpaper Engine playlist 策略映射仍需补。 |

## 能力矩阵

| 能力 | 当前 Gilder 路径 | 状态 | 验证目标 |
| --- | --- | --- | --- |
| 静态图片显示 | `static-image` entry | 完整 | Manifest 加载测试、GTK 静态 smoke、fit-mode render plan 测试。 |
| 视频循环播放 | `video` entry + GStreamer | 完整 | Codec smoke、Wayland video surface smoke、video runtime CSV。 |
| 视频音频意图 | `runtime.allow_audio` + `entry.muted` | 部分 | Converter 测试和 `playbin` flags 测试；PipeWire 采集/输出策略仍是后续工作。 |
| Slideshow / 普通动态图片 | `slideshow` entry | 完整 | Render plan 测试、GTK 定时切换/crossfade、adaptive、battery、fullscreen、unfocused、hidden 和 session `pause-dynamic` 测试。 |
| Playlist 条件选择 | `playlist` entry | 部分 | Manifest/schema 测试、Wallpaper Engine image/video/web/scene playlist 转换测试、power 条件 render plan 测试、本地时间/星期条件 selection 测试、稳定 weighted-random/weight 测试、battery `pause-dynamic` 静态选择测试；媒体/系统信息和复杂日历策略后续补。 |
| Web 壁纸资源 | `web` entry | 部分 | Converter 测试、manifest 校验和 fallback render plan 测试。 |
| Web runtime bridge | `assets/web/gilder-bridge.js` | fallback | 后续 web helper smoke、WebKitGTK/浏览器进程内存预算和属性更新测试。 |
| Scene fallback/snapshot | `scene-lite` entry display | 部分 | Converter 测试、scene-lite render plan 测试、静态 snapshot SVG cache 测试和 fallback/首图/纯色 GTK 显示路径。 |
| Scene layer 和 transform | `core::scene_lite` graph | 部分 | Headless scene graph 解析、shape/text/path layer、资源校验和 snapshot evaluator 测试。 |
| Timeline 动画 | `core::scene_lite` keyframes | 部分 | 确定性 timeline 曲线求值测试；原生 scene surface 和真实 renderer frame budget telemetry 后续补。 |
| Shader entry | `shader` entry | 部分 | Manifest/schema 测试、Wallpaper Engine Shader 转换测试、fallback render plan 测试和 `pause-dynamic` 释放测试；Shader compile、uniform 注入、GPU memory telemetry 和 Wayland surface smoke 后续补。 |
| Particle | 后续 scene/particle runtime | 阻塞 | 确定性 emitter 测试、资源预算 gate、adaptive pause 测试。 |
| Audio response | 后续可选 PipeWire input | 阻塞 | 显式权限测试、默认关闭/静音策略、延迟和资源 telemetry。 |
| 用户属性 | Manifest `properties` + scene-lite bindings | 部分 | Parser/schema 测试、scene-lite snapshot 属性绑定测试；video/web/native scene runtime 仍需按类型继续接入。 |
| 桌面状态性能优化 | policy/adaptive monitor | 当前动态类型完整 | Desktop policy smoke、resource baseline CSV、Wayland performance snapshot。 |
| 硬解证据 | video runtime decoder reports | 部分 | Codec smoke 和 Wayland runtime gates。 |
| Zero-copy 证据 | caps 和 sink memory feature reports | 部分 | DMABuf/GLMemory gates，以及后续 compositor presentation 证据。 |

## 实现顺序

1. 保持 static image、video 和 slideshow 作为性能基线。这些类型必须持续维持低 CPU、
   GPU、PSS、USS/private 和 wakeup。
2. 下一步优先让 `web` runtime 可用，并默认启用 sandbox，网络和音频必须显式授权。
   Web 壁纸常见，而且已经能映射到当前转换出的资源目录。
3. 在加入原生 shader runtime 或 particle 前，先把 `scene-lite` 扩展成真正的 2D scene runtime。
   Scene graph、transform、opacity 和 timeline 行为应可在无 compositor 环境确定性测试；
   当前 first-class render plan 已经给原生 scene surface 留出同步边界。
4. 原生 shader runtime 和 particle 能力从一开始就要接入 GPU/USS/PSS 预算 gate，以及 adaptive
   pause/throttle。
5. Audio-responsive 壁纸必须作为 opt-in 能力实现，使用 PipeWire input，并在 status/watch
   中清晰报告权限和采样状态。
6. Playlist 已按 first-match 条件和稳定 weighted-random 复用 static、video、slideshow、
   web、scene-lite、shader 的既有 renderer 逻辑，并支持本地时间窗口和本地星期条件；Wallpaper
   Engine playlist 已支持 image/video/web/scene 子项，后续再补复杂策略映射。

## Renderer 后端约束

后续扩展其他壁纸类型时，必须按 `docs/vulkan-migration.md` 的边界实现：

- manifest、conversion report、render plan 和 headless evaluator 不引用 GTK、GDK、wgpu、
  ash、GStreamer 等后端具体类型。
- WebKitGTK 可以作为 Web helper 的内部实现，但不能成为 daemon/core 的架构依赖。
- scene-lite 和 shader runtime 的属性、time、resolution、mouse 和资源输入必须可被 GTK/wgpu
  当前实现和未来纯 Vulkan 后端共同消费。
- 每个动态类型都要接入 `pause-dynamic`、fullscreen/hidden/session release、resource telemetry
  和 baseline matrix 预算，而不是只实现 active 显示。
- pure Vulkan 后端只有在 static/video/web/scene/shader/playlist 的同一类型矩阵都能通过后，
  才能替换当前默认后端。

## 转换报告要求

每次 Wallpaper Engine 转换都应继续维护：

- `detected_features`：从项目元数据或引用文件观察到的源能力。
- `converted_features`：已经表达在 Gilder 包中的能力。
- `unsupported_features`：明确丢弃或等待后续运行时的能力。
- `warnings`：用户需要看到的保真度或运行时 caveat。
- `errors`：executable wallpaper 等硬阻塞。

当某个能力从 `阻塞` 移到 `部分` 或 `完整` 时，转换器测试需要同时断言 manifest 输出
和 conversion report 的变化，避免 unsupported feature 账目静默漂移。
