# Video Codec Validation

When both `gtk-renderer` and `video-renderer` are enabled, Gilder tries to use
GStreamer `playbin` with `gtk4paintablesink` so the video sink exposes a
`GdkPaintable` that GTK can render inside the per-output layer-shell background
window. If that runtime plugin is missing or fails to initialize, the GTK
renderer keeps the poster/fallback image visible and reports the error.

The codec smoke path still uses `playbin` with a headless `fakesink` so it can
run in CI without a Wayland session. This proves package loading, decode
availability, pipeline lifecycle basics, and converter preview generation, but
it does not yet prove real compositor presentation performance.

## Smoke Script

Run the codec smoke:

```sh
scripts/video-codec-smoke.sh
```

Standalone CI jobs on supported Linux distributions can let the smoke script
install missing runtime dependencies:

```sh
scripts/video-codec-smoke.sh --install-missing --work-dir /tmp
```

Use a stable report directory when the CI job should upload artifacts:

```sh
scripts/video-codec-smoke.sh --install-missing --report-dir /tmp/gilder-video-codec-smoke
```

The script generates tiny synthetic samples and validates:

- MP4/H.264 can be generated with `ffmpeg` and decoded by GStreamer.
- WebM/VP9 can be generated with `ffmpeg` and decoded by GStreamer.
- WebM/AV1 can be generated with `ffmpeg` and decoded by GStreamer.
- `gilder-convert wallpaper-engine` can convert each generated video and create
  first-frame `poster.jpg` and `thumbnail.jpg` previews.

Useful options:

```sh
scripts/video-codec-smoke.sh --work-dir /tmp
scripts/video-codec-smoke.sh --report-dir /tmp/gilder-video-codec-smoke
scripts/video-codec-smoke.sh --preflight --report-dir /tmp/gilder-video-codec-preflight
scripts/video-codec-smoke.sh --install-missing --work-dir /tmp
scripts/video-codec-smoke.sh --allow-missing
scripts/video-codec-smoke.sh --no-convert
scripts/video-codec-smoke.sh --keep
```

Every run writes `metadata.txt`, `results.csv`, `gstreamer-elements.csv`, and
`summary.txt` inside the smoke work directory. `gstreamer-elements.csv` records
the required container demuxers, decoder candidates, and actual decoder
selected by `playbin` for each codec case. Actual decoder rows use
`role=actual-decoder` and `status=selected`, so strict reports can distinguish a
missing MP4/WebM demuxer, missing H.264/VP9/AV1 decoder candidates, and the
software or hardware decoder that was actually used. Use `--keep` for a
temporary work directory that should be preserved, or `--report-dir <dir>` when
CI needs a stable artifact path. The GitHub Actions workflow uploads
`/tmp/gilder-video-codec-smoke` as the `video-codec-smoke` artifact.
For example, `/tmp/gilder-video-codecs.uYc031/gstreamer-elements.csv` recorded
`avdec_h264`, `vp9dec`, and `dav1ddec` as the selected decoders for the current
local MP4/H.264, WebM/VP9, and WebM/AV1 smoke samples, which is a software
decode baseline on this host.

Use `--preflight` when validating a host before generating samples. It checks
the required tools, ffmpeg encoders, GStreamer playback/sink elements, demuxers,
and decoder candidates, then writes the same structured report files without
running GStreamer decode or `gilder-convert`.

`--install-missing` runs the matching codec dependency helper before strict
smoke checks on Ubuntu/Debian or Arch-like hosts so `ffmpeg`,
`gst-launch-1.0`, and the expected GStreamer plugin packages are available.
The current helpers are `scripts/install-video-codec-smoke-deps-ubuntu.sh` and
`scripts/install-video-codec-smoke-deps-arch.sh`. `--allow-missing` is intended
for developer machines where optional encoders or GStreamer plugins may not be
installed. CI should run the script in strict mode.

## Wayland Surface Smoke

Run inside a real niri, Hyprland, or other Wayland session:

```sh
scripts/wayland-video-surface-smoke.sh
```

The script builds Gilder with `gtk-renderer,video-renderer`, generates a tiny
video wallpaper, starts an isolated daemon with a temporary `GILDER_SOCKET`,
sets the wallpaper on one output, and writes status/log evidence under a
temporary work directory.

Useful options:

```sh
scripts/wayland-video-surface-smoke.sh --preflight --report-dir /tmp/gilder-wayland-video-preflight
scripts/wayland-video-surface-smoke.sh --output eDP-1
scripts/wayland-video-surface-smoke.sh --expect-compositor hyprland --require-video-runtime-row --visual-hold 20 --keep
scripts/wayland-video-surface-smoke.sh --expect-compositor hyprland --require-video-runtime-row --expect-decoder-class hardware --expect-zero-copy-evidence sink-dmabuf-caps --expect-gtk-frame-clock --expect-gtk-frame-clock-phase all --keep
scripts/wayland-video-surface-smoke.sh --all-outputs --visual-hold 20 --keep
scripts/wayland-video-surface-smoke.sh --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --visual-hold 20 --keep
scripts/wayland-video-surface-smoke.sh --simulate-power battery --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-output-state unfocused --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-output-state fullscreen --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --measure-fullscreen-resume --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-session locked --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --sample-paused --keep
scripts/wayland-baseline-matrix.sh --report-dir /tmp/gilder-wayland-baseline --sample-duration 30
scripts/wayland-baseline-matrix.sh --report-dir /tmp/gilder-wayland-baseline --budget-csv examples/wayland-memory-budget.example.csv
scripts/wayland-video-surface-smoke.sh --allow-missing
scripts/wayland-video-surface-smoke.sh --no-build --keep
```

Use `--preflight` first when validating a real compositor session. It checks
`WAYLAND_DISPLAY`, `XDG_RUNTIME_DIR`, required tools, built binaries, and the
GStreamer elements needed by the generated MP4/H.264 test wallpaper
(`playbin`, `gtk4paintablesink`, `qtdemux`, and an H.264 decoder candidate)
without starting the daemon or changing the current wallpaper. With
`--report-dir`, it writes stable `metadata.txt`, `checks.csv`,
`validation-report.txt`, and `summary.txt` evidence that can be attached before
a visual run. Missing GStreamer element rows include package hints for common
runtime gaps such as `gtk4paintablesink` (`gst-plugin-gtk4` on Arch-like
systems).

For a full Wayland video resource baseline, run
`scripts/wayland-baseline-matrix.sh --report-dir /tmp/gilder-wayland-baseline`.
It builds once, runs `wayland-video-surface-smoke.sh` with
`--sample-performance` across active, user-paused, battery, unfocused,
fullscreen, hidden, session-inactive, and session-locked states, and writes a
top-level `baseline.csv`. The CSV contains CPU/GPU, RSS/PSS/private/USS/shared
memory, retained and peak-over-first deltas, planned image-resource footprint,
package-cache retained source-resource footprint and configured byte budget,
runtime static image cache bytes and configured byte budget,
renderer output/static/slideshow/video surface counts, video pipeline counts,
decoder status, caps memory features, zero-copy evidence level, QoS, GTK frame
clock, and GDK timing fields. Each scenario keeps the original smoke evidence
under `scenarios/<name>/`, so budget regressions can be traced back to raw
`samples.csv`, `telemetry.csv`, `video-runtime.csv`, status snapshots, and
daemon logs. Add `--scenario fullscreen-resume` when the same run should also
measure fullscreen -> active resume after a file-backed output-state switch.
Pass `--budget-csv <path>` to make the matrix enforce per-scenario budgets.
The budget file is a simple CSV with `scenario,phase,metric,max`; `scenario`
and `phase` may be `*`, and `metric` must match a `baseline.csv` column:

```csv
scenario,phase,metric,max
active,active,max_uss_kib,250000
active,user-paused,retained_private_delta_kib,20480
*,*,max_pss_kib,320000
*,*,render_sync_package_cache_retained_unique_resource_bytes_latest,104857600
*,*,render_sync_package_cache_max_retained_unique_resource_bytes_latest,536870912
*,*,render_sync_static_image_cache_bytes_latest,104857600
fullscreen,fullscreen,renderer_video_pipelines_latest,0
```

Budget checks write `budget-results.csv` and fail the matrix when any matching
numeric baseline value is missing or above its limit. Prefer PSS, USS/private,
and retained delta limits for memory regression gates; keep RSS/shared limits
as supplemental audit signals because they include shared library mappings. The
repository includes `examples/wayland-memory-budget.example.csv` as a
conservative starting point for one-output active video and lifecycle
scenarios. Treat it as a guardrail template: update the values from your own
`baseline.csv` once a machine-specific baseline has been accepted.

The smoke is intentionally partly visual: after the script reports success,
confirm that the selected output shows the generated moving test video. Pass
`--expect-compositor hyprland|niri|generic-wayland|none` during full smoke runs
to make the captured `desktop.compositor` value a hard evidence gate; this is
recommended for the pending Hyprland validation so a generic fallback or niri
session cannot accidentally be filed as Hyprland evidence. Pass `--all-outputs`
to apply the same generated video wallpaper to every
daemon-reported output and assert that each target output has an active
`render_sync.video_plans` entry. It also checks that `gtk4paintablesink` is
available. Use `--visual-hold <sec>` to keep the applied wallpaper visible for a
fixed confirmation window before sampling or cleanup. With
`--sample-performance`, it also runs `performance-snapshot.sh` against the
isolated daemon and writes
`performance-active/samples.csv`, `summary.txt`, and status snapshots under the
same kept work directory. The smoke also accepts video runtime evidence gates
from `performance-snapshot.sh`: `--expect-decoder-policy-status`,
`--expect-decoder-class`, `--expect-memory-feature`,
`--expect-sink-memory-feature`, `--expect-zero-copy-evidence`,
`--expect-zero-copy-evidence-at-least`, `--expect-video-position-progress`,
`--expect-gtk-frame-clock`,
`--expect-gtk-frame-clock-phase before-paint|update|layout|paint|after-paint|all`,
and `--expect-gtk-frame-timings`. It also forwards process memory budget gates:
`--expect-max-rss-kib-at-most`, `--expect-max-pss-kib-at-most`,
`--expect-max-private-kib-at-most`, `--expect-max-uss-kib-at-most`, and
`--expect-max-shared-kib-at-most`, plus planned image-resource byte gates
`--expect-render-sync-planned-image-resource-reference-bytes-latest-at-most`
and `--expect-render-sync-planned-unique-image-resource-bytes-latest-at-most`,
and the runtime static-image cache byte gate
`--expect-render-sync-static-image-cache-bytes-latest-at-most`.
Supplying any of these options
automatically enables performance sampling. Video runtime checks apply only to
scenarios that should have an active video plan; process memory checks apply to
the sampled daemon in every performance scenario. The phase gate checks GTK
frame-clock phase counters from the video runtime summary; it is useful for
proving the GTK surface entered the expected frame-cycle stages, but it is
still not direct Wayland compositor presentation feedback. Pass `--require-video-runtime-row` during
real compositor validation to fail active phases where the render plan exists
but `renderer_runtime.video_pipelines` produced no CSV rows. This proves the
daemon exposed a live video pipeline snapshot for that phase; it does not prove
hardware decode or zero-copy on its own. With `--sample-paused`, it captures the
active sample, pauses the selected output, verifies a `user-paused` performance
decision, captures `performance-paused/`, and resumes the output.
Every run also writes `validation-report.txt` as a single audit entrypoint. It
summarizes the expected and actual compositor, selected outputs, whether the
scenario should contain an active video plan, video runtime row counts and phase
counts, the relevant performance/video-runtime summary artifact paths, and the
prefixed active/paused video runtime evidence for decoder policy/status,
actual decoders, decoder class, negotiated memory features, sink-side memory
features, zero-copy evidence level, GTK frame-clock phase counters, and GDK
frame timing counters. Use that file first when reviewing Hyprland, niri,
hardware decode, or zero-copy evidence, then drill into the referenced CSV and
status JSON files. Hardware decoder evidence alone is not treated as zero-copy
proof; look for DMABuf/GLMemory caps, especially sink-side caps, and compositor
presentation evidence for stronger validation.
With `--simulate-power battery`, it starts the isolated daemon with
`GILDER_POWER_STATE=battery`, verifies that status reports `power: battery`,
checks for a battery performance decision after applying the wallpaper, and
stores the primary performance sample under `performance-battery/`.
With `--simulate-output-state unfocused|fullscreen|hidden`, it starts the
isolated daemon with `GILDER_OUTPUT_STATE`, verifies that status reflects the
simulated output focus/visibility/fullscreen fields, and checks for the
matching performance decision. `unfocused` still expects an active video plan
with throttling, while `fullscreen` and `hidden` expect the paused/remove path.
With `--measure-fullscreen-resume`, it starts fullscreen through a
file-backed `GILDER_OUTPUT_STATE_FILE`, applies the video wallpaper while the
fullscreen policy removes the video plan, rewrites the override to `active`,
and records the time until status reports an interactive video plan again in
`fullscreen-resume-latency.csv` and `fullscreen-resume-latency.txt`.
With `--simulate-session inactive|locked`, it starts the isolated daemon with
`GILDER_SESSION_STATE`, verifies that status reflects the simulated logind
session state, and expects the paused/remove path with `session-inactive` or
`session-locked` as the performance reason.

## Runtime Packages

On Ubuntu-like systems the strict smoke path expects:

- `ffmpeg`
- `gstreamer1.0-tools`
- `gstreamer1.0-libav`
- `gstreamer1.0-plugins-base`
- `gstreamer1.0-plugins-good`
- `gstreamer1.0-plugins-bad`
- `gstreamer1.0-plugins-ugly`

On Arch-like systems the equivalent codec smoke packages are typically:

- `ffmpeg`
- `gstreamer`
- `gst-libav`
- `gst-plugin-dav1d`
- `gst-plugins-base`
- `gst-plugins-good`
- `gst-plugins-bad`
- `gst-plugins-ugly`

For Arch-like codec smoke, run:

```sh
scripts/install-video-codec-smoke-deps-arch.sh
```

The interactive GTK Wayland video surface path also needs:

- `gst-plugin-gtk4`
- `gtk4`
- `gtk4-layer-shell`
- `pkgconf`
- `wayland-protocols`

For Arch-like Wayland surface smoke, run:

```sh
scripts/install-wayland-video-smoke-deps-arch.sh
```

Then re-run:

```sh
scripts/wayland-video-surface-smoke.sh --preflight --report-dir /tmp/gilder-wayland-video-preflight
```

The GitHub Actions workflow installs the full CI dependency set through
`scripts/install-ci-deps-ubuntu.sh`; codec-only CI jobs can pass
`--install-missing` or run the distro-specific codec dependency helper first.
Do not run strict codec smoke on a fresh Ubuntu CI image without one of these
installers, because `gst-launch-1.0` is provided by `gstreamer1.0-tools`. As a
guardrail, `scripts/video-codec-smoke.sh` will auto-run the matching codec
dependency installer when strict mode is used on a GitHub Actions or generic
`CI=true` Ubuntu/Debian or Arch-like runner and `ffmpeg`, `gst-launch-1.0`, or
`gst-inspect-1.0` is missing. Local runs still fail explicitly unless
`--install-missing` or `--allow-missing` is passed.
If CI fails with `FAIL: gst-launch-1.0 is not available`, the dependency install
step did not run or did not complete before the codec smoke command. Use
`scripts/video-codec-smoke.sh --install-missing --work-dir /tmp` for strict
codec smoke jobs, and use `--allow-missing` only for optional smoke jobs where
missing codecs should be recorded as skips instead of failures.
If `gst-launch-1.0` exists but decode still fails, inspect
`gstreamer-elements.csv`. Missing `qtdemux` points to MP4/QuickTime demuxer
packages, missing `matroskademux` points to WebM/Matroska demuxer packages, and
missing decoder candidates such as `avdec_h264`, `avdec_vp9`, `dav1ddec`,
or `avdec_av1` point to codec plugin packages. Arch-like hosts may also expose
`av1dec` from the AOM plugin, but that decoder can fail the generated WebM/AV1
sample caps; use `gst-plugin-dav1d` or another plugin that provides `dav1ddec`
or `avdec_av1` for this smoke.

The GTK video surface path also needs a runtime plugin that provides
`gtk4paintablesink` such as `gst-plugin-gtk4`. Package names differ by
distribution, so Gilder probes it at runtime instead of making it a Rust build
dependency.
Use `gilderctl status` and check
`renderer_capabilities.video.gstreamer.elements` to confirm whether
`gtk4paintablesink` and the core GStreamer elements are available in the running
daemon environment.
When a video wallpaper is running, also inspect
`renderer_runtime.video_pipelines[].actual_decoders` from `gilderctl status` or
`gilderctl watch --snapshot`. This reports the decoder element selected by the
live daemon pipeline, while `renderer_capabilities` only reports which runtime
plugins are available. Empty `actual_decoders` means no known decoder element
has been observed yet, not that hardware decode is active.
`renderer_runtime.video_pipelines[].decoder_policy` reports the configured
intent from `[video].decoder`, and
`renderer_runtime.video_pipelines[].actual_decoder_reports[].class` classifies
observed decoder elements as `hardware`, `software`, or `unknown`.
`renderer_runtime.video_pipelines[].decoder_policy_status` reports whether the
observed decoder satisfies that policy: `not-applicable` for `auto`,
`not-observed` before a known decoder appears, `satisfied` when the selected
class matches the policy, `software-fallback` when `hardware-preferred` fell
back to software, `violated` when a required class was contradicted, and
`unknown-decoder` when GStreamer selected a decoder outside Gilder's current
diagnostic classification.
`renderer_runtime.video_pipelines[].caps_reports` records negotiated
`current_caps()` from live video pads. Each report includes the element, pad,
direction, caps string, media type list, caps features, and aggregated
`memory_features` such as `memory:DMABuf` or `memory:GLMemory` when GStreamer
exposes them on the negotiated path. Empty caps reports usually mean the
pipeline has not negotiated video caps yet.
`renderer_runtime.video_pipelines[]` also reports `position_ms`, `duration_ms`,
`frame_limiter_enabled`, `frame_limiter_max_fps`, and `frame_stats`.
`frame_stats` accumulates GStreamer QoS messages observed during bus polling,
including max processed/dropped values, stats format, jitter, and the latest
proportion scaled by 1000. On the GTK video surface path it also records
`gtk_frame_clock_*` values from passive `gtk::Picture` frame clock after-paint
observations: count, latest frame counter/time, frame interval, frame clock
FPS, refresh interval, and GDK's predicted presentation time. The same runtime
snapshot records frame clock phase counters for `before-paint`, `update`,
`layout`, `paint`, and `after-paint`, which helps diagnose whether GTK is
driving the surface through a full frame cycle or only exposing partial timing
evidence. It also records GDK `FrameTimings` observed/complete counts, frame
time, predicted presentation time, presentation time, presentation interval,
and refresh interval when GDK has timing history for the frame. These fields are runtime
playback/limiter/QoS/GTK frame-clock evidence: a moving `position_ms` proves the
pipeline playhead is advancing, `frame_limiter_max_fps` proves the applied sink
`throttle-time` limit, `qos_dropped_max` records GStreamer sink QoS drops when the
sink reports them, and `gtk_frame_clock_ticks` proves the GTK surface is being
driven by a frame clock. Completed GDK frame timings are stronger presentation
clues than after-paint ticks, but they are still not direct Wayland
`wp_presentation` protocol feedback or native compositor frame callback counts.
Use `gilderctl status --video-runtime-csv --from-file <status.json>` to turn a
saved status snapshot into compact decoder/caps/playback evidence with
sink-side memory features and `zero_copy_evidence_level`. The raw status JSON
remains the authoritative source for full caps strings.

The exact hardware decode path is left to the host GStreamer installation. The
smoke test intentionally uses `fakesink` so it can run in CI without a Wayland
session. Current Wayland surface smoke evidence should be treated as a
software-decoding or auto-selected-decoder baseline unless paired with codec
smoke evidence that records a hardware decoder element such as VAAPI/VDPAU/NVDEC.
The generated H.264 surface smoke does not force hardware decode.
The explicit decoder policy values are `auto`, `hardware-preferred`,
`hardware-required`, and `software`. Gilder currently applies these by adjusting
the GStreamer feature rank for its known H.264/VP9/AV1 hardware and software
decoder sets before constructing the pipeline. `hardware-preferred` raises known
hardware decoder rank while keeping software fallback available;
`hardware-required` raises known hardware decoders and disables known software
fallbacks; `software` disables known hardware decoders and raises known software
decoders; `auto` restores the host's original ranks.

Hardware decode is not the same thing as zero-copy presentation. A pipeline may
decode through VAAPI/VDPAU/NVDEC and still copy frames back through CPU memory
before GTK/Wayland presentation. Gilder now derives
`renderer_runtime.video_pipelines[].zero_copy_evidence.level` from the observed
decoder class and negotiated caps. The levels are ordered as `missing`,
`software-decode`, `hardware-decode`, `gpu-memory-caps`, `dmabuf-caps`,
`sink-gpu-memory-caps`, and `sink-dmabuf-caps`. They are evidence levels only:
zero-copy validation must still inspect the live `caps_reports`, especially
sink-side memory features such as DMABuf/GLMemory where available, and pair that
with CPU, GPU, PSS, USS, frame behavior, and compositor presentation evidence.
Seeing a hardware decoder or a configured hardware policy alone is not zero-copy
proof.

For muted video wallpapers, Gilder disables `playbin` audio stream selection
instead of routing decoded audio to `fakesink`, so muted wallpaper playback does
not spend CPU or memory decoding an unused audio stream. `runtime.allow_audio`
and entry-level mute settings still allow audio when a package explicitly asks
for it. The renderer keeps `playbin` flags minimal (`video` for muted playback,
`video+audio` for audible playback) so deinterlace, soft color balance, and
software volume elements are not kept in the wallpaper pipeline unless a later
renderer path explicitly needs them. FPS limiting is applied on the sink via
`throttle-time` instead of a `video-filter`, keeping `videorate` and
`capsfilter` out of the negotiated video path so GPU-memory caps have fewer
software-only elements to cross.

## Remaining Surface Work

The GTK paintable sink code path has been validated for one-output and
multi-output niri video presentation, plus niri simulated battery, unfocused,
fullscreen policy sampling, and file-backed fullscreen -> active resume
measurement. A recent niri validation run wrote evidence under
`/tmp/gilder-wayland-video.lLD2VR`: `fullscreen-resume-latency.txt` reported
`latency_ms: 642`, and `performance-resumed/summary.txt` reported resumed
video averages of `avg_cpu_percent: 7.77`, `avg_pss_kib: 204560`, and
`avg_uss_kib: 178865` over a 3-sample window. It still needs
compositor-facing checks:

- Hyprland video presentation;
- real compositor fullscreen resume latency;
- longer-duration CPU, memory, and GPU usage sampling while active and paused.

## Performance Sampling

For repeatable active/paused/fullscreen/battery comparisons, collect daemon
resource and status evidence while the scenario is running:

```sh
scripts/performance-snapshot.sh --label active-video --duration 30 --interval 1 --keep
scripts/performance-snapshot.sh --label paused-video --duration 30 --interval 1 --keep
scripts/desktop-policy-smoke.sh --keep
scripts/desktop-policy-smoke.sh --report-dir /tmp/gilder-desktop-policy-smoke
scripts/wayland-video-surface-smoke.sh --sample-performance --sample-duration 30 --keep
scripts/wayland-video-surface-smoke.sh --sample-paused --sample-duration 30 --keep
scripts/wayland-baseline-matrix.sh --report-dir /tmp/gilder-wayland-baseline --sample-duration 30
scripts/wayland-baseline-matrix.sh --report-dir /tmp/gilder-wayland-baseline --budget-csv ./wayland-budget.csv
```

The script finds a running `gilderd` process, samples `ps` CPU/RSS/VSZ fields,
and, on Linux, reads `/proc/<pid>/smaps_rollup` for PSS, USS/private, and
shared memory. RSS is the resident set including shared mappings; PSS is the
shared memory cost apportioned across processes; USS is the unique/private set
size, reported here as `Private_Clean + Private_Dirty`. It computes a small
`summary.txt` with min/average/max memory values and writes one `gilderctl`
status JSON snapshot per sample. It also asks
`gilderctl status --decisions-csv --from-file` to
produce `decisions.csv` and `decision-summary.txt`, so active/paused,
fullscreen, and battery scenarios can be compared by both resource usage and
the daemon's actual `mode/reason/max_fps` decision. The summary is generated
with a CSV-aware parser and includes decision row counts, unique samples and
outputs, `mode/reason` counts with FPS ranges, `max_fps` counts, action counts,
plan kinds, fit modes, muted video counts, and target FPS ranges. It also asks
`gilderctl status --telemetry-csv --from-file` to produce `telemetry.csv` and
`telemetry-summary.txt`, which report desktop refresh deltas, read-request
refresh skips, desktop change deltas, render-sync cache hit/miss deltas, and
renderer update queue queued/skipped counters. The telemetry summary also
reports latest package/archive cache entries, package/archive cache max entries,
package cache evictions, archive reuse/extraction counts, and archive
eviction/error deltas so long runs can catch manifest/package cache pressure or
`.gwp` unpack cache growth. It also reports static-image runtime cache entries,
generation/reuse counts, generation errors, and evictions so static oversized
source downscaling can be checked alongside memory samples. It reports planned
video source references, unique sources, duplicate source references, maximum
same-source fanout, and source bytes so multi-output duplicate decode candidates
can be identified before implementing pipeline sharing. It also reports renderer
video pipeline source references, unique sources, and source bytes so active,
paused, hidden, and resumed samples can prove whether the runtime pipeline is
still holding a video source. These source-byte metrics are not decoded frame
memory or USS. It also reports planned static image, video poster,
slideshow image, total image-reference, and unique image-resource counts so
paused/fullscreen/hidden evidence can prove the render plan stopped retaining
image resources before checking GTK/private memory. The same summary reports
planned static image, video poster, slideshow image, image-reference, and unique
image-resource source-file bytes. It also reports retained package-cache
manifest resource references, unique resources, reference bytes, and unique
bytes for packages still held by the per-render-sync package cache. These bytes
are source-file or source-directory footprint clues for large images/posters and
cached package manifests; they are not decoded texture memory or process USS.
Renderer telemetry summary fields
include latest/max output window, static surface, slideshow surface, video
surface, and video pipeline counts, which are used to prove that paused,
hidden, fullscreen, inactive, and locked states actually release GTK renderer
resources. They also include renderer static-surface and slideshow source
resource reference, unique-resource, and byte footprints, which show whether GTK
CSS providers or slideshow surfaces still point at large source images and
whether repeated references are to the same file or different files; these are
source-file clues, not decoded texture memory or process USS. When adaptive monitoring is
enabled, the same telemetry files also include adaptive refresh deltas, active
trigger counts, PSI CPU/memory pressure maxima, and thermal-zone maximum
temperature, power_supply AC/battery details, and daemon-side DRM
`gpu_busy_percent` samples when the driver exposes them. Adaptive action
columns report the observed action types, scopes, configured fallback actions,
and max FPS values so smoke tests can distinguish throttle, `pause-unfocused`,
and `pause-dynamic`. Renderer telemetry columns aggregate video pipeline frame
behavior from the daemon snapshot, including total QoS messages, max QoS dropped
count, total GTK frame clock ticks and phase tick counts, max GTK frame
interval, and max observed GTK frame-clock FPS, plus completed GDK frame timing
counts and presentation timing maxima.
Use `--expect-renderer-video-pipeline-lifecycle` in Wayland smoke runs when the
sampled scenario should prove lifecycle behavior: active and fullscreen-resumed
performance windows must report at least one renderer output window, video
surface, and video pipeline, while their latest static/slideshow surface counts
must be zero after the video surface has taken over. Paused, hidden,
fullscreen, inactive, and locked windows must end with zero output windows, zero
static/slideshow/video surfaces, zero renderer video pipelines, and zero
planned image resource references. The same gate also bounds runtime video
pipeline source footprint: renderable windows are allowed at most one video
source reference per selected output and one unique source for the generated
smoke wallpaper, while paused/hidden/fullscreen/inactive/locked windows must
end with zero video source references and zero source bytes. Renderable video
windows are allowed at most one planned poster reference per selected output
and one unique planned poster resource for the generated smoke wallpaper; that
planned poster is an error fallback and should not imply retained GTK
static-surface memory during active playback. This gate uses daemon telemetry
and complements
`--require-video-runtime-row`, which only proves that an active phase exposed a
live per-output runtime row. Use
`--expect-render-sync-planned-image-resource-references-latest-at-most <count>`
and
`--expect-render-sync-planned-unique-image-resources-latest-at-most <count>` to
set stricter planned-resource budgets for targeted runs. Use
`--expect-render-sync-planned-image-resource-reference-bytes-latest-at-most <bytes>`
and
`--expect-render-sync-planned-unique-image-resource-bytes-latest-at-most <bytes>`
when the budget should account for large source images or posters; the script combines
explicit user limits with its per-scenario lifecycle limits by taking the
stricter value. Use
`--expect-render-sync-package-cache-retained-resource-references-latest-at-most <count>`,
`--expect-render-sync-package-cache-retained-unique-resources-latest-at-most <count>`,
`--expect-render-sync-package-cache-retained-resource-bytes-latest-at-most <bytes>`,
and
`--expect-render-sync-package-cache-retained-unique-resource-bytes-latest-at-most <bytes>`
to set upper bounds for resources still referenced by retained package-cache
entries. Use
`--expect-render-sync-static-image-cache-bytes-latest-at-most <bytes>` to cap
the runtime downscaled static-image cache footprint reported in daemon
telemetry.
Use
`--expect-renderer-video-pipeline-source-references-latest-at-most <count>`,
`--expect-renderer-video-pipeline-source-reference-bytes-latest-at-most <bytes>`,
`--expect-renderer-video-pipeline-unique-sources-latest-at-most <count>`, and
`--expect-renderer-video-pipeline-unique-source-bytes-latest-at-most <bytes>`
to set runtime video-source footprint gates. These are useful for active,
paused, hidden, fullscreen, and resumed samples where the process memory
budget should be checked alongside proof that renderer video pipelines no
longer hold source references. When the lifecycle gate is enabled, explicit
runtime video-source limits are combined with the per-scenario lifecycle limits
by taking the stricter value.
The sampler also writes `video-runtime.csv`, which records each sample's
decoder policy status, actual decoder classes, caps report count, all memory
features, sink-side memory features, zero-copy evidence level, playback
position/duration, actual frame limiter state, and GTK frame clock phase
counters. It also writes
`video-runtime-summary.txt`, including `video_zero_copy_evidence_latest`,
`video_zero_copy_evidence.<level>` counts, `video_position_moving_outputs`,
`video_position_delta_ms_max`, `video_frame_limiter_enabled_rows`, limiter FPS
min/max,
`video_qos_messages_max`, `video_qos_dropped_max`,
`video_gtk_frame_clock_ticks_max`, GTK frame clock phase maxima, GTK frame
clock interval/FPS summaries, `video_gtk_frame_timings_complete_max`, and GDK
frame timing presentation interval/time summaries.
Use that table beside CPU, PSS, USS, and RSS when checking hard decode or
zero-copy behavior.
Use `--expect-decoder-policy-status`, `--expect-decoder-class`,
`--expect-memory-feature`, `--expect-sink-memory-feature`,
`--expect-zero-copy-evidence`, and `--expect-zero-copy-evidence-at-least` to
make the sampling run fail when live video runtime evidence does not contain
the expected decoder policy result, hardware/software class, negotiated caps
memory feature, sink-side memory feature, or zero-copy evidence level. For exact
evidence matching use `--expect-zero-copy-evidence`; for minimum acceptable
evidence use `--expect-zero-copy-evidence-at-least`, ordered as `missing`,
`software-decode`, `hardware-decode`, `gpu-memory-caps`, `dmabuf-caps`,
`sink-gpu-memory-caps`, then `sink-dmabuf-caps`. For example,
`--expect-decoder-class hardware` checks that the running pipeline observed a
known hardware decoder, `--expect-sink-memory-feature memory:DMABuf` checks for
sink-side DMABuf caps, and
`--expect-zero-copy-evidence-at-least sink-gpu-memory-caps` accepts either
sink-side GLMemory or the stronger sink-side DMABuf evidence level.
Use `--expect-video-position-progress`, `--expect-frame-limiter-enabled`, and
`--expect-frame-limiter-max-fps <fps>` to assert that playback moved during the
sample window and that the runtime frame limiter is active at the expected cap.
Position progress is measured as the largest minus smallest observed position
per output, so a looping video that wraps near the end of the sample window is
still counted as moving.
Use `--expect-video-qos` to require at least one QoS message and
`--expect-qos-dropped-max-at-most <count>` to fail runs where the observed QoS
dropped counter exceeds the selected threshold.
Use `--expect-gtk-frame-clock` to require GTK frame clock ticks from a real GTK
video surface sample. Use
`--expect-gtk-frame-clock-phase before-paint|update|layout|paint|after-paint|all`
to require specific frame clock phase ticks from the same sample. Use
`--expect-gtk-frame-timings` to require completed GDK frame timings from the GTK
surface path when the backend exposes them.
These checks are evidence gates only: hardware decoder evidence and
DMABuf/GLMemory caps should still be interpreted separately from compositor
presentation feedback and full zero-copy proof.
`performance-snapshot.sh` also writes `video-hardware-report.txt` next to the
process and video-runtime summaries. That report combines the same decoder,
caps, sink caps, zero-copy, CPU/GPU/PSS/USS/private fields with `ffprobe`
codec metadata for each sampled video source and DRM/NVIDIA GPU driver details
from sysfs or `nvidia-smi` when available. Wayland smoke reports link the
active and paused hardware report paths as
`performance_active_video_hardware_report` and
`performance_paused_video_hardware_report`, so codec/GPU/driver comparisons can
be attached without manually correlating separate files.
When available, `samples.csv` also includes `gpu_busy_percent_avg`,
`gpu_busy_percent_max`, and `gpu_busy_sources` from DRM sysfs
`gpu_busy_percent` or `nvidia-smi`. These fields are optional and may be empty
on drivers that do not expose a simple busy counter.
`telemetry-summary.txt` separately reports `daemon_gpu_busy_samples`,
`daemon_avg_gpu_busy_percent`, `daemon_max_gpu_busy_percent`, and
`daemon_gpu_busy_sources_latest` when adaptive monitoring captured GPU busy from
inside the daemon. It also reports `renderer_video_qos_messages_max`,
`renderer_video_qos_dropped_max`, `renderer_video_gtk_frame_clock_ticks_max`,
`renderer_video_gtk_frame_clock_*_ticks_max` phase counters,
`renderer_video_gtk_frame_clock_interval_us_max`, and
`renderer_video_gtk_frame_clock_fps_x1000_max`,
`renderer_video_gtk_frame_timings_complete_max`, and
`renderer_video_gtk_frame_timings_presentation_interval_us_max` from daemon
telemetry, which is a coarse health signal for video frame behavior before
drilling into `video-runtime-summary.txt`.
For memory comparisons, prefer `avg_uss_kib` or its equivalent
`avg_private_kib` for the process-private footprint and `avg_pss_kib` for the
shared-memory-adjusted footprint; `avg_rss_kib` includes shared mappings at
full size and is not private usage. Use
`--expect-max-uss-kib-at-most <kib>`,
`--expect-max-private-kib-at-most <kib>`, and
`--expect-max-pss-kib-at-most <kib>` to turn those budgets into hard sampling
gates. `--expect-max-rss-kib-at-most <kib>` and
`--expect-max-shared-kib-at-most <kib>` are also available for broader
auditing, but they should not be used as the only private-footprint signal.
The PSS/USS/private/shared gates require readable Linux
`/proc/<pid>/smaps_rollup` data; if that data is missing, the sampler reports
the expectation as unmet instead of treating zeroes as a valid pass.
`summary.txt` also records `first_*_kib`, `last_*_kib`,
`retained_*_delta_kib`, and `peak_over_first_*_kib` for RSS, PSS, private,
USS, and shared memory. Retained delta is the last sample minus the first
sample and is the quickest way to spot memory that remains after a paused,
hidden, fullscreen, or fullscreen-resumed sampling window. Peak-over-first is
kept separate so transient decode/GTK allocation spikes are not confused with
end-of-window private retention. Use
`--expect-retained-private-delta-kib-at-most <kib>`,
`--expect-retained-uss-delta-kib-at-most <kib>`,
`--expect-retained-pss-delta-kib-at-most <kib>`,
`--expect-peak-over-first-private-kib-at-most <kib>`,
`--expect-peak-over-first-uss-kib-at-most <kib>`, and
`--expect-peak-over-first-pss-kib-at-most <kib>` to turn these relative
private-footprint budgets into gates in desktop policy and Wayland smoke runs.
`desktop-policy-smoke.sh` forwards these fields into `resource-baseline.csv`,
and `wayland-video-surface-smoke.sh` includes the process memory and renderer
telemetry summaries in `validation-report.txt` for active, paused, and
fullscreen-resumed performance directories.

Current local release measurements for the generated 720p/30fps H.264 video
wallpaper are hardware- and driver-specific, but they define the latest
optimization baseline for this repository:

| Path | RSS max | PSS max | USS/private max | CPU avg | Zero-copy evidence |
| --- | ---: | ---: | ---: | ---: | --- |
| Idle daemon | 4100 KiB | 2018 KiB | 2000 KiB | n/a | n/a |
| Headless video renderer | 135820 KiB | 126127 KiB | 123376 KiB | 4.11% | sink-gpu-memory-caps |
| GTK/Wayland video surface | 356892 KiB | 272574 KiB | 241660 KiB | 14.69% | hardware-decode |

The headless path is now close to the decoder/GStreamer cost floor observed on
this host. The GTK/Wayland path is still the main memory target: it confirms
hardware decoding, but the sink caps only showed `memory:SystemMemory`, so it
may still copy frames through CPU memory instead of preserving a GPU/DMABuf
path to presentation.

Pass `--pid`, `--socket`, or `--gilderctl` when testing an isolated daemon such
as the Wayland surface smoke script. The CSV, summaries, and raw status files
are intended to be compared between scenarios.
Use `--expect-mode`, `--expect-reason`, `--expect-action`, `--expect-max-fps`,
and `--expect-plan-kind` to make a sampling run fail when the expected render
decision is not observed in `decision-summary.txt`. The Wayland video smoke
passes these expectations automatically for simulated battery, unfocused,
fullscreen, hidden, session, and user-paused scenarios. Use
`--expect-render-sync-cache-hit`, `--expect-desktop-refresh-skip`, and
`--expect-render-sync-update-queued` to make a sampling run fail when daemon
telemetry does not show render-sync cache reuse, read-request desktop refresh
throttling, or at least one renderer sync dispatch; the Wayland video smoke
enables these telemetry expectations for its performance samples. Use
`--expect-render-sync-update-skipped` for targeted repeated-state scenarios
where the same `render_sync` should be suppressed instead of sent to renderers
again. Use `--expect-adaptive-action throttle|pause-unfocused|pause-dynamic`
when adaptive monitoring is enabled and the sampled action itself must be
verified in telemetry.
`scripts/desktop-policy-smoke.sh` runs the same assertion path without GTK,
GStreamer, or a Wayland session by setting `GILDER_DESKTOP_OUTPUTS` to a
virtual output and covering active, battery, unfocused, fullscreen, hidden,
inactive, locked, adaptive CPU-pressure throttle, adaptive GPU-busy throttle,
adaptive `pause-unfocused`, adaptive focused-output fallback, adaptive
`pause-dynamic` static passthrough, adaptive CPU-pressure `pause-dynamic`
slideshow removal, adaptive low-battery `pause-dynamic` slideshow removal, and
per-output performance override scenarios, including battery `pause-dynamic`
static passthrough and slideshow removal, and fullscreen/unfocused/hidden/session
`pause-dynamic` static passthrough plus slideshow removal, against the default daemon build. It
asserts mode, reason, action,
plan kind, planned image resource references/unique resources, and expected
`max_fps` where the decision should remain renderable. The GitHub Actions
workflow runs it in strict mode and uploads `/tmp/gilder-desktop-policy-smoke`
as the `desktop-policy-smoke` artifact. The artifact includes top-level
`metadata.txt`, `matrix.csv`, `resource-baseline.csv`, and `summary.txt` files,
plus per-scenario status snapshots, daemon logs, decision summaries, and
telemetry summaries. `resource-baseline.csv` gives one row per scenario and
pulls the sampled CPU, GPU, RSS, PSS, private, USS, shared-memory, decision,
render-sync cache, static-image runtime cache, video source sharing candidates, renderer video pipeline source footprint, planned image resource count/byte footprint, renderer update,
package-cache retained resource footprint, renderer static/slideshow reference
and unique-resource footprint, adaptive-action, and renderer video telemetry summary values into one
table for quick baseline comparison.
The desktop policy smoke also forwards the same
`--expect-max-*-kib-at-most` memory budget gates to every scenario, which makes
it useful for CI-side private-memory regression checks once per-scenario
budgets have been established. It also forwards render-sync resource gates,
including `--expect-render-sync-static-image-cache-bytes-latest-at-most`, to
turn static runtime cache growth into a smoke failure.
For battery policy comparisons on machines that are not actually discharging,
run the daemon or smoke script with `GILDER_POWER_STATE=battery`; unset it to
return to sysfs-based power detection.
For compositor-state policy comparisons where changing the real desktop state
is awkward, use `GILDER_OUTPUT_STATE=unfocused`, `fullscreen`, or `hidden`; unset
it to return to compositor/GDK state detection.
For same-daemon transition measurements, use `GILDER_OUTPUT_STATE_FILE` or the
Wayland smoke's `--measure-fullscreen-resume` option so the validation can
switch fullscreen back to active without restarting `gilderd`.
For session-state policy comparisons where switching VT or locking the real
session is awkward, use `GILDER_SESSION_STATE=inactive` or `locked`; unset it to
return to logind state detection.
For adaptive policy comparisons where real PSI/thermal/GPU/battery pressure is
not stable, use `GILDER_ADAPTIVE_STATE=cpu-pressure`, `memory-pressure`,
`temperature`, `gpu-busy`, `low-battery`, or `all`; use `inactive` to force a
non-triggering adaptive sample.
