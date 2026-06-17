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

The script generates tiny synthetic samples and validates:

- MP4/H.264 can be generated with `ffmpeg` and decoded by GStreamer.
- WebM/VP9 can be generated with `ffmpeg` and decoded by GStreamer.
- WebM/AV1 can be generated with `ffmpeg` and decoded by GStreamer.
- `gilder-convert wallpaper-engine` can convert each generated video and create
  first-frame `poster.jpg` and `thumbnail.jpg` previews.

Useful options:

```sh
scripts/video-codec-smoke.sh --work-dir /tmp
scripts/video-codec-smoke.sh --allow-missing
scripts/video-codec-smoke.sh --no-convert
scripts/video-codec-smoke.sh --keep
```

`--allow-missing` is intended for developer machines where optional encoders or
GStreamer plugins may not be installed. CI should run the script in strict mode.

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
scripts/wayland-video-surface-smoke.sh --allow-missing
scripts/wayland-video-surface-smoke.sh --no-build --keep
```

The smoke is intentionally partly visual: after the script reports success,
confirm that the selected output shows the generated moving test video. It also
checks that `gtk4paintablesink` is available and that daemon status contains an
active `render_sync.video_plans` entry.

## Runtime Packages

On Ubuntu-like systems the strict smoke path expects:

- `ffmpeg`
- `gstreamer1.0-tools`
- `gstreamer1.0-libav`
- `gstreamer1.0-plugins-base`
- `gstreamer1.0-plugins-good`
- `gstreamer1.0-plugins-bad`
- `gstreamer1.0-plugins-ugly`

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
