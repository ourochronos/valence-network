# Protocol Evolution

## Principle

The protocol is not sacred. It evolves through the same proposal mechanism it provides. But it must never break — nodes that fall behind get dropped, not corrupted.

## Versioning

Every protocol version has:

- **Version identifier** — monotonic, unambiguous
- **Specification** — the complete message format, validation rules, and behavior
- **EOL date** — baked in at ratification. After this date, peers drop nodes still speaking this version
- **Grace period** — overlap window where both version N and N+1 are spoken

## Lifecycle

```
Proposed → Ratified → Active → Grace Period (N+1 ratified) → EOL → Dead
```

- **Proposed** — a protocol change proposal, subject to normal voting
- **Ratified** — enough nodes endorsed it. Adoption begins.
- **Active** — the current protocol version. Nodes speak it.
- **Grace period** — next version is ratified. Nodes that speak both coexist. Peers on N can still talk to peers on N+1.
- **EOL** — version is dead. Nodes still on it get dropped by peers.
- **Dead** — no peers speak it. Nodes on this version are effectively off the network.

## Upgrade Path

Upgrading should be trivial. The proposal *is* the upgrade — it contains the new specification. A node that adopts the proposal starts speaking the new version.

The goal: protocol as data, not code. Nodes interpret the protocol spec. A new version is a new spec to interpret. Adoption is mechanical, not a rebuild.

## EOL Timelines

Set by the community as part of the proposal. Factors:

- **Breaking scope** — how much changes? Minor refinements get short EOLs. Major restructuring gets long runways.
- **Network readiness** — what percentage of nodes need to adopt before the old version can safely die?
- **Contentiousness** — close votes suggest the community needs more time

## Safety

The protocol can evolve into anything the network decides. If the network decides to make bad choices, that's the network's problem — not a protocol failure.

The one hard constraint: **changes must not cause misinterpretation between nodes.** A node on version N talking to a node on version N+1 during the grace period must either:
1. Successfully negotiate down to version N, or
2. Cleanly fail to connect (no silent corruption)

Version negotiation is the handshake. If we can't agree on a version, we don't talk. We don't guess.

## Self-Hosting

Protocol evolution is itself governed by the protocol. There's no escape hatch, no benevolent dictator, no foundation that can override. The network is sovereign.

This means the first version (v0) must be good enough to:
1. Propose changes to itself
2. Vote on those changes
3. Propagate adoption
4. Enforce EOL

Everything after v0 is the network's responsibility.
