# 闭眼帧瞳孔可见：render/effect composite 根因

本文件记录 workshop scene `3742497499` 闭眼帧瞳孔可见的当前有效根因。
`docs/native-vulkan-we-eye-mdle-inverse-bind-root-cause.md` 已撤销：`MDLE0002`
不是已证明的 inverse-bind 输入，闭眼失败不应按 puppet 几何缺失修复。

代码级修复必须保持 first-class material/effect/pass 语义，不允许通过
sample-specific 隐藏、provenance-only runtime lookup、兼容字段或旧 opacity
shortcut 修复。

## 1. 当前结论

`node-77` 的 puppet 几何在当前 `MDLS`/`MDLA` skinning 下已经能闭合眼睑。单独
渲染 `node-77` 时，睁眼帧两个蓝色瞳孔完全露出，闭眼帧红褐色眼睑扫过并覆盖
瞳孔。Gilder 最终闭眼输出仍显示更清晰、带高光的瞳孔，说明问题发生在
render/effect composite 层：`node-77` 的 iris/effect pass 把瞳孔重新采样/绘制
到最终画面上。

旧版本把 `node-89` 的 opacity mask 路径列为主因，这只覆盖了真实缺口的一部分。
实测 node-89 的 opacity mask 在眼部/瞳孔区域大多为黑，node-89 不是把瞳孔重新
画回来的唯一或主要元凶。真正要建模的是 WE 的本地 effect pass 链和 final
composite 边界，尤其是 `node-77` 的 iris pass。

## 2. 已确认缺口

1. `"normal"` blend 未映射，材质 `"normal"` 会回退为 `SceneBlendMode::Alpha`。
   这会把应为 overwrite 的 pass 变成 alpha 混合。
2. `locktransforms=true` 只写入 provenance，未 lower 到
   `ScenePuppetAnimationLayer`，运行时 `sample_puppet_local_pose()` 不读取它。
3. opacity/iris mask 的 WE backing texture extents 未 first-class 保存；当前
   opacity mask UV scale 只能返回 identity stopgap。
4. `native-iris-mask` 仍是简化的 sampled-image shader 路径，未完整表达 WE 的
   local target、mask UV、pass ordering、final composite 和 alpha/blend 语义。

这些缺口都属于 render/effect composite 架构问题。它们应继续推进，但不依赖
MDLE inverse-bind 修复。

## 3. 推翻 MDLE 前置条件

- `MDLA` clip `730` frame `0` 等于 `MDLS` bind local；pose 和 bind 必须处于
  `MDLS` 系才能让静止帧得到单位 skinning。
- `MDLE0002` 对多数骨骼等于 `MDLS` 正向局部矩阵，不符合 inverse-bind 语义。
- 当前 computed inverse-bind 的 `node-77` 单独渲染已经能闭合眼睑；把 `MDLE`
  当 inverse-bind 会破坏睁眼状态。

因此后续计划不得再要求解析 `MDLE0002` 为 first-class inverse-bind，也不得保留
新旧两套 skinning 字段。

## 4. 修复路径

1. 先建立 `node-77` iris/effect composite 的 first-class 路径：本地目标、
   puppet base pass、iris mask/pass、final scene composite、pass labels、draw
   order和日志都要可见。
2. 增加显式 `normal` blend mode，并在 core scene、render-plan/draw-pass state、
   Vulkan blend equation 中映射为 `ONE / ZERO` overwrite。
3. 将 `locktransforms` lower 成 `ScenePuppetAnimationLayer` 的 first-class 字段。
   如果 WE 仍采样 opacity/material 通道，transform 锁定和非 transform 通道采样
   必须拆开，不要用 provenance fallback。
4. 保存 WE backing texture extents，并用它计算 iris/opacity mask UV scale。
   不要使用 decoded logical extents 代替 backing extents。
5. 保持 source `1530` / `node-89` 为独立后绘制 source；不要隐藏、折叠或按样本
   特判。

## 5. 验证

需要重新建立针对 `3742497499` 的证据日志：

- `node-77` 单独 base/effect/final pass 的 draw order、target、blend、alpha
  slot、mask slot、UV ranges。
- 闭眼帧中 iris pass 是否在眼睑几何之后重新输出瞳孔区域颜色。
- node-89 opacity mask 在瞳孔区域的 sampled mask coverage，证明它不是主重绘
  来源。
- HDMI-A-1 观察或同等 native Vulkan screenshot，对比睁眼帧、最低不透明度闭眼
  帧和最终 composite。
