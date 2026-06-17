# Video Codec Validation

Gilder's current video renderer uses GStreamer `playbin` with a headless
`fakesink` while the Wayland/layer-shell surface sink is still being wired.
This means codec validation can prove package loading, decode availability,
pipeline lifecycle, and converter preview generation, but it does not yet prove
real compositor presentation performance.

## Smoke Script

Run:

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

## Runtime Packages

On Ubuntu-like systems the strict smoke path expects:

- `ffmpeg`
- `gstreamer1.0-libav`
- `gstreamer1.0-plugins-base`
- `gstreamer1.0-plugins-good`
- `gstreamer1.0-plugins-bad`
- `gstreamer1.0-plugins-ugly`

The exact hardware decode path is left to the host GStreamer installation. The
smoke test intentionally uses `fakesink` so it can run in CI without a Wayland
session.

## Remaining Surface Work

After the video sink is bound to per-output Wayland/layer-shell surfaces, this
document should be extended with compositor-facing checks:

- one output, one video wallpaper;
- multiple outputs with the same source video;
- fullscreen pause and resume latency;
- battery/unfocused throttling behavior;
- CPU, memory, and GPU usage sampling while active and paused.
