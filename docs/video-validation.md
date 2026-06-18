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

Standalone Ubuntu CI jobs can let the smoke script install missing runtime
dependencies:

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
scripts/video-codec-smoke.sh --install-missing --work-dir /tmp
scripts/video-codec-smoke.sh --allow-missing
scripts/video-codec-smoke.sh --no-convert
scripts/video-codec-smoke.sh --keep
```

Every run writes `metadata.txt`, `results.csv`, and `summary.txt` inside the
smoke work directory. Use `--keep` for a temporary work directory that should be
preserved, or `--report-dir <dir>` when CI needs a stable artifact path. The
GitHub Actions workflow uploads `/tmp/gilder-video-codec-smoke` as the
`video-codec-smoke` artifact.

`--install-missing` is intended for Ubuntu-like CI runners and runs
`scripts/install-video-codec-smoke-deps-ubuntu.sh` before strict smoke checks so
`ffmpeg`, `gst-launch-1.0`, and the expected GStreamer plugin packages are
available. `--allow-missing` is intended for developer machines where optional
encoders or GStreamer plugins may not be installed. CI should run the script in
strict mode.

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
scripts/wayland-video-surface-smoke.sh --output eDP-1
scripts/wayland-video-surface-smoke.sh --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-power battery --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-output-state unfocused --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-output-state fullscreen --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --simulate-session locked --sample-performance --keep
scripts/wayland-video-surface-smoke.sh --sample-paused --keep
scripts/wayland-video-surface-smoke.sh --allow-missing
scripts/wayland-video-surface-smoke.sh --no-build --keep
```

The smoke is intentionally partly visual: after the script reports success,
confirm that the selected output shows the generated moving test video. It also
checks that `gtk4paintablesink` is available and that daemon status contains an
active `render_sync.video_plans` entry. With `--sample-performance`, it also
runs `performance-snapshot.sh` against the isolated daemon and writes
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

The GitHub Actions workflow installs the full CI dependency set through
`scripts/install-ci-deps-ubuntu.sh`; codec-only CI jobs can pass
`--install-missing` or run `scripts/install-video-codec-smoke-deps-ubuntu.sh`
first. Do not run strict codec smoke on a fresh Ubuntu CI image without one of
these installers, because `gst-launch-1.0` is provided by `gstreamer1.0-tools`.
As a guardrail, `scripts/video-codec-smoke.sh` will auto-run the codec
dependency installer when strict mode is used on a GitHub Actions or generic
`CI=true` Ubuntu runner and `ffmpeg` or `gst-launch-1.0` is missing. Local runs
still fail explicitly unless `--install-missing` or `--allow-missing` is passed.
If CI fails with `FAIL: gst-launch-1.0 is not available`, the dependency install
step did not run or did not complete before the codec smoke command. Use
`scripts/video-codec-smoke.sh --install-missing --work-dir /tmp` for strict
codec smoke jobs, and use `--allow-missing` only for optional smoke jobs where
missing codecs should be recorded as skips instead of failures.

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
session.

## Remaining Surface Work

The GTK paintable sink code path still needs compositor-facing checks:

- one output, one video wallpaper;
- multiple outputs with the same source video;
- fullscreen pause and resume latency;
- battery/unfocused throttling behavior;
- CPU, memory, and GPU usage sampling while active and paused.

## Performance Sampling

For repeatable active/paused/fullscreen/battery comparisons, collect daemon
resource and status evidence while the scenario is running:

```sh
scripts/performance-snapshot.sh --label active-video --duration 30 --interval 1 --keep
scripts/performance-snapshot.sh --label paused-video --duration 30 --interval 1 --keep
scripts/wayland-video-surface-smoke.sh --sample-performance --sample-duration 30 --keep
scripts/wayland-video-surface-smoke.sh --sample-paused --sample-duration 30 --keep
```

The script finds a running `gilderd` process, samples `ps` CPU/RSS/VSZ fields,
computes a small `summary.txt`, and writes one `gilderctl status` JSON snapshot
per sample. It also asks `gilderctl status --decisions-csv --from-file` to
produce `decisions.csv` and `decision-summary.txt`, so active/paused,
fullscreen, and battery scenarios can be compared by both resource usage and
the daemon's actual `mode/reason/max_fps` decision. The summary is generated
with a CSV-aware parser and includes decision row counts, unique samples and
outputs, `mode/reason` counts with FPS ranges, action counts, plan kinds, fit
modes, muted video counts, and target FPS ranges. It also asks
`gilderctl status --telemetry-csv --from-file` to produce `telemetry.csv` and
`telemetry-summary.txt`, which report desktop refresh deltas, read-request
refresh skips, desktop change deltas, and render-sync cache hit/miss deltas.
Pass `--pid`, `--socket`, or `--gilderctl` when testing an isolated daemon such
as the Wayland surface smoke script. The CSV, summaries, and raw status files
are intended to be compared between scenarios; GPU sampling remains
platform-specific follow-up work.
Use `--expect-mode`, `--expect-reason`, `--expect-action`, and
`--expect-plan-kind` to make a sampling run fail when the expected render
decision is not observed in `decision-summary.txt`. The Wayland video smoke
passes these expectations automatically for simulated battery, unfocused,
fullscreen, hidden, and user-paused scenarios.
For battery policy comparisons on machines that are not actually discharging,
run the daemon or smoke script with `GILDER_POWER_STATE=battery`; unset it to
return to sysfs-based power detection.
For compositor-state policy comparisons where changing the real desktop state
is awkward, use `GILDER_OUTPUT_STATE=unfocused`, `fullscreen`, or `hidden`; unset
it to return to compositor/GDK state detection.
For session-state policy comparisons where switching VT or locking the real
session is awkward, use `GILDER_SESSION_STATE=inactive` or `locked`; unset it to
return to logind state detection.
