# debian-pkg-skill

An agent skill for Debian package maintenance — gbp/pbuilder-backed local
builds, BTS and package-tracker checks, Debusine QA uploads, lintian, autopkgtest,
piuparts, and the commit/changelog habits used in day-to-day source package work.

The skill is harness-agnostic. The same `SKILL.md` + `references/` content drives
agents under Claude Code, OpenAI Codex, and any tool that reads `AGENTS.md`-style
instruction files. Harness-specific install metadata lives under `agents/`.

The guidance here is operational, not normative. It reflects one maintainer's
default workflow (gbp through pbuilder, amd64/sid test builds, debusine for
intermediate QA uploads). Debian Policy and the relevant team documentation
remain authoritative; adapt the defaults to your own packages and habits.

## Layout

- `SKILL.md` — entry point. Frontmatter conforms to the Claude Code / Codex
  Skills format (`name`, `description`). Body describes the operating model,
  first checks, workflow, and a reference map.
- `LOCAL.md` — maintainer-specific defaults (builder, arch/suite, upload
  targets). The one file a fork is expected to edit.
- `references/` — focused notes loaded on demand:
  - `policy.md` — Debian Policy, changelog/control/copyright/patch rules.
  - `tools.md` — gbp, pbuilder, sbuild, devscripts, uscan, quilt, pristine-tar.
  - `qa.md` — lintian, autopkgtest, piuparts, reproducibility, build logs.
  - `services.md` — tracker.d.o, BTS, Salsa, Debusine, buildd, mentors.
  - `workflows.md` — bug fixes, new upstream releases, NMUs, transitions,
    FTBFS, pre-upload checklist.
  - `extensions.md` — pointers and candidate future reference files.
- `agents/` — harness-specific install metadata (see below).
- `scripts/` — placeholder for any helper scripts the skill grows over time.

## Install

### Claude Code

Skills live under `~/.claude/skills/` (user) or `.claude/skills/` (project).
Symlink or copy the repository in:

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD" ~/.claude/skills/debian-pkg-skill
```

Claude Code auto-discovers `SKILL.md` and surfaces the skill by name and
description. Invoke implicitly ("inspect this Debian package…") or explicitly
via the skill name.

### OpenAI Codex

Skills live under `~/.codex/skills/` (user) or `.codex/skills/` (project).
Symlink or copy the repository in:

```bash
mkdir -p ~/.codex/skills
ln -s "$PWD" ~/.codex/skills/debian-pkg-skill
```

`agents/openai.yaml` provides the Codex display name and default prompt and
enables implicit invocation.

### Other AGENTS.md-aware tools

For tools that read a top-level `AGENTS.md` (Cursor, Aider, generic CLIs),
point them at this skill by adding a short pointer in your project's
`AGENTS.md`, for example:

```markdown
For Debian packaging tasks, follow the guidance in
`~/agent-skills/debian-pkg-skill/SKILL.md` and the matching
`references/*.md` files. Prefer reading only the reference needed for the
current task.
```

The skill itself does not need an `AGENTS.md` — `SKILL.md` is the entry point
for all harnesses.

## Customizing

Maintainer-specific defaults — builder choice, default arch/suite, log
location, commit-timing rules, and preferred upload targets — live in the
top-level `LOCAL.md`. Fork that file first; `SKILL.md` and the `references/`
are intended to be team-neutral and should rarely need editing.

Generic policy and agent-safety rules (e.g. how to invoke `dput`, when to
forward patches upstream, when to refuse a chroot operation) stay in their
topical reference and apply to every fork.
