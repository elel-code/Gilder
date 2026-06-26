# Video Optimization Plan

M8 历史路线已经归档。当前优化只按 FFmpeg + native Vulkan Video 推进。

## Priorities

1. 对齐 FFmpeg PacketQueue/FrameQueue/keep_last/serial/clock。
2. 减少 copy：AVPacket borrow、bounded bitstream ring、decoded image GPU ownership、
   descriptor heap sampling。
3. 固定容量：queue=3、DPB/image pool、present ring、timeline/fence retire。
4. 三格式 H.264/H.265/AV1 都以真 4K/240fps 连续 smoke 验证。
5. `Private_Dirty < 25MiB` 和 `239.999+fps` 同时达标才算完成。

## Non-goals

- 不恢复 decoded-frame frontend。
- 不恢复旧 daemon video runtime CSV 作为证据入口。
- 不引回 descriptor set。
