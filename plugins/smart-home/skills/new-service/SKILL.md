---
name: new-service
description: Scaffold a complete new Spring Boot microservice for the smart-home-automation-system organization — pom, source skeleton, Dockerfile, CI/release/sonar workflows, README with badges, CLAUDE.md, local port assignment, GitHub repo creation and org docs update. Use whenever the user wants to add or create a new microservice or service for the smart home system ("stwórz nowy serwis", "add a service for X", "new microservice"), even if they only name the domain (e.g. "potrzebuję serwisu do ogrodu").
---

# new-service

Creates a new microservice consistent with every existing one. The single source of truth
for structure is the **`water-service`** repository in the workspace — copy and adapt from
it rather than inventing files, so the new service inherits fixes made to the reference.

Run from the workspace root (the directory containing all repo clones). If started inside
another repo, ask the user to restart from the workspace root or use `/add-dir ..`.

## 1. Gather inputs

Ask only for what cannot be derived:

- **Name**: `<domain>-service` (lowercase, hyphenated).
- **Purpose**: one sentence — it goes into the pom `<description>`, README and CLAUDE.md.
- **Integrations**: which shared libraries it needs (`cholewa-commons`, `smart-home-sdk`,
  `shelly-client`) and whether it uses RabbitMQ.
- **Database**: ask explicitly, every time — *will this service talk to a database?* The
  reference does, so everything it needs for that would otherwise be copied along; the answer
  decides whether the database part of the scaffold (below) is kept or left out. Do not guess
  it from the purpose, and do not keep it "for later": a service that gets a database later
  adds the part in that task.

Derive the **local port**: read the service table in
`organization-repository/claude/organization.md` and take the next free `60xx` port; the
matching Actuator port is `80xx`. In-cluster every service listens on `6200` with Actuator
on `8200` — do not change that.

## 2. Scaffold from the reference

Copy from `water-service/` and adapt (names, packages, ports, only the requested deps):

- `pom.xml` — artifactId/name/description; keep the `<repositories>` block exactly as in
  the reference; drop unneeded dependencies — the database part by the answer above, see
  **The database part**. **Versions**: set `spring-boot-starter-parent`
  and `java.version` to the org **target toolchain** from the Conventions section of
  `organization-repository/claude/organization.md` — the reference repo may still be on
  older versions awaiting migration; new services always start on the target.
- `src/main/java/cloud/cholewa/<domain>/` — application class + minimal package layout;
  `src/test/...` — matching test skeleton (context-loads test at minimum).
- `src/main/resources/application.yaml` — `spring.application.name`, server port `6200`
  with `management.server.port: 8200`, a `local` profile with the assigned `60xx` /
  `80xx` pair, mirroring the reference structure.
- `Dockerfile` — copy verbatim.
- `.github/workflows/CI.yml`, `release.yml`, `sonar.yml` — adjust only: Docker image name
  (`magikabdul/<service-name>` in release.yml) and the Sonar project key
  (`smart-home-automation-system_<service-name>`).
- `.gitignore`, `lombok.config` and any other root config present in the reference.
- `README.md` — copy the **reference service's own `README.md`** (water-service) and
  substitute the repo name in every badge URL, then replace the Description section with
  the new service's purpose and local port. Do not use the shorter badge block from the
  org profile README — repo READMEs have a richer layout (CI/quality-gate/vulnerabilities,
  release info, separator, language/Java/Spring/coverage/LoC, repo stats) that must be
  preserved. Keep the Java/Spring version badges in sync with the pom.
- `CLAUDE.md` — fill in `assets/CLAUDE-template.md`.

The repo will be public: no secrets, tokens, IPs or private infrastructure details in any
file.

### The database part

Everything in the reference that exists only because `water-service` has a database. Read
the reference for the current list — it grows with the reference — but today it is:

- **pom.xml, runtime**: `spring-boot-starter-data-r2dbc`, `spring-boot-starter-flyway`,
  `flyway-database-postgresql`, `spring-jdbc`, `postgresql`, `r2dbc-postgresql`.
- **pom.xml, test**: `spring-boot-starter-data-r2dbc-test`, `spring-boot-starter-flyway-test`
  and the two Testcontainers artifacts (`testcontainers-junit-jupiter`,
  `testcontainers-postgresql`) with the comment above them.
- **application.yaml**: the `spring.flyway` block, the whole `database.*` group (the pool size
  included) and `spring.flyway.enabled: false` in the `test` document.
- **Sources**: `src/main/resources/db/migration/`, the `database` package, and in the tests
  the repository test and the test that pins `database.pool.max-size`.
- **README / CLAUDE.md**: the Database section, the connection properties under "Run
  locally", the note that `mvn verify` needs a running Docker.

**Without a database — leave all of it out.** None of the dependencies above stays in the
pom: a leftover Flyway or R2DBC starter brings auto-configuration that can ask for connection
settings the service does not have, and a leftover Testcontainers dependency — harmless at runtime
(test scope, nothing starts without a test that asks for a container) — is dead weight with
a comment pointing at a test the service does not have. After the scaffold, `mvn -q verify`
must pass and `mvn dependency:tree` must show none of the artifacts above; that is the check
that nothing of the part was left behind.

**With a database — keep all of it, the Testcontainers artifacts included**, and adapt:

- An own database on the managed cluster (`home-automation-<domain>`), created in the task
  that scaffolds the service; the manifest in `deployment-tools` carries the `env` block with
  the `database-*` properties and the Flyway URL, like the reference.
- `database.pool.max-size` is **not copied**: it is the share of the service in the 22
  backend connections of the managed database, and the split is in the Conventions of
  `organization.md`. Propose a share (2 unless the service needs more), re-do the rollout sum
  written there with the user, and pin the number with the test.
- `db/migration` starts with a `V1` of the new service; the migrations, the entity and the
  repository of the reference are not copied. The repository test of the reference is the
  pattern for a PostgreSQL-only statement, not a file to keep: keep the Testcontainers
  dependencies with a comment that names no test, and write the test with the first such
  statement. Until then `mvn verify` needs no Docker; say in CLAUDE.md that it will.

## 3. Git and GitHub

Ask before each outward-facing step — never assume consent for repo creation or pushes:

1. `git init -b main`, initial commit.
2. `gh repo create smart-home-automation-system/<name> --public --source=. --push`
3. Secrets: workflows expect `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `GH_PRV_USERNAME`,
   `GH_PRV_PASSWORD`, `SONAR_TOKEN` (plus automatic `GITHUB_TOKEN`). Check with
   `gh secret list -R smart-home-automation-system/<name>` whether they resolve from the
   organization level; if not, remind the user to add them.
4. Remind the user of the manual step: import the project in SonarCloud (the sonar.yml
   run fails until the project exists there).
5. Repo settings — a fresh repo does not match the org defaults; apply them after the
   scaffold commit is pushed (ask first, it is an outward-facing change):

   ```
   gh api -X PATCH repos/smart-home-automation-system/<name> \
     -f allow_squash_merge=true -F allow_merge_commit=false \
     -F allow_rebase_merge=false -F delete_branch_on_merge=true

   gh api -X PUT repos/smart-home-automation-system/<name>/branches/main/protection \
     --input protection.json
   ```

   with `protection.json` mirroring every other service — `enforce_admins: true`
   (the owner is an admin, so without it the rule protects nobody),
   `required_pull_request_reviews` with `required_approving_review_count: 0` (a
   solo developer cannot approve their own PR), `required_conversation_resolution: true`,
   `allow_force_pushes` / `allow_deletions` false, `required_status_checks: null`.
   The org rule is PR-only changes from `feature/HAS-<n>` branches; only the initial
   scaffold commit lands on `main` directly, so enable protection **after** it.
   Verify with an empty commit: the push must be rejected with `GH006`.

## 4. Update org docs

Add the service to the map in `organization-repository/claude/organization.md` (with its
port) and to `organization-repository/profile/README.md` (badge section). The
`sync-org-docs` skill does exactly this — follow its steps or invoke it.

## 5. Verify

- `mvn -q verify` in the new repo must pass locally; for a service without a database also
  the dependency check of **The database part**.
- After push, trigger `CI.yml` via `workflow_dispatch` (`gh workflow run CI.yml`) and
  confirm it goes green; report the result to the user.
