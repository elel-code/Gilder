# Native Vulkan Video Migration

当前迁移目标已经收敛为一条路线：

- FFmpeg frontend 负责 container demux、bitstream filter、AVPacket ownership、serial、
  clock/reference 语义。
- Gilder native Vulkan 负责 H.264/H.265/AV1 Vulkan Video decode、YUV sampling、render
  pass 和 Wayland/Vulkan present。
- Shader resource binding 使用 `VK_EXT_descriptor_heap`。descriptor set 路线不再存在。
- decoded-frame provider/importer 路线不作为当前主线保留。

## FFmpeg References

每个视频性能或 correctness 改动必须写明本地 FFmpeg 源码 reference：

- `references/ffmpeg/fftools/ffplay.c`: PacketQueue、FrameQueue、keep_last、serial、
  frame_timer 和 video_refresh。
- `references/ffmpeg/libavcodec/bsf.h`: bitstream filter send/drain contract。
- `references/ffmpeg/libavformat/av1dec.c`: AV1 frame merge reference。

## Success Gates

- H.264/H.265/AV1 任意入口连续 decode/present。
- 真 4K/240fps 源稳定 `239.999+`，不能因为 queue、async、present overlap 或 ring
  调整下降。
- `Private_Dirty < 25MiB`，并给出 smaps summary。
- `descriptor_sets=0`，descriptor model 为 `VK_EXT_descriptor_heap`。
- 所有 copy 都必须可命名：AVPacket borrow、bitstream ring upload、decoded-image handoff、
  render sampling、present。

## Verification

常规本地检查：

```sh
cargo fmt --check
cargo check --features native-vulkan-video --bin gilder-native-vulkan
cargo check --bin gilderd
cargo tree --features native-vulkan-video -i <deleted-video-frontend-dep>
```

最后一条如果返回 package 不存在，表示当前 feature 未引入已删除的视频前端依赖。

真实 Wayland smoke 使用 `native-vulkan-*-ready-prefix-video-smoke.sh`，并开启 validation
layer、performance snapshot 和真 4K/240fps 源。
