---
name: jira-backlog
description: Turn a feature description into well-scoped Jira tasks and, after explicit user acceptance, create them in the backlog of the HAS project (magikabdul.atlassian.net) with epic assignment, implementation details and a Definition-of-Done checklist. Use whenever the user describes functionality to plan, wants a feature broken down into tasks, or asks to add work to the backlog/Jira ("rozpisz funkcjonalność", "przygotuj taski", "dodaj do backlogu", "zaplanuj ten feature", "create Jira tasks for X").
---

# jira-backlog

Feature description in → reviewed task drafts → (acceptance) → issues in the HAS backlog.
The user handles everything after creation (sprints, transitions, closing) manually in
Jira — this skill only plans and creates.

Jira site: `https://magikabdul.atlassian.net`, project key `HAS`, board 3.
API details and ready curl templates: read `references/jira-api.md` before calling the API.

## Credentials

Read `<workspace-root>/.env` (the workspace directory containing all repo clones — this
file lives outside every git repo on purpose; it is shared by any skill needing secrets):

```
JIRA_EMAIL=<atlassian account email>
JIRA_API_TOKEN=<API token from id.atlassian.com → Security → API tokens>
```

Never print the token. If the file or variables are missing, stop and tell the user how
to create the token and the file. Verify auth with the `myself` endpoint before doing
anything else.

## Workflow

**1. Understand the feature.** If it touches specific repos, ground yourself in reality:
read the relevant code, `CLAUDE.md`s and `organization-repository/claude/organization.md`
(service map, ports, contracts) so task details reference actual endpoints, queues,
classes and conventions — not guesses. Ask the user about anything unclear or any scope
decision they need to make; do not invent requirements.

**2. Pick the epic.** Every feature must belong to an epic, and the epic represents the
**feature being implemented, not a system domain** — name it after the functionality
(e.g. "Room temperature scheduling", not "Heating"). Fetch open epics from HAS first:
reuse one only when it describes this same feature (a continuation of earlier planning);
otherwise propose a new feature-named epic (name + one-line goal). Ignore broad
domain-style epics as targets for new features. New epics are created only as part of
the accepted plan.

**3. Draft the tasks.** All content in **English**. Sizing rule: each backend task must
be implementable by the user in roughly **one day or less** — split anything bigger.
Frontend tasks (`web-application`) are exempt from the size limit because Claude
implements them; give them the label `frontend` and note in the description that
implementation follows the web-application workflow (feature branch → PR → user review).

Issue type is always **Task**. Each draft contains:

- **Title**: imperative and specific, prefixed with the repo, e.g.
  `heating-service: expose schedule API for room temperature profiles`.
- **Context** — why this task exists, 1–3 sentences, link to the feature.
- **Implementation details** — everything needed to start coding: affected repo and
  modules, endpoints/queues with payload shapes, config keys, edge cases, relations to
  other tasks of the feature ("blocked by", "provides contract for").
- **Definition of Done** — checkboxes. Always include the applicable standard items plus
  feature-specific acceptance criteria:
  - implementation complete, `mvn verify` green (backend) / build + tests green (frontend)
  - `service-review` + `/code-review` run on the changes (backend); PR opened with
    self-review and screenshots, reviewed and merged by the user (frontend)
  - documentation updated: repo README via `update-readme` when API/behavior changed,
    `CLAUDE.md` when conventions changed
  - when the task ends in a release: release performed per the `release` skill checklist
    and org docs refreshed via `sync-org-docs`

**4. Get acceptance.** Present all drafts (titles, full descriptions, DoD, epic
assignment) in the conversation. Iterate on feedback. **Nothing is created in Jira until
the user explicitly accepts** — acceptance of the plan is the trigger, task by task or as
a whole, as the user prefers.

**5. Create in Jira.** New epic first (if any), then tasks: parent = epic, label
`frontend` where applicable, description as ADF with a real checkbox task-list for the
DoD (templates in `references/jira-api.md`). New issues land in the backlog by default —
do not assign sprints, statuses or people. Finish with a table of created issue keys and
URLs, and verify each returned key with a GET.
