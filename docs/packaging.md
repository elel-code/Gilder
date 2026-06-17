# Packaging Notes

This document records the installable assets that are useful for distribution
packages and user-local installs.

## Binaries

Install these binaries into a directory visible to the user session:

- `gilderd`
- `gilderctl`
- `gilder-convert`

The daemon is normally built with `gtk-renderer` for static wallpaper display.
Video support also needs `video-renderer` plus host GStreamer plugins.
Wallpaper Engine video preview extraction in `gilder-convert` can use `ffmpeg`
from `PATH`; packages may declare it as an optional runtime dependency.

## Distribution Tarball

`packaging/build-dist.sh` builds and stages a tarball with binaries, man pages,
shell completions, and the systemd user service:

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
