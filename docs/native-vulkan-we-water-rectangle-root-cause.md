# WE 水纹矩形：root cause 和完整修复主线

Workshop scene: `3742497499`

## 结论

两个可见矩形不是 `.gscn`/binary 信息丢失。它们来自原始 WE scene 中真实存在的
普通 image 图层；Gilder 之前把带 WE image effect pass chain 的水纹图层当成
普通 sampled image 直接合成，导致 raw source rectangle 暴露。

完整修复主线不是隐藏图层，而是实现 WE/CWE image effect graph executor：
base material pass -> effect material passes -> optional `colorBlendMode`
passthrough -> final scene composite。细节记录在
`docs/native-vulkan-we-effect-graph-mainline.md`。

当前代码里的矩形 guard 只是临时正确性护栏：未实现完整 graph 前，不能把
water ripple/flow/caustics 这类 passthrough carrier 当普通 sampled image 直接画进
scene FBO。这个 guard 必须保持窄范围，不能吞掉 alpha/normal 的人物层，也不能全局隐藏
`waterwaves` 头发/身体层。

## 具体源图层

小矩形：

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

Large rectangle candidates under the character:

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
