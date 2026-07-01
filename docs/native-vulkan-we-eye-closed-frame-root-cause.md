# Native Vulkan WE Eye — 闭眼帧瞳孔可见根因

> **⚠ 本文档已更新。2026-07-01 MDLE 几何根因已撤销：**
> - 已撤销假设：`docs/native-vulkan-we-eye-mdle-inverse-bind-root-cause.md`
> - 当前根因：`docs/native-vulkan-we-eye-render-composite-root-cause.md`
>
> 本文档保留 iris/opacity 效应链路由的历史记录和实验证据。

本文件记录闭眼帧中瞳孔仍然显示（workshop scene `3742497499`）的代码级根因，
取代 `native-vulkan-we-eye-first-class-handoff.md` 中已被代码证伪的旧结论。

2026-07-01 更新：后续 HDMI-A-1 观察证明，把 iris 塞进当前
alpha shortcut 会重新造成整只眼丢失、错位或漂移。当前落点以
`native-vulkan-we-eye-first-class-handoff.md` 为准：`node-77` 直接画
puppet mesh；`node-89` 保持独立第二张 eye 图，并回到文档记录的稳定路径：
direct puppet mesh + material-UV opacity mask。不要再把当前半成品
local-target opacity 路径当成修复。

2026-07-01 更新二：闭眼调用链复查确认，旧的
`/tmp/gilder-we-3742497499-output-restored-placement` gscene 丢了原始
`models/眼睛_puppet.mdl` 的 MDLA opacity tail；原始 MDLA 在 transform tracks
后 5 字节处有 opacity block，最低值仍为 `0.265767`。运行时 loader 已增加
通用 backfill：不增加兼容字段；loader 通过节点的 `provenance.model.puppet`
匹配 packaged `role=we-puppet-mdl` / `original_source` 资源，并在 gscene 缺失
非默认 puppet opacity 时补回 clip frames。

2026-07-01 更新三：复查 `docs/native-vulkan-video.md` 的回退记录后，当前
工作目录已关闭 opacity local-target 尝试。正确的闭眼调用链应显示：
`node-77` 为 `direct-puppet-mesh`、`alpha_slot=None`；`node-89` 为
`we-opacity-effect-direct-puppet-mesh-material-uv`、`alpha_slot=Some(1)`，
且没有 `effect-target(index=0)` / final scene quad。

2026-07-01 更新四：上面“node-77 direct-puppet-mesh”的落点已被新的
iris first-class 修复取代。当前调用链是：`node-77` 先把 puppet mesh 画到
local effect target，再用 final scene quad 采样该 target 和 iris mask；
`node-89` 仍保持 direct puppet mesh + material-UV opacity mask。当前
HDMI-A-1 日志为 `/tmp/gilder-eye-iris-target-hdmi-a-1-20s-r2.log`，其中
`runtime.eye-overlap` 明确记录 `direct_base_swapchain=false`。以下旧根因仍
作为历史证据保留，但“effect chain 跳过”已经不是当前代码状态。

旧文档的核心错误：声称当前 renderer 可以安全把眼睛接进 first-class local
effect-target pass。HDMI-A-1 观察已经证明这个半成品路径会造成整眼漂移、
丢失或错乱；当前代码故意关闭该路径。以下每条根因都附带精确的文件路径、
行号与工具验证证据。

## 根因一（effect chain 跳过）

**文件**：`src/renderer/native_vulkan/scene/draw_pass.rs`  
**行号**：约 1565–1571（未提交，`git blame` 显示 `Not Committed Yet`）

```rust
fn native_vulkan_scene_effect_pass_uses_first_class_target(
    runtime: Option<&str>,
    effect_file: &str,
) -> bool {
    let _ = (runtime, effect_file);
    false
}
```

该函数现在故意不匹配 `"native-opacity-mask"` 或 `"native-iris-mask"`。
这会关闭当前半成品 local-target 路径，避免重现整眼漂移/丢失。

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
- opacity duplicate（source 1530，node-89）也保持 direct mesh + material-UV mask
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

## 根因四（1530 只影响自己的本地图像）

**文件**：`src/renderer/native_vulkan/scene/draw_pass.rs`

1530（opacity duplicate）现在按 direct puppet mesh 路径绘制，其 opacity
mask（`331×115`，R8）在 material UV 空间采样，只会改变 `1530` 这张图的
alpha。它不会、也不应该擦除 `node-77` 已经画到场景里的像素。若闭眼瞳孔
仍可见，下一步应继续查 `node-77` 的 iris/rest-bind/闭眼本地图像输出，而
不是隐藏或折叠 `1530`。

## 证据来源

- 源代码：`src/convert/wallpaper_engine.rs`、`src/core/scene.rs`、
  `src/renderer/native_vulkan/scene/draw_pass.rs`、`src/renderer.rs`
- SPIR-V 反汇编：`spirv-dis /tmp/frag.spv`（从
  `NATIVE_VULKAN_VULKANALIA_SCENE_FULL_SAMPLED_IMAGE_FRAGMENT_SPIRV` 提取）
- `git blame`：根因一所在函数为 `Not Committed Yet`（2026-07-01）
- `git diff`：效应链基础设施在工作目录中存在，但 iris 匹配条件未被加入

## 当前修复结论

不要把 `"native-iris-mask"` 接到旧的 alpha shortcut；当前修复是单独的
first-class iris target path：`node-77` local target base mesh -> iris final
scene quad。`"native-opacity-mask"` 仍保持 direct material-UV alpha mask
路径，避免重现文档记录的整眼漂移/丢失回退。

剩余风险：
1. 当前 native `mode=iris` shader 仍是简化 offset，不是完整 original
   `iris.vert` time/noise 常量语义。
2. `node-89` 必须继续保持独立本地 opacity pass，不隐藏、不折叠、不回退到
   opacity local-target shortcut。
