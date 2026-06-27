# Debian Packaging Workflows

The upstream-source boundary (when to read upstream code vs. just `debian/`, and which upstream agent-instruction files to avoid) is part of the operating model in `SKILL.md`. Re-read it before any workflow that might pull you into the upstream tree.

## Bug Fix In Debian Packaging

1. Read the BTS report, package tracker, and Salsa MRs/issues.
2. Reproduce locally if possible with the package's normal build/test path.
3. Decide whether the fix belongs in Debian packaging, upstream source, dependencies, or tests.
4. For upstream source in `3.0 (quilt)`, add or update a quilt patch with DEP-3 headers.
5. Update `debian/changelog` with a concise entry and `Closes: #NNNNNN` if appropriate.
6. Run focused validation, then full build/autopkgtest if the fix affects runtime behavior.

For quilt patches, use this default sequence:

```bash
quilt push -a
quilt new fix-specific-issue.patch
quilt add path/to/file
editor path/to/file
quilt diff
quilt refresh
quilt header -e
quilt pop -a
dpkg-source --before-build .
```

The patch header must explain provenance and forwarding state. Use `Forwarded: not-needed` only for genuinely Debian-specific changes; otherwise use `Forwarded: no` until the patch is sent upstream.

## New Upstream Release

1. Check tracker, watch file, upstream changelog, current Debian delta, and open bugs.
2. Run `uscan --report`; use `gbp import-orig --uscan` if the repository supports it.
3. If `pristine-tar = True`, import with pristine-tar enabled and verify the orig tarball is reproducible.
4. Refresh/drop Debian patches; forward remaining patches upstream where possible.
5. Review copyright and license changes.
6. Review build system changes and dependencies.
7. Generate a changelog draft with `gbp dch`, then edit it into release notes and add `Closes:` entries.
8. Run clean build, lintian, and autopkgtest.
9. Update changelog as `<upstream>-1` unless Debian versioning requires otherwise.

Common command sequence:

```bash
uscan --report
gbp import-orig --uscan --pristine-tar
quilt push -a
quilt refresh
quilt pop -a
gbp dch
editor debian/changelog
dpkg-buildpackage -S -us -uc
mypbuilder sid amd64 build ../<source>_<version>.dsc   # see LOCAL.md
```

Drop `--pristine-tar` only when the repository does not use pristine-tar.

## Packaging-Only Revision

Use for metadata, dependency, test, or packaging fixes with no upstream version change.

1. Increment Debian revision with `dch`.
2. Keep changes under `debian/` unless a quilt patch is required.
3. Commit each logical change with a changelog-ready subject, for example `d/control: add missing build dependency on foo`.
4. Use `gbp dch` when the packaging branch has meaningful commits to summarize.
5. Edit `debian/changelog` to remove commit noise and add accurate `Closes:` entries.
6. Validate with source build and relevant QA.
7. Avoid unrelated standards/version churn.

## Commit Before Changelog Generation

Before running `gbp dch`, review recent commit subjects:

```bash
git log --oneline <last-upload-tag>..HEAD
```

The subjects should be good raw material for changelog entries. If they are vague, mixed-purpose, or too implementation-heavy, prefer fixing/squashing local commits before generating the changelog when history is still private. Do not rewrite shared history without explicit maintainer approval.

Suggested commit granularity:

- one commit for a dependency/control change
- one commit for a quilt patch add/remove/refresh set
- one commit for autopkgtest changes
- one commit for symbols/install/rules metadata updates
- one release commit for final `gbp dch`/upload changelog state when repository practice uses it

This maintainer's commit-timing rule (commit non-changelog changes promptly; usually delay the release changelog commit until after upload) is recorded in `LOCAL.md`.

## NMU

Non-maintainer uploads require extra care.

1. Confirm the bug severity and NMU appropriateness in Developer's Reference.
2. Keep the diff minimal.
3. Use NMU versioning, usually `+nmu1` for non-native packages.
4. Include changelog entries that clearly identify the fixed bug.
5. Respect DELAYED queue conventions unless an immediate upload is justified.

## Security Update

Port a published fix into one or more released suites. Goal: the smallest
verifiable diff that resolves the issue — no refactors, no unrelated cleanups,
no version bumps. This workflow ends at "patches written, series and changelog
updated"; build, lintian, autopkgtest, and upload use the standard
`Pre-Upload Checklist` and the builder in `LOCAL.md`.

### 1. Identify the issue

When the task names a *source package* rather than a single CVE, start from the
package overview page:

```
https://security-tracker.debian.org/tracker/source-package/<src>
```

It tabulates every known issue against each suite. The work-list is exactly the
rows marked **vulnerable** for your target suite(s) — `fixed`, `no-dsa`,
`<not-affected>`, and `<unimportant>` rows need no patch. Treat the per-CVE
detail pages as the authoritative description; the overview table truncates.

Capture both names each issue may go by:

- **CVE-YYYY-NNNNN** if assigned.
- **TEMP-NNNNNNN-XXXXXX** if the security tracker only has a temporary name
  (the pre-CVE placeholder on security-tracker.debian.org).

Open `https://security-tracker.debian.org/tracker/<id>`. The page lists:

- A short description.
- Per-suite status (vulnerable / fixed) — source of truth for which releases
  need work and which one may already carry a reusable patch.
- A **fixed version** column. This names the *first Debian/upstream version
  that carries the fix* and is the key to finding the commit (see §3): it bounds
  the upstream git range to search (vulnerable version → fixed version).
- A **Notes** block with upstream advisory URLs, upstream commit links, and
  the Debian BTS report. The Notes are best-effort: frequently they contain
  only a GHSA / advisory URL and no commit hash — and the linked advisory page
  often omits the commit too. Do not assume the commit is findable from the
  tracker alone; §3 covers deriving it from the upstream repo.

Also capture the **Debian bug number** for `Closes:` in the changelog (a CVE
may have no Debian bug — that is fine, just omit `Closes:`).

### 2. Confirm scope with the user

Always ask which suites to target. Default: **stable + oldstable** security
suites (e.g. trixie-security and bookworm-security). Other suites are opt-in:

- unstable / experimental — often already fixed; confirm before duplicating.
- LTS / ELTS — different team and workflow; confirm before touching
  `dla-needed.txt` or producing `+deb11uN` uploads.

A multi-suite confirmation up front avoids redoing the port for the wrong
tree.

### 3. Source the fix

Cheapest source first:

1. **A reusable backport patch already shipped in another Debian release.**
   The tracker's "fixed" column is a hint, not a guarantee — a newer release
   can be marked fixed for several reasons:

   - It carries an explicit backport patch in `debian/patches/` (reusable).
   - It includes a newer upstream release that contains the fix in the
     `orig.tar.xz` (no targeted patch to copy).
   - The vulnerable code was rewritten or removed upstream, so the release
     is unaffected by construction (nothing to copy and nothing applies).

   Treat (a) as the only reusable case. Verify before fetching: open
   `debian/patches/series` and `debian/changelog` of that release and look
   for an entry matching the CVE / temp id / bug number. If both name it,
   the patch is reusable — pull its `.debian.tar.xz` (or the salsa
   `debian/<suite>` branch) and copy the DEP-3-headed patch. If only
   `debian/changelog` mentions it as a "New upstream release" with no
   matching patch, skip to source (2): the fix lives in upstream code that
   may or may not exist in your older version, and you need to compare to
   the upstream commit anyway.

2. **Upstream commit(s) from the Notes block.** Most git forges serve a raw
   mbox patch by appending `.patch` to a commit URL — gitlab.gnome.org,
   salsa.debian.org, GitHub, cgit all support this:

   ```bash
   curl -sL <commit-url>.patch -o /tmp/<short-hash>.patch
   ```

   When the Notes (and the GHSA / vendor advisory they link) name *no* commit,
   derive it from a local clone of the upstream repo. The tracker's
   **fixed version** column bounds the search: a CVE vulnerable in 3.7.1 and
   fixed in 3.11.0 lives somewhere in `git log 3.7.1..3.11.0`. Grep subjects
   and bodies for the vulnerability's topic, then confirm against the upstream
   release notes for the fixed version:

   ```bash
   git log --oneline <vuln-tag>..<fixed-tag> | grep -iE '<topic keywords>'
   git log -1 --format='%H%n%s%n%n%b' <candidate>     # read the body to confirm
   git show <fixed-tag>:news.rst | grep -iA3 CVE-YYYY-NNNNN   # or NEWS/ChangeLog
   ```

   The release-notes entry is the tiebreaker: it states what the project
   considers *the* fix and frequently cites the merging PR, which disambiguates
   when several nearby commits touch the same file.
3. **Vendor advisory** (Red Hat, SUSE, distro-patches list) if no upstream
   fix exists yet.

If multiple commits make up the fix, fetch each separately — they often want
to land as separate patches.

A CVE the release notes describe as "fixed in X.Y.Z" is not necessarily one
revertable commit: it may be a multi-commit hardening *series* whose later
commits depend on earlier refactors (changed signatures, new helpers). Confirm
applicability with `patch --dry-run` early — if the head commit's hunks fail
because the surrounding code was already rewritten upstream, you are in the
hand-recreated-backport case (§4 structural divergence, §5 DEP-3 format), not a
clean cherry-pick.

### 4. Adapt the patch to the target version

Upstream `main` is usually newer than the suite version. Check both paths and
context before trusting the patch:

- **File paths.** Upstream may have moved or renamed files. Rewrite the
  `--- a/…` / `+++ b/…` headers to the path that exists in the target
  version; verify with `find` or `git log --follow` on the upstream repo.
- **Hunk context.** Even when surrounding lines match, line numbers drift.
  `patch --dry-run` reports `Hunk #N succeeded at <line> (offset M lines)` —
  tolerable but worth fixing so the apply lands clean. Edit the
  `@@ -X,Y +X,Y @@` headers to match the target source.
- **Structural divergence.** If upstream refactored, renamed symbols, or
  introduced helpers that don't exist in the older release, preserve the
  *security intent*, not the textual diff. Recreate the equivalent check and
  document the divergence in a DEP-3 `Note:` paragraph.

Validate by dry-running:

```bash
patch -p1 --dry-run < debian/patches/<new>.patch
```

### 5. Write the patch files

- **One file per upstream commit. Do not combine.** Keep the upstream commit
  boundary intact even when the commits fix the same issue, are small, and
  always travel together. The cost of splitting is zero; the cost of
  unmerging later — partial revert, bisecting which half broke a test,
  selectively backporting one half to another suite — is real. Combining
  cherry-picks also makes the patch stop matching what `git show <hash>`
  produces upstream, which is what reviewers will compare against.

- **Order patches by upstream commit order.** When multiple commits make
  up the fix (or one upload addresses several CVEs), apply them in the
  order upstream did — author date, or the topological order from
  `git log --oneline --reverse <fix-range>` on the upstream repo. This
  avoids "applies but doesn't compile" intermediate states and matches
  what a reviewer would see cherry-picking them by hand. Reflect the same
  order in `debian/patches/series`.

- **Keep upstream test hunks.** Cherry-picks usually include changes to
  upstream tests and CI manifests; keep them. They are the cheapest way to
  document the fix and to catch a regression on rebuild, and dropping them
  invites the package's behaviour to drift from upstream. Drop test hunks
  only when retaining them would force pulling in an even larger earlier
  commit (e.g. a new test harness introduced a few commits before the
  fix) — and note the omission in the patch.

- **Format.**

  - **Clean upstream cherry-pick** — ship the file as `git format-patch`
    output, byte-identical to upstream except for two allowed edits: hunk
    line numbers tightened so apply is clean, and `--- a/…` / `+++ b/…`
    paths corrected when upstream moved the file. Keep the original
    `From <hash> Mon Sep 17 …` line, `From:`, `Date:`, `Subject:`, the
    commit body, and the `---` separator before the diff. **Do not add
    DEP-3 fields** (`Origin:`, `Bug:`, `Bug-Debian:`, `Forwarded:`,
    `Last-Updated:`) — the original `From <hash>` + author + subject
    already encode the provenance, and the changelog carries the CVE / bug
    citation. Extra headers make the patch diverge from upstream for no
    audit benefit and make future re-imports noisy to diff.

  - **Hand-recreated backport** — when the upstream patch can't be
    cherry-picked because of structural divergence (renamed symbols,
    missing helper, refactored call site), use DEP-3 headers and explain
    the divergence in a `Note:` paragraph. This patch is no longer the
    upstream commit, so it needs the provenance the `From:` line would
    otherwise have carried:

    ```
    From: <upstream author of the equivalent commit>
    Subject: <descriptive backport title>

    <description of the security fix>

    Origin: backport, <upstream commit URL>
    Bug: <upstream tracker URL>
    Bug-Debian: https://bugs.debian.org/<bugnum>
    Forwarded: not-needed
    Note: <how this differs from upstream — files renamed, helper inlined, etc.>
    ```

- **Filenames.** `CVE-YYYY-NNNNN.patch` when a single commit fixes a single
  CVE. With multiple commits per CVE, suffix each with the upstream short
  hash (`CVE-YYYY-NNNNN-d220aa2f.patch`) or an ordinal that matches
  upstream order (`CVE-YYYY-NNNNN-1.patch`, `-2.patch`). Without a CVE,
  use a descriptive slug (`sandbox-escape-1-no-ghelp-proc.patch`) or
  `bts-<bugnum>-*.patch`. Avoid temp tracker IDs in filenames — they are
  flagged "Not for external reference" and get superseded by a real CVE.

### 6. Wire into series and validate

Append the new patches at the end of `debian/patches/series` unless they need
to precede an existing Debian-only patch. Then exercise the full quilt cycle
in both directions:

```bash
QUILT_PATCHES=debian/patches quilt push -a   # all patches must apply
QUILT_PATCHES=debian/patches quilt pop -a    # and reverse cleanly
```

A `Hunk … succeeded at N (offset M lines)` warning is acceptable but signals
the line numbers should be tightened (see §4).

### 7. Update `debian/changelog`

Use **`dch --security`** (devscripts) — do not write the header by hand.
It opens a new stanza with the right shape derived from the current top
entry: the `+deb<N>u<M>` version suffix is incremented for the matching
suite, the distribution is set to `<codename>-security`, and urgency is
set to `high`. Hand-writing the header is error-prone — the suite number,
the `u<M>` counter, and the distribution name all have to agree, and
`dch --security` is the only thing that gets all three right at once.

```bash
cd <source-tree>
dch --security        # opens $EDITOR on a freshly-templated stanza
```

Then edit the body. Security uploads are per-suite source packages, so run
`dch --security` once per source tree (trixie tree, bookworm tree, …) —
not a single stanza spanning multiple suites.

Required content of the body:

- **NMU note** when the Security Team or another non-maintainer is
  uploading: start with `* Non-maintainer upload by the Security Team.`.
- **CVE / tracker reference**, when one exists. List each `CVE-YYYY-NNNNN`
  on its own line under the patch bullets so DSA tooling and search
  indexers pick it up. If the issue still has only a `TEMP-…` tracker id,
  cite it the same way; replace it with the real CVE on the next upload.
- **Debian bug closure** with `Closes: #NNNNNN` — this is what triggers
  the BTS to close the report when the upload reaches the archive.
- **Per-patch bullet** naming the file added and a one-line description of
  the change.

Example shape (with a CVE assigned):

```
yelp (42.2-4+deb13u1) trixie-security; urgency=high

  * Non-maintainer upload by the Security Team.
  * SECURITY UPDATE: <one-line description>
    - debian/patches/CVE-YYYY-NNNNN-<slug>.patch: <what it changes>.
    - CVE-YYYY-NNNNN
  * (Closes: #NNNNNN)

 -- <Uploader Name> <email>  <RFC2822 date>
```

If no CVE has been assigned yet, replace the `CVE-…` line with the
tracker temp id (e.g. `TEMP-1136299-DD6181`) and update on the next
revision once a CVE is allocated.

## FTBFS

1. Collect failing build log, architecture, suite, build profile, and dependency versions.
2. Try reproducing in clean pbuilder/sbuild matching the failure.
3. Distinguish missing Build-Depends from upstream/compiler/toolchain breakage.
4. Patch with the smallest change and include upstream forwarding status.
5. Validate at least a source build and clean binary build.

Compiler/toolchain fixes often belong upstream. If the patch is backported from upstream, set `Origin: upstream` or `Origin: backport`; if created in Debian, include `Bug-Debian` and forward it upstream when practical.

## Autopkgtest Regression

1. Read `debian/tests/control`, failing logs, triggered packages, and testbed architecture.
2. Reproduce with the same backend if possible; use `null` only for quick local checks.
3. Determine whether the regression is in the package, a dependency, test assumptions, or infrastructure.
4. Prefer fixing the package/test over adding broad restrictions.
5. If adding `Restrictions`, document why the test genuinely requires them.

## Library Transition

1. Identify ABI/API change, reverse dependencies, and release team coordination needs.
2. Update symbols/shlibs carefully; compare build logs and generated files.
3. Avoid breaking coinstallability or multiarch metadata.
4. Consider binNMUs and transition tracker state before upload.

## Pre-Upload Checklist

Run or explicitly defer:

```bash
git status --short
dpkg-parsechangelog
gbp config dump
dpkg-buildpackage -S -us -uc
mypbuilder sid amd64 build ../<source>_<version>.dsc       # builder per LOCAL.md
lintian ~/pbuilder/result/sid-amd64/<source>_<version>_<arch>.changes
mypbuilder sid amd64 autopkgtest ../<source>_<version>.dsc # autopkgtest per LOCAL.md
```

`mypbuilder` writes its own log to `~/pbuilder/logs/<DIST>-<ARCH>/`; do not add an external `tee` unless a copy is needed elsewhere.

Check:

- changelog distribution is correct, not `UNRELEASED`
- version is correct for target suite
- bug closures are intended
- source format and patch stack are clean
- generated files are cleaned
- Vcs fields point to the active repository
- test failures and lintian findings are understood
