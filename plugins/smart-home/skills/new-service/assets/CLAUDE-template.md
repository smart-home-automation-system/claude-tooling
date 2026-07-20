# <service-name>

<One-paragraph purpose of the service.>

Part of the smart-home-automation-system organization — org-wide conventions, the
repository map and working rules come from the workspace-level context
(`organization-repository/claude/organization.md`). The user writes the code in this
repository themselves; Claude's default role here is analysis, code review and security
review.

## Role in the system

- Talks to: <api-gateway routes? RabbitMQ queues? other services?>
- Uses libraries: <cholewa-commons / smart-home-sdk / shelly-client>
- Integrates with hardware: <Shelly / AMX / none>

## Build & run

- Build + tests: `mvn verify`
- Local run: `local` Spring profile, port `<60xx>`; in-cluster port `6200`
  (management `9200`).

## Specifics

<Anything non-obvious: quirks of the domain, external APIs, scheduled jobs, etc.>
