---
name: quality-gate
description: Quality gate for a Jira task in the HAS project — fetch the task's Definition of Done, judge which items actually apply at this stage, verify every applicable item against reality (merged PRs, CI, releases, repo files, org docs) and report pass/fail with evidence; after user confirmation, tick verified checkboxes in Jira and leave a gate report comment. Use when the user wants to know whether a task is really done, verify or close a task, or says "sprawdź DoD", "czy HAS-x jest skończony", "bramka jakości", "quality gate HAS-123".
---

# quality-gate

DoD checklists are written when a task is planned; reality decides which items still make
sense and whether they were actually done. This gate does three things: **judge
applicability**, **verify with evidence**, **report honestly**. It never marks anything
done in Jira without user confirmation and never transitions task status — that stays
manual.

Jira site/credentials: as in `jira-backlog` (`<workspace-root>/.env`); generic API
templates in `../jira-backlog/references/jira-api.md`, gate-specific ones (reading and
ticking ADF checkboxes, comments) in `references/dod-verification.md`.

## Workflow

**1. Identify the task.** Task key from the user (or the current `feature/HAS-<n>`
branch). Fetch summary, status, epic, description; extract the DoD task list (items +
TODO/DONE states) and the implementation details. The repo comes from the title prefix
(`repo-name: ...`).

**2. Applicability pass.** For each DoD item decide: **applicable**, **N/A at this
stage** (with a one-line why), or **not automatically verifiable** (needs the user).
Judge from what the task actually changed, not from the template. Canonical examples:

- `org docs synced via sync-org-docs` — applies only when the task changed something the
  org docs track: a repo added/renamed/archived, consumer lists, ports. A routine
  release or an internal migration changes none of that → N/A. (A rename task like
  amx-service → very much applicable.)
- `README updated via update-readme` — applies only when API/behavior/badges really
  changed. A toolchain migration does change the Java badge → applicable; a pure
  refactor → N/A.
- `released and deployed` — N/A for docs-only or repo-administration tasks.
- PR from `feature/HAS-<n>` — applicable whenever any repo content changed (org rule).

**3. Verify applicable items — evidence, not declarations.** Read-only checks; prefer
remote evidence (survives machine changes) over local state:

- **PR**: merged PR with head `feature/HAS-<n>` exists (`gh pr list --state merged`),
  squash-merged into `main`.
- **Build green**: latest CI run on `main` after the merge commit (`gh run list`);
  fall back to local `mvn verify` when CI is ambiguous.
- **Reviews**: evidence that `service-review`/`/code-review` happened (PR description or
  comments mention findings/clean result). No evidence → report as unverified and offer
  to run the review now on the merged diff — late review beats no review.
- **Release**: `gh release list` tag matching the change; artifact exists (Docker Hub
  manifest / GitHub Packages version).
- **Local run**: verifiable — start the service with the `local` profile
  (`mvn spring-boot:run -Dspring-boot.run.profiles=local` or the repo's documented
  command) and confirm it boots without errors; stop it afterwards.
- **Deployment**: verifiable — the workspace machine has `kubectl` with the
  `kind-smart-home-backend` context and the `smart-home` namespace. Run
  `kubectl -n smart-home rollout status deployment/<repo>`, then confirm the pod is
  Running with 0 restarts and that its container `imageID` digest **matches the digest of
  the released tag on Docker Hub** — that is what proves the cluster runs the artifact
  this task released, not a stale image. Finish with `kubectl logs deployment/<repo>` to
  confirm the started version and a clean startup. Only if the cluster is unreachable,
  fall back to asking the user. Verification is read-only; never `apply`, restart or scale
  anything as part of the gate.
- **Docs claims**: open the actual files — e.g. after a Java 21 migration the README
  badge must say 21, `organization.md` tables must reflect a rename.

**4. Report.** A table: DoD item | verdict (`DONE` / `MISSING` / `N/A` / `ASK`) |
one-line evidence or reason. Then the verdict: **GATE PASSED** (every applicable item
done) or **GATE FAILED** with the concrete list of what remains and offers to do the
parts Claude can (run reviews, update readme, sync docs).

**5. Update Jira (only after user confirmation).** Tick the verified items (taskItem
`state: DONE` in the description ADF), leave N/A items unticked, and add a comment with
the gate report including the N/A justifications — so the task history shows why boxes
stayed empty. Never change the task status.
