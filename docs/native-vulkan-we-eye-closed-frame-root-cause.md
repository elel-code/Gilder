# Native Vulkan WE Eye — 闭眼帧瞳孔可见根因

本文件记录闭眼帧中瞳孔仍然显示（workshop scene `3742497499`）的代码级根因，
取代 `native-vulkan-we-eye-first-class-handoff.md` 中已被代码证伪的旧结论。

2026-07-01 更新：后续 HDMI-A-1 观察证明，把 iris 或 opacity 继续塞进
当前半成品 effect-target/alpha shortcut 会重新造成整只眼丢失、错位或漂移。
当前落点以 `native-vulkan-we-eye-first-class-handoff.md` 为准：`node-77` 直接
画 puppet mesh，`node-89` 作为第二张 eye 图直接画 puppet mesh，并用 material
UV 采样 opacity mask 乘自己的 alpha。

旧文档的核心错误：声称 `native-iris-mask` 已被分类为 first-class local
effect-target pass。代码（包括工作目录中未提交的修改）从未实现这一点。
以下每条根因都附带精确的文件路径、行号与工具验证证据。

## 根因一（effect chain 跳过）

**文件**：`src/renderer/native_vulkan/scene/draw_pass.rs`  
**行号**：1542–1548（未提交，`git blame` 显示 `Not Committed Yet`）

```rust
fn native_vulkan_scene_effect_pass_uses_first_class_target(
    runtime: Option<&str>,
    effect_file: &str,
) -> bool {
    matches!(runtime, Some("native-opacity-mask"))
        || native_vulkan_scene_effect_file_is_opacity_mask(effect_file)
}
```

该函数只匹配 `"native-opacity-mask"`，不匹配 `"native-iris-mask"`。

而 iris 的 runtime 已在 `src/core/scene.rs:2981-2982` 正确设置为 `"native-iris-mask"`：

```rust
if file == "effects/iris/effect.json" || … {
    return Some("native-iris-mask".to_owned());
}
```

后果：

- `image_effect_pass_count` 对 base eye（source 1336，node-77）= 0
- `native_vulkan_scene_sampled_image_needs_we_effect_chain()` 返回 `false`
- base eye 走 else-if 分支（`draw_pass.rs:1478`），mesh 直接渲到 swapchain
- 无 effect-target FBO，无 iris pass，无 waterripple pass

## 根因二（opacity 最低值 ≠ 0）

**文件**：`src/convert/wallpaper_engine.rs:5146–5161`

`scene_parse_puppet_animation_opacity_tracks()` 从 MDLA f32 tail track 解析
per-bone opacity。bone 22 在帧 300 附近的最低值为 **~0.266**，不是 0.0。

**文件**：`src/core/scene.rs:3474–3489`

skinning 时加权平均各骨骼的 `pose.opacity` 到 vertex opacity：

```rust
local_pose.get(bone_index)
    .map(|pose| pose.opacity.clamp(0.0, 1.0) * weight)
```

bone 22 的 0.266 被传播到受影响顶点的 `SceneMeshVertex.opacity`。

**文件**：fragment shader SPIR-V（`spirv-dis /tmp/frag.spv` 已验证）

```spirv
%231 = OpLoad %float %v_opacity
%233 = OpLoad %float %232              ; color_1.a
%234 = OpFMul %float %233 %231         ; color_1.a *= v_opacity
```

`v_opacity = 0.266` 被乘到输出 alpha。0.266 ≠ 0 → 27% 不透明度 → 瞳孔可见。

## 根因三（bone 22 只覆盖 5% 顶点）

**文件**：`src/core/scene.rs:3480–3483`

54 根骨骼中仅 bone 22 的 MDLA opacity 非 1.0。bone 22 影响的顶点 UV 范围
约为 `u=0.250..0.311, v=0.104..0.243`。瞳孔核心 UV 很可能不在此范围内。

其余 95% 顶点 opacity = 1.0 → 完全不透明。

## 根因四（1530 opacity mask UV 映射失配）

**文件**：`src/renderer/native_vulkan/scene/draw_pass.rs:1542–1548`

1530（opacity duplicate）的 effect chain 被正确触发（runtime 匹配
`"native-opacity-mask"`），但其 opacity mask（`331×115`，R8）在
effect pass 中是按 pass-space 四边形采样的，mask UV 坐标与
base eye mesh 的顶点 UV 不匹配，导致 mask 无法擦除 node-77
已绘制的瞳孔像素。

## 证据来源

- 源代码：`src/convert/wallpaper_engine.rs`、`src/core/scene.rs`、
  `src/renderer/native_vulkan/scene/draw_pass.rs`、`src/renderer.rs`
- SPIR-V 反汇编：`spirv-dis /tmp/frag.spv`（从
  `NATIVE_VULKAN_VULKANALIA_SCENE_FULL_SAMPLED_IMAGE_FRAGMENT_SPIRV` 提取）
- `git blame`：根因一所在函数为 `Not Committed Yet`（2026-07-01）
- `git diff`：效应链基础设施在工作目录中存在，但 iris 匹配条件未被加入

## 修复（结论）

在 `native_vulkan_scene_effect_pass_uses_first_class_target` 中增加
`"native-iris-mask"` 分支是使 effect chain 执行的必要条件，但仅此不够。

瞳孔消失需要：
1. bone 22 的 MDLA opacity 能驱动到接近 0（或与 opacity mask 联动）
2. effect chain 正确执行 iris pass，产生正确的本地图像
3. 1530 的 opacity mask UV 映射到正确的 pass-space 坐标
