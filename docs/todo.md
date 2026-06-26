# TODO

当前主线只保留 FFmpeg frontend + native Vulkan Video decode/render/present。

## Video

- [ ] 对齐本地 FFmpeg 源码的 packet queue、frame queue、keep-last、serial、clock 和
  refresh/pacing 语义。
- [ ] 三种格式 H.264/H.265/AV1 均保持任意连续解码，真实 4K/240fps 源稳定
  `239.999+`。
- [ ] `Private_Dirty` 目标降到 25MiB 以下；所有保留队列、bitstream ring、DPB/image
  pool 和 upload buffer 都必须有 bounded ownership 证据。
- [ ] 删除 decoded-frame/provider/importer 旧路线；CPU frame copy 只允许作为显式失败证据，
  不能作为正常路径。
- [ ] descriptor model 必须保持 `VK_EXT_descriptor_heap`，不得引回 descriptor set。
- [ ] 对 H.264/H.265/AV1 分别记录 FFmpeg 源码 reference、迁移前 4K/240 基线、
  native Vulkan smoke summary 和 smaps/Private_Dirty 证据。

## Runtime

- [ ] 继续压缩 copy 成本：AVPacket payload 借用、bounded bitstream ring、固定 image
  pool、timeline/fence retire、present ring 和 queue=3 背压都要与 FFmpeg 语义一致。
- [ ] 音频/clock 后续也走 FFmpeg-style serial 和 master clock；视频路径不得因此引入
  decoded frame copy。
- [ ] 性能脚本以 native Vulkan smoke/runtime summary 为准，daemon 旧 video runtime CSV
  不再作为证据入口。

## Cleanup

- [ ] 每次删除旧代码后跑残留搜索：旧 feature 名、deleted scripts、
  decoded-frame frontend、descriptor set。
- [ ] 删除或重写旧文档和脚本，避免历史路线被当成当前架构。
- [ ] 通过 `cargo fmt`、`cargo check --features native-vulkan-video --bin
  gilder-native-vulkan`、`cargo check --bin gilderd` 后再做 smoke。
