# Unifying the four package aggregators

Written 2026-08-09, after the apk repo landed. This is an assessment of
what a single reusable workflow would take, and a recommendation on how
much of it is worth doing.

The four repos are [flatpak-repo], [apt-repo], [rpm-repo] and
[apk-repo]. Two apps ([lighthouse], [holler]) feed all four.

## Where the duplication actually is

```
aggregators                 app-side workflows
  flatpak-repo   147          lighthouse  flatpak  134   holler  flatpak  138
  apt-repo       215          lighthouse  deb      115   holler  deb      125
  rpm-repo       199          lighthouse  rpm      136   holler  rpm      137
  apk-repo       185          lighthouse  apk       61   holler  apk       61
  ---------------------      ------------------------------------------------
  746 lines, 4 files          907 lines, 8 files
```

The app-side files are the bigger and worse problem: 907 lines across
eight files, and `lighthouse/deb.yml` vs `holler/deb.yml` differ only in
the package name, the toolchain list and the verify commands. Adding a
third app means six more near-identical files.

The aggregators look similar but are less duplicated than they appear —
see the split below.

## Genuinely shared

These are near-identical today and are the real candidates:

| Concern | Where | Notes |
| --- | --- | --- |
| Signing-secret guard | 3 aggregators | Written three times, character-for-character apart from the secret names. |
| State-hash → skip deploy | 4 aggregators | Same shape: hash something, compare to live `state.txt`, skip if unchanged on a `schedule` run. |
| Landing page + `state.txt` | 4 aggregators | Same structure, different install snippet. |
| `configure-pages` → `upload-pages-artifact` → `deploy-pages` | 4 aggregators | Byte-identical, including the `if:` guards. |
| Rolling `continuous` release upload | 6 app workflows | The `gh release create … \|\| true; gh release upload --clobber` pair. |
| Aggregator nudge | 6 app workflows | Byte-identical. |
| Snapshot version stamp | 6 app workflows | Same idea (`date -u` + short sha, monotonic), four different syntaxes. |

## Genuinely different

These resist unification, and pretending otherwise produces a worse
result than the duplication:

| Concern | flatpak | apt | rpm | apk |
| --- | --- | --- | --- | --- |
| Artifact source | fetch tar | fetch tar | fetch tar | **builds from source** |
| Container | ubuntu | ubuntu | fedora:43 | alpine via docker |
| Index tool | `build-update-repo` | `apt-ftparchive` | `createrepo_c` | `apk index` |
| Signing | OSTree gpg-sign | detached OpenPGP | OpenPGP on **packages + repomd** | **RSA**, not OpenPGP |
| Key | shared OpenPGP | shared OpenPGP | shared OpenPGP | **its own RSA key** |
| Layout | OSTree repo | `dists/` + `pool/` | `fedora/$releasever/` | `v3.24/$arch/` |
| Verify step | — | — | `rpm -Kv` per package | install on clean Alpine |

The apk column is the outlier on four of seven rows. That is not
incidental complexity to be factored away — it follows from apk
verifying package signatures at install time, which forces the build
into the aggregator and the key into a different format.

## What it would take

### Tier 1 — composite actions (recommended)

Create `rob-land/.github` (or `rob-land/ci`) holding three composite
actions, and have all four aggregators plus the app workflows call them:

- `require-secrets` — takes a list of secret names, fails with a
  `::error::` naming the missing ones and the command to set them.
- `publish-pages` — takes a site path and a state string; does the
  compare-to-live, the `state.txt` write, and the three Pages actions.
- `continuous-release` — takes a list of asset paths; does the
  create-or-ignore plus `upload --clobber`.

**Cost:** one new repo, ~120 lines of action YAML, and a one-line edit
in each caller. **Saves:** roughly 240 lines of duplication and, more
usefully, means a fix to the skip-deploy logic lands once. Composite
actions can be referenced as `rob-land/.github/.github/actions/x@main`,
so there is no release dance.

**Caveat:** a bug in a shared action breaks all four repos at once,
where today a bad edit breaks one. Pin to a tag rather than `@main` once
it settles.

### Tier 2 — reusable `workflow_call` for app-side builds

Collapse the eight app workflows into three called workflows
(`build-deb.yml`, `build-rpm.yml`, `build-apk.yml`) in the shared repo,
each taking inputs for package name, container image, toolchain packages
and verify commands. The app repos keep a ~15-line caller each.

**Cost:** the parameterisation becomes a small language — `toolchain:` as
a string blob, `verify:` as a shell snippet passed as an input. That is
noticeably harder to read and debug than the duplication it replaces,
and shell-in-YAML-in-input has its own quoting hazards.

**Recommendation: not yet.** At two apps the duplication is cheaper than
the abstraction. At four or more apps it flips — the crossover is when
you would otherwise be hand-editing the same block six-plus times.
Revisit when the third app needs packaging.

### Tier 3 — one aggregator repo for all formats

A single repo with a matrix over `[flatpak, apt, rpm, apk]`, each format
providing `acquire`/`index`/`sign` hooks, sharing checkout, state and
publish.

**Recommendation: don't.** Two reasons, one of them decisive:

1. **It changes every published URL.** Pages is per-repo, so one repo
   means one site with subpaths — `…/apt-repo/dists/…` becomes
   `…/packages/apt/dists/…`. Your `sources.list`, `.repo` file,
   `/etc/apk/repositories` and flatpak remote are all already deployed
   on the phone and the desktop. That is a migration on every device for
   no user-visible gain.
2. **It couples four things that fail independently.** Today a broken
   Fedora build leaves the apt repo publishing happily. In a matrix,
   shared-harness changes put all four at risk at once, and the hourly
   cron becomes one job whose slowest leg (apk, which builds from
   source) sets the cadence for all.

The saving would be perhaps 200 lines. Not worth either cost.

## Suggested order

1. Do Tier 1 when next touching any aggregator — it is small and the
   `require-secrets` guard has already been written three times.
2. Leave Tier 2 until a third app needs packaging.
3. Skip Tier 3 unless the URLs are being changed for some other reason
   anyway.

A cheaper win than any of these: **six of the eight app workflows differ
only in strings.** Before reaching for reusable workflows, consider
whether `deb.yml` and `rpm.yml` should be generated from a small
per-app manifest by a script in the app repo, checked in and diffable.
That keeps the workflows readable and greppable while removing the
hand-editing.

[flatpak-repo]: https://github.com/rob-land/flatpak-repo
[apt-repo]: https://github.com/rob-land/apt-repo
[rpm-repo]: https://github.com/rob-land/rpm-repo
[apk-repo]: https://github.com/rob-land/apk-repo
[lighthouse]: https://github.com/rob-land/lighthouse
[holler]: https://github.com/rob-land/holler
