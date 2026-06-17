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

当前实现支持静态图片、视频、Web 和保守 `scene-lite` 项目的 `.gwpdir` 输出；application/executable 项目会生成转换报告并拒绝转换。缺失预览图时，静态图片项目会从源图生成 poster/thumbnail，视频项目会优先通过本机 `ffmpeg` 从首帧生成 poster/thumbnail，失败时回退到 metadata-based SVG fallback；Scene 项目会生成 metadata-based SVG fallback。

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

## 类型映射

| Wallpaper Engine 类型 | Gilder 类型 | 支持等级 | 策略 |
| --- | --- | --- | --- |
| Image / Scene from image | `static-image` 或 `scene-lite` | 高/中 | 纯图片无损复制；含效果时转 scene-lite 子集或静态 fallback |
| Video | `video` | 高 | 复制可播放视频；必要时转码；生成 poster |
| Web | `web` | 中 | 复制 HTML/CSS/JS/资源；注入兼容 bridge；默认禁网 |
| Scene | `scene-lite` / `video` / `static-image` | 低到中 | 复制 Scene 入口元数据并生成 fallback；复杂效果记录为 unsupported |
| Application / executable | 不支持 | 无 | 拒绝转换，仅生成报告 |

## 静态图片转换

适合情况：

- Wallpaper Engine 项目只有单张图片。
- Scene 项目只包含一个背景图且没有动画/effect。

输出：

- 原图复制到 `assets/`。
- 预览复制或生成到 `previews/`；缺失 preview 时从源图复制生成 poster 和 thumbnail。
- `entry.type = "static-image"`。
- `fit` 根据源项目 alignment/scaling 映射，无法识别时使用 `cover`。

可选优化：

- 生成 AVIF/WebP variant。
- 保留原图作为无损源。
- 为常见比例生成裁剪 variant，如 16:9、21:9、9:16。

## 视频转换

适合情况：

- Wallpaper Engine video wallpaper。
- Web 项目或 Scene 项目中可识别出主循环视频。

输出：

- 视频复制到 `assets/`。
- poster 复制；缺失 preview 时优先调用 `ffmpeg` 从第一帧生成 `previews/poster.jpg` 和 `previews/thumbnail.jpg`，如果 `ffmpeg` 不在 `PATH` 或解码失败，则生成 SVG fallback 并在转换报告写入 warning。
- `entry.type = "video"`。
- 默认 `loop = true`、`muted = true`。

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
- 用户属性转为 Gilder `properties`。
- 生成 `assets/web/gilder-bridge.js`，提供基础属性桥接，并在后续 Web runtime 中适配常见 `window.wallpaperPropertyListener.applyUserProperties` 行为。

限制：

- 默认禁止网络请求。
- 默认禁止访问包根之外的本地文件。
- 音频可视化、媒体集成、RGB 硬件接口先记录为 unsupported feature。
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
- 自定义 shader。
- 复杂粒子、音频响应、RGB 设备联动。
- 3D model 行为。

Scene 转换策略按优先级：

1. 当前先生成保守 `entry.type = "scene-lite"`，复制 Scene 入口文件到 `assets/`。
2. 如果项目提供 preview，则作为 `fallback`；缺失时生成 SVG fallback。
3. SceneScript、shader、复杂粒子和音频响应记录为 unsupported，不执行也不翻译。
4. 后续如果能识别主要视频或图片，可降级为 `video` 或 `static-image`。

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
  "detected_features": ["image-layer", "timeline", "scenescript"],
  "converted_features": ["image-layer", "timeline"],
  "unsupported_features": ["scenescript"],
  "warnings": ["SceneScript was not executed or converted."],
  "errors": []
}
```

## 版权与分发

转换器只处理用户本地已有资源。Gilder 包不应默认上传、重分发或修改许可证字段。`license = "unknown"` 时 UI 应提醒用户确认授权范围。

## 官方资料链接

- [Wallpaper Engine Scene Guide](https://docs.wallpaperengine.io/en/scene/overview.html)
- [Wallpaper Engine Web Guide](https://docs.wallpaperengine.io/en/web/overview.html)
- [Wallpaper Engine Web User Properties](https://docs.wallpaperengine.io/en/web/customization/properties.html)
