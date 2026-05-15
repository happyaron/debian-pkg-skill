# debian-pkg-skill

This repository contains a Codex skill for Debian package maintenance.

It is intentionally tuned to Shengqi Chen's personal Debian packaging workflow:
`gbp`, pbuilder-backed local builds, BTS/package-tracker checks, Debusine QA
uploads, and the commit/changelog habits used in day-to-day package work.

The guidance here is not a general Debian packaging policy and may not fit every
maintainer, team, package, or archive workflow. Treat it as an operational
memory for this maintainer's environment, with Debian Policy and the relevant
team documentation remaining authoritative.

The skill entry point is `SKILL.md`; detailed notes live under `references/`.
