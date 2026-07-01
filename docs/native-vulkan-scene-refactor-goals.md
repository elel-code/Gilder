# Native Vulkan Scene Refactor Goals

This document extracts the native scene renderer goals from
`docs/native-vulkan-video.md` into a standalone refactor target. The scope here
is the native scene renderer architecture, not the video pipeline and not the
deferred eye/closed-eye investigation.

## Scope

- Refactor the native scene renderer around first-class scene, material, effect,
  geometry, and binding concepts.
- Move effect evaluation toward the retained final per-frame boundary described
  in the Vulkan/CWE notes.
- Keep runtime evidence logging as part of the architecture, not as optional
  temporary debugging.
- Prepare the renderer for a binary scene format and retained/partial update
  path.

## Non-Goals

- Do not continue the eye/closed-eye fix in this work stream.
- Do not spend this work stream on video pipeline fixes.
- Do not treat file splitting as the refactor by itself.
- Do not optimize JSON trimming, whole-snapshot rebuilding, or compatibility
  branches as substitutes for retained scene architecture.
- Do not add Workshop-specific patches, preview fallback behavior, or hidden
  legacy lowering paths.

## Architecture Targets

1. Texture-slot and descriptor binding

   Introduce an explicit boundary for resource indices, texture slots,
   descriptor groups, sampler state, alpha/mask slots, and future third/fourth
   `g_TextureN` inputs.

2. Blend policy and equations

   Move blend, alpha, depth, cull, translucency, and effect blend semantics out
   of ad hoc draw branching and into typed render/material state.

3. Solid quads

   Isolate solid/color rectangle recording, transform handling, full-extent
   clears, text-as-solid geometry, and related batching decisions.

4. Sampled images

   Isolate sampled-image geometry, UV ranges, resource binding, tint/color
   treatment, alpha handling, and image-specific recording payloads.

5. Puppet and skinned geometry

   Isolate puppet mesh extraction, material UV selection, mesh bounds, bone or
   skinning inputs, per-frame mesh deltas, and geometry upload decisions.

6. Material and effect passes

   Represent `shader`, `blending`, `combos`, texture-pass metadata, masks,
   opacity, iris, water ripple/waves/flow/caustics, blur, sway, shake, and
   drift-like effects as first-class material/effect records instead of scattered
   string checks or one-off runtime branches.

   Dedicated effect modules must be introduced for the effect families that are
   currently mixed into generic draw/runtime branches:

   - `effect/opacity_mask`: opacity/mask sampling, mask coverage, mask UVs, alpha
     slot selection, sampled mask evidence, and final material alpha output.
   - `effect/iris`: iris-specific texture slots, UV transforms, alpha policy, and
     retained per-frame parameters.
   - `effect/water_ripple`: ripple parameters, UV disturbance inputs, time
     uniforms, and material pass lowering.
   - `effect/water_waves`: wave field parameters, UV/time evaluation, texture
     slots, and retained uniform updates.
   - `effect/water_flow`: flow direction/speed, flow-map or texture-pass inputs,
     UV offset evaluation, and descriptor binding.
   - `effect/water_caustics`: `genericimage2`/caustic material semantics,
     translucent blending, depth-disabled state, caustic texture slots, and
     regression logging.
   - `effect/blur`: blur pass parameters, source texture binding, pass ordering,
     and fullscreen/utility layer behavior.
   - `effect/sway_shake`: sway/shake transform parameters, per-frame deltas, and
     retained transform/effect uniform updates.
   - `effect/flutter`: hair, ribbon, clothing, skirt, accessory, and loose-part
     flutter semantics; wind/noise/time parameters; anchored-parent origin
     handling; per-layer phase/weight inputs; and retained vertex/material
     uniform updates. Flutter must resolve at the final retained per-frame
     transform/material/vertex boundary, after base node transforms, parent
     origins, timelines, and material parameters are known. It must not be
     implemented as scattered JSON sampling, early layer rewrites, per-frame CPU
     snapshot rebuilds, or unrelated mesh mutation before final composition.
   - `effect/drift`: hair, ribbon, clothing, and skirt drift parameters, geometry
     or vertex semantics, and changed-topology detection.
   - `effect/composite_layer`: `composelayer`/`fullscreenlayer` utility pass
     semantics, source layer binding, full-extent behavior, and pass ordering.
   - `effect/user_bindings`: user color/property bindings such as
     `newproperty5`/`newproperty6` lowered into material parameters.

7. Debug and evidence logging

   Keep targeted logs at render-plan, runtime composition, draw-pass, and Vulkan
   command-recording boundaries. Logs must explain visual gaps with draw order,
   layer ids, resource indices, texture slots, descriptor groups, alpha modes,
   push constants, blend mode, mesh bounds, UV ranges, mask coverage, sampled
   mask values, and per-frame transform/mesh deltas where relevant.

8. Binary scene format and retained updates

   Replace the current JSON runtime shape with a versioned binary scene format
   covering resource tables, compact transform/timeline/puppet data,
   material/effect pass records, texture-slot tables, blend/depth/cull state,
   random-access loading, and retained GPU binding/state IDs.

   The binary format must be designed as renderer input, not as a compressed copy
   of the existing JSON. Required chunk families:

   - `header`: magic, version, endian/alignment policy, feature flags, and chunk
     table offset.
   - `resource_table`: stable resource ids, image/video/buffer references,
     dimensions, color space, sampler defaults, and upload hints.
   - `node_table`: layer/node ids, draw order, parent/child links, visibility,
     and retained runtime ids.
   - `transform_timeline`: compact transform channels, parent origin data,
     interpolation mode, default values, and random-access frame ranges.
   - `geometry`: solid quad records, sampled-image quads, puppet meshes, mesh
     bounds, vertex/index streams, material UV sets, and topology-change ids.
   - `texture_slots`: first-class `g_TextureN` slot records, resource index,
     sampler state, UV source, alpha/mask role, and future third/fourth texture
     inputs.
   - `material_pass`: shader/material kind, combos, blend/depth/cull state,
     tint/user properties, descriptor layout id, and retained pipeline key.
   - `effect_pass`: effect kind, parameter block, source/target texture slots,
     time/uniform inputs, pass ordering, and evidence-log labels.
   - `flutter_state`: wind/noise curves, phase offsets, anchor/origin binding,
     affected vertex/material ranges, per-layer weights, and retained dirty
     ranges for hair/clothing/ribbon/skirt/accessory flutter. The chunk must
     preserve enough data for final-boundary evaluation instead of baking
     flutter into precomposed transforms or rewritten geometry.
   - `puppet`: bone/skinning data, mesh-to-material mapping, dynamic bounds, and
     retained skinning/update ids.
   - `render_state`: clear policy, full-extent utility passes, layer target
     routing, blend equations, depth/cull state, and pass dependencies.
   - `retained_gpu_state`: stable descriptor ids, buffer ids, image binding ids,
     pipeline ids, dirty ranges, and partial-update masks.
   - `debug_names`: compact ids for layer names, material/effect names, resource
     labels, and log correlation.

## Execution Order

1. Stabilize native scene renderer structure around typed draw-pass planning and
   recording boundaries.
2. Extract texture-slot/resource binding, blend policy, solid quad, sampled
   image, puppet geometry, material/effect pass, and evidence logging modules
   with real ownership of behavior.
3. Move effect evaluation to retained per-frame material/effect semantics rather
   than per-frame JSON/document sampling or large CPU geometry rebuilds.
4. Introduce the binary scene format and retained/partial update path, then
   migrate tests and fixtures to the new model.
5. Delete obsolete compatibility branches instead of preserving dual paths.

## Regression Guard

`Water Caustic` (`node-57-models-workshop-2790231929-wc-test-json`) remains the
concrete native scene regression guard for material/effect semantics:

- `shader=genericimage2`
- `blending=translucent`
- depth disabled
- ordered immediately after the long-hair layers

The expected fix path is first-class gscene/material/effect semantics, not
preview fallback, legacy loader mapping, resource probing, compatibility
branches, or Workshop-specific patches.

## Acceptance Criteria

- Draw-pass planning exposes typed routes and payloads instead of one giant
  inline branch chain.
- Recording code consumes typed solid, sampled-image, puppet, material, effect,
  and binding inputs.
- Effect decisions are represented as material/effect records that can be logged
  and retained across frames.
- Runtime logs can explain draw order, bindings, blend state, UV/mask inputs, and
  per-frame deltas for effect-related visual gaps.
- The design has a direct path to binary scene chunks and retained partial GPU
  updates.
