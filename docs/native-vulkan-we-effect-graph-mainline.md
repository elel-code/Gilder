# Native Vulkan WE Effect Graph Mainline

This is the main implementation direction for Wallpaper Engine image effects in
the native Vulkan renderer. The rectangle guard is only a temporary correctness
fence; it is not the target architecture.

## Goal

Implement the WE/CWE image material/effect graph as first-class native Vulkan
passes:

1. Render the base image/model material pass into a per-image local target.
2. Execute every visible image effect material pass in order.
3. Resolve pass-local FBOs and texture binds, including `previous`, `_rt_*`, and
   `_alias_*`.
4. Ping-pong intermediate passes through the image main/sub targets.
5. Apply optional `colorBlendMode` through the `materials/util/effectpassthrough`
   pass at the end of the local chain.
6. Composite only the final pass to the scene FBO, with the original first-pass
   blend mode moved to that final scene pass.

The renderer must not treat an image layer with an unimplemented effect chain as
a plain sampled-image quad. That was the source of the earlier raw carrier
rectangles in scene `3742497499`.

2026-07-02 correction: the final one-large/one-small rectangle pair that the
user confirmed fixed was a separate pass-boundary bug on `waterwaves` character
layers. Those layers were valid visible image layers and had to stay recorded;
the bug was that Gilder promoted the effect material pass `blending="normal"`
to the final swapchain composite instead of keeping the base material
`blending="translucent"` as the scene blend.

## Hard Requirements

Effects and blend are first-class renderer data, not labels attached to a final
quad:

- `effect_file`, shader, combos, constants, target, binds, texture slots, and
  visibility must survive lowering as typed pass records.
- Blend/depth/cull state belongs to the pass that actually writes the target.
  In a multi-pass image chain, the original first-pass blend must move to the
  final scene composite, matching CWE.
- WE `colorBlendMode` is a material passthrough pass in the chain, not a single
  fixed-pipeline blend flag on the raw source image.
- An effect approximation may change math only inside its owned effect module.
  It may not bypass local targets, texture bindings, pass order, or final blend
  routing.
- A missing effect implementation must report an unsupported graph boundary or
  use a narrow guard; it must not silently draw the carrier texture as if no
  effect existed.

## CWE Reference Shape

Primary local reference files:

- `references/cwe/src/WallpaperEngine/Render/Objects/CImage.cpp`
- `references/cwe/src/WallpaperEngine/Render/Objects/Effects/CPass.cpp`
- `references/cwe/src/WallpaperEngine/Data/Parsers/EffectParser.cpp`
- `references/cwe/src/WallpaperEngine/Data/Parsers/ObjectParser.cpp`

Important behavior already confirmed from CWE:

- `CImage::setup()` appends model material passes, visible image effect passes,
  optional compatibility passes, and optional `colorBlendMode` passthrough.
- If there is more than one pass, the first pass blend is moved to the last pass
  and the first pass becomes `normal`.
- `CImage::setupPasses()` routes intermediate passes through image-local main/sub
  FBOs and sends only the final pass to the scene FBO.
- `configurePassTarget()` resolves explicit effect `target` FBOs and keeps a
  target-effect sequence input.
- `pinpongFramebuffer()` swaps the local image main/sub FBOs after ordinary
  intermediate passes.
- `CPass::setupTextureUniforms()` resolves shader, material, user override, and
  effect bind textures into `g_Texture0` through `g_Texture7`.
- `EffectParser::parseBinds()` and `parseFBOs()` are part of the graph model, not
  optional metadata.

## Required Renderer Model

Introduce a typed WE image effect graph lowerer before draw-pass recording:

- `WeImagePassChain`: ordered material/effect/passthrough passes for one image.
- `WeImageLocalTarget`: main/sub per-image targets plus effect-declared FBOs.
- `WeTextureBinding`: `g_TextureN`, source role, sampler, resource/FBO target,
  override source, and resolution uniform.
- `WePassState`: shader id, combos, constants, blend/depth/cull state, target,
  input, previous input, geometry space, UV space, and final-scene flag.
- `WeEffectRuntime`: per-effect time uniforms, user properties, audio inputs, and
  retained dirty state.

The Vulkan draw pass should record graph passes, not infer effect behavior from
string checks at the last minute.

Current implementation progress:

- Native draw-pass planning now builds a typed `we_image_pass_chain` for sampled
  images that need WE graph semantics. The chain records base material,
  effect-material, and `colorBlendMode` passthrough roles; local image target
  requirement; ping-pong requirement; first-blend-moved-to-final policy; final
  scene blend mode; temporary execution mode; and the specific unsupported
  reason when the chain is guarded instead of executed.
- Runtime snapshots expose `we_image_pass_chain` on affected sampled-image quads,
  so regressions can prove whether a layer is direct, first-class target,
  temporary raw fallback, or suppressed until the graph executor exists.
- The rectangle guard now consumes that pass-chain decision. It no longer lives
  as an unstructured "skip water" rule in draw-pass visibility logic.
- Draw-pass planning also emits a first-class WE image graph step list:
  chain counts, suppressed/fallback/first-class-target chain counts, and each
  base/effect/passthrough step with layer id, chain index, input, target,
  execution mode, final scene blend, and unsupported reason. This is the bridge
  for replacing the evidence-only chain with actual Vulkan local-target
  recording.
- Graph planning now also emits first-class local target records. Each chain
  target has a stable target index, endpoint (`image-local-main`,
  `image-local-sub`, or `first-class-effect-target`), extent, execution mode,
  first write step, write count, sampled-by-following-pass flag, and
  scene-composite-source flag. Steps carry input/output target indices, so pass
  order, ping-pong, and final scene routing are no longer implicit.
- WE pass records now preserve texture slot lists, combo keys, parameter keys,
  shader, effect file, blend/depth/cull state, input endpoint, output endpoint,
  and final-scene flag. This makes effects and blend renderer data instead of
  late string checks.
- WE graph passes now also carry a typed `render_state` snapshot with blend
  equation, depth-test/depth-write, and cull mode. This makes the CWE
  first-blend-to-final-pass move executable data rather than a descriptive flag:
  intermediate local passes can be `normal`, while the final scene pass owns the
  original WE `colorBlendMode` equation.
- Draw-pass planning now also emits a global WE effect-pass inventory across
  all visible draw ops, not only sampled-image graph steps. This exposes
  effect kind counts and the number of effect-bearing non-image layers, so text
  and other renderable effect passes are visible as first-class pending graph
  work instead of disappearing behind sampled-image-only evidence.
- Runtime snapshots now also preserve per-draw-op typed effect pass records for
  non-image layers. A text or rectangle layer with `scroll`, `color-key`,
  `clipping-mask`, or `rounded-mask` now exposes the same effect file, shader,
  pass index, bindings, and family classification as sampled-image graph
  inputs.
- WE effect pass graph fields are now preserved from the converter forward:
  `command`, `source`, `target`, and `bind`/`binds` are merged from the effect
  file pass, material-pass shader state, and object pass overrides. The render
  plan, draw-pass effect records, graph pass records, runtime snapshots, and
  `.gscn` binary ingest path all carry these fields as typed data. Effect file
  passes are preserved even when the object instance omits its own `passes`
  array, so file-declared graph passes do not disappear during conversion.
- WE effect-declared FBOs are now typed scene data. The converter preserves
  `fbos:[{name, format, scale, unique}]`; scene snapshot/render pass records
  carry them; the graph planner creates `named-fbo` targets with scaled extent,
  format, uniqueness, first-write count, and sampled-by-following-pass evidence.
- Binary scene format version `14` stores effect pass `command/source/target`
  names directly in the `effect_pass` record and stores pass binds/FBOs as typed
  `effect_parameter` records with `PASS_BIND` and `EFFECT_FBO` roles. Direct
  `.gscn` ingest reconstructs `binds`, FBOs, combos, and constant shader values
  for `SceneRenderImageEffectPass`, so binary no longer drops graph pass fields.
- Each graph step now carries a typed `g_TextureN` binding plan. The current
  lowering records `g_Texture0` as the source texture for base-material passes,
  as the previous graph target for effect/passthrough passes, and records
  effect-declared pass texture slots such as `waterripple` normal maps as
  separate `pass-texture-slot` bindings with source path and resolution. Runtime
  snapshots resolve those bindings back to sampled-image resource indices when
  file resources are allocated, and to Vulkan effect-target resource indices
  when the target is already executable.
- Effect pass binds override ordinary slot resolution the way CWE does:
  `previous` becomes a `previous-graph-target` binding, while non-`previous`
  names become `named-fbo-bind` bindings. If the named FBO is declared or
  targeted by a pass, the bind resolves to that graph target index and extent;
  otherwise the name is still preserved as an unresolved FBO bind.
- The existing executable opacity/iris first-class effect-target path is now
  linked back to the WE graph plan. Allocated effect targets record their graph
  chain index, graph target index, and graph endpoint; recorded draw steps carry
  graph chain/step and input/output target indices where the current executor
  really matches a graph step. Multi-pass chains that the old first-class route
  still collapses remain explicitly unlinked for the collapsed step instead of
  pretending the full graph executed.
- Runtime graph target snapshots now mark whether a target is an actual
  allocated Vulkan effect target or only planned until the graph executor exists.
  Allocated target records carry both the graph target index and the Vulkan
  effect-target resource index, and Vulkan effect target resource labels include
  the graph chain/target endpoint.
- Runtime snapshots now also expose a first-class WE graph resource table. It
  contains file texture inputs and graph target outputs in one stable resource
  index space. Texture resources carry their source path and extent; graph
  target resources carry layer/chain/endpoint/allocation state. Texture
  bindings point at `planned_graph_resource_index` even when the target is not
  allocated yet, and already executable opacity/iris bindings also point at the
  real Vulkan effect-target resource. This is the resource boundary needed
  before descriptor binding can execute the graph.
- Final scene material selection now preserves the base image material blend
  when an effect pass says `normal`. In WE/CWE, `normal` on an effect pass is
  the local FBO overwrite state, not permission to overwrite the swapchain.
  The converter/runtime now recognizes material `translucent` as
  `SceneBlendMode::Alpha`, binary material lowering prefers
  `node.properties.material.passes[0]` over effect passes, and draw-pass
  recording no longer lets an effect pass override the scene composite blend.
  This keeps visible `waterwaves` hair/body layers in the plan while preventing
  their transparent source rectangles from overwriting the character/background.

The next coding boundary is no longer "detect the rectangle"; it is to bind the
typed graph targets and per-step texture binding plans to descriptor sets, then
execute pass-specific shader/material modules against those targets.

## Effect Families

The first compatibility target is all effects observed in workshop scene
`3742497499`, plus the built-in families that share the same graph machinery:

- `opacity`: mask texture sampling, alpha policy, effect UV transform, local
  target final composite.
- `iris`: iris mask texture slots, UV transform, final alpha coverage.
- `waterripple`: fragment UV refraction using normal texture input and `g_Texture0`
  resampling.
- `waterflow`: flow/phase texture input, mask input, time-driven UV movement.
- `waterwaves`: wave parameters and mask input. These can appear on character
  hair/body layers and must not be globally hidden.
- `watercaustics`: caustic/generic image material behavior, translucent state,
  and FBO/texture routing.
- `foliagesway`, `auto_sway`, `shake`: vertex/transform motion effects, with
  graph pass participation preserved.
- `scroll`, `skew`, `cloudmotion`, `lightshafts`, `colorkey`: material graph
  passes with their own shader constants, texture slots, and blend state.
- Workshop effects in this scene:
  `tech_circle`, `clipping_mask`, `enhanced_simple_audio_bars`, `rounded_mask`,
  `foliagesway`, workshop `waterripple`, and `auto_sway`.

Every family should have a module that owns parameter decoding, texture slots,
combos, time uniforms, and evidence logging. A native approximation may be used
temporarily only when it preserves the pass-chain contract.

## Temporary Rectangle Guard

The current guard is deliberately narrow by effect family, not by blend mode:

- It applies to water ripple/flow/caustics chains that have no first-class
  graph executor target yet.
- It prevents the incorrect raw source carrier from being composited directly,
  including plain `normal` water-ripple chains. Plain water-ripple carriers are
  still WE material graph inputs; they are not safe raw sampled-image fallbacks.
- It must not suppress ordinary alpha/normal character layers, and it must not
  hide `waterwaves` hair/body layers.
- It must be deleted once the graph executor can run those passes.

This guard fixes the symptom of a missing graph executor. The complete fix is
the graph executor described above.

## Regression Requirements

For scene `3742497499` the evidence must prove all of the following:

- `node-108-models-json` (`id=544`, `方块`) is not recorded as a raw
  `colorBlendMode=32` sampled-image quad.
- `node-5-models-json` (`id=202`, `水纹`) and `node-57...wc-test` are not recorded
  as raw passthrough/source rectangles.
- Character image layers with alpha/normal blend and `waterwaves` remain recorded.
- The final implementation records local effect targets/passes for those nodes
  instead of simply omitting them.
- Runtime snapshots expose non-zero `draw_pass_sampled_image_we_graph_target_count`
  and per-step `input_target_index`/`output_target_index` for affected chains.
- First-class opacity/iris effect target recording steps expose
  `we_graph_chain_index`, `we_graph_step_index`,
  `we_graph_input_target_index`, and `we_graph_output_target_index` where the
  recorded pass corresponds to the planned graph step.
- Runtime graph target snapshots expose `allocation` and
  `vulkan_effect_target_index`, so allocated graph targets can be matched to the
  real Vulkan image resources.
- Runtime graph resources expose texture-source records plus graph-target
  records with source extents, layer id, chain index, endpoint, execution mode,
  allocation status, and Vulkan resource linkage when present. Planned-only water
  main/sub targets must remain visible here instead of disappearing because they
  are not allocated yet.
- Runtime graph step snapshots expose per-step `texture_bindings` with
  `g_TextureN`, source role, source path or graph target, resolution, and
  `planned_graph_resource_index`; they also expose the allocated resource index
  when the current executor has one.
- Release conversion is from the original WE directory, not hand-edited JSON.
- Release smoke uses the latest rebuilt binaries.

Latest local evidence after the 2026-07-02 `waterwaves` blend-boundary fix:

- Converted directly from
  `artifacts/wallpaper-engine-workshop/steamcmd-root/steamapps/workshop/content/431960/3742497499`
  to `/tmp/gilder-we-3742497499-alpha-blend-fix`.
- Snapshot:
  `/tmp/gilder-we-3742497499-alpha-blend-fix-runtime.json`.
- Smoke:
  `/tmp/gilder-we-3742497499-alpha-blend-fix-smoke.json`.
- User visual confirmation: the previously unchanged one-large/one-small
  rectangle pair is gone.
- The remaining rectangle pair was localized to `effects/waterwaves` visible
  image layers, not to `.gscn`/binary loss and not to the earlier
  `node-108/id=544/name=方块` water-ripple carrier.
- Large rectangle location: bottom hair/body waterwaves layers
  `node-43..48` and `node-51..56`, source resources
  `resource-73/78/81/86/91/96`, with runtime bboxes around
  `836..3333 x 291..2224` and `858..3355 x 320..2252`.
- Small rectangle location: head/hair waterwaves layers starting at
  `node-70`, including `resource-149-1-frame-0.gtex`, with runtime bboxes
  around `1341..2125 x 1309..2124`.
- Pre-fix snapshot
  `/tmp/gilder-we-3742497499-no-raw-water-ripple-runtime.json` showed
  `waterwaves` swapchain recording steps as `35 normal + 1 multiply`; the
  affected `node-43..47`, `node-51..56`, `node-70..76`, and `node-79` steps
  were `normal`.
- Post-fix snapshot shows the same sources and geometry still present, but
  `waterwaves` swapchain recording steps are `35 alpha + 1 multiply`; the
  affected large/small rectangle layers are now `alpha`. `node-48` remains
  `multiply` because that is its authored/effect chain state, not the artifact.
- Mathematical cause:
  `waterwaves.frag` computes a UV offset and returns
  `texSample2D(g_Texture0, texCoord)`, so source alpha is preserved. Correct
  scene composite is `Cout = As * Cs + (1 - As) * Cd`; the old promoted
  `normal` state was `Cout = Cs`. For transparent source pixels `As=0`,
  alpha composite leaves `Cd`, while normal overwrite writes the source RGB
  rectangle. This exactly predicts the observed hard rectangles and their
  disappearance after the blend fix.
- CWE reference:
  `references/cwe/src/WallpaperEngine/Render/Objects/CImage.cpp:750-775`
  appends the color-blend passthrough and moves the first blend mode to the
  last pass for multi-pass images; `references/cwe/src/WallpaperEngine/Render/Objects/Effects/CPass.cpp:130-140`
  defines `Translucent` as source-alpha blending and `Normal` as `ONE,ZERO`.
- Release smoke presented `305` frames in `6014 ms`, average
  `50.70766027941938 FPS`, with `68` GPU draw calls (`60` sampled-image,
  `8` solid), `19` pipeline binds, `3839` sampled-image recording steps,
  `248` WE graph chains, `547` WE graph steps, `275` WE graph targets,
  `343` WE graph resources, and `99` visible effect passes including
  `76` `water-waves` passes.
- Release tests/build used the rebuilt release binaries:
  `keeps_scene_alpha`,
  `draw_pass_plan_keeps_alpha_waterwaves_character_quad`,
  `plain_unimplemented_water_ripple`, and
  `cargo build --release --features native-vulkan-video,native-vulkan-vulkanalia --bin gilder-native-vulkan --bin gilder-convert`.

Latest local evidence after the graph pass-field/binary preservation change:

- Converted directly from
  `/tmp/gilder-we-download-3742497499/steamapps/workshop/content/431960/3742497499`
  to `/tmp/gilder-we-3742497499-resource-model-v14`.
- Snapshot:
  `/tmp/gilder-we-3742497499-resource-model-v14-runtime.json`.
- `draw_pass_sampled_image_we_graph_chain_count = 248`.
- `draw_pass_sampled_image_we_graph_step_count = 547`.
- `draw_pass_sampled_image_we_graph_target_count = 275`.
- `draw_pass_sampled_image_we_graph_resource_count = 343`, split into
  `68` texture-source resources and `275` graph-target resources.
- Global visible effect inventory: `draw_pass_effect_pass_count = 99`, including
  `draw_pass_effect_pass_non_image_layer_count = 5`. The current release
  snapshot reports first-class effect family counts for `water-waves`,
  `water-ripple`, `water-flow`, `opacity-mask`, `iris`, `foliage-sway`,
  `auto-sway`, `scroll`, `color-key`, `clipping-mask`, and `rounded-mask`.
- Per-draw-op effect snapshots are present for `51` draw ops, including `5`
  non-image layers. Examples: `node-28-text` has `scroll`,
  `node-29-text` has `color-key + scroll`, `node-30-text` has
  `scroll + clipping-mask`, and two solid rectangle layers have `rounded-mask`.
- All `547` graph steps expose at least one typed `texture_bindings` entry.
- Allocated graph targets: `2`.
- Planned-only graph targets: `273`.
- `draw_pass_sampled_image_we_graph_final_scene_step_count = 248`.
- `suppressed = 4`, `temporary_raw = 242`, `first_class = 2`.
- Target rectangle carriers
  `node-5-models-json`, `node-6-models-workshop-2790231929-wc-test-json`,
  `node-57-models-workshop-2790231929-wc-test-json`, and
  `node-108-models-json` have no raw sampled-image recording steps; their
  planned steps route through image-local targets and final
  `color-blend-passthrough`.
- Executable first-class effect targets now link to graph targets:
  `node-77-models-json` maps graph chain `33`, graph target `56` to Vulkan
  effect target resource `0`; and `node-89-models-json` maps graph chain `44`,
  graph target `70` to Vulkan effect target resource `1`. The final `node-77`
  collapsed draw step remains unlinked because the planned graph still has an
  intermediate target before its final water ripple pass.
- The graph resource table reports all graph texture sources plus all planned
  graph targets. Allocated opacity/iris targets carry both
  `planned_graph_resource_index` and `vulkan_effect_target_index`; water
  carrier image-local main/sub targets are still present as
  `planned-until-graph-executor` resources.
- `node-108-models-json` now shows the exact unresolved WE chain: base
  `g_Texture0 -> source-texture`, water ripple `g_Texture0 ->
  image-local-main` plus `g_Texture2 -> waterripplenormal`, and final
  `color-blend-passthrough g_Texture0 -> image-local-sub`.
- The same `node-108-models-json` chain now proves blend is pass-local data:
  base and ripple steps write local targets with `render_state.blend.mode =
  normal`, while the final scene `color-blend-passthrough` owns
  `render_state.blend.mode = modulate` with `src_color = dst-color` and
  `dst_color = one`.
- Smoke:
  `/tmp/gilder-we-3742497499-resource-model-v14-smoke.json` presented `150`
  frames at about `49.96 FPS` via the release binary on `HDMI-A-1`;
  Vulkan effect target resource labels include graph chain/target endpoint ids.
- Performance diagnosis from that smoke: the scene is not issuing `3843` GPU
  draw calls. Runtime planning has `3843` sampled-image recording steps, but the
  Vulkan command stream is coalesced to `71` draws (`63` sampled-image and `8`
  solid), `26` pipeline binds, and `63` descriptor-heap resource binds. The
  unnecessary cost was that any allocated effect target disabled recorded
  command-buffer reuse, so the two pre-existing opacity/iris targets forced a
  full per-frame command-buffer reset/re-record even though the current
  sampled-image shader path does not consume elapsed-time push constants for
  those targets. Command reuse is now keyed by material
  `uses_elapsed_push_constants` instead of `effect_target_count > 0`: current
  static opacity/iris target passes can reuse recorded swapchain-image command
  buffers, while a future real water-ripple/material shader can opt out until
  its time uniform moves to retained dynamic data.
- Release tests proving this boundary:
  `draw_pass_plan_preserves_we_effect_bind_overrides_as_graph_bindings`,
  `draw_pass_plan_resolves_named_fbo_bindings_to_graph_targets`,
  `draw_pass_plan_counts_effect_passes_across_image_and_non_image_layers`,
  `effect_family_modules_classify_core_native_effects`,
  `scene_runtime_snapshot_reports_we_graph_resources_as_first_class`,
  `scene_runtime_snapshot_reports_native_draw_ready_layers`,
  `preserves_wallpaper_engine_effect_pass_graph_fields_from_effect_file`,
  `gscn_direct_ingest_preserves_effect_graph_pass_fields_from_binary_payload`,
  release `draw_pass_plan`, and release `binary`.
