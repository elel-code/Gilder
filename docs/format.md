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
- `kind`：`static-image`、`video`、`slideshow`、`web`、`scene-lite`。
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
Wallpaper Engine 静态图转换器会在 `ffprobe`/`ffmpeg` 可用且源图足够大时生成
`landscape-*`、`ultrawide-*` 和 `portrait-*` 这类常见比例 PNG variant，作为降低
常见输出解码内存的保守默认；原图仍保留为 entry source 和无损 fallback。静态图
entry 带有 `width`/`height` 且没有可覆盖输出的 variant 时，daemon 还可以按当前输出
尺寸生成 `$XDG_CACHE_HOME/gilder/static-image-cache/` 下的运行时 PNG 缓存；显式
`--variant` 不会被这个机制覆盖。

`runtime.pause_when_fullscreen` 和 `runtime.pause_when_unfocused` 会参与 daemon 的桌面
状态性能决策。包内 runtime 策略只能让当前输出更保守，例如从 active/throttled
变为 paused；用户暂停、输出隐藏、会话 inactive 等更强决策不会被包内策略放宽。
`runtime.allow_audio` 默认为 `false`。视频计划会把 `entry.muted || !runtime.allow_audio`
作为最终静音状态；只有包显式允许音频且 video entry 未静音时，GStreamer 路径才会
允许音频输出。

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
- `width` / `height`: 可选源图像像素尺寸。转换器能探测到尺寸时会写入，daemon 可用它判断静态大图是否需要生成输出尺寸级缓存，避免直接让 GTK/GDK 解码原始超大图。

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
启用 `video-renderer` feature 时，GStreamer worker 会把该上限应用到视频 pipeline。
`entry.poster` 会优先作为视频占位图；如果没有设置，会回退到 manifest 的
`preview.poster`。有可用 poster 时，daemon 会额外生成一条静态渲染计划供
`gtk-renderer` 显示。

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
转成 `render_sync.slideshow_plans`，GTK renderer 会按 `interval_ms` 切换源图；
`transition = "crossfade"` 当前作为格式意图保留，早期运行时先执行稳定的即时切换。

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

### Scene-lite

`scene-lite` 是 Gilder 对 Wallpaper Engine 场景壁纸的可迁移子集，不追求完整兼容：

- 2D 图层。
- 变换、透明度、简单时间线。
- 基础粒子或 shader 需要逐项白名单。
- 不执行 SceneScript。
- 当前渲染器优先显示 `fallback`，完整 scene-lite runtime 后续实现。

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
