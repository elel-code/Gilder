# Native Vulkan WE Eye First-Class Handoff

> **⚠ 本文档已更新。2026-07-01 MDLE 几何根因已撤销：**
> - 已撤销假设：`docs/native-vulkan-we-eye-mdle-inverse-bind-root-cause.md`
> - 当前根因：`docs/native-vulkan-we-eye-render-composite-root-cause.md`
>
> 本文档保留 iris/opacity 效应链路由和 puppet 格式的历史证据。

This document preserves the evidence for the Wallpaper Engine eye-rendering bug
in workshop scene `3742497499`. It exists so the next implementation step does
not restart from guesses, repeat extraction work, or replace first-class runtime
support with sample-specific hiding.

## Active Update Protocol

- Every new implementation step must update this document in the same turn.
- When a new conclusion replaces an old one, delete the old wording instead of
  leaving both descriptions side by side.
- Keep `1530` described as an independent later-drawn source under parent `937`;
  do not describe it as an effect child of `1336`.

## Current Landing Point

- 2026-07-01 iris first-class routing update: `native-iris-mask` is no longer
  metadata-only for the base eye. The renderer now keeps `node-77`'s scene
  layer `alpha_slot=None`, but builds an explicit two-step first-class path:
  first draw the puppet mesh into a local effect target, then draw a final
  scene quad that samples that target as `g_Texture0` and the iris mask as
  slot 1 with `mode=iris`. This does not hide, fold, or remove `node-89`, and
  it does not re-enable the rejected opacity local-target route.
- 20s HDMI-A-1 evidence after the change:
  `/tmp/gilder-eye-iris-target-hdmi-a-1-20s-r2.log`. Key lines show
  `node-77` base step as `target=effect-target(index=0, clear=true)` with
  `geometry_semantics=we-iris-effect-local-target-base-mesh`, followed by
  `node-77` final step as `target=swapchain`, `texture_slot_resource_indices=[50, 29]`,
  `alpha_slot=Some(1)`, `mode=iris`, and
  `geometry_semantics=we-iris-effect-final-scene-quad`. The Vulkan command log
  binds slot 0 to `we-image-effect-target-layer-63-0` and slot 1 to
  `resource-175-iris-mask-7c584a3b-frame-0.gtex`. Runtime overlap now reports
  `direct_base_swapchain=false`, meaning the base pupil mesh is no longer
  emitted directly to the swapchain before the opacity duplicate.
- 2026-07-01 closed-frame call-chain correction: the old converted scene at
  `/tmp/gilder-we-3742497499-output-restored-placement` was missing the MDLA
  opacity tail from `models/眼睛_puppet.mdl`, so `sample_puppet_animation()`
  produced `base_opacity_range=1.000..1.000` for both `node-77` and `node-89`.
  The original MDLA does contain the opacity block after a 5-byte tail preamble:
  one bone has non-default opacity with minimum `0.265767`. Runtime gscene
  loading now does not add a compatibility field; it resolves the puppet from
  `provenance.model.puppet` to the packaged `role=we-puppet-mdl` /
  `original_source` resource and backfills missing WE puppet opacity tracks
  before validation/snapshotting. The relevant call chain is now
  `load_scene_document()` -> puppet opacity backfill -> `snapshot_sampled_layers_at(time_ms)`
  -> `sample_puppet_animation()` -> sampled mesh vertices -> native
  sampled-image draw. Runtime eye debug logs `native-iris-mask` base-eye layers
  even when `alpha_slot=None`, so closed frames expose both `node-77` and
  `node-89`.
- 2026-07-01 rollback correction still applies to the old shortcut: do not set
  the layer's own alpha slot for iris and do not send iris through the
  rejected alpha/local-target shortcut. The current path is narrower: the layer
  stays `alpha_slot=None`; only the generated final effect draw step receives
  the iris mask slot so the shader can sample `g_Texture1`.
- The attempted mask UV scale `alpha_texture_extent / base_texture_extent` was
  also rejected. Gilder's `SceneTextureSlot` dimensions are decoded logical
  extents, not Wallpaper Engine backing texture extents. Until the converter
  preserves those backing extents separately, opacity material UV scale must
  stay identity `(1.0, 1.0)`.
- 2026-07-01 opacity rollback correction: the current attempted
  `native-opacity-mask` local-target route repeated the documented eye
  drift/disappear failure. The active renderer is back to the documented
  stable boundary: `node-77` (`1336` base eye) draws as a direct puppet mesh
  with `alpha_slot=None`; `node-89` (`1530`) remains an independent later-drawn
  duplicate and draws as a direct puppet mesh with `alpha_slot=Some(1)`,
  `mode=multiply`, and material-UV opacity-mask sampling. This keeps the
  two-image behavior: only `node-89`'s own alpha is multiplied by
  `masks/opacity_mask_d2f87f99`; it does not erase pixels already drawn by
  `node-77`. Do not hide, fold, or remove `1530`.
- The active visual result still needs user confirmation. Runtime evidence now
  proves the old direct base-eye draw path is gone for `node-77`, but the
  current native iris shader is still the existing simplified `mode=iris`
  sampled-image shader, not a complete port of all original `iris.vert`
  time/noise constants.
- Keep MDLE/rest-bind investigation as a fallback, not the primary next step.
  The immediate code path is now: `node-77` local effect target base mesh,
  `node-77` iris final scene quad, then later independent `node-89` direct
  duplicate mesh with its own opacity mask. Do not reintroduce the rejected
  whole-puppet y/v migration; keep mesh storage as `x = raw_x`, `y = raw_y`,
  `v = 1.0 - raw_v`.

## User-Visible Failure

- Closed-eye frames still show the pupil.
- Some attempted changes made the pupil disappear even when the eye was open,
  and some made unrelated face details such as eyebrows disappear.
- Treat runtime logs and frame-level evidence as primary. Do not claim this is
  fixed until logs and user observation agree.

## Required Constraints

- Follow `docs/native-vulkan-video.md`: no short-term substitutes,
  sample-specific fixes, magic offsets, layer hiding, resource re-export hacks,
  preview fallbacks, or temporary render paths.
- Do not hide, remove, fold, or special-case source/layer `1530`.
- Python commands must use `uv run`.
- Live tests must use `--output-name HDMI-A-1`.
- The correct direction is first-class Wallpaper Engine material/effect
  semantics, not another alpha-only shortcut.

## Original JavaScript Status

The user explicitly asked to extract the complete original JavaScript first.
That work is complete and must not be redone unless source input changes.

- Extracted directory:
  `artifacts/we-3742497499-js/`
- Key files:
  - `manifest.json`
  - `all-scripts.concatenated.js`
  - `README.md`
  - `scripts/*.js`
  - `unique/*.js`
- Manifest result:
  - `script_count`: `65`
  - `unique_script_count`: `11`
  - `load_errors`: `[]`

Important conclusion: extracted scripts do not directly control eye sources
`1336` or `1530`. The scripts using `ani.setFrame` target other layers such as
`1146`, `1148`, `1150`, `947`, `951`, and `953`. The eye failure is therefore
not explained by missing eye-specific JavaScript control.

## Scene Facts

Original source data:

- Source `1336`: base eye.
  - `image`: `models/眼睛.json`
  - `attachment`: `眼睛`
  - `parent`: `937`
  - `size`: `663 230`
  - animation clip `730`, rate `0.80000001`
  - effects:
    - `effects/iris/effect.json`
    - `effects/waterripple/effect.json`
- Source `1530`: opacity duplicate, not a disposable helper.
  - `image`: `models/眼睛.json`
  - `attachment`: `眼睛`
  - `parent`: `937`
  - `locktransforms`: `true`
  - `size`: `663 230`
  - animation clip `730`, rate `0.80000001`
  - effect:
    - `effects/opacity/effect.json`
    - mask `masks/opacity_mask_d2f87f99`

Generated runtime names/resources observed in the converted scene:

- Base eye node: `node-77-models-json`
- Opacity duplicate node: `node-89-models-json`
- Eye texture: `resource-173-frame-0.gtex`, `663x230`, BC7
- Iris mask: `resource-175-iris-mask-7c584a3b-frame-0.gtex`, `331x115`, R8
- Opacity mask: `resource-206-opacity-mask-d2f87f99-frame-0.gtex`,
  `331x115`, R8

Original hierarchy/order under parent source `937`:

- array index `67`, source `1072`, `裙子零件`, attachment `裙子`.
- array index `68`, source `1128`, `顶部头发`, attachment `头发`.
- array index `76`, source `1336`, `眼睛`, attachment `眼睛`, effects
  `iris` and `waterripple`.
- array index `77`, source `808`, `顶部头发`, attachment `头发`.
- array index `88`, source `1530`, `眼睛`, `locktransforms: true`,
  attachment `眼睛`, effect `opacity`.

Therefore source `1530` is an independent later-drawn duplicate under the same
body parent, not a child effect of `1336` and not a removable helper. The
first-class implementation must preserve that late-layer source ordering while
executing `1530`'s own local material/effect pass chain.

Observed draw-order context:

- Main body/source `937` appears as `node-67-models-json`.
- Base eye `node-77-models-json` appears around layer index `63`.
- Opacity duplicate `node-89-models-json` appears around layer index `74`.
- The nearby range includes hair/face/eye related layers, so parent/child
  flattening and first-class pass ordering matter.

## Material And Shader Evidence

Original materials:

- `/tmp/gilder-we-3742497499-extracted/materials/眼睛.json`
  - shader `genericimage4`
  - blending `translucent`
- `/tmp/gilder-we-3742497499-extracted/materials/effects/opacity.json`
  - shader `effects/opacity`
  - blending `normal`
- `/tmp/gilder-we-3742497499-extracted/materials/effects/iris.json`
  - shader `effects/iris`
  - blending `normal`

Opacity shader evidence:

- `opacity.vert` keeps base UV in `v_TexCoord.xy`.
- `opacity.vert` computes mask UV in `v_TexCoord.zw` by scaling with
  `g_Texture1Resolution.zw / g_Texture1Resolution.xy`.
- In the image/effect chain, opacity is not the first puppet material pass; its
  `a_TexCoord` is the later full pass/scene quad texcoord, not the puppet mesh
  material UV.
- `opacity.frag` samples previous pass texture `g_Texture0` at
  `v_TexCoord.xy`.
- `opacity.frag` samples mask `g_Texture1` at `v_TexCoord.zw`.
- `opacity.frag` applies `albedo.a *= mask * g_UserAlpha`.

Iris shader evidence:

- `iris.vert` computes both image UV and mask UV.
- `iris.frag` samples `g_Texture0`, samples mask `g_Texture1`, computes an iris
  offset, and assigns `albedo = iris`.
- Therefore iris is a real image-space effect pass, not metadata and not just an
  alpha mask.

## CWE Reference Semantics

The closest checked reference is CWE:

- `references/cwe/src/WallpaperEngine/Render/Objects/CImage.cpp`
- `references/cwe/src/WallpaperEngine/Render/Objects/Effects/CPass.cpp`
- `references/cwe/src/WallpaperEngine/Data/Parsers/MaterialParser.cpp`

CWE image/effect chain:

- First material pass draws the image or puppet mesh into a local FBO.
- Effect passes operate in pass space, usually as fullscreen/pass quads, reading
  the previous FBO as `g_Texture0`.
- A final pass composites the result back to the scene FBO.
- For a first pass with puppet geometry, `CImage::setupPasses` forces
  `BlendingMode_Translucent` and installs the puppet geometry callback.
- `CPass::setupRenderFramebuffer` maps `Translucent` to
  `SRC_ALPHA / ONE_MINUS_SRC_ALPHA` for both color and alpha, and maps `Normal`
  to `ONE / ZERO`.

This means the native renderer must model material/effect passes as a chain for
each image source. Source `1530` still remains an independent later-drawn scene
source. Long term its opacity effect belongs in that source's local pass chain,
but the current local-target attempt is disabled because live HDMI-A-1 evidence
matched the documented eye drift/disappear rollback. Collapsing it into `1336`
or hiding it is not equivalent to Wallpaper Engine semantics.

## Current Native Runtime State

Current native behavior is intentionally narrow:

- `effects/iris/effect.json` now creates a first-class effect target for
  `node-77` only. The base puppet mesh is local-target pass-space geometry; the
  final scene quad samples the local target and iris mask. The scene layer's
  own `alpha_slot` remains `None`.
- `effects/opacity/effect.json` leaves `node-89` as a direct second puppet
  mesh. Its `alpha_slot=Some(1)` mask is sampled in material UV space and
  multiplies only this duplicate image's alpha.
- The rejected local-target path for opacity must stay disabled until the full
  WE pass-space shader chain is implemented without causing eye drift or
  disappearance.

Consequence: keep iris first-class routing separate from the rejected alpha
shortcut, and do not re-enable the rejected opacity local-target shortcut as a
partial fix.

## Puppet Format Evidence

The previous handoff hypothesis that the whole native puppet format should be
migrated to `y = -raw_y` / `v = raw_v` is now contradicted by HDMI-A-1 evidence.
That migration broke established component placement: the character became a
scatter of wrongly positioned parts. Do not reapply it as an eye fix.

CWE `CImage::loadPuppetMesh` reads raw MDL vertex data as:

- `x = reader.nextFloat()`
- `y = reader.nextFloat()`
- `z = reader.nextFloat()`
- `u = reader.nextFloat()`
- `v = reader.nextFloat()`

CWE `CImage::updatePuppetPositionBuffer(size)` converts raw puppet coordinates
to pass-space positions as:

- `x = size.x / 2.0 + raw_x`
- `y = size.y / 2.0 - raw_y`
- `z = raw_z`
- UV keeps original `v`.

Gilder's existing solved scene convention, introduced by the puppet-runtime
work and covered by converter/runtime tests, stores puppet mesh and attachment
data differently because the rest of the native scene stack already uses its
own centered/y-down transform math:

- `x = raw_x`
- `y = raw_y`
- `v = 1.0 - raw_v`

MDLS/MDAT attachments, bind transforms, MDLA translations, and rotations must
remain in that established convention unless a full scene-space migration is
designed and proven across the already closed puppet-placement cases. For the
eye bug, CWE's pass-space equations are still useful inside the image/effect
FBO chain, but they must not be implemented by changing the global puppet
converter.

2026-07-01 raw-format inspection status:

- Verified: `models/眼睛_puppet.mdl` is `MDLV0023` with `MDLS0004`,
  `MDLA0006`, and `MDLE0002`.
- Verified: `MDLS` has `54` bones. The mesh block before `MDLS` has `4106`
  vertices and `23988` indices with 80-byte vertices. Skin weights sum to
  approximately `1.0`, use bone indices `0..53`, and are not the immediate
  cause of the leak.
- Verified: `MDLA` has one clip, id `730`, fps `30`, frame count `600`, loop
  playback, and `601` sampled frames per bone. Source layers `1336` and `1530`
  both play this clip at rate `0.80000001`.
- Verified: the current `pose_world * inverse_bind_world` skinning changes the
  eye around frame `300`, but the `node-77` local base FBO still contains
  visible dark pupil pixels. That is a real closed/blink-time output problem,
  not just a missing final opacity multiply.
- Superseded: `MDLE0002` contains `3456` bytes, exactly `54 * 64`, but it is
  not established as inverse-bind data. Follow-up evidence shows `MDLA` clip
  `730` frame `0` equals `MDLS` local bind transforms for all `54` bones, and
  `MDLE` matches `MDLS` forward local matrices for `45/54` bones. Treating
  `MDLE` as inverse-bind breaks the static/open-eye identity pose.
- Current conclusion: native `MDLS`/`MDLA` skinning is not the closed-eye root
  cause. The `node-77` geometry can close the eyelids; the active failure is
  render/effect composite, especially iris pass routing.

2026-07-01 offline blend/cull check status:

- Verified: culling alone does not explain the leak. The eye mesh triangle
  winding is overwhelmingly one-sided in the current pass-space output; dropping
  the small opposite-winding set does not remove the pupil.
- Verified: forcing normal overwrite in the local base pass removes too much and
  creates blocky wrong output even when open. That is not a viable fix and
  should not be implemented.
- Current interpretation: the desired closed-eye behavior is not `1530` setting
  the previous pupil alpha to zero. The base eye source `1336` must produce the
  correct deformed/covered local image before `1530` overlays its own opacity
  result.

## Known Debug Artifacts

Do not regenerate these unless needed:

- `/tmp/gilder-eye-assets/eye.png`
- `/tmp/gilder-eye-assets/eye.ppm`
- `/tmp/gilder-eye-assets/eye-alpha.pgm`
- `/tmp/gilder-eye-assets/opacity.png`
- `/tmp/gilder-eye-assets/resource-207-opacity-mask-d2f87f99-frame-0.pgm`
- `/tmp/gilder-eye-assets/iris.png`
- `/tmp/gilder-eye-assets/resource-175-iris-mask-7c584a3b-frame-0.pgm`
- `/tmp/gilder-eye-cpu-render/*.png`
- `/tmp/gilder-eye-sweep-render/contact.png`
- `/tmp/gilder-eye-sweep/snapshot-0.json` through
  `/tmp/gilder-eye-sweep/snapshot-24000.json`

Useful snapshot command:

```bash
target/release/gilder-native-vulkan --scene-runtime-snapshot \
  --source /tmp/gilder-we-3742497499-output-restored-placement/assets/scene.gscene.json \
  --scene-root /tmp/gilder-we-3742497499-output-restored-placement \
  --scene-time-ms 12000 \
  --fit cover
```

Useful live command shape:

```bash
GILDER_NATIVE_VULKAN_EFFECT_DEBUG=1 target/release/gilder-native-vulkan \
  --run-scene \
  --duration 35 \
  --output-name HDMI-A-1 \
  --source /tmp/gilder-we-3742497499-output-restored-placement/assets/scene.gscene.json \
  --scene-root /tmp/gilder-we-3742497499-output-restored-placement \
  --fit cover
```

## Implementation Direction

Longer-term direction: extend the first-class Wallpaper Engine image material
and effect pass chain. The current opacity local-target attempt is disabled;
opacity, iris, and other effects still need real shader-pass semantics.

1. Represent WE image rendering as:
   - first material pass into a local image/effect target,
   - ordered effect passes reading the previous pass as `g_Texture0`,
   - final scene composite.
2. Model pass-space geometry separately from scene-space geometry.
3. Preserve source `1530` as the independent later-drawn duplicate under
   parent `937`. In the current renderer, keep it as a direct duplicate puppet
   mesh with material-UV opacity masking; do not hide it, merge it into `1336`,
   or re-enable the rejected partial local-target route.
4. Source `1336` iris is now connected as a first-class two-step pass. Remaining
   shader work is to replace the simplified native `mode=iris` offset with the
   full original `iris.vert` time/noise constant semantics if visual evidence
   still differs.
5. Keep the established native puppet storage convention. Use CWE coordinate and
   UV semantics only inside the local pass-space effect chain where needed; do
   not migrate the whole scene again.
6. Add frame-level logs before live testing. Required fields:
   - time/frame and animation clip frame,
   - node id/layer index,
   - pass chain being executed,
   - material/effect file names,
   - input/output target identity,
   - texture slots and mask resources,
   - position/UV/effect-UV bounds,
   - alpha/blend mode,
   - sampled coverage summaries for eye/pupil/mask regions when feasible,
   - explicit `alpha_zero_reason` text that distinguishes original iris
     sampling from opacity-mask alpha multiplication,
   - opacity material UV scale, the texture/resource dimensions being reported,
     and whether those dimensions are decoded logical extents or real backing
     texture extents.

## Explicit Non-Solutions

- Do not remove or hide `1530`.
- Do not make `1530` invisible in closed-eye frames by sample id/name.
- Do not treat `effects/iris/effect.json` as a simple alpha shortcut.
- Do not keep tuning mask polarity or UV flips without resolving the pass-chain
  and puppet-format evidence above.
- Do not mark the bug fixed from build/test success alone. The required proof is
  targeted logs plus user observation on `HDMI-A-1`.
