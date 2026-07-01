# 闭眼帧瞳孔可见：渲染合成层根因

本文件记录 workshop scene `3742497499` 闭眼帧瞳孔可见的**渲染合成层根因**。
几何层根因见 `docs/native-vulkan-we-eye-mdle-inverse-bind-root-cause.md`。

代码级缺口已确认（2026-07-01）。实现计划必须保持 first-class material/effect
语义，不允许通过兼容字段、provenance-only runtime lookup、sample-specific
分支或旧 opacity shortcut 修复。

---

## 1. 概述

原始 WE 的 node-1530 通过 **opacity effect pass chain** 实现多 pass FBO
合成：material pass → 本地 FBO → opacity effect pass（全屏四边形，Normal blend
写入 scene）。Gilder 的 node-89 以 **direct puppet mesh + material UV mask +
Alpha blend** 路径替代，存在三个独立缺陷：

1. `"normal"` blend 未映射，回退为 Alpha blend — **已确认**
   (`src/core/scene.rs`)
2. `lock_transforms=true` 未进入 puppet animation runtime — **已确认**
   (`src/core/scene.rs`)
3. opacity mask UV backing extent 未 first-class 表达，当前只能 identity
   stopgap — **已确认** (`src/renderer/native_vulkan/present/render_plan.rs`)

三者叠加导致 node-89 无法正确执行 WE 的"选择性替换"语义。

---

## 2. 证据

### 2.1 纹理 alpha

眼睛纹理（663×230 PNG，提取自 `materials/眼睛.tex`）眼睑区域 alpha 分布：

```
眼睑区域 (顶部 80/230 行，v>0.65):
  非零像素: 5185 (占 9.8% 面积，其余为透明背景)
  alpha min=1, max=255, median=255, avg=180
  alpha>=64:   83.0%
  alpha>=128:  63.5%
  alpha>=255:  53.8%
```

约一半的眼睑像素 alpha < 255。这是 WE 纹理的**故意设计**——多 pass FBO
alpha 累积会使最终合成中 alpha 更接近于所需值。验证脚本: `scripts/check_eye_alpha.py`

### 2.2 Opacity mask 值

`resource-206-opacity-mask-d2f87f99-frame-0.gtex`, 331×115 R8：

```
整体: mask=0: 68%, mask=255: 18%, 其他: 14%
眼睑区域 (顶部 40 行): mask=0: 72%, mask=255: 14%, 其他: 15%
瞳孔区域 (行 40-74):  mask=0: 66%, mask=255: 18%, 其他: 16%
```

眼睑区域 **72% mask=0**（应保留 node-1336），14% mask=255（应替换为 node-1530）。

验证脚本: `scripts/check_mask_values.py`

### 2.3 Blend 模式映射错误 [已确认]

WE 的 opacity 材质 blending = **"normal"**。CWE 参考实现将其映射为 **ONE / ZERO**（直接替换）。

Gilder 的 `scene_blend_mode_from_material()`（`src/core/scene.rs:4419-4435`）：
```rust
.find_map(|blending| match blending.to_ascii_lowercase().as_str() {
    "additive" | "add" => Some(SceneBlendMode::Additive),
    "multiply" => Some(SceneBlendMode::Multiply),
    "screen" => Some(SceneBlendMode::Screen),
    _ => None,  // ← "normal" 落入此分支，回退为默认 Alpha
})
```

`SceneBlendMode` 枚举中没有 `Normal` 变体。`"normal"` 回退为
`SceneBlendMode::Alpha`（SRC_ALPHA / ONE_MINUS_SRC_ALPHA for both color and alpha）。

**代码追踪确认**：
- node-89 的 snapshot layer 在 `push_sampled_image_snapshot_layers()`
  (`src/core/scene.rs:1203-1254`) 中通过 `scene_blend_mode_from_properties()`
  → `scene_blend_mode_from_material()` → `None` → `unwrap_or_default()` →
  `SceneBlendMode::Alpha`（第 1217, 1244 行）
- 该 `blend_mode` 直接传递到 `NativeVulkanSceneSampledImageQuad` 的
  `material_pass.render_state.blend.mode`
  (`src/renderer/native_vulkan/scene/draw_pass.rs:543-545`)
- Vulkan pipeline 最终使用 SRC_ALPHA / ONE_MINUS_SRC_ALPHA

### 2.4 lock_transforms 未实现 [已确认]

`src/convert/wallpaper_engine.rs:5666-5667` 将 `locktransforms=true` 写入
provenance metadata，但运行时**完全不使用**该字段。

**代码追踪确认**：
- `ScenePuppetAnimationLayer`（`src/core/scene.rs:4060-4074`）所有字段：
  `clip_id, name, additive, blend, visible, rate, initial_phase`
  — 无 `lock_transforms` 字段
- `sample_puppet_local_pose()`（`src/core/scene.rs:3619-3637`）对每个
  animation layer 无条件执行动画采样（第 3629-3637 行的 lerp/additive_blend），
  无 `lock_transforms` 检查
- 搜索结果：`lock_transform|lock_transforms` 在 `src/` 下仅在
  `convert/wallpaper_engine.rs`（写入 provenance）和测试文件中出现，
  渲染代码中零引用

### 2.5 Mask UV 缩放 [已确认]

`src/renderer/native_vulkan/present/render_plan.rs:489-499`:
```rust
pub fn native_vulkan_scene_opacity_effect_material_uv_scale(
    _base_width: Option<u32>, _base_height: Option<u32>,
    _alpha_width: Option<u32>, _alpha_height: Option<u32>,
) -> (f64, f64) {
    // decoded logical extents are not WE backing extents
    (1.0, 1.0)
}
```

原始 WE opacity.vert 的正确缩放：
```glsl
v_TexCoord.zw = vec2(
    v_TexCoord.x * 331.0 / 663.0,   // = 0.5
    v_TexCoord.y * 115.0 / 230.0);  // = 0.5
```

Gilder 当前强制 `(1.0, 1.0)` 是 stopgap：现有 `SceneTextureSlot` 尺寸是 decoded
logical extents，不是 WE backing texture extents。直接用这些字段改成 `331/663`
会重现之前记录的错误半尺寸采样。正确修复是先把 backing extent 作为
first-class texture/effect-pass 输入保留下来，再按 backing extent 计算 mask UV。

**代码追踪确认**：
- 该函数通过 `draw_pass.rs:78` import 供给所有调用者
- node-89 的 direct mesh 路径通过
  `native_vulkan_scene_opacity_effect_uv_space_from_snapshot_layer()`
  (`draw_pass.rs:417-432`) → `native_vulkan_scene_opacity_effect_material_uv_scale_for_scene_slots()`
  → `native_vulkan_scene_opacity_effect_material_uv_scale()` 使用该 scale
- 返回的 `NativeVulkanSceneEffectUvSpace` 传递给 fragment shader 作为
  `v_effect_uv`，用于 alpha mask 纹理采样

---

## 3. 数学证明

### 3.1 符号定义

| 符号 | 含义 | 典型值 |
|------|------|--------|
| T_rgb | 纹理颜色 | eyelid_color |
| T_a | 纹理 alpha | ~0.7 (avg 180/255) |
| V_a | vertex opacity | 1.0 (上眼睑) |
| M | opacity mask 值 | 0 或 1 |
| src_a | fragment 输出 alpha | T_a × V_a × M |
| src_rgb | fragment 输出颜色 | T_rgb |

### 3.2 原始 WE 公式（node-1530）

```
Pass 1: material (translucent = SRC_ALPHA / ONE_MINUS_SRC_ALPHA) → local FBO, clear=透明
  FBO_a   = (T_a × V_a)² = T_a²   (V_a=1)
  FBO_rgb = T_rgb × T_a × V_a = T_rgb × T_a

Pass 2: opacity (Normal = ONE / ZERO) → scene FBO
  scene_a   = FBO_a × M × 1 + dst_a × 0 = T_a² × M
  scene_rgb = FBO_rgb × 1 + dst_rgb × 0 = T_rgb × T_a

M=0: scene_a=0     → 完全透明 → node-1336 内容保留
M=1: scene_a=T_a²  → 0.49     → node-1530 覆盖 node-1336

控制语义: mask 在 node-1336(闭眼) 和 node-1530(睁眼,lock_transforms) 之间切换
```

### 3.3 Gilder 当前公式（node-89）

```
Node-89: direct puppet mesh → swapchain (Alpha blend = SRC_ALPHA / ONE_MINUS_SRC_ALPHA)
  src_a = T_a × M

  scene_a   = src_a × src_a + dst_a × (1 - src_a)
            = (T_a × M)² + dst_a × (1 - T_a × M)

  scene_rgb = src_rgb × src_a + dst_rgb × (1 - src_a)
            = T_rgb × T_a × M + dst_rgb × (1 - T_a × M)

M=0: scene_a=dst_a, scene_rgb=dst_rgb → node-77 保留（NOT 擦除）
M=1: scene_a=T_a²+dst_a×(1-T_a) → 两 layer 混合，透明度稀释

控制语义: mask 仅控制混合权重；lock_transforms 未实现 → 两 layer 动画相同
```

### 3.4 差异对比

| 条件 | WE (Normal=ONE/ZERO) | Gilder (回退 Alpha blend) |
|------|---------------------|---------------------------|
| M=0 | duplicate pass 透明；是否保留 previous scene 由正确 pass-target/final-composite 边界决定 | direct alpha path 仅按混合权重保留 dst |
| M=1 | node-1530 覆盖 (scene_a=T_a²) | node-89 与 node-77 **混合** |
| 语义 | local pass chain + final composite 的选择语义 | 双 layer **混合** |

### 3.5 lock_transforms 对差异的放大

若 lock_transforms 正确实现（node-89 使用 bind pose 无动画）：

```
Frame 300 (闭眼):
  node-77 (1336): animated → 眼睑覆盖瞳孔
  node-89 (1530): lock_transforms → 始终睁眼

Mask 控制:
  M=0 → scene = node-77 → 闭眼，瞳孔不可见 ✓
  M=1 → scene = node-89 → 睁眼，瞳孔可见

Mask 动画: 随时间渐变 M=1→0，实现平滑闭眼过渡
```

Gilder 当前：node-77 和 node-89 动画相同 → mask 毫无作用。

### 3.6 与纹理 alpha 的交互

WE 中即使 T_a < 1.0，Pass 1 的 translucent blend 将 alpha 平方（T_a²），
再乘以 mask。但即使 T_a²=0.49，在 Normal(ONE/ZERO) blend 下，眼睑仍然会
以 alpha=0.49 覆盖 scene，导致半透明 — 这是 WE 的**有意设计**：
半透明眼睑 + 背景混合 = 自然渐变。

Gilder 中 Alpha blend 进一步稀释透明度，使眼睑更透明。

---

### 3.7 实际像素值模拟 [已验证]

代入真实纹理采样值进行端到端模拟（脚本: `scripts/eye_pixel_simulation.py`）：

**输入数据**（从 `materials/眼睛.tex` 提取）：
- 眼睑平均: RGB=(0.583,0.558,0.559), T_a=0.707
- 瞳孔平均: RGB=(0.752,0.668,0.666), T_a=0.612
- 身体肤色: RGB=(0.92,0.85,0.78), bg_a=1.0

```
=== 模拟: 闭眼帧眼睑区域像素 ===

                Mask=0.0      Mask=0.5      Mask=1.0
                ────────      ────────      ────────
WE (lt=True):
  scene_a       0.000         0.187         0.374
  final RGB     (0.920,       (0.834,       (0.748,
                 0.850,        0.767,        0.685,
                 0.780)        0.710)        0.641)

Gilder (lt=False, Alpha):
  scene_a       0.750         0.610         0.719
  final RGB     (0.730,       (0.747,       (0.695,
                 0.679,        0.697,        0.654,
                 0.636)        0.656)        0.627)

Fixed (lt=True, Normal):
  scene_a       0.000         0.306         0.612
  final RGB     (0.920,       (0.869,       (0.817,
                 0.850,        0.794,        0.739,
                 0.780)        0.745)        0.710)
```

**关键发现**：

1. **mask 在 Gilder 中完全无效**：M=0/0.5/1.0 产生的 scene_a 分别为
   0.750/0.610/0.719，差异仅 0.14。因为 lock_transforms 未实现，
   node-77 和 node-89 动画相同，mask 在两个相同的渲染之间切换无意义。

2. **眼睑始终半透明**：Gilder 的 scene_a 始终在 0.61-0.75 范围。
   纹理 alpha 0.707 经过三次 Alpha blend（EffectTarget → final quad →
   node-89 direct mesh）后被稀释，从未接近 1.0。

3. **与 WE 的语义差异**：WE 中 opacity pass 处在 local pass chain 与 final
   composite 边界内，M=0/M=1 是 source-local 选择语义；Gilder 当前的 direct
   alpha path 只是在已存在 scene 上叠加第二张 mesh，无法表达该边界。

4. **叠加几何根因后的完整现象**：
   - MDLE 缺失：眼睑仅覆盖瞳孔 25% → 75% 瞳孔直接暴露于 node-77 的 FBO
   - Alpha blend 链：覆盖的 25% 眼睑也是半透明 (scene_a≈0.7)
   - lt 未实现：node-89 的 mask 切换无效
   - **结果**：75% 瞳孔直接可见 + 25% 半透明眼睑覆盖 = 用户观察到的
     "瞳孔始终半透明可见"

5. **修复后的预期**（Fixed 行）：
   - M=0/M=1 必须在 source-local pass chain 和 final composite 边界内验证；
     不再用 direct mesh alpha blend 近似。
   - `lock_transforms` 生效后，node-89 才能与 node-77 产生不同 pose 输入。
   - MDLE inverse-bind 修复后，node-77 base-eye local output 才有足够闭眼
     几何覆盖量。

---

## 4. 修复路径

### 4.1 增加 Normal blend mode

```rust
// src/core/scene.rs — SceneBlendMode 枚举
pub enum SceneBlendMode {
    Alpha,
    Additive,
    Multiply,
    Screen,
    Max,
    Normal,  // ← 新增
}
```

映射：
```rust
// scene_blend_mode_from_material()
"normal" => Some(SceneBlendMode::Normal),
```

Vulkan blend equation（需同时在 `blend.rs` 和 `present.rs` 中添加）：
```rust
SceneBlendMode::Normal => NativeVulkanSceneBlendEquation {
    src_color: One,
    dst_color: Zero,
    color_op: Add,
    src_alpha: One,
    dst_alpha: Zero,
    alpha_op: Add,
}
```

### 4.2 实现 lock_transforms

`ScenePuppetAnimationLayer` 增加 first-class 字段；转换器直接 lower
`locktransforms`，运行时不再从 `provenance` 临时查询：
```rust
pub struct ScenePuppetAnimationLayer {
    // ... 现有字段 ...
    pub lock_transforms: bool,
}
```

`sample_puppet_local_pose()`（`src/core/scene.rs:3619`）中：
```rust
for layer in layers {
    if !layer.visible || layer.blend <= 0.0 { continue; }
    if layer.lock_transforms {
        // transform channels stay at bind/rest pose for this layer
        continue;
    }
    // ... 现有采样逻辑 ...
}
```

如果后续证据表明 WE 仍采样非 transform 通道（例如 opacity），需要把 transform
锁定与 opacity/material 参数采样拆成独立 runtime inputs；不要用 provenance
分支补丁。

### 4.3 修复 mask UV 缩放

先在 converter/core scene/texture-slot records 中保存 WE backing texture extent，
再在 `effect/opacity_mask` 模块中计算：

```text
scale_u = alpha_backing_width  / base_backing_width
scale_v = alpha_backing_height / base_backing_height
```

不要用现有 decoded logical extent 字段替代 backing extent；不要保留 identity
shortcut 与 backing extent scale 的双路径。

### 4.4 修复优先级

1. **Normal blend mode** — 最小改动，修复最根本的语义差异
2. **lock_transforms** — 中改动，使 mask 变得有意义
3. **backing extent + mask UV 缩放** — 中改动，使 mask 作用于正确位置

三项与 MDLE inverse-bind 修复合并验证后，闭眼帧瞳孔才可以判定为修复。

---

## 5. 与几何层根因的关系

渲染合成层根因解决 **alpha/混合/合成语义**问题。几何层根因解决**顶点蒙皮**
问题。两者**独立且互补**。任一未修复都将导致瞳孔可见。

---

## 6. 验证

- `scripts/check_eye_alpha.py` — 纹理 alpha 分析
- `scripts/check_mask_values.py` — opacity mask 值分析
- `scripts/eye_skinning_compare.py` — CPU 蒙皮对比
- `scripts/mdle_compare.py` — MDLE 解析和对比
- `scripts/eye_pixel_simulation.py` — 像素级端到端模拟
