---
name: cleanup-releases
description: Delete old GitHub releases and GitHub Packages versions of a shared library (smart-home-sdk, shelly-client, cholewa-commons, cholewa-security), keeping only the 3 newest. Takes the library name as argument; without it, asks the user which library to clean. Use when the user wants to prune or clean up library releases ("posprzątaj release'y", "usuń stare wersje releasów", "wyczyść releasy cholewa-commons", "cleanup releases").
---

# cleanup-releases

Prune ONE shared library so that only the **3 newest versions** remain: delete the old
GitHub **releases** and the matching **GitHub Packages versions**. **Git tags are never
touched** — the source of every version stays tagged. Deleting a package version breaks
any consumer whose pom pins it, so consumer poms are checked before anything is deleted.

## 1. Resolve the library

The eligible libraries are the shared libraries listed in
`organization-repository/claude/organization.md` (library table). Owner mapping per the
note under that table:

- org account: `smart-home-automation-system/<library>` (e.g. `smart-home-sdk`,
  `shelly-client`)
- personal account: `magikabdul/cholewa-commons`, `magikabdul/cholewa-security`

If the skill was invoked **with an argument**, match it against that list. If invoked
**without an argument** — or the argument matches nothing — ask the user which library to
clean (AskUserQuestion), listing all eligible libraries by name. Never guess and never
run against a service repo: this skill is for libraries only.

## 2. List releases and package versions (read-only)

```
gh release list -R <owner>/<library> -L 200
```

Package versions (maven package name = `<groupId>.<artifactId>`, e.g.
`cloud.cholewa.smart-home-sdk`):

- org library: `gh api /orgs/smart-home-automation-system/packages/maven/<name>/versions --paginate`
- personal library: `gh api /user/packages/maven/<name>/versions --paginate`
  (requires `gh` authenticated as `magikabdul`; token needs the `delete:packages` scope
  for step 4 — check with `gh auth status`)

Then:

- Order releases by **semver, descending** (tags have no `v` prefix). Cross-check against
  the list order and the `Latest` marker — on mismatch, stop and show both orderings
  instead of deleting anything.
- Keep set = the 3 newest versions. Delete set = everything else, including drafts and
  pre-releases older than the keep set.
- If there are **3 or fewer releases**, report that there is nothing to delete and stop.
- Note package versions that have no matching release (and vice versa) — list them in the
  plan instead of silently skipping.

## 3. Consumer safety check (read-only)

Deleting a package version is **irreversible and breaks every consumer pinned to it**.
For each consumer of the library (consumer table in `organization.md`), read the version
referenced in its pom on `main` (local clone or `gh api .../contents/pom.xml`). If any
consumer pins a version from the delete set, **exclude that version from deletion**,
flag it in the plan, and let the user decide explicitly whether to delete it anyway.

## 4. Present the plan and get confirmation

Show a short table: versions kept (with dates) and versions to delete — for each deleted
version both the release and the package version id. State explicitly that git tags
remain. **Delete only after the user confirms** — both deletions are irreversible.

## 5. Delete

For each version in the delete set:

```
gh release delete <tag> -R <owner>/<library> --yes
gh api -X DELETE /orgs/smart-home-automation-system/packages/maven/<name>/versions/<version-id>   # org
gh api -X DELETE /user/packages/maven/<name>/versions/<version-id>                                # personal
```

No `--cleanup-tag` — tags stay. Never delete the release marked `Latest`. If a package
version delete fails (e.g. GitHub refuses for a public package with >5000 downloads),
report it and continue with the rest — do not retry blindly.

## 6. Verify and report

- `gh release list -R <owner>/<library>`: exactly the kept versions remain, `Latest`
  still pointing at the newest.
- Package versions endpoint again: same surviving set.
- Spot-check that tags survived: `git ls-remote --tags` (or the repo's tags page).

Report deleted count (releases and package versions separately), the surviving versions,
and anything excluded or failed.
