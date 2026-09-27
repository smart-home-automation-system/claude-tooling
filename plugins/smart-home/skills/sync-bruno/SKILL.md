---
name: sync-bruno
description: Keep the user's Bruno API collection (D:\OneDrive\Bruno\home) in sync with the smart-home services — add requests for new endpoints, fix requests whose path, method, params or body drifted from the code, flag requests for removed endpoints, add a folder for a new service — always following the collection's existing conventions and always showing every change in the conversation for the user's approval before writing a single file. Use whenever the user mentions Bruno, the Bruno collection or .bru files, asks to add/update/fix requests for a service or endpoint, or after an API change in a service ("zaktualizuj Bruno", "dodaj request do Bruno", "zsynchronizuj kolekcję", "brakuje endpointu w Bruno", "update the bruno collection").
---

# sync-bruno

The Bruno collection at `D:\OneDrive\Bruno\home` is the user's hand tool for calling the
services, locally and on the cluster. It drifts every time an API changes: new endpoints
have no request, renamed paths keep the old URL, bodies stay copied from another endpoint.
This skill brings it back in line with the code — the controllers are the source of truth,
the collection's own conventions decide how a request looks.

**The one hard rule: nothing is written to the collection without the user's explicit
approval of that exact change in the conversation.** The collection lives outside git (on
OneDrive), so there is no review or undo step after a write — the conversation is the only
review. Show first, write after "tak"/"ok"/"akceptuję"; if the user accepts only part of the
proposal, write only that part.

## 1. Read the collection first, every time

Read `references/collection-standard.md` (the conventions, with templates), then look at
the current state of the collection: the folder tree, the `folder.bru` of the service
folder you will touch, and every request in it. The standard describes what was true when
this skill was written; the files are what is true now — if they disagree, follow the files
and mention the difference to the user.

Never print secret values you come across (tokens in environment or header values) — refer
to variables by name.

## 2. Build the list of endpoints from the code

For each service in scope (the one the user names, or the one just changed in this
conversation), read its controllers — `@RestController` classes with their
`@RequestMapping`/`@GetMapping`/… annotations, `@PathVariable`, `@RequestParam` and
`@RequestBody` types — plus `spring.webflux.base-path` in `application.yaml` (it is `/home`
everywhere today). For each endpoint note: method, full path, path variables, query
parameters (required or not), body type and its fields (for `smart-home-sdk` models read
the generated class or the OpenAPI schema in `smart-home-sdk/swagger/`), and what it
returns. The repo's README API table helps, but the code wins when they differ.

## 3. Compare and draft the proposal

Match endpoints to requests by method + path (ignoring the base URL and port variables).
Sort the differences into:

- **New** — an endpoint without a request → a new `.bru` file.
- **Changed** — a request whose method, path, path/query params, body shape or field names
  no longer match the endpoint (typos in field names count; a body copied from another
  endpoint counts) → a modification.
- **Orphaned** — a request for an endpoint that no longer exists → propose deletion, but
  only as a question; the user may keep it on purpose.
- **Structure** — a new service needs its folder (`folder.bru` with the port script) and,
  if the service is new to the collection, its local port from the org conventions.

Example values must pass the endpoint's validation (e.g. an E.164 phone, a lowercase
colon-separated MAC, a 3–50 character name). **Values that already exist in a request are the
user's — keep them, real data included**; when one no longer passes validation, change only
its format and keep the value (`696027072` → `+48696027072`), never swap it for another.
**Values you introduce** — in a new request, or a new field or path variable in an existing
one — are obviously fictional (`Test`, `+48500000001`, `02:00:00:00:00:01`), so that running a
new request cannot touch a real record by accident.

## 4. Show the proposal and wait

Present it in the conversation in this shape, then stop and ask for approval:

```
## Bruno: <service> — proposed changes

| # | Change | File | Why |
|---|---|---|---|
| 1 | new | database/household/activateHousehold.bru | POST /home/household/member/{name}/activate has no request |
| 2 | changed | database/household/addHousehold.bru | field "phpne" → "phone", phone to E.164 |
| 3 | delete? | … | endpoint removed in … |

### 1. database/household/activateHousehold.bru (new)
<the full file content in a ```bru block>

### 2. database/household/addHousehold.bru (changed)
<a diff (```diff) of the old and new content, or the full new content when most of it changes>
```

Number the changes so the user can accept selectively ("1–3 tak, 4 nie"). If the user asks
for adjustments, show the adjusted version again before writing.

## 5. Write only what was accepted

Write the accepted files exactly as shown (Write/Edit on the paths under
`D:\OneDrive\Bruno\home`), delete only the accepted deletions, and then list what was
written. Do not touch anything that was not in the accepted proposal — environments,
other folders, `seq` numbers of untouched requests. If writing reveals something the
proposal did not cover (e.g. a `seq` clash), stop and show the extra change first.

## When this runs as part of other work

After a service task changes an API (new endpoint, changed path or body), offer this sync
at the end of the task — it is how the collection stays current. The approval rule still
applies: propose, wait, write.
