# 闭眼帧瞳孔可见：几何层根因 — MDLE0002 逆绑定矩阵缺失

本文件记录 workshop scene `3742497499` 闭眼帧瞳孔可见的**几何层根因**。
渲染合成层根因见 `docs/native-vulkan-we-eye-render-composite-root-cause.md`。

旧文档 `docs/native-vulkan-we-eye-first-class-handoff.md` 和
`docs/native-vulkan-we-eye-closed-frame-root-cause.md` 中关于 iris/opacity
效应链路由的讨论仍然有效，但不覆盖本文档分析的顶点层面问题。

2026-07-01 计划审阅结论：本文档的几何层根因可以采纳。修复必须按
first-class puppet skin 数据推进，不允许增加兼容字段，不允许保留运行时
computed inverse-bind fallback，也不允许同时维护旧/新两套 skinning 字段。

---

## 1. 概述

`models/眼睛_puppet.mdl` 包含一个 `MDLE0002` 区段（3456 字节 = 54 × 64），
存储每根骨骼的逆绑定位姿（inverse-bind）4×4 矩阵。Gilder 转换器**完全不解析**
`MDLE` 区段，运行时通过 MDLS bind 矩阵的 parent chain 累积求逆计算
`inverse_bind_world`。该计算结果与原始 MDLE 矩阵在**所有 54 根骨骼**上存在
显著平移差异（最高 721 单位），导致蒙皮后眼睑顶点位置严重错误。

---

## 2. 证据

### 2.1 MDLE 解析

```
文件: /tmp/gilder-we-3742497499-extracted/models/眼睛_puppet.mdl
MDLE0002 偏移: 0x001A9304
数据起始:      0x001A9315
数据大小:      3456 bytes = 54 bones × 64 bytes (4×4 f32, column-major LE)
```

验证脚本: `scripts/mdle_compare.py`

### 2.2 MDLE vs Gilder inverse_bind 对比

MDLS 54 根骨骼中，Gilder 的 computed `inverse_bind_world` 与 MDLE 的旋转分量
几乎一致（差 ~0.00006），但**平移分量差异巨大**：

| 骨骼 | Parent | 平移差 (tx,ty,tz) | 最大差 |
|------|--------|-------------------|--------|
| 1 | 0 | (721.6, 16.0, 0.0) | 721.6 |
| 4 | 0 | (641.2, 24.5, 0.0) | 641.2 |
| 0 | -1 (root) | (525.0, 35.3, 0.0) | 525.0 |
| 9 | 0 | (120.6, 140.3, 0.0) | 140.7 |
| 2 | 1 | (2.4, 131.9, 0.0) | 131.9 |
| 5 | 4 | (28.8, 122.4, 0.0) | 122.4 |

所有 54 根骨骼都存在亚像素级旋转差 + 大平移差。差异集中在 x/y 平移，z 为 0。
根骨骼（bone 0）的平移差异 525 单位 — 意味着整个模型空间坐标系在转换器中
被偏移了 500+ 单位。

### 2.3 CPU 蒙皮对比

脚本: `scripts/eye_skinning_compare.py`

用 MDLE 和 Gilder inverse_bind 分别对 4106 顶点执行蒙皮（clip 730, frame 300 闭眼帧）：

| 指标 | Gilder vs MDLE |
|------|---------------|
| 全体顶点 RMS 位移 | dx=42.7, dy=82.4 |
| 瞳孔区域 (1003 顶点, UV 0.3-0.7) 平均位移 | **48.5 单位** |
| 瞳孔区域最大位移 | **dx=75.5, dy=140.7** |
| 上眼睑 (v>0.65, 1579 顶点) 变形量 Δy | Gilder: -17.2, **MDLE: -56.0** |
| 上眼睑最大变形量 | Gilder: -43.0, **MDLE: -169.6** |

MDLE 版本的上眼睑变形量是 Gilder 的 **3.3 倍**（平均）到 **3.9 倍**（最大）。

### 2.4 眼睑覆盖深度

闭眼帧 (frame 300) 上眼睑最低 Y 值 vs 瞳孔最高 Y 值：

| | Gilder | MDLE |
|--|--------|------|
| 眼睑最低 Y | 22.5 | 14.6 |
| 瞳孔最高 Y | 39.5 | 137.6 |
| 覆盖深度 | **17.0** | **123.1** |
| 瞳孔被遮盖比例 | ~25% | ~50% |

Gilder 的 computed inverse_bind 导致眼睑只覆盖瞳孔的 1/4。

### 2.5 根源：Gilder 的 inverse_bind 计算公式

```
当前计算（src/core/scene.rs:3597-3604）:

bind_world[i] = parent_bind_world × MDLS_bind_local[i]   (parent chain 累积)
inverse_bind_world[i] = inv(bind_world[i])                (数学求逆)
```

问题：当 MDLS bind_local 矩阵与 MDLE 中的原始逆绑定矩阵不一致时（在 9 根关键
骨骼上差异很大），`inv(bind_world)` 产生错误的平移分量。`inv()` 的平移
= `-R⁻¹ × t`，其中 R 和 t 来自 bind_world 的旋转和平移。任何 parent chain
累积误差都会被放大到逆矩阵的平移中。

---

## 3. 修复路径

### 3.1 转换器端：解析 MDLE0002

`src/convert/wallpaper_engine.rs` 的 `scene_parse_puppet_attachment_map()` 中，
在 MDLA 解析后增加 MDLE 解析：

```
MDLA0006 [...] MDLE0002 [3456 bytes = 54 × 64]
           ↑ mdla_end 处或之后找 MDLE
```

MDLE 区段头部与 MDLS/MDLA 相同（TAG + version + metadata + count），
每根骨骼的 4×4 矩阵使用相同的 `scene_take_mdl_matrix()` 读取。

### 3.2 gscene 存储

在 `SceneMeshSkinBone` 中增加 first-class 逆绑定矩阵字段。该字段是 puppet
skinning 的正式输入，不是兼容字段：

```rust
pub struct SceneMeshSkinBone {
    pub parent: Option<usize>,
    pub bind: ScenePuppetTransform,
    pub inverse_bind: [f64; 16],
}
```

重新生成并更新所有相关 gscene fixture，使每根 bone 都有 `inverse_bind`。不要用
`Option`，不要在同一 schema 中保留“缺失时走旧逻辑”的双路径。

### 3.3 运行时使用

`src/core/scene.rs` 的 skinning 热路径直接读取 `skin.bones[*].inverse_bind`：

```rust
let inverse_bind_world = skin
    .bones
    .iter()
    .map(|bone| bone.inverse_bind)
    .collect::<Vec<_>>();
```

删除运行时从 `bind_world` 数学求逆作为 puppet skinning 输入的旧路径。对于没有
MDLE 的非 WE puppet source，转换器也必须在转换阶段 materialize
`inverse_bind`，运行时只消费一种字段形态。

### 3.4 验证

重新转换 workshop scene，用 `gilder-native-vulkan --run-scene --output-name HDMI-A-1`
验证闭眼帧瞳孔是否被正确遮盖。验证脚本产生的证据文件路径：
`/tmp/mdle_comparison.json`, `/tmp/eye_skinning_frame_*.json`

---

## 4. 与渲染合成层根因的关系

MDLE 修复解决的是**几何量级问题**：眼睑变形幅度从正确值的 1/4 恢复到正常。

渲染合成层根因解决的是**alpha/混合语义问题**：即使几何正确，WE 的多 pass
FBO 混合语义也需要正确实现才能产生不透明眼睑。

两者**独立且互补**。都修复后闭眼帧瞳孔才完全不可见。
