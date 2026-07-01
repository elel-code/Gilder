# 已撤销：闭眼帧瞳孔可见不是 MDLE0002 逆绑定缺失

本文件原先把 workshop scene `3742497499` 的闭眼帧瞳孔可见归因于
`models/眼睛_puppet.mdl` 中 `MDLE0002` 未作为 inverse-bind 矩阵解析。该结论
已于 2026-07-01 被新证据推翻。不要把本文档旧版本当作实现依据。

正确方向见 `docs/native-vulkan-we-eye-render-composite-root-cause.md`：当前问题在
render/effect composite 层，重点是 `node-77` 的 iris/effect pass 会在已经闭合
的眼睑几何之上重新采样/绘制瞳孔。

## 推翻证据

1. `MDLA` clip `730` 的第 `0` 帧局部变换精确等于 `MDLS` 局部矩阵，覆盖全部
   `54` 根骨骼，包括旧文档列为差异关键点的 root 直接子骨骼。蒙皮 pose 来自
   `MDLA`，静止帧要映射为单位矩阵，就必须使用 `MDLS` bind 系。
2. `MDLE0002` 对 `45/54` 根骨骼等于 `MDLS` 正向局部矩阵；逆绑定矩阵不应等于
   正向局部矩阵。旧文档把 `MDLE` 的正向/局部形态矩阵与 `inv(bind_world)` 的
   平移直接相减，得到的数百单位差异是伪差异。
3. 用当前 `MDLS` parent-chain 计算的 inverse-bind 渲染 `node-77` 单独输出时，
   睁眼帧正常露出瞳孔，闭眼帧眼睑会扫过并覆盖瞳孔。把 `MDLE` 当 inverse-bind
   会破坏静止帧单位映射，符合旧 handoff 记录中的“睁眼瞳孔消失、眉毛消失”
   失败现象。

## 实现结论

- 不要向 `SceneMeshSkinBone` 增加 `inverse_bind` 字段。
- 不要解析 `MDLE0002` 作为 puppet skinning 的 inverse-bind 输入。
- 不要删除当前运行时从 `MDLS` bind parent-chain 计算 inverse-bind 的路径。
- 后续如果重新研究 `MDLE0002`，必须先证明它的语义和 `MDLA` pose 空间一致；
  在此之前它不是 eye closed-frame 修复路径。

## 当前根因

`node-77` 的 puppet 几何已经能在闭眼帧闭合；Gilder 最终输出仍然显示带高光的
瞳孔，更符合 iris/effect pass 在最终合成阶段把瞳孔重新画回来的现象。修复应
集中在 first-class iris/effect composite、pass target、blend/effect UV 语义，
而不是 puppet 几何绑定。
