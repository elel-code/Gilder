# 原始 Wallpaper Engine 眼睛渲染语义反汇编

本文件直接反汇编 workshop scene `3742497499` 中眼睛相关的原始 WE 资产，
包含原始 GLSL shader、material、effect 定义和 layer 结构。不依赖 Gilder 转换层。

---

## 1. Layer 结构

### Layer 1336（base eye）

```
id: 1336
name: 眼睛
parent: 937（主身体）
image: models/眼睛.json
attachment: 眼睛
size: 663×230
animation: clip 730, rate 0.8
effects: [effects/iris/effect.json, effects/waterripple/effect.json]
```

### Layer 1530（opacity duplicate）

```
id: 1530
name: 眼睛
parent: 937（主身体）
image: models/眼睛.json
attachment: 眼睛
locktransforms: true
size: 663×230
animation: clip 730, rate 0.8（同一动画）
effects: [effects/opacity/effect.json]
```

1530 和 1336 使用**同一份 puppet mesh**（models/眼睛_puppet.mdl）、
**同一份 animation clip 730**、**同一个 attachment 骨骼**。
区别仅在于 1530 的 `locktransforms=true` 和 effect 链不同。

---

## 2. Material 定义

### models/眼睛.json — 眼睛材质

```json
{
  "passes": [{
    "shader": "genericimage4",
    "blending": "translucent",
    "textures": ["眼睛"],
    "cullmode": "normal",
    "depthtest": "disabled",
    "depthwrite": "disabled"
  }]
}
```

WE 语义：第一道 material pass 使用 `genericimage4` shader 将 puppet mesh
渲染到本地 FBO，blend mode 为 **translucent**（SRC_ALPHA / ONE_MINUS_SRC_ALPHA）。

### materials/effects/iris.json — Iris 效果材质

```json
{
  "passes": [{
    "shader": "effects/iris",
    "blending": "normal",
    "cullmode": "nocull"
  }]
}
```

### materials/effects/opacity.json — Opacity 效果材质

```json
{
  "passes": [{
    "shader": "effects/opacity",
    "blending": "normal",
    "cullmode": "nocull"
  }]
}
```

**关键语义**：所有 effect pass 的 blend mode 均为 **normal**（ONE / ZERO），
即**完全覆盖**前一个 pass 的输出。effect pass 通过全屏四边形在 pass space
中执行，采样前一个 pass 的 FBO 作为 `g_Texture0`。

---

## 3. Effect 定义

### effects/iris/effect.json

```json
{
  "version": 1,
  "replacementkey": "iris",
  "passes": [{
    "material": "materials/effects/iris.json"
  }]
}
```

### effects/opacity/effect.json

```json
{
  "version": 1,
  "replacementkey": "opacity",
  "passes": [{
    "material": "materials/effects/opacity.json"
  }]
}
```

---

## 4. 原始 GLSL Shader（完整反汇编）

### shaders/effects/iris.vert

```glsl
#include "common.h"

uniform mat4 g_ModelViewProjectionMatrix;
uniform float g_Time;
uniform vec2 g_Scale;
uniform float g_Speed;
uniform float g_Rough;
uniform float g_NoiseAmount;
uniform float g_PhaseOffset;

#if MASK
uniform vec4 g_Texture1Resolution;
#endif

attribute vec3 a_Position;
attribute vec2 a_TexCoord;

varying vec4 v_TexCoord;
varying vec2 v_TexCoordIris;

void main() {
    gl_Position = mul(vec4(a_Position, 1.0), g_ModelViewProjectionMatrix);
    v_TexCoord = a_TexCoord.xyxy;

#if MASK
    v_TexCoord.zw = vec2(
        v_TexCoord.x * g_Texture1Resolution.z / g_Texture1Resolution.x,
        v_TexCoord.y * g_Texture1Resolution.w / g_Texture1Resolution.y);
#endif

    float time = (g_Time * g_Speed) + g_PhaseOffset;
    float lowDt = floor(time);
    vec2 motion2 = sin(1.9 * (lowDt + vec2(0, 1)));
    vec4 motion4 = sin(2.5 * (lowDt + vec4(0, 0, 1, 1)) + vec4(1, 2, 1, 2));
    vec2 moveStart = motion2.xx + motion4.xy;
    vec2 moveEnd = motion2.yy + motion4.zw;
    vec2 da = mix(moveStart, moveEnd,
        smoothstep(1.0 - g_Rough, 1.0,
            cos(frac(time) * M_PI) * -0.5 + 0.5));

    da.x += sin(time) * g_NoiseAmount;
    da.y += cos(time) * g_NoiseAmount;
    da *= g_Scale * 0.001;
    v_TexCoordIris = da.xy;
}
```

**语义**：
- `v_TexCoord.xy` = pass-space 四边形 UV（全屏 0..1）
- `v_TexCoord.zw` = mask UV，通过 `g_Texture1Resolution` 缩放为 mask 纹理空间坐标
- `v_TexCoordIris` = 虹膜偏移向量（基于时间 + 噪声 + 参数的动态扰动）

### shaders/effects/iris.frag

```glsl
varying vec4 v_TexCoord;
varying vec2 v_TexCoordIris;

uniform sampler2D g_Texture0; // 前一个 pass 的 FBO
uniform sampler2D g_Texture1; // iris mask（R8，331×115）
uniform vec3 g_EyeColor;

void main() {
    vec4 albedo = texSample2D(g_Texture0, v_TexCoord.xy);
    float mask = 1.0;

#if MASK
    mask *= texSample2D(g_Texture1, v_TexCoord.zw).r;
    vec4 iris = texSample2D(g_Texture0, v_TexCoord.xy + v_TexCoordIris.xy * mask);
    float irisMask = texSample2D(g_Texture1, v_TexCoord.zw + v_TexCoordIris.xy * mask).r;
#if BACKGROUND
    iris.rgb = mix(g_EyeColor, iris.rgb, irisMask);
#endif
#else
    vec4 iris = texSample2D(g_Texture0, v_TexCoord.xy + v_TexCoordIris.xy);
#endif

    albedo = iris;   // ← 完全替换为偏移采样结果
    gl_FragColor = albedo;
}
```

**语义**：
- 从 `g_Texture0`（前一个 pass 的 FBO）以偏移 UV 采样得到 `iris`
- **albedo = iris** — 完全替换前一个 pass 的输出
- iris mask（`g_Texture1`，R8 格式）控制偏移强度：mask 越接近 1，偏移越大
- 不修改 alpha 通道——alpha 来自 `g_Texture0` 在偏移位置的采样值
- blend mode 为 normal（ONE/ZERO）→ 直接覆盖 pass FBO

### shaders/effects/opacity.vert

```glsl
uniform mat4 g_ModelViewProjectionMatrix;
uniform vec4 g_Texture1Resolution;

attribute vec3 a_Position;
attribute vec2 a_TexCoord;

varying vec4 v_TexCoord;

void main() {
    gl_Position = mul(vec4(a_Position, 1.0), g_ModelViewProjectionMatrix);
    v_TexCoord.xy = a_TexCoord;
    v_TexCoord.zw = vec2(
        v_TexCoord.x * g_Texture1Resolution.z / g_Texture1Resolution.x,
        v_TexCoord.y * g_Texture1Resolution.w / g_Texture1Resolution.y);
}
```

### shaders/effects/opacity.frag

```glsl
varying vec4 v_TexCoord;

uniform sampler2D g_Texture0; // 前一个 pass 的 FBO
uniform sampler2D g_Texture1; // opacity mask（R8，331×115）
uniform float g_UserAlpha;    // 用户滑块，值 = 1.0

void main() {
    vec4 albedo = texSample2D(g_Texture0, v_TexCoord.xy);
#if MASK
    float mask = texSample2D(g_Texture1, v_TexCoord.zw).r;
#else
    float mask = 1.0;
#endif
    albedo.a *= mask * g_UserAlpha;
    gl_FragColor = albedo;
}
```

**语义**：
- 从 `g_Texture0` 采样前一个 pass 的完整 RGBA
- **albedo.a *= mask * g_UserAlpha** — 仅修改 alpha 通道
- mask 来自 `g_Texture1`（opacity mask，R8，331×115），在 mask UV 空间采样
- `g_UserAlpha` = 1.0（用户未调）
- blend mode 为 normal（ONE/ZERO），即**直接写入 scene FBO**

---

## 5. 完整语义链

### Layer 1336（base eye）渲染流程

```
Step 1 — material pass:
  shader: genericimage4
  geometry: puppet mesh（已由 animation clip 730 skinning）
  render target: 本地 FBO
  blend: translucent (SRC_ALPHA / ONE_MINUS_SRC_ALPHA)
  输出: 眼睛 puppet mesh 渲染到 FBO

Step 2 — iris effect pass:
  shader: iris.frag
  geometry: 全屏 pass-space 四边形
  render target: 同一本地 FBO（覆盖）
  blend: normal (ONE / ZERO)
  g_Texture0: Step 1 的 FBO
  g_Texture1: iris mask（331×115，R8）
  操作: 从 g_Texture0 以 iris 偏移采样，完全替换 albedo
  输出: 虹膜偏移后的图像

Step 3 — waterripple effect pass:
  类似，对前一步 FBO 做水波纹处理

Step 4 — 最终复合:
  将本地 FBO 内容以 translucent blend 写入 scene FBO
```

### Layer 1530（opacity duplicate）渲染流程

```
Step 1 — material pass:
  shader: genericimage4
  geometry: puppet mesh（同一 animation clip 730，同一 attachment 眼睛）
  render target: 本地 FBO
  blend: translucent (SRC_ALPHA / ONE_MINUS_SRC_ALPHA)
  输出: 眼睛 puppet mesh 渲染到 FBO

Step 2 — opacity effect pass:
  shader: opacity.frag
  geometry: 全屏 pass-space 四边形
  render target: scene FBO（直接写入）
  blend: normal (ONE / ZERO)
  g_Texture0: Step 1 的 FBO
  g_Texture1: opacity mask（331×115，R8）
  操作: albedo.a *= mask.r
  输出: 选择性透明/不透明地覆盖 scene FBO 上已有的像素
```

### 1530 覆盖 1336 的语义

1530 的 layer 索引（74）晚于 1336（63），渲染时 scene FBO 上已有
1336 的输出。

1530 的 opacity pass 使用 **blend=normal（ONE/ZERO）** 直接写入 scene：

- **mask=0 的像素**：albedo.a=0 → 完全透明 → scene FBO 该位置不改变 → **1336 的输出被保留**
- **mask=1 的像素**：albedo.a=eye_alpha → 不透明 → scene FBO 该位置被 1530 的渲染覆盖 → **1336 的输出被擦除**

这意味 opacity mask 控制**哪些区域的 1336 输出被保留 vs 被 1530 覆盖**。

### 闭眼动画的语义

Animation clip 730 驱动 54 根骨骼的变换（translation/rotation/scale）。
两个 layer 共用同一 clip。骨骼变形驱动眼睑遮盖瞳孔。

1530 的 opacity mask（`masks/opacity_mask_d2f87f99`）控制选择性覆盖。
mask 的 R8 值域 [0,1] 决定哪些像素从 1530 写入 scene、哪些保留 1336 的。

**在原始 WE 中，闭眼帧瞳孔消失是因为：**
1. Animation clip 730 使眼睑骨骼变形，几何上遮盖瞳孔
2. Opacity mask 选择性保留/覆盖不同区域的像素

---

## 6. 与 Gilder 当前实现的差异映射

| 原始 WE | Gilder 当前 |
|---------|------------|
| genericimage4 material pass → 本地 FBO | 直接渲染到 swapchain（无本地 FBO） |
| iris effect pass 偏移采样并替换 FBO | iris 不被识别为 first-class target，effect chain 跳过 |
| opacity effect pass 仅修改 alpha | opacity 被识别但 mask UV 映射可能错误 |
| bone 变形驱动眼睑遮盖瞳孔 | bone 变形正确计算但 FBO 输出仍含瞳孔像素 |
| 1530 blend=normal 选择性覆盖 1336 | 1530 blend 行为不同 |
