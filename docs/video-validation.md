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
scripts/wayland-video-surface-smoke.sh --all-outputs --visual-hold 20 --keep
scripts/wayland-video-surface-smoke.sh --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --visual-hold 20 --keep
scripts/wayland-video-surface-smoke.sh --simulate-power battery --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-output-state unfocused --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-output-state fullscreen --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --measure-fullscreen-resume --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-session locked --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --sample-paused --keep
scripts/wayland-video-surface-smoke.sh --allow-missing
scripts/wayland-video-surface-smoke.sh --no-build --keep
```

Use `--preflight` first when validating a real compositor session. It checks
`WAYLAND_DISPLAY`, `XDG_RUNTIME_DIR`, required tools, built binaries, and the
GStreamer elements needed by the generated MP4/H.264 test wallpaper
(`playbin`, `gtk4paintablesink`, `qtdemux`, and an H.264 decoder candidate)
without starting the daemon or changing the current wallpaper. With
`--report-dir`, it writes stable `metadata.txt`, `checks.csv`, and `summary.txt`
evidence that can be attached before a visual run. Missing GStreamer element
rows include package hints for common runtime gaps such as `gtk4paintablesink`
(`gst-plugin-gtk4` on Arch-like systems).

The smoke is intentionally partly visual: after the script reports success,
confirm that the selected output shows the generated moving test video. Pass
`--all-outputs` to apply the same generated video wallpaper to every
daemon-reported output and assert that each target output has an active
`render_sync.video_plans` entry. It also checks that `gtk4paintablesink` is
available. Use `--visual-hold <sec>` to keep the applied wallpaper visible for a
fixed confirmation window before sampling or cleanup. With
`--sample-performance`, it also runs `performance-snapshot.sh` against the
isolated daemon and writes
`performance-active/samples.csv`, `summary.txt`, and status snapshots under the
same kept work directory. With `--sample-paused`, it captures the active sample,
pauses the selected output, verifies a `user-paused` performance decision,
captures `performance-paused/`, and resumes the output.
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

The exact hardware decode path is left to the host GStreamer installation. The
smoke test intentionally uses `fakesink` so it can run in CI without a Wayland
session. Current Wayland surface smoke evidence should be treated as a
software-decoding or auto-selected-decoder baseline unless paired with codec
smoke evidence that records a hardware decoder element such as VAAPI/VDPAU/NVDEC.
The generated H.264 surface smoke does not force hardware decode.

For muted video wallpapers, Gilder disables `playbin` audio stream selection
instead of routing decoded audio to `fakesink`, so muted wallpaper playback does
not spend CPU or memory decoding an unused audio stream. `runtime.allow_audio`
and entry-level mute settings still allow audio when a package explicitly asks
for it.

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
- daemon status/watch reporting of running video pipeline decoder elements;
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
renderer update queue queued/skipped counters.
For memory comparisons, prefer `avg_uss_kib` or its equivalent
`avg_private_kib` for the process-private footprint and `avg_pss_kib` for the
shared-memory-adjusted footprint; `avg_rss_kib` includes shared mappings at
full size and is not private usage.
Pass `--pid`, `--socket`, or `--gilderctl` when testing an isolated daemon such
as the Wayland surface smoke script. The CSV, summaries, and raw status files
are intended to be compared between scenarios; GPU sampling remains
platform-specific follow-up work.
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
again.
`scripts/desktop-policy-smoke.sh` runs the same assertion path without GTK,
GStreamer, or a Wayland session by setting `GILDER_DESKTOP_OUTPUTS` to a
virtual output and covering active, battery, unfocused, fullscreen, hidden,
inactive, locked, and per-output performance override scenarios against the
default daemon build. It asserts mode, reason, action, plan kind, and expected
`max_fps` where the decision should remain renderable. The GitHub Actions
workflow runs it in strict mode and uploads `/tmp/gilder-desktop-policy-smoke`
as the `desktop-policy-smoke` artifact. The artifact includes top-level
`metadata.txt`, `matrix.csv`, and `summary.txt` files, plus per-scenario status
snapshots, daemon logs, decision summaries, and telemetry summaries.
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
