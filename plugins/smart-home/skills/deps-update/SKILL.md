---
name: deps-update
description: Audit dependency and toolchain versions across every repo of the smart-home-automation-system workspace — Spring Boot parent, Spring Cloud, shared cloud.cholewa libraries, Java versions (pom vs Dockerfile), key plugins — and produce a divergence report plus an ordered upgrade plan. Use when the user wants to bump Spring Boot or any dependency across services, asks which versions are in use, or suspects version drift ("podbij springa", "jakie wersje mamy w serwisach", "upgrade dependencies", "sprawdź zależności").
---

# deps-update

Solo-maintained multi-repo means version drift accumulates silently — ten services bump at
ten different moments. This skill makes the drift visible and turns a bump into an ordered
plan instead of ten ad-hoc edits.

**Default mode is report + plan.** Apply pom changes only when the user explicitly asks
(the user maintains backend code themselves).

Run from the workspace root.

## 1. Collect (all repos with a pom)

For each repo record:
- `spring-boot-starter-parent` version; Spring Cloud BOM version if present
- versions of consumed `cloud.cholewa` libraries (commons, sdk, shelly-client, …)
- `java.version` from the pom **and** the JDK image version in `Dockerfile` — these can
  legitimately differ (compile vs runtime), but the divergence should be a conscious org
  decision, not an accident; flag mismatches across repos either way
- workflow toolchain: `java-version` in `.github/workflows/*.yml`
- for the frontend repo: Angular major + Node version in CI

Parallelize with subagents if the workspace is large; otherwise plain grep is fine.

## 2. Report

A matrix (rows = repos, columns = the versions above) with divergent cells called out,
followed by:
- the version the majority is on vs the newest in the workspace vs the newest available
  (check Maven Central for the Spring Boot line the org uses)
- anything pinned by a constraint (e.g. a library holding a service back)

## 3. Upgrade plan (when a bump is wanted)

Order matters because services consume the libraries via GitHub Packages:

1. Libraries first (`cholewa-commons`, `cholewa-security`, `smart-home-sdk`,
   `shelly-client`) — bump, verify `mvn verify`, release via the `release` skill flow.
2. Then services, consuming the fresh library versions — grouped so RabbitMQ/API contract
   partners move together when a contract is affected.
3. Frontend and workflows (Node/Java versions in CI) last.

For each step list: repo, edit, verification command, release needed or not. Present the
plan; execute only the parts the user asks for.
