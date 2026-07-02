# 闭眼帧瞳孔可见：iris effect UV / pass graph 根因

本文件是 workshop scene `3742497499` 闭眼帧瞳孔可见的**当前有效根因**。
旧交接、MDLE 和不完整 composite 诊断文档已删除；源语义证据保留在
`docs/native-vulkan-we-eye-original-semantics.md`。

---

## 1. 根因

**Gilder 的 iris effect 在眼睑区域错误地应用了偏移采样，从 EffectTarget FBO
的瞳孔区域取了颜色，替换掉闭眼帧中本该显示的眼睑像素。**

### 1.1 触发机制

Gilder 的 `mode=iris` sampled-image fragment shader 核心逻辑
（SPIRV 反汇编：`NATIVE_VULKAN_VULKANALIA_SCENE_FULL_SAMPLED_IMAGE_FRAGMENT_SPIRV`）：

```spirv
%259 = OpFunctionCall %v2float %iris_motion_offset_     // 计算 iris motion 偏移
%260 = OpLoad %float %mask_0                             // 加载 alpha mask 值
%261 = OpVectorTimesScalar %v2float %259 %260            // offset = motion * mask
OpStore %iris_offset %261
%265 = OpLoad %v2float %v_uv                             // 当前 pass-space UV
%267 = OpFAdd %v2float %265 %266                         // uv + offset
%268 = OpImageSampleImplicitLod %v4float %263 %267       // 在偏移位置采样 g_Texture0
OpReturnValue %268                                       // 返回 → 替换当前像素
```

其中 `mask_0` 通过对 `v_effect_uv` 调用 `raw_alpha_mask()` 获得——即采样
iris mask texture（resource-175, 331×115 R8）。

### 1.2 为什么眼睑区域 mask_0 不是 0

旧运行时把 iris/opacity mask 的 effect UV 当作 identity (1.0, 1.0) 使用；
这会让 331×115 的 iris mask 被当成 663×230 的 eye pass 空间采样。眼睑区域的
effect UV 因此落到 iris mask 的非零中央带；实测该非零带在眼睛 UV 中约为
x[0.47,0.76]、y[0.10,0.91]，会覆盖闭合眼睑片元并触发 iris offset。

这里不能把“正确修复”简化成 alpha/base 的 331/663、115/230 均匀缩放。该
比例会把 mask 有效区移到错误半区，离真实瞳孔位置 x≈0.63 很远，导致睁眼帧
iris 视差落空。正确方向是 converter 保存 WE 原始 pass/effect UV 变换、backing
extent 和 slot 语义，runtime 只消费这些 typed records。

### 1.3 闭合眼睑如何被瞳孔颜色替换

1. Base mesh pass 将 puppet 几何渲染到 EffectTarget（闭眼帧中，眼睑正确覆盖瞳孔）
2. Final scene quad 以 `mode=iris` shader 采样 EffectTarget + iris mask
3. 眼睑像素处：`mask_0 > 0`（UV 错位导致）→ `offset = motion * mask ≠ 0`
4. 偏移后的 UV 指向 EffectTarget 中紧邻的瞳孔区域
5. 瞳孔颜色被采样回来，**完全替换**当前眼睑像素
6. 因为 offset 随像素和时间变化，不同像素被替换的程度不同 → 斑驳的"半透明"视觉效果

---

## 2. 已推翻的假设

### 2.1 MDLE0002 inverse-bind 缺失（已推翻）

旧 MDLE 诊断把 `MDLE0002` 当成逆绑定矩阵，并把闭眼失败归因于
inverse-bind 缺失。该方向已被三条证据推翻：

1. MDLA clip 730 frame 0 的局部变换精确等于 MDLS 局部矩阵。蒙皮 pose 来自 MDLA
   系，静止帧必须 `pose × inv(bind) = I`，只能使用 MDLS bind。
2. MDLE 对 45/54 骨骼等于 MDLS 正向局部矩阵——逆绑定矩阵不应等同正向矩阵。
   旧对比中数百单位的差异是正向 vs 逆向的伪差异。
3. 当前 computed inverse-bind 的 node-77 单独渲染已能闭合眼睑。把 MDLE 当
   inverse-bind 会破坏睁眼状态，表现为睁眼时瞳孔/眉毛消失。

MDLE 不是闭眼帧修复路径；不要为该 bug 增加 MDLE/inverse-bind 字段或分支。

### 2.2 node-89 opacity mask 重绘瞳孔（已降级）

node-89 的 opacity mask 在眼部/瞳孔区域 66-72% 为黑（mask=0），node-89 在瞳孔
区域基本不可见。它不是把瞳孔画回来的主要元凶。`SceneBlendMode::Normal` 和
`locktransforms` 是真实 first-class 语义缺口，但不是闭眼帧瞳孔可见的主因。

---

## 3. 附带缺口

以下代码缺陷在分析过程中确认存在。`normal` blend 和 `locktransforms` 可以独立
推进，但它们不是闭眼帧瞳孔可见的主因；主修复仍是 first-class WE effect-UV
语义。

| 缺口 | 位置 | 影响 |
|------|------|------|
| `normal` blend | core/draw-pass/Vulkan blend state | WE Normal(ONE/ZERO) 替换语义必须显式建模 |
| `locktransforms` provenance-only | puppet animation layer state | transform 通道和 opacity/material 通道必须拆开采样 |
| mask UV identity | effect UV transform | iris/opacity mask 都不能靠 identity 或 decoded extent 比例猜测 |

---

## 4. 修复

### 4.1 主修复：first-class WE effect-UV transform

必须在 converter 中把 WE 原始 effect-UV 语义 first-class 落到 gscene：

1. 保存 WE 原始 backing texture extents 与 effect/material pass 的 UV 变换输入。
2. 表达完整 effect-UV transform，而不是只有 scale；必要时包含 offset、pass-space
   到 mask-space 的映射、texture region 和 backing/logical extent 区分。
3. runtime 只消费这些 first-class 记录，不再从 decoded logical extents 临时推断。
4. iris 与 opacity 共享的 mask UV 逻辑必须走同一套 typed effect-UV transform，
   但各自的 shader 语义和 alpha 输出语义仍保持独立。
5. `.gscn` binary 必须携带这些 typed records；runtime 不能回读 `.gscene.json`
   或保留大块 JSON 派生 payload。

明确禁止把下面这种比例作为修复落地：

```rust
scale = alpha_decoded_extent / base_decoded_extent
```

该比例可以作为证据实验记录，但不是 renderer 修复。它会把本场景的 iris mask
有效区挪到错误半区，破坏睁眼帧 iris 视差。

### 4.2 2026-07-02 当前修复进展

`node-77-models-json` 的 graph 现在不再被折叠成旧的 scene-final iris quad：

1. step 0: base puppet mesh -> `first-class-effect-target` target 56
2. step 1: `iris` -> `image-local-sub` target 57
3. step 2: `waterripple` -> scene

同时修正了一个新的残缺成因：把 iris 从 scene-final pass 移到
image-local-sub pass 后，中间 pass 的 quad 曾经退回 identity effect UV
`u/v=0..1`。这会让 331×115 iris mask 在 663×230 eye pass 上采样错误区域，
表现为眼睛图案被裁掉、错位或只剩局部色块。当前中间 iris pass 已恢复 converter
保存的 effect-UV transform，运行时证据为 `u=0..2.003, v=0..2`，与旧 final pass
一致。

这条修复仍是同一主线：effect/material/FBO graph 和 effect UV 都必须 first-class。
不是二进制丢信息，也不是临时隐藏图层。

### 4.3 验证标准

修复后必须同时满足：

1. 闭眼帧：`node-77` eyelid 几何闭合后，iris final pass 不再把瞳孔区域采样回
   眼睑片元。
2. 睁眼帧：iris 视差仍落在真实瞳孔区域，不能因为错误的 0.5 缩放而偏到左/右
   半区。
3. 日志能显示 final pass 的 effect-UV transform、mask sampled range、texture
   slot、target、blend=normal 和 draw order。

---

## 5. 验证

- SPIRV 反汇编：`spirv-dis /tmp/sampled_image_frag.spv` — 确认 iris offset 逻辑
- iris mask 值分析：`scripts/check_mask_values.py`（需扩展 iris mask）
- node-77 单独渲染对照：验证几何正确闭合
- Gilder 实际输出对照：验证 iris pass 重绘瞳孔
- 修复后验证：`gilder-native-vulkan --run-scene --output-name HDMI-A-1`，
  闭眼帧瞳孔应不可见，睁眼帧 iris 视差仍应对齐瞳孔
