# claude-tooling

Claude Code plugin marketplace for the
[smart-home-automation-system](https://github.com/smart-home-automation-system) organization.

Contains the `smart-home` plugin with skills shared across all repositories:

| Skill | Purpose |
|---|---|
| `new-service` | Scaffold a complete new Spring Boot microservice (pom, Dockerfile, workflows, README, CLAUDE.md, GitHub repo) |
| `new-library` | Scaffold a new shared Maven library (GitHub Packages release flow) |
| `service-review` | Review code against organization conventions (reactive rules, shared libs, contracts, versions) |
| `release` | Guided release checklist for services (Docker Hub) and libraries (GitHub Packages) |
| `sync-org-docs` | Keep `organization.md` and the org profile README in sync with reality |
| `deps-update` | Audit dependency versions across all workspace repos, plan upgrades |
| `update-readme` | Bring a repo README up to org standard: badges, description, API docs |
| `jira-backlog` | Feature description → scoped Jira tasks with DoD checklists, created in the HAS backlog after user acceptance (credentials in `<workspace>/.env`) |
| `jira-sprint` | Monthly sprint management on the HAS board: status, close at ≥7 Done, carry-over into a new element-named sprint, top-up to ≥10 tasks with the user |

## Installation

Once, on any machine:

```
/plugin marketplace add smart-home-automation-system/claude-tooling
/plugin install smart-home@smart-home-tooling
```

For local development of the skills, add the marketplace from the workspace path instead:

```
/plugin marketplace add /path/to/workspace/claude-tooling
```

Enable the plugin globally (all projects) in `~/.claude/settings.json` so the skills are
available in every repository of the workspace.

## Assumptions

The skills assume the standard workspace layout: all organization repositories cloned side
by side, with the shared context in `organization-repository/claude/organization.md`
(see `organization-repository/claude/SETUP.md`).
