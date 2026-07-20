---
name: release
description: Guide a release of a smart-home-automation-system service (Docker image on Docker Hub) or library (artifact on GitHub Packages) — pre-release checks, semver proposal, exact commands, artifact verification. Use whenever the user wants to release, publish or tag a version ("zrób release", "wydaj nową wersję", "opublikuj bibliotekę", "release water-service").
---

# release

A guided checklist. **This skill does not execute the release** — the user runs the
release commands themselves; your job is to verify preconditions, propose the version,
print the exact commands, and verify the outcome afterwards. Read-only git/`gh` queries
are fine; anything that creates tags, releases or pushes is the user's move.

## 1. Detect the release type

Look at `.github/workflows/`:
- `release.yml` → **service**: the workflow builds the jar, sets the Maven version from
  the git tag, and pushes a Docker image `magikabdul/<repo>` tagged `latest` + the tag.
- `package.yml` → **library**: the workflow sets the Maven version from the tag and
  deploys to GitHub Packages.

Both trigger on **GitHub release created** — the whole release is one `gh release create`.
Poms keep a dev version; nothing needs editing before a release.

## 2. Pre-release checks (do these yourself, read-only)

- Working tree clean, branch `main`, in sync with `origin/main`.
- CI green on main: `gh run list --workflow=CI.yml -L 3`.
- Sonar state worth a glance: `gh run list --workflow=sonar.yml -L 1`.
- Last released version: `gh release list -L 3` (and `git tag --sort=-v:refname | head`).
- For a **library**: scan `git log <last-tag>..HEAD` for public-API changes; if breaking,
  say so — it changes the version proposal and consumers (from
  `organization-repository/claude/organization.md`) will need coordinated bumps.

Report each check as pass/fail. Stop and say what's wrong if anything fails.

## 3. Propose the version

Read `git log <last-tag>..HEAD --oneline` and propose the next semver
(fix-only → patch, new functionality → minor, breaking → major), with a one-line
justification. Let the user decide.

## 4. Hand over the commands

Print, ready to copy:

```
gh release create <X.Y.Z> --title "<X.Y.Z>" --generate-notes
```

(Match the tag format to the repo's existing tags — check whether they use a `v` prefix.)

## 5. Verify the outcome (after the user confirms they ran it)

- `gh run watch` / `gh run list --workflow=<release.yml|package.yml> -L 1` until green.
- Service: confirm the Docker Hub tag exists —
  `docker manifest inspect magikabdul/<repo>:<X.Y.Z>` (or the Hub web UI).
- Library: confirm the package version —
  `gh api /orgs/smart-home-automation-system/packages/maven/<group.artifact>/versions`.
- For a library release that consumers are waiting for: list the consumer repos and offer
  to prepare the version-bump checklist for them.
