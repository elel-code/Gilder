# Packaging Notes

This document records the installable assets that are useful for distribution
packages and user-local installs.

## Binaries

Install these binaries into a directory visible to the user session:

- `gilderd`
- `gilderctl`
- `gilder-convert`

The daemon is normally built with `gtk-renderer` for static wallpaper display.
Video support also needs `video-renderer` plus host GStreamer plugins. MP4/H.264
and WebM/VP9/AV1 smoke validation expects `gstreamer1.0-tools`,
`gstreamer1.0-libav`, and the base, good, bad, and ugly plugin sets on
Ubuntu-like systems.
Real GTK/layer-shell video display additionally needs a runtime plugin that
provides `gtk4paintablesink`, such as `gst-plugin-gtk4`; Gilder probes it at
runtime and keeps the poster visible when it is unavailable.
Wallpaper Engine video preview extraction in `gilder-convert` can use `ffmpeg`
from `PATH`; packages may declare it as an optional runtime dependency.

## Distribution Tarball

`packaging/build-dist.sh` builds and stages a tarball with binaries, man pages,
shell completions, the systemd user service, docs, and validation helpers:

```sh
packaging/build-dist.sh
```

By default it builds with `--features gtk-renderer,video-renderer` and writes to
`dist/gilder-<version>-<system>-<arch>.tar.gz`.

Useful options:

```sh
packaging/build-dist.sh --features gtk-renderer
packaging/build-dist.sh --profile debug --no-build --dest /tmp/gilder-dist
```

Validation helpers are installed under
`share/doc/gilder/scripts/video-codec-smoke.sh` and
`share/doc/gilder/scripts/wayland-video-surface-smoke.sh`, with
`share/doc/gilder/scripts/performance-snapshot.sh` for compositor-session
resource sampling.

## systemd User Service

The example user service is in `packaging/systemd/gilder.service`.

Recommended package install location:

```sh
/usr/lib/systemd/user/gilder.service
```

Recommended user-local install location:

```sh
~/.config/systemd/user/gilder.service
```

Enable it for the current user session:

```sh
systemctl --user daemon-reload
systemctl --user enable --now gilder.service
```

The unit uses `ExecStart=gilderd`, so packaged installs should put `gilderd`
on systemd's user-service executable search path, usually `/usr/bin`. For
ad-hoc installs under `~/.cargo/bin` or `~/.local/bin`, either install a wrapper
into `/usr/bin` or override `ExecStart` with an absolute path.

## Man Pages

Man pages are stored in `docs/man/`:

- `gilderd.1`
- `gilderctl.1`
- `gilder-convert.1`

Recommended package install location:

```sh
/usr/share/man/man1/
```

## Shell Completions

Bash completions are stored in `completions/bash/`.

Recommended package install location:

```sh
/usr/share/bash-completion/completions/
```

Zsh completions are stored in `completions/zsh/`.

Recommended package install location:

```sh
/usr/share/zsh/site-functions/
```
