---
name: new-library
description: Scaffold a new shared Maven library for the smart-home-automation-system organization — pom with GitHub Packages publishing, CI/package/sonar workflows, README, CLAUDE.md, GitHub repo creation and org docs update. Use whenever the user wants to create a new shared library, extract common code from services into a library, or says things like "stwórz bibliotekę", "wydziel to do biblioteki", "new shared client for X".
---

# new-library

Creates a library consistent with the existing ones. The reference repository is
**`smart-home-sdk`** — copy and adapt from it rather than inventing files.

Run from the workspace root. Before scaffolding, sanity-check the premise: the org rule
(see `organization.md`) is that a library earns a separate repo only with **two or more
consumers** or when it defines a cross-service contract. If the code has a single
consumer, say so and suggest keeping it inside that service — scaffold anyway if the user
confirms.

## 1. Gather inputs

- **Name**: lowercase, hyphenated (existing examples: `smart-home-sdk`, `shelly-client`,
  `cholewa-commons`).
- **Purpose** and **initial consumers** — both go into the README and into the consumer
  table in `organization.md`.

## 2. Scaffold from the reference

Copy from `smart-home-sdk/` and adapt:

- `pom.xml` — artifactId/name/description; keep groupId `cloud.cholewa` and the
  `<distributionManagement>` block, changing only the repo name in its URL
  (`https://maven.pkg.github.com/smart-home-automation-system/<name>`, server id `github`).
  **Versions**: use the org target toolchain (Java, and Spring Boot parent if the library
  uses it) from the Conventions section of
  `organization-repository/claude/organization.md`, even if the reference repo lags.
- `src/main/java/cloud/cholewa/<package>/` + test skeleton.
- `.github/workflows/CI.yml`, `package.yml`, `sonar.yml` — adjust the Sonar project key
  (`smart-home-automation-system_<name>`). `package.yml` publishes to GitHub Packages on
  a GitHub release (the workflow sets the Maven version from the git tag — poms keep a
  dev version).
- `.gitignore`, `lombok.config` if present; `readme.md` — copy the reference library's
  own `readme.md` (note the lowercase filename convention in libraries), substitute the
  repo name in every badge URL and replace the description line; keep its full layout
  (CI/release badges, language/Java, repo stats) rather than the shorter org-profile
  block. Short `CLAUDE.md`: purpose, consumers, "user codes here himself", release via
  `package.yml`.

Public repo: no secrets, IPs, or private infrastructure details.

## 3. Git and GitHub

Same as for services — ask before each outward-facing step:

1. `git init -b main`, initial commit.
2. `gh repo create smart-home-automation-system/<name> --public --source=. --push`
3. Verify workflow secrets resolve (`SONAR_TOKEN`, `GH_PRV_*`; publishing itself uses the
   automatic `GITHUB_TOKEN`), remind about SonarCloud project import.
4. Remind the user to enable branch protection on `main` (require pull requests) — the
   org rule is PR-only changes from `feature/HAS-<n>` branches; only the initial
   scaffold commit lands on `main` directly.

## 4. Update org docs

Add the library with its consumers to the table in
`organization-repository/claude/organization.md` and a badge section in
`organization-repository/profile/README.md` (or invoke `sync-org-docs`).

## 5. Verify

`mvn -q verify` locally, then a `workflow_dispatch` run of `CI.yml` after push.
