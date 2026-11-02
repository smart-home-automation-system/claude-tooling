---
name: cleanup-dockerhub
description: Clean up the magikabdul Docker Hub account — scan all repositories and for each keep only the 2 newest version tags plus latest; everything else is presented to the user for acceptance and deleted only after confirmation. Use when the user wants to prune Docker Hub images/tags ("wyczyść docker huba", "posprzątaj obrazy dockera", "usuń stare tagi z docker hub", "cleanup docker hub").
---

# cleanup-dockerhub

Prune ALL repositories of the `magikabdul` Docker Hub account: per repository keep the
**2 newest version tags + `latest`**, delete the rest — but only after showing the full
deletion plan and getting the user's acceptance. `latest` is always kept, in every repo.

## 1. Credentials

From `<workspace>/.env` (same convention as `jira-backlog`):

```
DOCKERHUB_USERNAME=magikabdul
DOCKERHUB_TOKEN=<personal access token>
```

The PAT needs **Read, Write, Delete** permissions (hub.docker.com → Account Settings →
Personal access tokens). If the keys are missing, tell the user exactly that and stop —
never ask for the token in chat.

Claude never reads `.env` itself (the auto-mode classifier refuses it): every call goes through
`organization-repository/claude/scripts/dockerhub-api.sh`, run from the workspace root, which
loads the file, logs in (the PAT in the `password` field) and passes the JWT to `curl` without
printing either (SETUP.md, section 4a):

```
bash organization-repository/claude/scripts/dockerhub-api.sh 'repositories/<user>/?page_size=100'
bash organization-repository/claude/scripts/dockerhub-api.sh - [curl args] < paths.txt   # one login, one call per line
```

`<user>` in a path is replaced by the account name. There is no `jq` on the machine: keep the
answers in the scratchpad (one JSON body per line) and parse them with PowerShell
(`ConvertFrom-Json`) or `node`. A path list written by PowerShell has CRLF line endings — pass it
through `tr -d '\r'` before the script reads it, or curl rejects every URL and nothing happens.

## 2. Inventory (read-only)

- Repositories: `GET https://hub.docker.com/v2/repositories/<user>/?page_size=100`
  (follow `next` until null).
- Tags per repository:
  `GET https://hub.docker.com/v2/repositories/<user>/<repo>/tags/?page_size=100`
  (follow `next`; note `name`, `last_updated`, `digest`).

Per repository compute:

- **Keep set** = `latest` + the 2 newest version tags (semver descending; cross-check
  against `last_updated` — on a suspicious mismatch flag it instead of trusting either
  order blindly).
- **Delete set** = every other tag. Non-version tags (branch names, `dev`, sha…) go into
  the delete set but must be **flagged separately** in the plan.
- Repos with ≤2 version tags: nothing to delete — list them as "already clean".

## 3. Deployment safety check (read-only)

The k8s manifests in `deployment-tools` pin `magikabdul/<service>:<tag>`. Grep them for
every tag in the delete set; any tag still referenced by a manifest is **excluded from
deletion** and flagged — deleting it would break image re-pulls and rollbacks on the
cluster. The user may still delete it, but only by explicit decision.

## 4. Present the plan and get acceptance

One table per repository: kept tags (with dates) and tags to delete, flagged entries
(non-version tags, manifest-referenced tags) clearly marked. Then ask for acceptance
(AskUserQuestion) — nothing is deleted before the user accepts, and tag deletion on
Docker Hub is irreversible. With many repos, offer per-repo acceptance as an option.

## 5. Delete

The accepted tags as one path per line (`repositories/<user>/<repo>/tags/<tag>/`), then one call:

```
bash organization-repository/claude/scripts/dockerhub-api.sh - -X DELETE -o /dev/null -w '%{http_code} %{url}' < delete-paths.txt
```

204 = success; count the statuses before trusting the result. On an error report the repo/tag and continue with the rest — no blind
retries. Untagged images are garbage-collected by Docker Hub itself; nothing more to do.

## 6. Verify and report

Re-list tags of every touched repository: exactly the keep set should remain. Report per
repo: deleted count, surviving tags, anything excluded, flagged or failed.
