# Local Maintainer Defaults

Personal defaults for this fork of `debian-pkg-skill`. Everything here
describes *this maintainer's* environment, not generic Debian practice —
`SKILL.md` and the `references/` are intended to be team-neutral and should
not encode personal preferences. Forks should edit this file first.

Load this file whenever the task needs a concrete builder command, default
architecture/suite, or upload target.

## Builder

The local builder is **`mypbuilder`**, a personal shell wrapper around
`pbuilder` at `~/usr/bin/mypbuilder`. It is **not** `gbp buildpackage`;
do not assume `gbp buildpackage --git-pbuilder` is the build path. `gbp`
is still used for non-build tasks (see *gbp usage* below).

### Interface

```
mypbuilder DIST ARCH OPERATION [DSCFILE] [DEPSDIR] [-- extra pbuilder args]
```

- `DIST` — distribution. A `bpo` suffix marks a backport target (e.g.
  `bookwormbpo`); the suffix is stripped before debootstrap.
- `ARCH` — target Debian architecture. The wrapper rejects cross-builds where
  target bits > host bits, and uses a `linux32` personality when target
  bits < host bits.
- `OPERATION` — one of `create`, `update`, `clean`, `build`, `autopkgtest`,
  `login`, `execute`, `edit`, `edit-script`, `debuild`.
- `DSCFILE` — required for `build` and `autopkgtest`.
- `DEPSDIR` — optional. If supplied, `mypbuilder` runs `apt-ftparchive
  packages .` in it and bind-mounts it as a
  `deb [trusted=yes] file://` source inside the chroot. Useful for testing
  against locally built dependencies without a full repo.

### State layout

Everything lives under `~/pbuilder/`:

- `etc/<DIST>-<ARCH>` — pbuilderrc per dist-arch (required; the wrapper
  fails if missing or empty).
- `base/<DIST>-<ARCH>-base.tgz` — base tarball.
- `hooks/` — pbuilder hook scripts.
- `logs/<DIST>-<ARCH>/<OP>_<BASENAME>_<YYYYMMDD-HHMM>.log` — written
  automatically; **do not add a separate `tee` log**, the wrapper handles it.
- `result/<DIST>-<ARCH>/` — built `.deb` / `.changes` land here.

The wrapper invokes `pbuilder` via `sudo` and sets
`DEB_BUILD_OPTIONS=parallel=$(nproc)` automatically.

### Default arch / suite

- Architecture: `amd64`.
- Suite: `sid` (equivalent to `unstable`; preserve the spelling actually used
  in commands when documenting or rerunning).

## Standard build / test commands

Produce the source package first (e.g. `dpkg-buildpackage -S -us -uc` or
`gbp buildpackage -S` from the source tree), then drive `mypbuilder`:

```bash
# clean-chroot binary build
mypbuilder sid amd64 build ../<source>_<version>.dsc

# autopkgtest against the dsc, chroot derived from the base tarball
mypbuilder sid amd64 autopkgtest ../<source>_<version>.dsc

# build with a local deps directory bind-mounted as an apt source
mypbuilder sid amd64 build ../<source>_<version>.dsc /path/to/deps
```

Built `.changes` and `.deb` files land in
`~/pbuilder/result/sid-amd64/` (or the matching `<DIST>-<ARCH>` directory),
not in `..` — adjust `lintian`, `debdiff`, `dput`, and similar follow-ups
accordingly. Result files older than the current build are not
automatically pruned.

Chroot management:

```bash
mypbuilder sid amd64 create     # create base tarball (network; ask first)
mypbuilder sid amd64 update     # refresh existing base tarball
mypbuilder sid amd64 login      # throwaway interactive shell
mypbuilder sid amd64 edit       # interactive shell, changes persisted to base
mypbuilder sid amd64 execute -- /path/to/script.sh
```

`create` and `update` need network; ask before running them.

## gbp usage

`gbp` is the preferred tool for the rest of the packaging workflow, just
**not** the builder:

- `gbp dch` — generate changelog drafts from git history.
- `gbp import-orig` — import a new upstream tarball (with `--pristine-tar`
  when the repo uses it).
- `gbp pq import` / `gbp pq export` — manage the quilt patch queue.
- `gbp tag` — apply the release tag after upload.

Do not reach for `gbp buildpackage` or `gbp buildpackage --git-pbuilder`
unless the user explicitly asks for it.

## Commit and release-changelog timing

- Commit non-changelog packaging changes promptly so `gbp dch` has useful
  raw input.
- Usually delay the final release changelog commit until *after* the upload
  succeeds. A pre-upload build may therefore have exactly one uncommitted
  file, `debian/changelog`, with distribution already set to `unstable`.

## Upload targets

- QA / intermediate uploads: `dput debusine.debian.net <changes>`. Debusine
  accepts unsigned or work-in-progress source uploads; treat this as
  different from archive upload.
- Archive uploads: only with explicit user instruction for the exact target;
  do not select ftp-master implicitly.
- Generic `dput` invocation safety is in `references/tools.md`
  (Upload Helpers); the archive-vs-debusine policy is in
  `references/services.md`. This file records only *which* targets this
  maintainer uses.

## Privileged operations

`mypbuilder` invokes `pbuilder` via `sudo`. The first run in a session will
prompt for a password. `create`/`update` operations need network and write
to `~/pbuilder/base/`; ask before running them.
