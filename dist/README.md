# Distribution packaging

Packaging sources for distributing Rune through OS package managers.
The goal is inclusion in the official distro repositories (Arch
`[extra]`, Debian, Ubuntu, ...), so the Linux packages build from
source. The Homebrew cask repackages the signed, notarized DMG because
macOS artifacts cannot be reproduced from source.

| Platform             | Path            | Mechanism                                   |
|----------------------|-----------------|---------------------------------------------|
| Arch Linux / Omarchy | `arch/PKGBUILD` | Source build, AUR first                     |
| Debian / Ubuntu      | `debian/`       | Source build via `dpkg-buildpackage`        |
| Homebrew (macOS)     | `homebrew/`     | Cask over the notarized DMG release asset   |

Packages are built from `github.com/unstablebuild/rune`, which needs no
submodules and no code generation: `cmd/rune/docs` is vendored in-tree
and embedded via `go:embed`.

## Install layout (Linux)

Both Linux packages install the `rune.app` bundle layout under
`/usr/lib` and expose the binary through a symlink:

```
/usr/lib/rune.app/bin/rune              # the binary
/usr/lib/rune.app/share/zdot/.z*        # zsh bootstrap files
/usr/bin/rune -> ../lib/rune.app/bin/rune
/usr/share/applications/rune.desktop
/usr/share/icons/hicolor/{512x512,1024x1024}/apps/rune.png
```

The `rune.app/bin` layout is load-bearing: `linuxAppDir`
(`cmd/rune/main.go`) only enables the bundled-zdot bootstrap when the
executable path lives in `.../rune.app/bin/`. The `/usr/bin/rune`
symlink is safe because `os.Executable()` resolves through
`/proc/self/exe` to the real path. Unlike the portable tarball,
packages ship no `rune.app/lib`: the binary links the distro's own
shared libraries.

Rune's ELF binary links only libc and libm. It loads the X11/OpenGL
client libraries only when it opens a GUI window, so `rune --tui` and
`rune --headless` work on a machine with no graphical stack installed.
The Linux packages keep the GUI libraries as optional/recommended
dependencies rather than making every headless node install them.

Both packages bake the same version stamps and production endpoints
that `RUNE_ENV=prod` sets in `cmd/rune/Makefile`
(`internal/debug.{Tag,Commit,Package}` plus the four
`cmd/rune/ide/apiclient.Default*` addresses).

They also set `-X unstable.build/rune/internal/debug.OSPackaged=true`,
which disables the in-product upgrader: the package manager owns the
install prefix, so an in-place upgrade would fight it over files it
does not own. With it set, the background update check never starts
(so no upgrade nag appears) and the console's `upgrade` command
reports:

> This build was distributed by an OS package manager, so auto-updates
> are disabled. Check your distribution's package manager for updates.

Any future source packaging (RPM, Nix) must set the same flag.

## Build requirements (source packages)

Both Linux packages build inside Docker, so no Debian or Arch host is
needed:

```bash
make pkg-deb              # .deb for the host arch      -> target/pkg/
make pkg-deb-amd64        # .deb for linux/amd64
make pkg-deb-arm64        # .deb for linux/arm64
make pkg-arch             # Arch package                 -> target/pkg/
make pkg-arch-srcinfo     # regenerate dist/arch/.SRCINFO only (fast)
make pkg-clean
```

`.github/workflows/packages.yml` runs all of it on amd64 runners and
uploads the artifacts, which is the supported way to produce release
packages.

Both Linux builds package the working tree, not a published tag, so CI
validates the commit under review. `arch/PKGBUILD` keeps its
`git+https://...#tag=v$pkgver` source for AUR users; the container
replaces it with the staged tree, because a git worktree's `.git` is a
file pointing outside the build context and cannot be cloned.

Caveats:

- **`make pkg-arch` needs an x86_64 builder.** `archlinux` is
  published for amd64 only and the Go toolchain segfaults under
  `qemu-user`, so it cannot be emulated on an arm64 host. Use CI or a
  native amd64 machine. `make pkg-arch-srcinfo` only parses the
  PKGBUILD, so it works anywhere.
- **`DEB_BASE_IMAGE` sets the `.deb`'s glibc floor**, because the
  package links the distro's own libraries. The default
  `debian:bookworm` (glibc 2.36) covers Debian 12+ and Ubuntu 24.04+.
  Override for a wider reach, e.g.
  `make pkg-deb DEB_BASE_IMAGE=debian:bullseye` for glibc 2.31
  (Ubuntu 22.04), accepting an EOL build base.

Building outside Docker needs Go >= 1.26, a C toolchain (cgo), the dev
headers listed in `debian/control`, and network access for
`go mod download`.

## Arch Linux / Omarchy

`arch/PKGBUILD` builds from the release tag. Omarchy is Arch-based and
installs from the AUR, so the same package covers it.

```bash
make pkg-arch                    # in Docker (amd64 builder required)
cd dist/arch && makepkg -si      # on an Arch host
```

Release procedure:

1. Bump `pkgver`, reset `pkgrel=1`.
2. `make pkg-arch-srcinfo` (regenerates `.SRCINFO`; CI fails if it
   drifts from `PKGBUILD`).
3. Push `PKGBUILD` + `.SRCINFO` to `ssh://aur@aur.archlinux.org/rune.git`.

Once the AUR package has traction, ask an Arch package maintainer to
adopt it into `[extra]`.

The GUI libraries are `optdepends`, not `depends`: Rune loads them only
when opening a window. This keeps `rune --tui` and `rune --headless`
usable on minimal systems with no graphical stack.

## Debian / Ubuntu

`debian/debian/` is a standard debhelper source package;
`debian/build-deb.sh` stages the working tree into a temp directory,
generates `debian/changelog`, and runs `dpkg-buildpackage -b`.
`make pkg-deb` runs it in a container; run it directly only on a
Debian/Ubuntu host:

```bash
./dist/debian/build-deb.sh          # -> dist/out/rune_<version>_<arch>.deb
```

`-d` is passed so the upstream Go toolchain satisfies the build instead
of requiring the distro `golang-go` package. `RUNE_TAG` / `RUNE_COMMIT`
override the version when the tree has no git metadata, which is how
the container build works.

The package passes `lintian` clean and the binary is built
`-buildmode=pie`. `Depends` contains only the ELF-linked libraries;
X11/OpenGL is `Recommends`, so APT installs it for the normal GUI path
while `--no-install-recommends` supports TUI and headless installations.
Both container builds assert that `DT_NEEDED` lists nothing but
`libc.so.6` and `libm.so.6`, and the Debian one installs the package
without Recommends and runs `rune --version` on a bare image to prove
the loader starts it with no graphical stack present.

Official inclusion later means vendored or distro-packaged Go
dependencies (no-network build), `3.0 (quilt)` source format with orig
tarballs, and an ITP bug. This directory is the seed for that.

## Homebrew (macOS)

`homebrew/Casks/rune.rb` installs `Rune.app` from the DMG published as
a GitHub release asset and links the `rune` CLI. Publish it in a tap
repo (`github.com/unstablebuild/homebrew-rune`, casks under `Casks/`):

```bash
brew tap unstablebuild/rune
brew trust --tap unstablebuild/rune   # Homebrew 6 gates third-party taps
brew install --cask rune
```

or as a one-liner, without tapping first:

```bash
brew install --cask unstablebuild/rune/rune
```

Note that `brew install --cask unstablebuild/rune` does not work:
Homebrew only accepts a bare token or the full `user/repo/token` form,
and resolves a two-part `user/repo` against the default taps instead.

On each release, regenerate the cask — version and sha256 are read from
the `manifest-darwin-<arch>.json` assets that `cmd/rune/dist.sh`
uploads next to the DMGs — then commit the result to the tap:

```bash
./dist/homebrew/update-cask.sh      # rewrites Casks/rune.rb
```

The committed cask is pinned to v1.2.0 and passes `brew audit` and
`brew style`.

## Keeping packages in step with releases

Both Linux packages and the cask are pinned to a released tag, so each
release needs:

1. `update-cask.sh`, then commit `Casks/rune.rb` to the tap.
2. `pkgver` bumped in `arch/PKGBUILD` (`pkgrel=1`), regenerate
   `.SRCINFO`, push to the AUR.

A tag alone is not enough for either: both resolve assets from a
published GitHub *release*. `arch/PKGBUILD` additionally needs the
`v<pkgver>` tag to exist, since its source is
`git+https://...#tag=v$pkgver`.

## Future targets

- Fedora/RHEL: RPM spec reusing the same install layout; COPR first,
  then a Fedora package review.
- Flatpak/AppImage if demand appears; the portable tarball already
  covers "no package manager" installs.
