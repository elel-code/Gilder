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
- Use the validated WE eye evidence as regression-driven architecture work for
  first-class material/effect composition, iris routing, blend state, puppet
  animation-layer state, and pass targets.

## Non-Goals

- Do not treat the eye/closed-eye work as a sample-specific patch stream. Eye
  work in this plan must advance first-class puppet, material, effect, blend, and
  pass-chain architecture.
- Do not spend this work stream on video pipeline fixes.
- Do not treat file splitting as the refactor by itself.
- Do not optimize JSON trimming, whole-snapshot rebuilding, or old dual branches
  as substitutes for retained scene architecture.
- Do not add Workshop-specific patches, preview fallback behavior, or hidden
  old lowering paths.

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

   Workshop scene `3742497499` no longer treats `MDLE0002` as a proven
   inverse-bind source. Current evidence shows `MDLA` frame `0` matches `MDLS`
   bind local transforms and current `MDLS` parent-chain inverse skinning can
   close `node-77` by itself. Do not add MDLE/inverse-bind scene fields or dual
   skinning paths unless new evidence proves the semantics in the `MDLA` pose
   space.

6. Material and effect passes

   Represent `shader`, `blending`, `combos`, texture-pass metadata, masks,
   opacity, iris, water ripple/waves/flow/caustics, blur, sway, shake, and
   drift-like effects as first-class material/effect records instead of scattered
   string checks or one-off runtime branches.

   WE image pass chains must model `node-77` iris/effect composition as
   first-class local-target and final-composite passes. They must also model
   `normal` blend as an explicit overwrite blend equation, `locktransforms` as
   first-class puppet animation-layer input, and opacity/iris mask effect-UV
   transform from preserved WE pass metadata and backing texture extents.
   Existing decoded logical extents, alpha/base extent ratios, or sample-specific
   constants must not be used as substitutes for the original WE UV transform.

   Current implementation progress:

   - `SceneEffectPass` now carries a typed `SceneEffectUvTransform` for
     opacity/iris mask passes. The converter lowers WE pass constants,
     mask/source slots, node extents, and mask backing extents into this record
     instead of deriving UV scale from decoded alpha/base image ratios.
   - Render-plan, runtime, and draw-pass UV generation consume the typed
     transform as `scale + offset`, and the old identity scale helper has been
     removed from the active path.

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
   - `transform_keyframes`: compact time/value/curve records referenced by
     `transform_timeline`; timeline channels must not remain summary-only if
     renderer-side sampling needs their full keyframe stream.
   - `geometry`: solid quad records, sampled-image quads, puppet meshes, mesh
     bounds, vertex/index streams, material UV sets, and topology-change ids.
   - `particle_emitter`: typed emitter settings, deterministic seed/lifetime,
     spawn/particle extents, velocity/gravity ranges, shape, color, fade/loop
     flags, and retained runtime ids.
   - `texture_slots`: first-class `g_TextureN` slot records, resource index,
     sampler state, UV source, alpha/mask role, and future third/fourth texture
     inputs.
   - `material_pass`: shader/material kind, combos, blend/depth/cull state,
     tint/user properties, descriptor layout id, and retained pipeline key.
   - `effect_pass`: effect kind, parameter block, source/target texture slots,
     time/uniform inputs, pass ordering, and evidence-log labels.
   - `effect_uv_transform`: source/mask texture slots, input/mask/backing
     extents, transform scale, transform offset, mapping kind, and retained
     state ids for opacity/iris mask UV evaluation.
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

   Current implementation progress:

   - Binary version `12` uses a fixed chunk-table format with typed chunks for
     resources, nodes, transform timelines, transform keyframes, geometry
     streams, particle emitters, texture slots, material/effect passes, effect
     parameters, effect-UV transforms, flutter state, puppet skin bones/vertices,
     attachments, clips, frames, animation layers, render state, retained GPU
     state, and debug names.
   - The native Vulkan CLI can now build render layers from `.gscn` directly for
     static binary-scene smoke with a header/table/range reader: resource paths,
     node visual state, default transforms, mesh vertex/index streams, material
     texture slots, effect passes, alpha state, effect-UV transforms, and scalar
     transform/opacity/extent timelines are reconstructed without reading
     `.gscene.json` or retaining the full binary payload. Puppet skinning now
     reconstructs skin bones, vertex weights, clips, frames, and animation layers
     from binary ranges and samples mesh vertices on the `.gscn` path. Particle
     emitters now expand from typed binary records on the direct `.gscn` path
     instead of reading JSON properties. Remaining binary render gaps are
     material/effect graph execution, effect target compositing, and retained
     runtime updates.
   - `.gscn` node records now carry resolved default user-condition visibility,
     text/font payloads, and parent-composed transform/opacity state on direct
     binary ingest. The workshop `3742497499` eye binary smoke no longer draws
     the hidden pure-color theme layer, no longer places child utility layers at
     local origin, and no longer rejects text layers for missing text payloads.
   - `node_table` now carries direct child/subtree, transform-range, material,
     geometry, puppet, and static visual-state data: opacity, packed color,
     packed stroke color, stroke width, corner radius, and fit mode.
     `transform_timeline` records point to `transform_keyframes` ranges, so
     targeted timeline sampling can be driven from binary records instead of
     JSON summaries.
   - Stream ingest validates chunk shape with record-sized reads and keeps
     keyframes/geometry streams as counted record ranges rather than retaining
     full JSON-derived tables.
   - The current `3742497499` binary eye smoke reports `unsupported_layer_count=0`,
     `draw_op_count=3862`, `sampled_image_layer_count=3845`, `chunk_count=24`,
     `effect_pass_count=120`, and `retained.record_count=1714`. The remaining
     visible failure is no longer missing particle support; it is the
     `shader-material-graph` boundary: large effect/material patches and missing
     composites until WE pass targets, material graph execution, UV transforms,
     and blend state are first-class end to end.
   - Conversion now emits a `.gscn` binary scene asset from the typed
     `SceneDocument`, and the native Vulkan CLI can accept `.gscn` sources for
     direct binary-scene smoke/ingest without routing through the JSON scene
     loader.

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
5. Delete obsolete dual branches instead of preserving parallel old paths.

## Regression Guard

`Water Caustic` (`node-57-models-workshop-2790231929-wc-test-json`) remains the
concrete native scene regression guard for material/effect semantics:

- `shader=genericimage2`
- `blending=translucent`
- depth disabled
- ordered immediately after the long-hair layers

The expected fix path is first-class gscene/material/effect semantics, not
preview fallback, old loader mapping, resource probing, dual branches, or
Workshop-specific patches.

`WE Eye Closed Frame` (workshop scene `3742497499`) is now an accepted
architecture regression guard. Obsolete eye handoff and MDLE/composite
misdiagnosis documents have been deleted so the active plan has a single root
cause:

- `docs/native-vulkan-we-eye-iris-mask-uv-root-cause.md`: current root cause and
  fix direction. The `node-77` iris pass samples the effect target at an offset
  driven by a nonzero iris mask in eyelid pixels. Identity mask UVs were a
  trigger in the old path, but the fix is full WE effect-UV
  transform/backing-extent semantics plus first-class pass target composition,
  not a decoded alpha/base ratio or any MDLE/inverse-bind change.
- `docs/native-vulkan-we-eye-original-semantics.md`: source evidence for the WE
  layer/material/effect semantics used by the implementation plan.

The required fix path is:

1. Preserve the current MDLS/MDLA skinning path for the eye bug; do not add
   MDLE/inverse-bind fields or dual branches.
2. Build first-class `node-77` iris/effect composite routing with local target,
   mask/effect slots, final scene composite, draw order, and evidence logs.
3. Add explicit `normal` blend semantics through core scene state, render-plan
   state, and Vulkan blend equations.
4. Lower `locktransforms` into first-class puppet animation-layer state instead
   of reading provenance at runtime.
5. Preserve WE backing texture extents and effect-UV transform inputs in
   texture/effect records. Use those records to compute iris/opacity mask UVs.
   Do not use current decoded logical extents, alpha/base ratios, or
   sample-specific constants as substitutes. The first typed gscene and binary
   records for this are now in place; remaining work is full WE pass-chain
   routing and visual validation on the closed/open eye frames.
6. Keep source `1530` as an independent later-drawn source; do not hide, fold,
   or special-case it. Validate the final pass chain with targeted logs and
   HDMI-A-1 observation.

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
