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

Login (PAT goes in the `password` field), the JWT is used for all later calls:

```
curl -s -X POST https://hub.docker.com/v2/users/login \
  -H "Content-Type: application/json" \
  -d '{"username": "'$DOCKERHUB_USERNAME'", "password": "'$DOCKERHUB_TOKEN'"}'
# → {"token": "<jwt>"}
```

Never print the token or the JWT into the conversation; keep them in shell variables.

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

For each accepted tag:

```
curl -s -X DELETE -H "Authorization: Bearer $JWT" \
  https://hub.docker.com/v2/repositories/<user>/<repo>/tags/<tag>/
```

204 = success. On an error report the repo/tag and continue with the rest — no blind
retries. Untagged images are garbage-collected by Docker Hub itself; nothing more to do.

## 6. Verify and report

Re-list tags of every touched repository: exactly the keep set should remain. Report per
repo: deleted count, surviving tags, anything excluded, flagged or failed.
