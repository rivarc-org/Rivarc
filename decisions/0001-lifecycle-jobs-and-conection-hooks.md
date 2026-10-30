# 0001: Manifests describe lifecycle jobs and connection hooks, not just services

- **Status:** Proposed
- **Date:** 2026-10-30
- **Test case:** Medusa v2
- **Discussion:** https://github.com/medusajs/medusa/discussions/17501#discussioncomment-19197059

## Context

The prototype ([CyanStack](https://github.com/rivarc-org/CyanStack)) treated an app as one
long-running service. Connections only injected environment variables, plus an optional
seed command. That ran a demo, but it doesn't describe how production software is deployed.

To test the model, I wrote down how Rivarc would deploy Medusa v2 and posted it on
Medusa's GitHub Discussions. A community member replied; Medusa's maintainers have not
reviewed it yet. Together, my draft and the reply show what a Medusa deployment needs:

1. **One image, several roles.** The same backend image runs twice: once as the API and
   admin server, once as a background worker (`MEDUSA_WORKER_MODE=server` / `worker`).
2. **Ordered one-off jobs.** Database migrations must run to completion before the server
   or worker starts, on install and again on upgrade. Otherwise both may start against an
   incomplete schema.
3. **Bootstrap steps.** An initial admin user must exist before anything can call the
   Admin API. *(From the community answer; to be verified against Medusa's docs.)*
4. **Requirements that depend on topology.** Redis is optional for a single process, but
   required once server and worker run separately, since events and workflows must
   travel between them. *(From the community answer; to be verified.)*
5. **Connections that create resources inside the app.** Connecting a storefront means
   creating a publishable API key through the Admin API, linking it to a sales channel,
   and injecting it into the storefront. Disconnecting must revoke that key. An
   environment variable alone cannot express this.
6. **Version-specific commands.** The prototype's manifest used a migration command from
   an older Medusa CLI. Lifecycle commands belong to a specific app version.

An open point from the reply, not yet verified: Medusa's migrations may need Postgres
extensions that only a privileged role can create. If true, some provisioning must run
with more rights than the app's own database role.

## Options considered

**A. Environment variables only (the prototype's model).**
Simple, but cannot express points 2–5. Every missing step falls back to the user, which
is the problem Rivarc exists to solve.

**B. App-specific runtime code, like Juju charms or operators.**
Can express anything, but every app needs code someone must write and maintain, inside
or alongside the runtime. That brings back the bottleneck Rivarc avoids, where
community members can describe an app without writing software for it.

**C. Declarative lifecycle jobs and connection hooks in the manifest.**
The manifest lists roles, ordered one-off jobs and hooks that run when a connection is
made or removed. Hooks run commands or requests against the app using credentials the
runtime holds.

## Decision

**Option C.** A v0 manifest can declare:

- **Roles:** several long-running services from one image, each with its own settings.
- **Jobs:** one-off commands with triggers (`install`, `upgrade`) and ordering
  (`before` / `after` roles or other jobs). A failed job stops the deployment.
- **Connection hooks:** `on_connect` and `on_disconnect` actions per connection type.
  Whatever `on_connect` creates, `on_disconnect` must remove. A connection type without
  a working revoke hook is invalid.
- **Version scope:** a manifest names the app versions it was tested against.

**Sub-decision: no conditional requirements in v0.** Rather than "Redis is required only
when roles are split", the v0 Medusa manifest always runs server and worker separately,
so Redis is always required. This keeps manifests free of conditional logic until real
cases show it's needed.

Illustrative sketch, not final syntax:

```yaml
id: com.medusajs.backend
versions: ">=2.0 <3.0"

roles:
  server: { env: { MEDUSA_WORKER_MODE: server } }
  worker: { env: { MEDUSA_WORKER_MODE: worker } }

requires:
  database: { kind: postgres, bind: { DATABASE_URL: url } }
  cache:    { kind: redis,    bind: { REDIS_URL: url } }

jobs:
  migrate:
    run: ["medusa", "db:migrate"]
    on: [install, upgrade]
    before: [server, worker]
  bootstrap-admin:
    run: ["medusa", "user", "-e", "{{secrets.admin_email}}", "-p", "{{secrets.admin_password}}"]
    on: [install]
    after: [migrate]

provides:
  storefront-api:
    on_connect:    create a publishable key via the Admin API, link it to a sales channel, inject it into the peer
    on_disconnect: revoke that key
```

## Consequences

- **Good:** Medusa can be described without changing its code, and disconnecting a
  storefront actually removes its access instead of leaving a key behind.
- **Good:** The ordering rules give the runtime what it needs to handle install,
  upgrade and failure consistently across apps.
- **Cost:** Hooks run with elevated credentials (here, an Admin API secret). The runtime
  must hold these and never expose them to connected nodes.
- **Cost:** Manifests become longer and tied to app versions, so they need maintenance.
- **Risk:** If many apps need unusual hooks, manifests may slide toward option B. The
  4–5 test apps will show whether that happens; the spec will record any such limit
  rather than hide it.

## Open questions

1. How are hooks expressed: a command run in a one-off container of the app's image, an
   HTTP request, or both?
2. What happens when `on_disconnect` fails, for example because the app is down? Retry
   later, block the disconnect, or cut the network and revoke on next start?
3. Should privileged provisioning (e.g. creating Postgres extensions) be a property of
   the database connection, run by the database node with its own rights?
4. Where do admin credentials for hooks live, and how are they rotated?
5. Do conditional requirements ever become necessary, or does fixing the topology per
   manifest stay enough?