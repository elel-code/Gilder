# Native Vulkan Scene Refactor Goals

This is the active native scene renderer refactor target. The scope here is the
native scene renderer architecture, not the video pipeline and not a deferred
eye/closed-eye investigation.

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

   Blend is first-class pass state. WE `colorBlendMode` must be represented as
   the same passthrough material pass CWE appends to the image chain, and the
   first-pass blend must be moved to the final scene-composite pass when a chain
   contains multiple passes. Raw sampled-image fallback must never stand in for
   this pass routing.

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

   The mainline is full WE/CWE image effect graph execution, documented in
   `docs/native-vulkan-we-effect-graph-mainline.md`. The renderer must model the
   same pass-chain contract as CWE: base material pass, visible effect passes,
   declared effect FBOs/binds, image-local ping-pong targets, optional
   `colorBlendMode` passthrough, and final scene composite. Hiding an
   unsupported effect carrier is only a temporary guard against drawing the wrong
   raw source quad; it is not the architecture target.

   Effects are first-class pass records. No effect family should be reduced to a
   late draw-pass string check, a hidden layer flag, or a global "skip water"
   rule. The effect module owns parameter decoding and shader/material graph
   participation; the graph executor owns ordering, targets, bindings, and final
   blend routing.

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
   - Opacity mask effects now route through a first-class local effect target:
     the base mesh renders to a retained target, the final scene quad samples
     `g_Texture0` from that target and `g_Texture1` from the mask, `normal`
     blend remains an overwrite equation, and final alpha uses coverage semantics
     instead of a direct material-UV alpha-mask draw.
   - A narrow temporary guard prevents raw direct composition of unimplemented
     water ripple/flow/caustics passthrough carriers while their WE
     fragment/material graph is not implemented. The guard must not suppress
     alpha/normal character layers or waterwaves hair/body layers. The permanent
     fix remains the full WE image effect graph executor in
     `docs/native-vulkan-we-effect-graph-mainline.md`; rectangle evidence is in
     `docs/native-vulkan-we-water-rectangle-root-cause.md`.
   - A typed `we_image_pass_chain` is now produced for sampled-image quads that
     need WE graph semantics. It records local target/ping-pong requirements,
     base/effect/passthrough pass roles, final-scene blend routing, execution
     mode, and unsupported reason. This is the first implementation step toward
     replacing raw direct fallback with a real graph executor.
   - Draw-pass planning now promotes those chains into a first-class graph step
     plan with chain/step counts and per-step input/target/final-blend evidence.
     The plan also carries first-class graph target records for image-local
     main/sub targets and first-class effect targets, plus target indices on
     every graph step. Pass records preserve texture slots, shader/effect file,
     parameter keys, combo keys, blend/depth/cull state, and final-scene routing.
     The existing opacity/iris executable effect-target path now records its
     graph chain/target/step linkage, so the old first-class target route is
     being folded into the general WE graph executor instead of remaining a
     special case. Collapsed legacy steps are left unlinked when they do not
     correspond to the planned graph, which keeps remaining multi-pass work
     visible instead of faking completion. Runtime graph target snapshots now
     distinguish allocated Vulkan effect targets from planned-only graph
     targets and carry the Vulkan effect-target resource index. Graph steps now
     also carry typed `g_TextureN` binding plans: base passes bind source
     texture input, effect/passthrough passes bind the previous graph target as
     `g_Texture0`, and effect-owned texture slots such as water ripple normal
     maps remain explicit pass texture bindings with source path and resolution.
     Effect pass `command/source/target/binds` are now preserved from the WE
     effect file through render planning, graph planning, runtime snapshots, and
     direct `.gscn` binary ingest. Effect-declared FBOs are also preserved as
     typed scene data and graph targets with name, format, scale, uniqueness,
     scaled extent, write counts, and sampled-by-following-pass evidence. Binary
     format version `14` stores `command/source/target` on `effect_pass` and
     stores pass binds/FBO declarations as typed `PASS_BIND`/`EFFECT_FBO`
     effect parameters. Runtime snapshots annotate texture bindings with
     sampled-image or allocated effect-target resource indices when available,
     and bind overrides now distinguish `previous-graph-target` from resolved
     or unresolved `named-fbo-bind`. The next implementation boundary is to
     allocate named FBO targets, bind all executable graph targets/texture
     bindings to real Vulkan descriptors, and execute pass-specific
     shader/material modules.
   - Runtime snapshots now expose a first-class WE graph resource table: file
     texture sources and graph targets share one planned resource index space.
     Texture bindings carry `planned_graph_resource_index`; allocated
     opacity/iris targets additionally carry the real Vulkan effect-target
     index, while unallocated water main/sub targets remain visible as
     `planned-until-graph-executor` resources. This keeps descriptor/resource
     work on the main graph-executor path instead of hiding it behind the
     temporary rectangle guard.
   - WE graph passes now carry typed render state as well as blend labels:
     runtime snapshots expose the resolved blend equation, depth flags, and cull
     mode per pass. This is the evidence boundary that blend has moved into
     first-class pass execution data instead of staying as a final-quad label.
   - Native draw-pass planning now inventories all visible WE effect passes,
     including non-image draw ops. Runtime snapshots report the total effect
     pass count, effect-bearing non-image layer count, and counts by typed
     effect family. This keeps text/material effects such as `scroll`,
     `colorkey`, `clipping_mask`, and `rounded_mask` on the main graph-executor
     work list instead of treating the sampled-image water case as the whole
     problem.
   - Runtime draw-op snapshots now preserve the typed effect pass records on
     those non-image layers too. The executor still has to run them, but their
     effect file, shader, pass index, binds/FBO metadata, and classified family
     now survive into renderer evidence instead of collapsing to a count.
   - Performance evidence is tracked as part of the same first-class execution
     boundary. The current scene `3742497499` smoke produced `3843` sampled-image
     recording steps but only `71` Vulkan draw calls after command coalescing.
     The immediate regression was not the step count by itself: two allocated
     opacity/iris effect targets disabled recorded command-buffer reuse and
     forced per-frame re-recording. Reuse is now controlled by a typed material
     `uses_elapsed_push_constants` flag, so static first-class effect targets do
     not pessimize the whole scene; actual time-driven material effects must opt
     into per-frame command data until their uniforms are retained/dynamic.
   - 2026-07-02 release smoke after the `waterwaves` blend-boundary fix ran the
     real scene for `10.009s` on `HDMI-A-1`: `479` frames, `47.854 FPS`,
     `68` Vulkan draw calls (`60` sampled-image, `8` solid), `19` pipeline
     binds, `3839` sampled-image recording steps, `248` WE graph chains,
     `547` WE graph steps, `275` WE graph targets, `343` WE graph resources,
     and `99` visible effect passes. The user confirmed that the one-large/
     one-small rectangles disappeared, and also observed that other thick
     outline artifacts around patterned layers disappeared. This reinforces
     that blend state must remain first-class pass data: the old bug was
     effect-pass `normal` escaping into final scene composite.
   - Remaining visual gaps from the same smoke are now tracked as first-class
     semantics, not water-specific patches:
     1. Hair does not follow head/body motion. The affected hair groups
        (`node-61`, `node-63`, `node-69`, `node-78`, etc.) are WE puppet
        attachment groups named `头发`, parented to `node-59/node-67`
        `主身体` puppet attachments. The converter preserves attachment
        metadata (`bone_index=42`, `placement_source=mdls-bone-matrix-chain`),
        but native runtime still has to apply the sampled parent puppet bone
        pose every frame to child attachment transforms, including group
        children with waterwaves/material graph participation.
     2. Eyes are still visually incomplete. The scene has two eye layers:
        `node-77` (`effects/iris` + `waterripple`) and `node-89`
        (`effects/opacity`), both attached to the `眼睛` puppet attachment on
        the body puppet and both using clip `730`. WE semantics require
        base puppet mesh -> local effect target -> iris/opacity material pass
        -> final scene composite with `normal` overwrite where authored. The
        current opacity/iris target path is first-class evidence, but the full
        two-layer FBO/mask/effect-UV/composite behavior remains the next
        executor boundary.

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
   - `effect_pass`: effect kind, `command/source/target`, parameter block,
     source/target texture slots, pass binds, time/uniform inputs, pass
     ordering, and evidence-log labels.
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
   - `.gscn` sampled-image present now uses a binary runtime sampler instead of
     falling back to a single static frame. The sampler keeps only the file
     reader, chunk table, debug-name index, resource table, package root, and
     scene constants; each frame re-reads the required binary ranges for
     transform timelines, opacity, puppet meshes, and particle emitters, builds
     vertex-only dynamic updates against the retained draw topology, then drops
     frame layers. It does not load `.gscene.json` and does not retain the
     binary payload.
   - `3742497499` release binary eye smoke now presents dynamic `.gscn` frames:
     `frames_presented=600`, `average_present_fps=29.99` at `--target-fps 30`,
     `vertex_buffer_count=4`, `particle_emitter_count=10`, and
     `payload_retention_model=read-header-table-stream-records-drop-source-bytes`.
     The scene is no longer frozen by the binary path. The remaining visible
     failure is still `shader-material-graph`: WE pass-target composition,
     material/effect graph execution, blend, and effect-UV semantics must be
     completed rather than patched per sample.
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
   - Binary format version `14` extends effect records for WE graph execution:
     `effect_pass` carries `command/source/target`, `effect_parameter` carries
     pass binds with `PASS_BIND` and effect-declared FBOs with `EFFECT_FBO`, and
     direct binary ingest reconstructs binds, FBOs, combos, and constant shader
     values into render effect passes.

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
architecture regression guard. Obsolete handoff documents and the old
video-scene catch-all document have been deleted so the active scene plan has a
single root cause:

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
