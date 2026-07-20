---
name: sync-org-docs
description: Synchronize the smart-home-automation-system organization docs with reality — the repository map and consumer tables in organization-repository/claude/organization.md and the badge sections in the org profile README. Use after any repo is added, renamed, archived or removed, when library consumers change, or when the user says the org README/organization.md/badges are stale ("zaktualizuj dokumentację organizacji", "dodaj repo do readme", "popraw badge").
---

# sync-org-docs

Keeps two files honest:

1. `organization-repository/claude/organization.md` — loaded into every Claude session;
   staleness here directly misleads future work.
2. `organization-repository/profile/README.md` — the public face of the org.

Run from the workspace root (needs to see all repos).

## 1. Build the actual inventory

Ground truth is code, not memory:

- Local: workspace directories with their `pom.xml` / `package.json`; classify each repo —
  service (`spring-boot-starter-parent` + `Dockerfile`/`release.yml`) vs library
  (`package.yml` / `distributionManagement`) vs other.
- Remote: `gh repo list smart-home-automation-system --limit 100 --json name,isArchived`
  — catches repos not cloned locally and archived ones.
- Consumers: grep every service pom for the shared-library artifactIds to rebuild the
  consumer table.
- Ports: `server.port` / local profile ports from each service's `application.yaml`.

## 2. Diff against the docs

Compare the inventory with both files. Look for: missing repos, removed/archived repos
still listed, renamed repos (old name in tables, links or badge URLs), wrong consumer
lists, wrong ports, and pending changes in the "Pending architecture changes" section of
`organization.md` that have actually been completed — move those into the regular tables
and delete the pending entry.

## 3. Apply the edits

- `organization.md`: keep the existing table formats.
- Profile README: for a new repo, copy a complete badge block from an existing entry of
  the same kind (service vs library) and substitute the repo name everywhere — the badge
  URL patterns (CI workflow badge, SonarCloud quality gate with project key
  `smart-home-automation-system_<repo>`, shields.io language/commit/release badges) are
  already correct in the file; don't compose them from scratch. For renames, substitute
  the name in every URL of the block.

Leave commits to the user; finish with a summary of what changed in each file and a
reminder to review with `git -C organization-repository diff`.
