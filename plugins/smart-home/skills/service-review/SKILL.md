---
name: service-review
description: Review Spring Boot service or library code against smart-home-automation-system organization conventions — reactive (WebFlux/Reactor) discipline, reuse of shared libraries, cross-repo contracts (api-gateway routes, RabbitMQ payloads), dependency version consistency and public-repo hygiene. Use whenever the user asks to review changes or a whole repo in this organization ("zrób review", "sprawdź ten serwis", "przejrzyj moje zmiany"), alongside or instead of a generic code review; also before releases.
---

# service-review

Organization-specific review. Generic bug-hunting is `/code-review`'s job and security
scanning is `/security-review`'s — this skill checks what those cannot know: the
conventions and cross-repo contracts of this particular system. When the user asks for a
full review, run this checklist in addition to `/code-review`.

**Do not modify code.** The user writes backend code themselves; deliver findings only
(apply fixes only on an explicit request).

## Setup

Read `organization-repository/claude/organization.md` first (repository map, consumer
table, conventions). Scope: the working diff if there are uncommitted/branch changes,
otherwise the whole repo.

## Checklist

**Reactive discipline** — everything is WebFlux/Reactor; blocking silently poisons event
loops, so treat these as high severity:
- `.block()`, `.blockFirst()`, `.blockLast()`, `.toIterable()`, `.toStream()` outside tests
- blocking clients/IO (RestTemplate, JDBC, `Thread.sleep`, synchronous file IO) on
  reactive paths; `WebClient` and reactive drivers are the norm
- swallowed errors (empty `onErrorResume`, missing error handling on fire-and-forget
  subscriptions), `subscribe()` calls with no error consumer
- misused `Schedulers` (e.g. `boundedElastic` wrapping that hides a blocking call without
  a comment explaining why it is unavoidable)

**Reuse** — code that re-implements something already in `cholewa-commons`,
`smart-home-sdk` or `shelly-client`. Point to the existing type instead.

**Cross-repo contracts** — changes here break other repos at runtime, not at compile time:
- REST paths/DTOs that `api-gateway-service` routes to, or that other services call
- RabbitMQ exchange/queue names and message payload shapes (consumers:
  `gateway-service`, `heating-service`, `notification-service`)
- For a **library**: any breaking public-API change → list the affected consumers from the
  organization.md table and state the required semver bump (breaking → major).

**Version consistency** — `spring-boot-starter-parent` and shared-library versions that
diverge from the rest of the workspace (spot-check 2–3 sibling repos). Report, don't fix.

**Public-repo hygiene** — secrets, tokens or credentials in committed files. These belong
in `deployment-tools` (private) or in env/config outside git. Note: private LAN IPs in
service configs are an accepted risk per `organization.md` — do not report them.

**Tests** — new logic without tests; reactive code tested without `StepVerifier` or
`WebTestClient` where those fit naturally.

## Output

Findings ranked by severity, each with `file:line`, a one-sentence problem statement and
a concrete suggestion. If a finding affects other repos, name them explicitly. End with
anything checked and found clean (one line), so the user knows what was covered.
