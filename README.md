# Rivarc

Self-hosting open-source software is hard: the relationships between apps,
databases, storage, network access and credentials are scattered across config
files. Rivarc puts them in one graph users can see and edit.

- **Connecting** two nodes configures both and gives the connection its own
  isolated network and credentials.
- **Disconnecting** removes them, so access is cut.
- **Infrastructure is a node too.** Moving an app's storage from your laptop to a
  hosting provider means reconnecting one node; Rivarc migrates the data.

Rivarc works with existing software as it is. Anyone can write a manifest for an
app without its maintainers changing any code.

## Status

**Early design phase. Not usable yet.** This repository holds the design work for v0:
decision records, draft schemas and example manifests. Code will follow here.

The earlier prototype ([CyanStack](https://github.com/rivarc-org/CyanStack),
[cyan-ui](https://github.com/rivarc-org/cyan-ui)) is archived. It showed the idea
works and that the contracts need careful design before anything is built on them.

## Planned for v0

- **Manifest and graph format:** what an app needs and provides, and how nodes connect
- **Runtime:** reconciles a single Linux host to the graph using Podman
- **Connection-scoped access:** per-connection networks and credentials, revoked on disconnect
- **Storage rebinding:** move an app's data to another storage node, with verification and rollback
- **Management UI:** see and edit the graph; the UI is itself a node in the graph
- **Real-world manifests** for 4–5 unmodified open-source apps (e.g. Immich, Medusa)
- **Spec v0**, written from the implementation, with its limits and open questions stated plainly

Later: remote access, an SDK for apps that want finer-grained capabilities,
multi-host execution, WASI and a conformance suite.

## How decisions are made

Design decisions are recorded in [`decisions/`](decisions/) as short decision
records: the problem, the options considered, and what was chosen and why.
Feedback is welcome on any of them. Open an issue or reply to the relevant discussion.

## Get involved

Rivarc especially needs perspectives from:

- **Maintainers** of self-hosted apps: does a manifest describe your software correctly?
- **Self-hosters:** which setups would you want to express as a graph?
- **Hosting providers:** what would it take to offer Rivarc-compatible infrastructure?

Contact: rivarc@bluda.tech or [GitHub Discussions](https://github.com/rivarc-org/rivarc/discussions)

## Licences

- Software: Apache-2.0
- Specification and documentation: CC BY 4.0
- Example manifests and graphs: CC0