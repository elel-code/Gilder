# Wallpaper Engine 转换设计

本项目不会运行 Windows Wallpaper Engine，也不会承诺完整还原其私有场景运行时。转换器的目标是尽量把可迁移资源转成 Gilder 原生格式，并在报告中明确损失。

参考资料：

- Wallpaper Engine 官方 Scene 文档说明其编辑器可基于图片加入元素、效果、粒子、时间线、音频响应、视差、SceneScript、shader 等能力。
- Wallpaper Engine 官方 Web 文档说明 Web wallpaper 基于 HTML/CSS/JavaScript，并提供用户属性、音频、FPS 等接口。
- Wallpaper Engine 官方 Web 用户属性文档列出 color、slider、bool、combo、text、file、directory 等属性类型。

## 输入类型

转换器入口：

```sh
gilder-convert wallpaper-engine <source-project-dir> <dest.gwpdir>
```

当前实现支持静态图片、视频、Web、明确 Shader、保守 `scene-lite` 和 image/video/web/scene/shader 子项组成的 playlist/collection 项目的 `.gwpdir` 输出；application/executable 项目会生成转换报告并拒绝转换。明确 Shader 项目会复制 shader source，生成 `entry.type = "shader"`、uniform schema 和 fallback poster；Scene 内嵌 custom shader/effect graph 仍作为 Scene 转换缺口记录，不会执行或翻译。缺失预览图时，静态图片项目会从源图生成 poster/thumbnail，视频项目会优先通过本机 `ffmpeg` 从首帧生成 poster/thumbnail，失败时回退到 metadata-based SVG fallback；Scene 和 Shader 项目会生成 metadata-based SVG fallback。静态大图在本机同时有 `ffprobe` 和 `ffmpeg` 时，会额外生成 16:9、21:9/ultrawide 和 9:16 portrait PNG variants，供 daemon 按输出尺寸自动选择，降低常见输出上无意义解码超大原图的概率。

已支持：

```sh
gilder-convert wallpaper-engine --pack <source-project-dir> <dest.gwp>
```

后续可扩展：

```sh
gilder-convert wallpaper-engine --transcode video=webm <source> <dest.gwpdir>
gilder-convert wallpaper-engine --allow-web <source> <dest.gwpdir>
```

## 识别流程

1. 检查源目录是否存在 `project.json`。
2. 读取项目元数据、标题、描述、预览图、类型、用户属性。
3. 根据项目类型和入口文件选择转换策略。
4. 扫描资源引用，复制到 `assets/`、`previews/`、`metadata/`。
5. 生成 `manifest.gilder.json`。
6. 生成 `metadata/source.json` 和 `metadata/conversion-report.json`。
7. 校验包内路径、文件存在性、manifest schema。

转换器默认生成规范 JSON manifest。手写或二次编辑 `.gwpdir` 时可以改用
`manifest.gilder.toml`，但 `.gwp` 打包会重新写出规范化的 `manifest.gilder.json`。

## 类型映射

完整类型/能力矩阵见 [`docs/wallpaper-types.md`](wallpaper-types.md)。该矩阵区分
“可转换为 Gilder 包”和“已有原生运行时”，避免把 fallback 误判为完整兼容。

| Wallpaper Engine 类型 | Gilder 类型 | 支持等级 | 策略 |
| --- | --- | --- | --- |
| Image / Scene from image | `static-image` 或 `scene-lite` | 高/中 | 纯图片无损复制；含效果时转 scene-lite 子集或静态 fallback |
| Video | `video` | 高 | 复制可播放视频；必要时转码；生成 poster |
| Web | `web` | 中 | 复制 HTML/CSS/JS/资源；注入兼容 bridge；默认禁网 |
| Scene | `scene-lite` / `video` / `static-image` | 低到中 | 复制 Scene 入口元数据并生成 fallback；复杂效果记录为 unsupported |
| Shader / effect | `shader` / `scene-lite` fallback | 低 | 明确 Shader 项目复制 shader source，生成 uniform schema 和 fallback；Scene 内 custom shader 仍记录为缺口 |
| Playlist / collection | `playlist` | 中 | 将 image/video/web/scene/shader 子项复制为一等 playlist item；保留 item weight；web 子项注入 bridge；scene 子项生成独立 scene-lite fallback graph；shader 子项生成 `shader` fallback entry |
| Application / executable | 不支持 | 无 | 拒绝转换，仅生成报告 |

## 静态图片转换

适合情况：

- Wallpaper Engine 项目只有单张图片。
- Scene 项目只包含一个背景图且没有动画/effect。

输出：

- 原图复制到 `assets/`。
- 预览复制或生成到 `previews/`；缺失 preview 时从源图复制生成 poster 和 thumbnail。
- 对足够大的光栅图片，若 `ffprobe`/`ffmpeg` 可用，会生成 16:9 `landscape-*`、21:9/ultrawide `ultrawide-*` 和 9:16 `portrait-*` PNG variants；源图不足对应尺寸、工具缺失或解码失败时跳过并写入转换报告 warning。
- `entry.type = "static-image"`。
- `fit` 根据源项目 alignment/scaling 映射，无法识别时使用 `cover`。

可选优化：

- 生成 AVIF/WebP variant，减少 PNG variant 的磁盘体积。
- 保留原图作为无损源。
- 增加更多设备档位或按用户输出 profile 生成 variant。

## 视频转换

适合情况：

- Wallpaper Engine video wallpaper。
- Web 项目或 Scene 项目中可识别出主循环视频。

输出：

- 视频复制到 `assets/`。
- poster 复制；缺失 preview 时优先调用 `ffmpeg` 从第一帧生成 `previews/poster.jpg` 和 `previews/thumbnail.jpg`，如果 `ffmpeg` 不在 `PATH` 或解码失败，则生成 SVG fallback 并在转换报告写入 warning。
- `entry.type = "video"`。
- 默认 `loop = true`、`muted = true`。如果 `project.json` 有明确音频开关或音频文件字段，
  转换器会设置 `runtime.allow_audio = true` 并把 video entry `muted` 设为 `false`。

转码策略：

- 如果源视频是 MP4/H.264、WebM/VP9/AV1 且系统可播放，默认复制。
- 如果源视频格式罕见，提供 `--transcode` 选项。
- 不默认保留音频，除非用户显式允许。

## Web 转换

适合情况：

- Wallpaper Engine web wallpaper。
- 项目入口是 HTML/CSS/JS。

输出：

- Web 根目录复制到 `assets/web/`。
- `entry.type = "web"`。
- 如果有 preview/fallback，当前 renderer 会先用 fallback 生成静态渲染计划。
- 用户属性转为 Gilder `properties`。
- 生成 `assets/web/gilder-bridge.js`，提供基础属性桥接，并在后续 Web runtime 中适配常见 `window.wallpaperPropertyListener.applyUserProperties` 行为。

限制：

- 默认禁止网络请求。
- 默认禁止访问包根之外的本地文件。
- 音频可视化、媒体集成、RGB 硬件接口先记录为 unsupported feature；检测到 Web
  或 Scene 音频意图时会写入转换报告，但不会打开 `runtime.allow_audio`。
- Web runtime 本身会记录为 `web-runtime`，检测到网络、音频 listener 或媒体集成时会额外
  记录 `web-permissions`，用于提醒后续需要 sandbox、权限和低功耗策略。
- `directory` 属性可迁移为普通 `file`/`directory` schema，但运行时能力后置实现。

## Scene 转换

Wallpaper Engine scene 能力很大，v1 只实现可解释子集。

可迁移：

- 背景图片层。
- 静态前景层。
- 基础 transform：位置、缩放、旋转、透明度。
- 简单循环时间线。
- 部分粒子系统的静态 fallback。

暂不迁移：

- SceneScript。
- Scene 内自定义 shader graph/source。明确 Shader 项目可转为 `shader` fallback entry，但 Scene graph 中的 shader/effect 仍不执行、不翻译。
- 复杂粒子、音频响应、RGB 设备联动。
- 3D model 行为。

Scene 转换策略按优先级：

1. 当前先生成保守 `entry.type = "scene-lite"`，`entry.source` 指向 Gilder
   `assets/scene-lite.json`。该文档至少包含 fallback image layer，后续可扩展为原生
   scene graph。
2. 原始 Wallpaper Engine Scene 入口文件保留到 `metadata/source-scene.*`。
3. 如果项目提供 preview，则作为 `fallback`；缺失时生成 SVG fallback。
4. 完整 Scene runtime 仍记录为 `scene-runtime`；SceneScript、shader、复杂粒子、timeline、
   parallax 和音频响应会分别记录为 `scenescript`、`custom-shader`、
   `complex-particles`、`timeline-animation`、`parallax` 和 `audio-runtime`。
5. 后续如果能识别主要视频或图片，可降级为 `video` 或 `static-image`。

## Playlist 转换

适合情况：

- Wallpaper Engine playlist 或 collection 项目中列出多个图片、视频、Web 或 Scene 资源。
- `project.json` 使用 `items`、`playlist`、`wallpapers`、`entries` 或 `children` 数组描述条目。

输出：

- 顶层 `entry.type = "playlist"`，`selection = "first-match"`。
- image 子项转换为 `static-image` entry，video 子项转换为 muted `video` entry。
- web 子项复制到独立 `assets/playlist-<index>-web/` 根目录并注入 `gilder-bridge.js`。
- scene 子项保留原始 metadata，并转换为独立的 `scene-lite` fallback graph。
- `weight`、`probability` 或 `chance` 数值会转换为 Gilder playlist item `weight`。
- image/video 子项资源复制到 `assets/playlist-<index>.*`，item id 使用稳定索引和源标题/name/id 生成。

限制：

- Wallpaper Engine playlist 的复杂日历、媒体状态、随机策略和嵌套 playlist 还未完整映射。
- application 和嵌套 playlist 子项仍不转换，会写入 `playlist-item:*` unsupported feature 和 warning。
- playlist 音频意图暂不打开全局 `runtime.allow_audio`；video 子项默认静音。

## 用户属性映射

| Wallpaper Engine 属性 | Gilder 属性 | 备注 |
| --- | --- | --- |
| color | `color` | 转为 CSS hex 或 float RGB 数组 |
| slider | `range` | 保留 min/max/default/step |
| bool | `bool` | 直接迁移 |
| combo | `choice` | 保留显示文本和值 |
| textinput | `text` | 直接迁移 |
| file | `file` | 限制到包内或用户显式选择 |
| directory | `file` 或后续 `directory` | 早期只记录 schema，不承诺运行时完整行为 |

## 转换报告

`metadata/conversion-report.json` 必须包含：

- `source_type`
- `detected_features`
- `converted_features`
- `unsupported_features`
- `copied_assets`
- `generated_assets`
- `warnings`
- `errors`

示例：

```json
{
  "source_type": "scene",
  "detected_features": ["audio-response", "image-layer", "scenescript", "shader", "timeline"],
  "converted_features": ["scene-lite"],
  "unsupported_features": [
    "audio-runtime",
    "custom-shader",
    "scene-runtime",
    "scenescript",
    "timeline-animation"
  ],
  "copied_assets": ["metadata/source-scene.json"],
  "generated_assets": ["assets/scene-lite.json", "previews/poster.svg", "previews/thumbnail.svg"],
  "warnings": ["Converted Scene project to a scene-lite fallback graph; original scene metadata was preserved at metadata/source-scene.json."],
  "errors": []
}
```

`detected_features` 现在会同时扫描 `project.json` 和可读取的入口文件内容。Web 项目会识别
`wallpaperPropertyListener`、`wallpaperRegisterAudioListener`、网络 URL 和媒体集成线索；
Scene 项目会识别 layer/image、timeline/keyframe、SceneScript、shader、particle、
parallax、playlist 和音频响应线索。该检测用于报告迁移缺口，不能替代后续原生 runtime。

## 版权与分发

转换器只处理用户本地已有资源。Gilder 包不应默认上传、重分发或修改许可证字段。`license = "unknown"` 时 UI 应提醒用户确认授权范围。

## 官方资料链接

- [Wallpaper Engine Scene Guide](https://docs.wallpaperengine.io/en/scene/overview.html)
- [Wallpaper Engine Web Guide](https://docs.wallpaperengine.io/en/web/overview.html)
- [Wallpaper Engine Web User Properties](https://docs.wallpaperengine.io/en/web/customization/properties.html)
