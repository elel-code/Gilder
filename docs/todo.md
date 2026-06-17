# TODO

## M0: 项目骨架

- [x] 使用单个 Cargo package。
- [x] 提供 `gilderd`、`gilderctl`、`gilder-convert` 三个入口。
- [x] 定义基础 IPC socket 路径和命令。
- [x] 写入设计文档、格式文档、转换文档和 TODO。
- [x] 采用 `src/foo.rs` + `src/foo/` 的 Rust 模块组织方式。
- [x] 初始化 Git 仓库。
- [x] 添加 CI：`cargo fmt`、`cargo check`、`cargo test`。

## M1: 格式与加载器

- [x] 为 `manifest.gilder.json` 定义 Rust 数据结构。
- [x] 引入 `serde`、`serde_json`、`camino` 或等价路径处理。
- [x] 实现 `.gwpdir` 加载。
- [x] 实现路径逃逸校验。
- [x] 实现 preview、entry、variant 校验。
- [x] 添加 manifest schema 测试。
- [x] 添加示例静态壁纸包。

## M2: IPC 与状态

- [x] 用真实 JSON parser 替换当前占位字符串匹配。
- [x] 实现 JSON-RPC 错误响应。
- [x] 添加 `outputs`。
- [x] 添加 `properties set/get`。
- [x] 添加 `watch`。
- [x] 状态写入 `$XDG_STATE_HOME/gilder/state.json`。
- [x] 配置读取 `$XDG_CONFIG_HOME/gilder/config.toml`。
- [x] socket 权限和 stale socket 处理。
- [x] daemon 单实例检测。

## M3: GTK/Wayland 静态壁纸

- [ ] 引入 GTK-rs。
- [ ] 选择并接入 layer-shell 支持。
- [ ] 为每个输出创建 background layer 窗口。
- [ ] 实现静态图片解码和显示。
- [ ] 实现 fit mode：cover、contain、stretch、tile、center。
- [ ] 支持输出热插拔。
- [ ] 支持按 output 设置不同壁纸。

## M4: 视频壁纸

- [ ] 引入 GStreamer。
- [ ] 实现视频 entry 加载。
- [ ] 实现 loop、muted、poster。
- [ ] 实现 pause/resume/stop。
- [ ] 实现 max_fps 或 pipeline throttling。
- [ ] 验证 MP4/H.264、WebM/VP9、WebM/AV1。
- [ ] 添加 fullscreen 暂停策略接口。

## M5: 合成器适配

- [x] 定义合成器输出/桌面状态快照模型。
- [x] 定义 fullscreen、unfocused、battery 等性能策略决策层。
- [ ] 通用 GDK monitor 后端。
- [x] Hyprland IPC 后端。
- [x] niri IPC 后端。
- [x] 输出名称稳定映射。
- [x] 工作区/fullscreen 状态感知。
- [x] 配置中允许禁用特定适配器。

## M6: Wallpaper Engine 转换器

- [x] 解析 `project.json`。
- [x] 识别 image/video/web/scene/application 类型。
- [x] 静态图片转换到 `static-image`。
- [x] 视频转换到 `video`。
- [x] 复制 preview 为 poster 和 thumbnail。
- [ ] 缺失 preview 时从图片/视频生成 poster 和 thumbnail。
- [x] Web 项目复制与 bridge 注入。
- [x] 用户属性映射。
- [x] 生成 `metadata/conversion-report.json`。
- [x] 拒绝 executable/application 类型并输出清晰错误。
- [ ] Scene 子集转换到 `scene-lite`。

## M7: 打包与发布

- [x] 实现 `.gwp` 打包。
- [x] 实现 `.gwp` 解包或只读读取。
- [ ] 添加 man page。
- [ ] 添加 systemd user service 示例。
- [ ] 添加 shell completions。
- [ ] 准备发行包脚本。
