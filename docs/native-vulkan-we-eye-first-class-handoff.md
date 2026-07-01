# Native Vulkan WE Eye First-Class Handoff

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

- 2026-07-01 rollback correction: the attempted `native-iris-mask`
  first-class/alpha-texture shortcut was rejected because it makes the whole
  eye render through the wrong pass path. Until the real WE iris shader is
  implemented, `effects/iris/effect.json` stays effect metadata only:
  `node-77` (`1336` base eye) must render as direct puppet mesh with
  `alpha_slot=None`, not through an iris alpha slot or effect-target final
  quad. This matches the rollback note in `docs/native-vulkan-video.md`.
- The attempted mask UV scale `alpha_texture_extent / base_texture_extent` was
  also rejected. Gilder's `SceneTextureSlot` dimensions are decoded logical
  extents, not Wallpaper Engine backing texture extents. Until the converter
  preserves those backing extents separately, opacity material UV scale must
  stay identity `(1.0, 1.0)`.
- The current intended topology is: `node-77` (`1336` base eye) renders its
  puppet mesh directly to the scene, while `node-89` (`1530`) remains an
  independent later opacity-masked image under parent `937`. The
  `native-opacity-mask` FBO/effect-target route is backed out for now because
  it reintroduced whole-eye loss/misalignment; the active code path is the
  simple two-image behavior: draw the duplicate puppet mesh and multiply that
  image's alpha by the opacity mask sampled in material UV space. Do not hide,
  fold, or remove `1530`.
- The active visible bug remains the closed-eye/blink pupil leak until HDMI-A-1
  observation confirms otherwise. Before the rejected mask-UV-scale attempt, at
  `time_ms=12000` in `/tmp/gilder-eye-hdmi-a-1-iris-firstclass-rerun.log`,
  `node-77` still had visible dark base texels while
  `base_opacity_range=0.266..1.000 below_one=203/4106`.
- Keep MDLE/rest-bind investigation as a fallback, not the primary next step.
  The immediate code path is the simple per-image effect semantics: opacity
  pass samples the previous image and multiplies that image's alpha by its
  mask; iris samples the previous image through its own mask UV. Do not
  reintroduce the rejected whole-puppet y/v migration; keep mesh storage as
  `x = raw_x`, `y = raw_y`, `v = 1.0 - raw_v`.

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
- Opacity mask: `resource-207-opacity-mask-d2f87f99-frame-0.gtex`,
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
source, but its opacity effect is applied through that source's local pass chain;
collapsing it into `1336`, hiding it, or treating it as a scene-level
alpha-textured replacement is not equivalent to Wallpaper Engine semantics.

## Current Native Runtime State

Current native behavior is intentionally narrow:

- `effects/iris/effect.json` remains metadata only until the real iris shader
  pass exists; `node-77` must stay a direct puppet mesh with no alpha slot.
- `effects/opacity/effect.json` keeps its mask texture slot, but the renderer
  no longer routes `node-89` through an effect target. The retained and dynamic
  paths both draw it as a second puppet mesh with `alpha_slot=Some(1)` and
  `effect_uv` equal to material UV.
- The next validation target is therefore not "previous layer erasure"; it is
  whether the second eye image is present and its masked part becomes alpha 0
  without making the whole eye disappear or drift.

Consequence: do not re-enable the rejected iris alpha shortcut or the opacity
effect-target route unless a real WE pass implementation replaces the current
sampled-image alpha-mask shader.

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
- Verified: `MDLE0002` contains `3456` bytes, exactly `54 * 64`, which parses
  cleanly as one 4x4 matrix per bone. Matrices `45/54` match the MDLS local
  matrix convention, while `9` parent-chain/key bones differ. The strongest
  discrepancy is in root/parent-0 child bones, where x translation offsets are
  much larger than the current computed bind-chain values. Do not ignore
  `MDLE0002` as padding; it remains the most important candidate for missing
  original rest/inverse-bind semantics.
- Verified: CWE only implements raw puppet mesh pass-space drawing for this
  path. It does not implement the full MDLS/MDLA/MDLE skinning semantics needed
  by this scene, so CWE can be used as a reference for image/effect FBO pass
  space, material pass chaining, and blend setup, but not as proof that native
  MDLS/MDLA skinning is currently correct.

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

Longer-term direction: implement a real first-class Wallpaper Engine image
material and effect pass chain. Current repair must keep the simple direct
opacity-mask path until that real pass implementation exists.

1. Represent WE image rendering as:
   - first material pass into a local image/effect target,
   - ordered effect passes reading the previous pass as `g_Texture0`,
   - final scene composite.
2. Model pass-space geometry separately from scene-space geometry.
3. Preserve source `1530` as the independent later-drawn duplicate under
   parent `937`. In the current renderer, run it as a direct puppet mesh whose
   alpha is multiplied by the opacity mask in material UV space; do not hide it,
   merge it into `1336`, or route it through the rejected effect-target shortcut.
4. Run source `1336` iris as a real pass that samples `g_Texture0` and the iris
   mask, rather than leaving `native-iris-mask` unconnected.
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
