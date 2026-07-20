---
name: update-readme
description: Bring a smart-home-automation-system repository README up to the org standard — badge block, accurate description, local run info, and short API documentation generated from the actual controllers (services) or an installation/usage section (libraries). Use when the user asks to update, fix or write a README, add badges, or document a service's API ("popraw readme", "dodaj badge", "udokumentuj API w readme", "opisz ten serwis").
---

# update-readme

READMEs are the public face of each repo and they rot: badges missing, descriptions
outdated, endpoints undocumented. This skill regenerates a README from the actual state
of the code — never invent an endpoint or capability that the code doesn't show.

Works on the current repo (or the one the user names). Detect the type first: service
(`release.yml`/`Dockerfile`) vs library (`package.yml`).

## Standard structure — service

```markdown
# <repo-name>
<badge block>
One-paragraph description (what it controls/integrates, its role in the system).
## Run locally
Build (`mvn verify`), local profile port `60xx`, in-cluster port 6200.
## API
Endpoint table generated from code.
## Messaging          <- only if the service uses RabbitMQ
Queues/exchanges consumed and produced, with payload types.
```

- **Badge block**: copy from this repo's existing README if present; otherwise from this
  repo's entry (or a sibling's) in `organization-repository/profile/README.md`,
  substituting the repo name in every URL. Standard set: CI workflow badge, SonarCloud
  quality gate (`smart-home-automation-system_<repo>`), top language, last commit,
  release date, release version.
- **API table**: read `@RestController`/`@RequestMapping` classes and `RouterFunction`
  beans; produce `| Method | Path | Description |` rows. Derive descriptions from method
  names, javadoc and DTOs — keep them one line. Include the gateway-facing base path if
  the service is routed through `api-gateway-service`.
- **Messaging**: find Rabbit listeners/templates, list queue/exchange names and payload
  classes.

## Standard structure — library

```markdown
# <repo-name>
<badge block — library set (no local port)>
One-paragraph description + current consumers (from organization.md).
## Installation
Maven dependency snippet with current released version (gh release list) and the
GitHub Packages <repositories> block from a consumer pom.
## Usage
Short example of the main public API (from the actual classes).
```

## Rules

- Preserve any valuable hand-written content — merge, don't clobber; show the user a
  summary of what changed.
- The released version comes from `gh release list`, never from the pom — poms
  intentionally keep a dev version (e.g. `0.0.1-SNAPSHOT`); release workflows set the
  real version from the git tag. Do not flag this mismatch as a problem.
- Org rule: changes land via PR, never directly on `main`. Branch name `feature/HAS-<n>`
  (the Jira task number); when no task covers this README work, confirm with the user
  before branching.
- Public repos: no secrets, private hosts or IPs.
- If the API surface is large, document the main resources and link to the code rather
  than exhaustively listing every endpoint.
