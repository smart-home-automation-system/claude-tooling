# quality-gate — verification and Jira update templates

Auth as everywhere: Jira through `bash organization-repository/claude/scripts/jira-api.sh <path below /rest/>
[curl args]`, run from the workspace root (it loads `JIRA_EMAIL` / `JIRA_API_TOKEN` from
`<workspace-root>/.env` and never prints them — never source the `.env` yourself); `gh` CLI
(GitHub).

## Read the DoD with checkbox states

```bash
bash organization-repository/claude/scripts/jira-api.sh 'api/3/issue/HAS-<n>?fields=summary,status,parent,description'
```

DoD = the `taskList` node in `fields.description.content`; each `taskItem` has
`attrs.localId` and `attrs.state` (`TODO`/`DONE`) and text in `content[0].text`.

## Evidence commands (read-only)

```bash
# merged PR from the convention branch
gh pr list -R smart-home-automation-system/<repo> --state merged \
  --search "head:feature/HAS-<n>" --json number,title,mergedAt,mergeCommit

# CI on main after the merge
gh run list -R smart-home-automation-system/<repo> --branch main --limit 5 \
  --json name,conclusion,headSha,createdAt

# release + artifacts
gh release list -R smart-home-automation-system/<repo> -L 3
docker manifest inspect magikabdul/<repo>:<version> >/dev/null && echo "image exists"
gh api /orgs/smart-home-automation-system/packages/maven/<group.artifact>/versions --jq '.[].name'

# k8s rollout (the current kubectl context - `home` on the owner's machine; namespace smart-home;
# pods are labelled name=<repo>)
kubectl -n smart-home rollout status deployment/<repo> --timeout=30s
kubectl -n smart-home get pods -l name=<repo> \
  -o jsonpath='{.items[*].status.containerStatuses[*].imageID}{"\n"}{.items[*].status.containerStatuses[*].restartCount}'
kubectl -n smart-home logs deployment/<repo> --tail=40
```

For `cholewa-commons`/`cholewa-security` substitute the personal account
(`-R magikabdul/<repo>`, packages under `/users/magikabdul/packages/...`).

The `imageID` digest must equal the digest of the released Docker Hub tag — that is the
check that the cluster runs *this* release. If the `docker` CLI is unavailable (it often is
on WSL), read the digest from the Hub API instead: log in at
`https://hub.docker.com/v2/users/login/` with `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` from
`<workspace-root>/.env` and GET
`https://hub.docker.com/v2/repositories/magikabdul/<repo>/tags`.

## Tick verified checkboxes

GET the full description ADF, set `attrs.state = "DONE"` on the verified `taskItem`s
(match by text, not position), PUT back the whole description:

```bash
bash organization-repository/claude/scripts/jira-api.sh api/3/issue/HAS-<n> -o /dev/null -w "%{http_code}" \
  -X PUT -H "Content-Type: application/json" \
  -d @updated-description.json   # {"fields": {"description": <full modified ADF>}}
```

Through API v3 only — v2 flattens the checkboxes of the DoD.

Modify only checkbox states — do not touch other content. Verify with a follow-up GET.

## Gate report comment

```bash
bash organization-repository/claude/scripts/jira-api.sh api/3/issue/HAS-<n>/comment -X POST \
  -H "Content-Type: application/json" -d @comment.json
```

`comment.json` body is ADF: heading "Quality gate report" + a bulletList with one line
per DoD item (`DONE — evidence`, `N/A — reason`, `MISSING — what remains`), ending with
the PASSED/FAILED verdict paragraph.
