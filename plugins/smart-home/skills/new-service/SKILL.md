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

Derive the **local port**: read the service table in
`organization-repository/claude/organization.md` and take the next free `60xx` port.
In-cluster every service listens on `6200` (management `9200`) — do not change that.

## 2. Scaffold from the reference

Copy from `water-service/` and adapt (names, packages, ports, only the requested deps):

- `pom.xml` — artifactId/name/description; keep the `<repositories>` block exactly as in
  the reference; drop unneeded dependencies. **Versions**: set `spring-boot-starter-parent`
  and `java.version` to the org **target toolchain** from the Conventions section of
  `organization-repository/claude/organization.md` — the reference repo may still be on
  older versions awaiting migration; new services always start on the target.
- `src/main/java/cloud/cholewa/<domain>/` — application class + minimal package layout;
  `src/test/...` — matching test skeleton (context-loads test at minimum).
- `src/main/resources/application.yaml` — `spring.application.name`, server port `6200`,
  a `local` profile with the assigned `60xx` port, mirroring the reference structure.
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
5. Remind the user to enable branch protection on `main` (require pull requests) — the
   org rule is PR-only changes from `feature/HAS-<n>` branches; only the initial
   scaffold commit lands on `main` directly.

## 4. Update org docs

Add the service to the map in `organization-repository/claude/organization.md` (with its
port) and to `organization-repository/profile/README.md` (badge section). The
`sync-org-docs` skill does exactly this — follow its steps or invoke it.

## 5. Verify

- `mvn -q verify` in the new repo must pass locally.
- After push, trigger `CI.yml` via `workflow_dispatch` (`gh workflow run CI.yml`) and
  confirm it goes green; report the result to the user.
