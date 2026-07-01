# WE 水纹矩形：root cause 和完整修复主线

Workshop scene: `3742497499`

## 结论

矩形问题不是 `.gscn`/binary 信息丢失。它们来自原始 WE scene 中真实存在的
普通 image 图层，根因在 native Vulkan 没有按 WE/CWE 的 pass/material/blend
边界执行：

- 早期 water-ripple/flow/caustics 矩形：Gilder 把未执行完整 image effect
  graph 的 carrier 当普通 sampled image 直接合成，导致 raw source rectangle
  暴露。
- 2026-07-02 最终修复的一大一小 `waterwaves` 矩形：图层本来应该可见，但
  Gilder 把 effect pass 的 `normal` local overwrite 误当成最终 scene composite
  blend，覆盖了 base material 的 `translucent` alpha 合成。

完整修复主线不是隐藏图层，而是实现 WE/CWE image effect graph executor：
base material pass -> effect material passes -> optional `colorBlendMode`
passthrough -> final scene composite。细节记录在
`docs/native-vulkan-we-effect-graph-mainline.md`。

当前代码里的矩形 guard 只是临时正确性护栏：未实现完整 graph 前，不能把
water ripple/flow/caustics 这类 passthrough carrier 当普通 sampled image 直接画进
scene FBO。这个 guard 必须保持窄范围，不能吞掉 alpha/normal 的人物层，也不能全局隐藏
`waterwaves` 头发/身体层。

## 2026-07-02 最终矩形修复记录

这次用户视觉确认修复的“一大一小矩形”不是 `.gscn`，也不是 `id=544/name=方块`
那条 water-ripple carrier。`方块` 属于更早已经被 narrow guard 处理的
water-ripple raw carrier 问题。剩下的两个矩形来自 `effects/waterwaves`
人物发片/底发层：

- 大矩形：`node-43..48` 和 `node-51..56`，资源
  `resource-73/78/81/86/91/96`，其中 `resource-73-1-frame-0.gtex`
  来自 `materials/底发1.tex`，源尺寸 `2318 x 1794`。runtime bbox 约为
  `x=836..3333, y=291..2224` 和 `x=858..3355, y=320..2252`，所以视觉上是
  人物下方/身体范围的一张大矩形。
- 小矩形：`node-70+` 头发/发卡等 waterwaves 层，例如
  `node-70-models-1-json`，资源 `resource-149-1-frame-0.gtex`，
  `materials/头发右1.tex`，源尺寸 `728 x 757`。runtime bbox 约为
  `x=1341..2125, y=1309..2124`，所以视觉上是人物上方/头发区域的小矩形。

这些节点的 WE source 形态是：

```text
base material pass:
  shader = genericimage4
  blending = translucent
  texture = hair/body image

effect material pass:
  shader = effects/waterwaves
  blending = normal
  g_Texture0 = base image/local previous pass
  g_Texture1 = waterwaves mask
```

`waterwaves.frag` 的关键输出在
`artifacts/wallpaper-engine-workshop/steamcmd-root/assets/effects/waterwaves/shaders/effects/waterwaves.frag:66-84`：

```glsl
float strength = g_Strength * g_Strength;
texCoord += val1 * s1 * offset * strength * mask;
gl_FragColor = texSample2D(g_Texture0, texCoord);
```

也就是说 `waterwaves` 只扰动采样 UV，然后把 `g_Texture0` 采样结果原样输出；
它不创建新 alpha，也不应该把图层透明区域变成不透明矩形。因此矩形硬边只能来自
最终 scene composite 的 blend 边界，而不是 shader 本身。

### CWE 依据

本地 CWE 参考明确把 image effect pass 和 scene final blend 分开：

- `references/cwe/src/WallpaperEngine/Render/Objects/CImage.cpp:750-766`
  在 `colorBlendMode > 0` 时追加 `materials/util/effectpassthrough.json`
  作为额外 pass。
- `references/cwe/src/WallpaperEngine/Render/Objects/CImage.cpp:769-775`
  当 image 有多个 pass 时，把 first pass 的 blend 移到 last pass，
  并把 first pass 改成 `BlendingMode_Normal`。这说明 effect pass 的
  `normal` 是 local pass overwrite，不是最终 scene composite。
- `references/cwe/src/WallpaperEngine/Render/Objects/Effects/CPass.cpp:130-140`
  中 `Translucent` 是
  `SRC_ALPHA, ONE_MINUS_SRC_ALPHA`，而 `Normal` 是 `ONE, ZERO`。

### 数学证明

令 waterwaves shader 输出的源采样颜色为 `S = (Cs, As)`，当前 framebuffer
颜色为 `D = (Cd, Ad)`。

WE base material 是 `translucent`，正确最终 scene composite 的颜色为：

```text
Cout_alpha = As * Cs + (1 - As) * Cd
```

Gilder 之前错误地把 first effect pass 的 `blending="normal"` 提升成最终
scene composite，实际变成：

```text
Cout_normal = 1 * Cs + 0 * Cd = Cs
```

在发片矩形透明背景区域，`As = 0` 或非常小。于是：

```text
Cout_alpha  = Cd          when As = 0
Cout_normal = Cs          when As = 0
```

只要透明区域纹理 RGB `Cs` 和背景 `Cd` 不完全相同，`normal` overwrite 就必然把
整张源纹理矩形的 RGB 写到 swapchain 上，形成硬矩形。`alpha` composite 则让
`As=0` 的区域完全保留背景，所以矩形消失。因为 `waterwaves.frag` 最后只是
`texSample2D(g_Texture0, texCoord)`，它保持源 alpha 语义；因此这个数学差异就是
矩形是否出现的充分原因。

这也解释了现象：

- 修复前：同样的 waterwaves 源图、同样 bbox、同样 shader，最终 blend 是
  `normal`，所以一大一小矩形稳定可见。
- 修复后：源图和 geometry 仍存在，waterwaves 层仍然被记录，但最终 blend 回到
  `alpha/translucent`，透明区域不再覆盖背景，所以两个矩形消失。

### 代码修复点

这次不是隐藏图层，而是修正 WE pass/material blend 边界：

- `src/core/scene.rs`
  - `scene_blend_mode_from_material_blending()` 新增
    `translucent | alpha -> SceneBlendMode::Alpha`。
- `src/core/scene/binary.rs`
  - `SceneBinaryMaterialState::from_node()` 优先读取
    `node.properties.material.passes[0]` 的 base material pass。
  - 只有没有 base material pass 时才 fallback 到 first effect pass。
  - 删除“属性 blend 是 alpha 时用 first effect pass normal 覆盖”的逻辑。
  - 单测：
    `binary_material_pass_keeps_scene_alpha_when_effect_material_blends_normal`。
- `src/renderer/native_vulkan/scene/draw_pass/effect.rs`
  - sampled-image material pass 默认不再用 effect pass blend 覆盖 scene blend。
  - 单测：
    `material_pass_keeps_scene_alpha_when_effect_material_blends_normal`。
- `src/renderer/native_vulkan/scene/draw_pass.rs`
  - recording step 也强制 `use_effect_blend=false`，避免 runtime 记录阶段重新把
    effect `normal` 提升到 swapchain composite。
  - 单测：
    `draw_pass_plan_keeps_alpha_waterwaves_character_quad` 断言
    waterwaves 人物 quad 的 render state 是 `SceneBlendMode::Alpha`，
    pipeline label 是 `sampled-image-alpha-blend`。

### 验证记录

修复前的 release snapshot：

```text
/tmp/gilder-we-3742497499-no-raw-water-ripple-runtime.json
waterwaves swapchain steps:
  normal   = 35
  multiply = 1

关键层：
  node-43..47, node-51..56, node-70..76, node-79 都是 normal
```

修复后的 release 转换和 snapshot：

```text
target/release/gilder-convert wallpaper-engine \
  artifacts/wallpaper-engine-workshop/steamcmd-root/steamapps/workshop/content/431960/3742497499 \
  /tmp/gilder-we-3742497499-alpha-blend-fix

target/release/gilder-native-vulkan --scene-runtime-snapshot \
  --source /tmp/gilder-we-3742497499-alpha-blend-fix/assets/scene.gscn \
  --scene-root /tmp/gilder-we-3742497499-alpha-blend-fix \
  --output-name HDMI-A-1 --target-fps 60 \
  > /tmp/gilder-we-3742497499-alpha-blend-fix-runtime.json
```

修复后结果：

```text
/tmp/gilder-we-3742497499-alpha-blend-fix-runtime.json
waterwaves swapchain steps:
  alpha    = 35
  multiply = 1

大矩形相关层：
  node-43..47 = alpha
  node-51..56 = alpha
  node-48 = multiply, 保留原 authored blend/effect 语义

小矩形相关层：
  node-70..76, node-79 = alpha
```

release smoke：

```text
timeout 8s env WAYLAND_DISPLAY=wayland-1 \
  target/release/gilder-native-vulkan --run-scene \
  --source /tmp/gilder-we-3742497499-alpha-blend-fix/assets/scene.gscn \
  --scene-root /tmp/gilder-we-3742497499-alpha-blend-fix \
  --output-name HDMI-A-1 --target-fps 60 --duration 6 \
  > /tmp/gilder-we-3742497499-alpha-blend-fix-smoke.json
```

smoke 输出：

```text
scene_present_route = sampled-image
frames_presented = 305
average_present_fps = 50.70766027941938
runtime_elapsed_ms = 6014
draw_call_count = 68
sampled_image_draw_call_count = 60
solid_quad_draw_call_count = 8
pipeline_bind_count = 19
draw_pass_sampled_image_recording_step_count = 3839
draw_pass_sampled_image_we_graph_chain_count = 248
draw_pass_sampled_image_we_graph_step_count = 547
draw_pass_sampled_image_we_graph_target_count = 275
draw_pass_sampled_image_we_graph_resource_count = 343
draw_pass_effect_pass_count = 99
draw_pass_effect_pass_kind_counts["water-waves"] = 76
```

release tests/build：

```text
cargo test --release --features native-vulkan-video,native-vulkan-vulkanalia \
  keeps_scene_alpha -- --nocapture

cargo test --release --features native-vulkan-video,native-vulkan-vulkanalia \
  draw_pass_plan_keeps_alpha_waterwaves_character_quad -- --nocapture

cargo test --release --features native-vulkan-video,native-vulkan-vulkanalia \
  plain_unimplemented_water_ripple -- --nocapture

cargo build --release --features native-vulkan-video,native-vulkan-vulkanalia \
  --bin gilder-native-vulkan --bin gilder-convert
```

用户视觉确认：最新 release 运行后，一开始稳定存在的一大一小矩形消失。

## 具体源图层

早期 water-ripple raw carrier 小矩形：

- Source id: `544`
- Name: `方块`
- Node: `node-108-models-json`
- Texture: `resource-231-frame-0.gtex`
- Source material texture: `materials/方块.tex`
- Size: `3450 x 3000`
- Origin: `(1978.70886, 1226.24512)`
- Scale: `1.17133`
- Blend: `colorBlendMode = 32`
- Effects: `effects/waterripple/effect.json`
- Draw order: after character nodes, so it appears above the character/head area.

早期 water-ripple/water-caustics raw carrier 大矩形候选：

- Source id `202`, name `水纹`, texture `resource-8-frame-0.gtex`, size `1287 x 1080`,
  scale `3.45134`, blend `colorBlendMode = 2`, effects `waterripple + waterflow`.
- Source id `164/2942`, name `Water Caustic`, texture `resource-14-wc-test-frame-0.gtex`,
  size `2048 x 1024`, scale `1.91655 x 2.20292`, blend `colorBlendMode = 7`,
  effects `foliagesway + waterripple`.
- These nodes are before the character group, so their visible artifact is below the character.

## 数学边界

For a WE image quad without rotation, screen-space bounds are:

```text
left   = origin.x - width  * scale.x * anchor.x
right  = origin.x + width  * scale.x * (1 - anchor.x)
top    = origin.y - height * scale.y * anchor.y
bottom = origin.y + height * scale.y * (1 - anchor.y)
```

For `方块`:

```text
width'  = 3450 * 1.17133 = 4041.0885
height' = 3000 * 1.17133 = 3513.99
left    = 1978.70886 - 4041.0885 / 2 = -41.83539
right   = 1978.70886 + 4041.0885 / 2 = 3999.25311
top     = 1226.24512 - 3513.99 / 2 = -530.74988
bottom  = 1226.24512 + 3513.99 / 2 = 2983.24012
```

The rectangle spans nearly the whole 3840-wide viewport and overlays the character because the
node is drawn after character nodes.

For `水纹`:

```text
width'  = 1287 * 3.45134 = 4441.37458
height' = 1080 * 3.45134 = 3727.4472
left    = 2198.66162 - 4441.37458 / 2 = -22.02567
right   = 2198.66162 + 4441.37458 / 2 = 4419.34891
top     = 311.70001 - 3727.4472 / 2 = -1552.02359
bottom  = 311.70001 + 3727.4472 / 2 = 2175.42361
```

This explains the large water rectangle under the character.

## WE reference behavior

CWE `CImage::setup()` and `CImage::setupPasses()` build a per-image pass list:

- material pass first,
- effect material passes next,
- optional `colorBlendMode` passthrough pass last,
- intermediate passes ping-pong through the image FBO,
- only the final pass writes back to the scene FBO.

`waterripple` itself has no explicit `target` field, but it is still an image effect pass in this
chain. Its fragment shader samples `g_Texture0`, disturbs UV using `g_Texture2` normal map, and
outputs the disturbed source. Gilder does not yet execute that fragment/material graph for water
effects; its current `native-effect-motion` is only a geometry approximation.

## Full implementation line

The renderer must implement the CWE pass-chain contract instead of relying on
the temporary guard:

1. Build an ordered pass chain for every image: model material pass, visible
   effect passes, compatibility passes if needed, then `colorBlendMode`
   passthrough if present.
2. Preserve effect-declared FBOs and texture binds. `g_Texture0..7`, `previous`,
   `_rt_*`, and `_alias_*` must be first-class bindings.
3. Allocate image-local main/sub targets and ping-pong intermediate passes.
4. Move the first material blend to the final scene pass when the chain contains
   multiple passes, matching CWE.
5. Execute water ripple/flow/waves/caustics, opacity, iris, scroll, skew,
   colorkey, lightshafts, cloudmotion, shake/sway, clipping/rounded masks,
   audio bars, and workshop effects through the same graph path.
6. Replace the current rectangle guard with real local effect target recording
   once the affected water passes are executable.

Current code progress on this line:

- Every affected image now lowers to a typed `we_image_pass_chain`.
- The draw-pass graph plan emits image-local target records and per-step
  input/output target indices.
- Pass records preserve texture slots, shader/effect file, parameter keys,
  combo keys, blend/depth/cull state, and final-scene routing.
- Pass records now also preserve `command`, `source`, `target`, and
  `bind`/`binds` from WE effect files and object pass overrides. Runtime graph
  snapshots include these fields on both generic effect records and WE image
  pass records. Effect file passes are still emitted when an object instance has
  no local `passes` array, so file-declared material graph passes survive
  conversion.
- Effect-declared `fbos` are preserved as typed scene data and graph targets:
  target name, format, scale, unique flag, scaled extent, first write count, and
  sampled-by-following-pass status are visible in runtime snapshots.
- `.gscn` binary format version `14` carries those graph fields too:
  `command/source/target` live on `effect_pass`, while bind slots are stored as
  typed `PASS_BIND` effect parameters and FBO declarations are stored as typed
  `EFFECT_FBO` parameters. Direct binary ingest reconstructs binds, FBOs,
  combos, and constants.
- The already executable opacity/iris effect-target path now carries WE graph
  chain/target/step linkage where it really matches the planned graph. Collapsed
  legacy steps remain unlinked, so the remaining multi-pass gap stays visible.
- Allocated graph targets are now marked with their Vulkan effect-target
  resource index; planned-only targets remain visible as graph targets without
  claiming a Vulkan resource.
- Runtime snapshots now include a WE graph resource table that puts source
  textures and graph targets into one planned resource index space. Source
  texture resources carry extents; already executable opacity/iris targets map
  their planned resource to the real Vulkan effect-target index; water carrier
  main/sub targets stay visible as `planned-until-graph-executor` resources
  until the full graph executor allocates them.
- Every graph step now exposes typed `g_TextureN` bindings. For the small
  `方块` carrier, the unresolved chain is visible as source `g_Texture0`,
  ripple `g_Texture0` from image-local main plus `g_Texture2` normal map, and
  final color-blend passthrough `g_Texture0` from image-local sub.
- Graph steps also expose typed render state. For the small `方块` chain the
  base/local ripple passes resolve to `normal` local-target writes, and the
  final color-blend passthrough owns the original `Modulate`/`colorBlendMode=32`
  equation. That proves blend must be executed at the graph pass boundary, not
  attached to the raw source rectangle.
- `previous` binds lower to `previous-graph-target`; named FBO binds lower to
  `named-fbo-bind`. Declared/targeted named FBO binds now resolve to graph
  target indices and extents. Vulkan allocation/execution for those named FBOs
  is still pending, but the graph data is no longer discarded.
- The temporary guard is still present only because the Vulkan executor does not
  yet allocate/run the target graph.
- Latest release snapshot from the original WE source
  `/tmp/gilder-we-3742497499-resource-model-v14-runtime.json` shows
  `graph_chains=248`, `graph_steps=547`, `graph_targets=275`, and
  `final_scene_steps=248`; all `547` graph steps expose texture bindings. The
  graph resource table has `343` resources: `68` texture sources and `275`
  graph targets. `2` graph targets are already backed by Vulkan effect target
  resources and `273` remain planned-only. For
  `方块`, local base/ripple writes are `normal` and only the final scene pass is
  `modulate`. The four rectangle carrier nodes have zero raw sampled-image
  recording steps and do have typed image-local graph targets/resources.
- The same snapshot reports `99` visible WE effect passes across the draw plan,
  including `5` effect-bearing non-image layers. Non-water effect families such
  as `scroll`, `color-key`, `clipping-mask`, and `rounded-mask` are now counted
  as typed first-class pending graph work.
- Per-draw-op effect snapshots now show those non-image passes directly:
  `node-28-text` owns `scroll`, `node-29-text` owns `color-key + scroll`,
  `node-30-text` owns `scroll + clipping-mask`, and solid util layers own
  `rounded-mask`. This keeps the rectangle fix tied to the complete graph
  architecture instead of a water-only patch.

The current scene `3742497499` contains these effect files and they all belong
to this mainline rather than sample-specific patches:

```text
effects/cloudmotion/effect.json
effects/colorkey/effect.json
effects/iris/effect.json
effects/lightshafts/effect.json
effects/opacity/effect.json
effects/scroll/effect.json
effects/shake/effect.json
effects/skew/effect.json
effects/watercaustics/effect.json
effects/waterflow/effect.json
effects/waterripple/effect.json
effects/waterwaves/effect.json
effects/workshop/2123274886/tech_circle/effect.json
effects/workshop/2790231929/foliagesway/effect.json
effects/workshop/2790231929/waterripple/effect.json
effects/workshop/2800594362/clipping_mask/effect.json
effects/workshop/3082978660/enhanced_simple_audio_bars/effect.json
effects/workshop/3083593512/rounded_mask/effect.json
effects/workshop/3392386920/auto_sway/effect.json
```

## Why this is not binary/gscn

The converted binary already carries:

- original source ids and names,
- dimensions and transforms,
- texture resources,
- `image_effect_passes`,
- effect texture slots,
- pass `command/source/target`,
- pass `binds`,
- effect `fbos`,
- pass combos and constant shader values,
- `colorBlendMode`.

The artifact exists because runtime treated unsupported water effect pass chains as ordinary
visible sampled-image quads. That is an app capability/execution-boundary bug, not a scene format
loss. The correct final state is not "omit the carrier"; it is "execute the
carrier's WE pass graph and composite only the graph output."
