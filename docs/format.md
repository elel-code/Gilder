# Gilder Wallpaper Package v1

Gilder 的壁纸格式同时支持目录形态和归档形态：

- `.gwpdir`：开发、转换和调试用的普通目录，可使用 JSON 或 TOML manifest。
- `.gwp`：发布用归档文件，使用规范化后的 JSON manifest。

v1 建议 `.gwp` 使用 ZIP 容器。理由是随机访问成熟、跨平台工具多、易于只读挂载式读取。图片、视频等已压缩资源默认使用 store 或低压缩级别，避免浪费 CPU。

manifest 语法约定：

- `manifest.gilder.json`：规范格式，转换器默认输出，`.gwp` 发布包必须包含。
- `manifest.gilder.toml`：`.gwpdir` 作者友好格式，适合手写和 Linux 配置风格。
- 如果 `.gwpdir` 同时存在 JSON 和 TOML，loader 优先读取 JSON。
- `gilder-convert pack` 会校验 `.gwpdir`，然后把读取到的 manifest 序列化为
  `manifest.gilder.json` 写入 `.gwp`；TOML 作者文件不会作为运行 manifest 打包。

当前 CLI 支持：

```sh
gilder-convert pack <source.gwpdir> <dest.gwp>
gilder-convert unpack <source.gwp> <dest.gwpdir>
```

打包前会先校验 `.gwpdir`，解包时会拒绝路径逃逸的 ZIP entry，并在解包后再次校验 manifest 和资源引用。

## 目录结构

```text
example.gwpdir/
  manifest.gilder.json        # canonical, or manifest.gilder.toml for authoring
  assets/
    main.avif
    loop.webm
  previews/
    poster.jpg
    thumbnail.jpg
  metadata/
    source.json
    conversion-report.json
```

必需文件：

- `.gwpdir`：`manifest.gilder.json` 或 `manifest.gilder.toml`。
- `.gwp`：`manifest.gilder.json`。

保留目录：

- `assets/`：壁纸运行资源。
- `previews/`：预览图、缩略图、静态 fallback。
- `metadata/`：来源、转换报告、许可证说明等非运行关键数据。

## Manifest 示例

JSON 是规范格式：

```json
{
  "format": "gilder.wallpaper",
  "format_version": 1,
  "id": "org.example.neon-rain",
  "version": "1.0.0",
  "title": "Neon Rain",
  "authors": ["Example Author"],
  "license": "unknown",
  "kind": "video",
  "tags": ["city", "rain", "loop"],
  "preview": {
    "thumbnail": "previews/thumbnail.jpg",
    "poster": "previews/poster.jpg"
  },
  "entry": {
    "type": "video",
    "source": "assets/loop.webm",
    "poster": "previews/poster.jpg",
    "loop": true,
    "muted": true,
    "fit": "cover",
    "max_fps": 60
  },
  "variants": [
    {
      "id": "uhd",
      "source": "assets/loop.webm",
      "width": 3840,
      "height": 2160,
      "scale": 1.0
    }
  ],
  "properties": {
    "fit": {
      "type": "choice",
      "default": "cover",
      "choices": ["cover", "contain", "stretch", "tile"]
    }
  },
  "runtime": {
    "pause_when_fullscreen": true,
    "pause_when_unfocused": false,
    "allow_network": false,
    "allow_audio": false
  }
}
```

同一数据也可以在 `.gwpdir` 中写成 TOML：

```toml
format = "gilder.wallpaper"
format_version = 1
id = "org.example.neon-rain"
version = "1.0.0"
title = "Neon Rain"
authors = ["Example Author"]
license = "unknown"
kind = "video"
tags = ["city", "rain", "loop"]

[preview]
thumbnail = "previews/thumbnail.jpg"
poster = "previews/poster.jpg"

[entry]
type = "video"
source = "assets/loop.webm"
poster = "previews/poster.jpg"
loop = true
muted = true
fit = "cover"
max_fps = 60

[[variants]]
id = "uhd"
source = "assets/loop.webm"
width = 3840
height = 2160
scale = 1.0

[properties.fit]
type = "choice"
default = "cover"
choices = ["cover", "contain", "stretch", "tile"]

[runtime]
pause_when_fullscreen = true
pause_when_unfocused = false
allow_network = false
allow_audio = false
```

## 基本字段

- `format`：固定为 `gilder.wallpaper`。
- `format_version`：当前为 `1`。
- `id`：包 ID，推荐反向域名或稳定 slug。
- `version`：包版本。
- `title`：展示名称。
- `authors`：作者列表。
- `license`：许可证或 `unknown`。
- `kind`：`static-image`、`video`、`slideshow`、`web`、`scene-lite`、`shader`、`playlist`。
- `tags`：搜索和管理用标签。
- `preview`：缩略图和 fallback poster。
- `entry`：默认运行入口。
- `variants`：面向分辨率、比例、编码的资源变体。
- `properties`：用户可配置项 schema。
- `runtime`：权限和性能策略。

Wallpaper Engine 输入类型、Gilder 当前 kind、运行时支持等级和后续缺口见
[`docs/wallpaper-types.md`](wallpaper-types.md)。

当前渲染路径会在壁纸绑定携带 variant ID 时使用 `variants[].source` 替代
`entry.source`。这适用于静态图片和视频 entry；CLI 可以通过
`gilderctl set <wallpaper> --variant <id>` 绑定指定变体。没有显式 variant 时，daemon
会根据输出尺寸和 scale，在能覆盖目标尺寸的 variant 中选择像素面积最小的资源；没有
可覆盖 variant 时继续使用 entry 默认资源，避免把大屏输出降级到小资源。
`playlist` 子 entry 当前不会自动套用顶层 `variants`，避免条件选择出的子项被全局
variant 误替换。
Wallpaper Engine 静态图转换器会在 `ffprobe`/`ffmpeg` 可用且源图足够大时生成
`landscape-*`、`ultrawide-*` 和 `portrait-*` 这类常见比例 PNG variant，作为降低
常见输出解码内存的保守默认；原图仍保留为 entry source 和无损 fallback。静态图
entry 带有 `width`/`height` 且没有可覆盖输出的 variant 时，daemon 还可以按当前输出
尺寸生成 `$XDG_CACHE_HOME/gilder/static-image-cache/` 下的运行时 PNG 缓存。缓存触发会按
`fit` 估算实际显示尺寸，`contain` 的极端长图/竖图和 `stretch` 的大面积源图也可以在
不依赖额外 manifest variant 的情况下被降采样；显式 `--variant` 不会被这个机制覆盖。

`runtime.pause_when_fullscreen` 和 `runtime.pause_when_unfocused` 会参与 daemon 的桌面
状态性能决策。包内 runtime 策略只能让当前输出更保守，例如从 active/throttled
变为 paused；用户暂停、输出隐藏、会话 inactive 等更强决策不会被包内策略放宽。
`runtime.allow_audio` 默认为 `false`。视频计划会把 `entry.muted || !runtime.allow_audio`
作为最终静音状态；只有包显式允许音频且 video entry 未静音时，视频/audio runtime
才会允许音频输出。

## Entry 类型

### Static Image

```json
{
  "type": "static-image",
  "source": "assets/wallpaper.avif",
  "fit": "cover",
  "background": "#000000",
  "orientation": "from-metadata",
  "width": 3840,
  "height": 2160
}
```

支持策略：

- `fit`: `cover`、`contain`、`stretch`、`tile`、`center`。
- `background`: `contain` 或透明图像下的填充色。
- `orientation`: `from-metadata` 或 `ignore`。
- `width` / `height`: 可选源图像像素尺寸。转换器能探测到尺寸时会写入，daemon 可用它判断静态大图是否需要生成输出尺寸级缓存，避免运行时直接解码原始超大图。

### Video

```json
{
  "type": "video",
  "source": "assets/loop.webm",
  "poster": "previews/poster.jpg",
  "loop": true,
  "muted": true,
  "fit": "cover",
  "max_fps": 60,
  "start_offset_ms": 0
}
```

视频壁纸必须可无音频运行。即使源视频包含音轨，默认也应丢弃或静音。
daemon 会把 video entry 转成 `render_sync.video_plans`，其中 `manifest_max_fps`
保留包内声明的上限，`target_max_fps` 是 manifest 与桌面状态性能策略合成后的运行上限。
启用 native Vulkan video feature 时，FFmpeg frontend 只负责 demux/bitstream filter；
最终显示由 native Vulkan Video decode/render/present 路径负责。
默认 muted 视频不启动音频输出；只有 `runtime.allow_audio = true` 且 entry 未 muted
时，后续音频/clock 路线才可接入。
`entry.poster` 会优先作为视频元数据；如果没有设置，会回退到 manifest 的
`preview.poster`。daemon 不再为 video poster 额外生成静态渲染计划。

### Slideshow

```json
{
  "type": "slideshow",
  "sources": ["assets/a.avif", "assets/b.avif"],
  "interval_ms": 300000,
  "transition": "crossfade",
  "fit": "cover"
}
```

Slideshow 是 v1 的普通动态壁纸，不需要脚本运行时。daemon 会把 slideshow entry
转成 `render_sync.slideshow_plans`，计划层保留 `interval_ms`、fit 和 transition 语义；
native Vulkan runtime 后续负责可见切换和 crossfade。

### Playlist

```json
{
  "type": "playlist",
  "selection": "first-match",
  "items": [
    {
      "id": "battery-static",
      "conditions": {
        "power": "battery"
      },
      "entry": {
        "type": "static-image",
        "source": "assets/battery.avif",
        "fit": "cover"
      }
    },
    {
      "id": "workday",
      "conditions": {
        "weekdays": ["monday", "tuesday", "wednesday", "thursday", "friday"],
        "local_time": {
          "start": "08:30",
          "end": "18:00"
        }
      },
      "entry": {
        "type": "static-image",
        "source": "assets/workday.avif",
        "fit": "cover"
      }
    },
    {
      "id": "default-video",
      "entry": {
        "type": "video",
        "source": "assets/loop.webm",
        "muted": true,
        "fit": "cover",
        "max_fps": 60
      }
    }
  ]
}
```

Playlist 是一等包类型，用来在一个 `.gwpdir` 内根据当前桌面状态选择已有 entry。
当前 `selection` 支持 `first-match` 和 `weighted-random`。`first-match` 会按顺序选择
第一个所有条件都满足的 item；`weighted-random` 会先过滤满足条件的 item，再按 item
的 `weight` 做稳定加权选择。`weight` 默认是 1，必须大于 0。稳定 seed 来自输出名、
本地分钟、本地星期、当前桌面状态和候选 item id/weight，因此同一分钟内状态栏轮询不会
导致壁纸跳变。选中的 `entry` 会继续转换为 static/video/slideshow/web/scene-lite/shader 的既有
render plan。
支持的条件字段：

- `outputs`：输出名列表，空列表表示不限制输出。
- `power`：`unknown`、`ac` 或 `battery`。
- `local_time`：本地时间半开区间 `{ "start": "HH:MM", "end": "HH:MM" }`，
  使用系统时区；`start < end` 表示同日区间，`start > end` 表示跨午夜区间。
- `weekdays`：本地星期数组，支持 `monday` 到 `sunday`，也接受 `mon`、`tue`、
  `wed`、`thu`、`fri`、`sat`、`sun` 短写；空数组表示不限制星期。
- `focused`、`visible`、`fullscreen`：匹配当前输出状态。
- `session_active`、`session_locked`：匹配当前 logind/session 状态。

如果没有 item 匹配，render plan 会报告 `playlist did not match any item`。Playlist
可以用于性能策略，例如电池供电时选择静态图，AC 时选择视频；这样 `pause-dynamic`
会按实际选中的子 entry 判断是否需要释放资源。CI 或 smoke 可以用
`GILDER_PLAYLIST_LOCAL_TIME=HH:MM` 固定本地时间条件，并用
`GILDER_PLAYLIST_LOCAL_WEEKDAY=<weekday>` 固定本地星期条件。媒体信息、系统信息和更复杂
日历策略仍是后续格式扩展。

### Web

```json
{
  "type": "web",
  "root": "assets/web",
  "index": "index.html",
  "fallback": "previews/poster.jpg",
  "max_fps": 30
}
```

Web 运行时默认受限：

- 不允许访问包根之外的本地文件。
- 默认不允许网络。
- 用户属性通过 Gilder bridge 注入，而不是直接暴露宿主 API。

当前 daemon 不直接执行 WebKit 内容；Web runtime 需要独立 helper，通过 frame stream
或可导入 texture 交给 native Vulkan 后端。如果 `fallback` 存在，renderer 会先显示
fallback 静态图，并把 `web` 视为动态壁纸参与 `pause-dynamic` 策略。缺少 `fallback`
的 Web 包会在渲染计划中报告 unsupported，避免静默显示空背景。

### Shader

```json
{
  "type": "shader",
  "source": "shaders/main.frag",
  "fallback": "previews/poster.svg",
  "language": "glsl",
  "max_fps": 60,
  "uniforms": [
    { "name": "u_time", "source": "time" },
    { "name": "u_resolution", "source": "resolution" },
    { "name": "u_mouse", "source": "mouse" },
    { "name": "u_intensity", "source": "property", "property": "intensity" }
  ]
}
```

`shader` 是 v1 为 GLSL/WGSL 风格动态壁纸预留的一等 entry：

- `source` 指向包内 shader 源文件。
- `fallback` 指向静态 poster；当前 renderer 尚未编译或执行 shader，会显示该 fallback。
- `language` 可为 `auto`、`glsl` 或 `wgsl`，默认 `auto`。
- `max_fps` 必须大于 0，后续原生 shader runtime 会与桌面状态性能策略合成目标 FPS。
- `uniforms` 声明 runtime 需要注入的 uniform。`source` 支持 `time`、`resolution`、`mouse`
  和 `property`；`property` uniform 必须设置 `property` 字段，其它来源不能设置
  `property`。uniform 名称不能为空且不能重复。

缺少 `fallback` 的 shader 包会在当前渲染计划中报告 unsupported，避免把未实现的
GPU shader runtime 误显示为空背景。`shader` 会作为动态壁纸参与 `pause-dynamic`
资源释放策略。

### Scene-lite

`scene-lite` 是 Gilder 对 Wallpaper Engine 场景壁纸的可迁移子集，不追求完整兼容：

- `entry.source` 指向一个 Gilder scene-lite JSON 文档。
- 2D `image`、`color`、`rectangle`、`ellipse`、`text`、`path` 和 `group` 图层。
- 变换、透明度、keyframe timeline 和动画曲线。
- 用户属性到图层属性的 binding schema。
- 基础粒子或 shader 需要逐项白名单。
- 不执行 SceneScript。
- 当前核心层会解析、校验并可确定性求值 scene graph；daemon 会生成
  `render_sync.scene_lite_plans`，并把 scene snapshot 和 image layer 纳入计划层资源
  footprint。native Vulkan/Vulkanalia 路线已经具备 solid/color quad 的可见 present
  边界，并在推进 sampled image 的 decode/upload/descriptor/draw 接线；headless
  snapshot fallback 仍可生成受控 SVG 资源作为兼容占位。原生动画 scene-lite surface
  后续继续补齐 text、path、effect 和 timeline。

最小 scene-lite 文档：

```json
{
  "version": 1,
  "size": { "width": 1920, "height": 1080 },
  "layers": [
    {
      "id": "background",
      "type": "image",
      "source": "assets/background.avif",
      "fit": "cover",
      "opacity": 1.0,
      "transform": {
        "x": 0,
        "y": 0,
        "scale_x": 1.0,
        "scale_y": 1.0,
        "rotation_deg": 0
      },
      "animations": [
        {
          "property": "opacity",
          "loop": true,
          "keyframes": [
            { "time_ms": 0, "value": 0.75 },
            { "time_ms": 1000, "value": 1.0, "curve": "ease-in-out" }
          ]
        }
      ]
    },
    {
      "id": "panel",
      "type": "rectangle",
      "color": "#102030",
      "stroke_color": "#ffffff",
      "stroke_width": 2,
      "corner_radius": 16,
      "width": 640,
      "height": 360,
      "transform": { "x": 100, "y": 80 }
    },
    {
      "id": "glow",
      "type": "ellipse",
      "color": "#80ffaa",
      "width": 240,
      "height": 160,
      "opacity": 0.5
    },
    {
      "id": "title",
      "type": "text",
      "text": "Gilder & Wayland",
      "color": "#f0f4ff",
      "font_size": 48,
      "font_family": "Inter",
      "font_weight": "700",
      "text_align": "middle",
      "width": 1920,
      "transform": { "y": 96 }
    },
    {
      "id": "wave",
      "type": "path",
      "path": "M 0 80 C 120 20 240 140 360 80",
      "stroke_color": "#80ffaa",
      "stroke_width": 4,
      "transform": { "x": 200, "y": 160 }
    }
  ],
  "property_bindings": [
    {
      "property": "scene_opacity",
      "target": "opacity",
      "layer": "background"
    }
  ]
}
```

Shape layer 的 `color` 是 fill 色；`stroke_color`、`stroke_width`、`corner_radius`、
`width` 和 `height` 均为可选。`width`/`height` 缺省时使用当前 snapshot 尺寸，
适合全屏色块或遮罩；指定本地尺寸后可以继续通过 `transform` 定位、缩放和旋转。
Text layer 的 `text` 和 `color` 必填，`font_size`、`font_family`、`font_weight`、
`text_align` 和 `width` 可选。`text_align` 支持 `start`、`middle`、`end`，用于映射 SVG
`text-anchor`；文本内容会作为 SVG text node 转义，不作为 SVG/HTML 片段执行。
Path layer 的 `path` 是 SVG path data；`color` 作为 fill，可省略为 `none`；
`stroke_color` 和 `stroke_width` 可选。`path` 会写入 SVG `d` attribute 并做 XML 属性转义。

支持的动画属性：`opacity`、`x`、`y`、`scale-x`、`scale-y`、
`rotation-deg`。支持的曲线：`linear`、`step`、`ease-in`、`ease-out`、
`ease-in-out`。scene-lite 文档内的 image layer `source` 也会在包加载时做路径
存在性校验。

`property_bindings` 已接入 render sync：`gilderctl properties set` 写入的数值属性会在
下一次 scene-lite 渲染计划中映射到对应 layer，输出级属性覆盖全局属性。运行时当前支持
`number`、`range`、`bool` 和可解析为数值的字符串；`bool` 映射为 `1/0`，`opacity`
会限制在 `0..1`，非正 scale 值会被忽略。绑定属性会进入 scene-lite snapshot 缓存键，
避免不同属性状态复用旧的 snapshot SVG；未被当前 scene-lite plan 声明的 IPC 属性不会让
daemon render sync 缓存失效。

## 用户属性

Gilder v1 属性类型：

- `bool`
- `number`
- `range`
- `choice`
- `color`
- `text`
- `file`

属性只描述 UI 和值域，不允许携带可执行代码。

## 资源路径规则

- 路径必须是相对路径。
- 禁止 `..`、绝对路径、空路径、NUL 字符。
- 运行时只读取 manifest 引用的资源。
- 转换器生成包时应记录原始来源到 `metadata/source.json`。

## 编码建议

图片：

- 首选 AVIF/WebP，保留 PNG/JPEG 输入的无损迁移能力。
- 预览图使用 JPEG 或 WebP。

视频：

- 首选 WebM/VP9、WebM/AV1 或 MP4/H.264。
- 转换器默认不重新编码已经可播放的视频，只复制并记录 codec。
- 需要转码时优先生成 WebM，除非用户指定兼容模式。

## 版本兼容

`format_version` 只在破坏性变更时递增。运行时遇到更高版本应拒绝加载并给出明确错误；遇到未知字段应忽略并保留。
