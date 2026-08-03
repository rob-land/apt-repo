# rob-land apt repo

The shared, GPG-signed Debian/Ubuntu package repository for rob-land's
mobile Linux apps, published to <https://rob-land.github.io/apt-repo/>.

```sh
sudo curl -fsSLo /usr/share/keyrings/rob-land.gpg \
  https://rob-land.github.io/apt-repo/rob-land.gpg
echo "deb [signed-by=/usr/share/keyrings/rob-land.gpg] \
https://rob-land.github.io/apt-repo stable main" \
  | sudo tee /etc/apt/sources.list.d/rob-land.list
sudo apt update
sudo apt install lighthouse
```

## Why this exists alongside the Flatpak repo

Most of the suite ships as Flatpaks from [flatpak-repo]. A few apps
can't: anything that needs a **systemd user unit**, host binaries like
`wpctl`, or unsandboxed LAN access is crippled by the Flatpak sandbox.
Those ship as `.deb`s here instead. See [Which apps belong
here](#which-apps-belong-here).

The two repos are independent — an app can be in both (GUI Flatpak,
daemon `.deb`), and installing one does not shadow the other.

## How it works

The same two-tier shape as [flatpak-repo]: each app repo builds its own
package and uploads it to its rolling `continuous` GitHub release, so
**app repos hold no secrets**. The [aggregate
workflow](.github/workflows/aggregate.yml) here runs hourly (and on
manual dispatch), downloads every app's `debs.tar` anonymously, lays
them out in a pool, generates the indexes with `apt-ftparchive`, signs
`InRelease` + `Release.gpg` with the repo GPG key, and deploys to
GitHub Pages. Deploys are skipped when no package changed.

An apt repo is just static files, so there is no OSTree equivalent to
merge — the published tree is:

```
/dists/stable/InRelease                       signed inline
/dists/stable/Release, Release.gpg
/dists/stable/main/binary-{amd64,arm64}/Packages{,.gz}
/pool/main/l/lighthouse/lighthouse_*.deb
/rob-land.gpg, /rob-land.asc                  public key, both forms
```

`apt-ftparchive` is used rather than `reprepro`/`aptly` deliberately:
it is a pure function of the pool directory, with no database to carry
between runs, which is what lets the aggregator stay stateless.

## Adding an app

1. Add a `debian/` directory to the app (see [lighthouse] for the
   reference: `--buildsystem=meson`, `Architecture: all`).
2. Add a `.github/workflows/deb.yml` that builds in a `debian:trixie`
   container and uploads `debs.tar` to the `continuous` release.
3. Add the app's name to `APPS` in
   [`aggregate.yml`](.github/workflows/aggregate.yml).

Packages are built `Architecture: all` — the suite is pure Python plus
GObject introspection, with nothing linked against a C library — so one
build serves both amd64 and arm64. If an app ever grows a compiled
extension it needs a per-arch build matrix and the `--arch` filtering
in the aggregator will start doing real work.

Building in a `debian:trixie` container (rather than on the Ubuntu
runner directly) matters: it pins the dependency *names and versions*
to what Debian-derived phone distros actually ship.

## Which apps belong here

| App | Why | Status |
| --- | --- | --- |
| [lighthouse] | systemd user unit for the always-on find-my agent; needs `wpctl` to override silent mode and raw LAN access for KDE Connect | packaged |
| [holler] | warm-resident systemd user unit, so an incoming call reaches an already-connected client instead of paying the cold-start tax | packaged |
| [pulse] | headless health-data daemon other apps talk to over D-Bus; a system-wide service is a poor fit for a per-user sandbox | candidate |

Holler was called Patch until it was packaged: GNU patch owns both the
`patch` package name and `/usr/bin/patch`, and dpkg will not let a second
package take that path. It's a good illustration of the tax a host
package pays that a sandboxed one doesn't — worth checking a name against
`apt-cache policy` and `apt-file search /usr/bin/<name>` before adding an
app here.

Everything else in the suite is a plain sandboxed GUI and should stay
Flatpak-only.

## Secrets

Set on this repo (the app repos need none):

| Secret | Value |
| --- | --- |
| `APT_GPG_PRIVATE_KEY` | armored `gpg --export-secret-keys --armor <KEY_ID>` |
| `APT_GPG_KEY_ID` | the key id used to sign `InRelease` |

This can be the same key that signs [flatpak-repo]; the secrets are
per-repo, so it has to be set here regardless.

[flatpak-repo]: https://github.com/rob-land/flatpak-repo
[lighthouse]: https://github.com/rob-land/lighthouse
[holler]: https://github.com/rob-land/holler
[pulse]: https://github.com/rob-land/pulse
