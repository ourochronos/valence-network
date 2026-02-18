# Architecture

## Overview

Valence Network is a P2P protocol where the core primitive is the **proposal loop**. Everything else — storage, inference, trust, verification — is built through that loop. The network governs and extends itself.

## Core Concepts

### Node

A participant in the network. Could be an agent, a human, or both. Each node has:

- A cryptographic identity
- Local reputation scores for peers (gossip-derived)
- Local storage capacity it may offer
- The ability to propose, vote, refine, and adopt

### Proposal

The atomic unit of change. A proposal is:

- **Content** — a diff, document, skill, config, or code
- **Claims** — verifiable assertions about what this does
- **Provenance** — who proposed it, what it builds on
- **State** — draft → open → converging → ratified | rejected | withdrawn

Proposals can exist:
- **In response to a request** — someone asked for X, this proposes a solution
- **Standalone** — "I figured something out, here's an improvement"

### Request

A broadcast need. "I need a skill that does X." Requests attract proposals.

### Edit Proposal

A proposed modification to an existing proposal. Refinement is collaborative — someone says "good idea but handle this edge case" and the proposal evolves. Edit proposals are themselves subject to voting.

### Vote

A signal from a node endorsing or rejecting a proposal. Votes are weighted by the voter's reputation. Voting is not binding — each node decides independently whether to adopt. Consensus is a signal, not an authority.

### Reputation

A node's accumulated trust, derived from:

- Proposals that get adopted
- Claims that verify
- Edits that improve proposals
- Storage contributions (verified via hash challenges)
- Verification work
- Inference contributions (future)

Reputation is:
- **Local** — each node computes its own view of peer reputations
- **Gossip-propagated** — nodes share reputation observations with peers
- **Recursive** — a peer's assessment of someone is weighted by your trust in that peer
- **Not transferable** — you can't give reputation away
- **Decayable** — inactive nodes gradually lose weight

## Network Services (Self-Proposed)

These aren't built in from day one. They're proposed and ratified through the protocol itself.

### Storage

- Nodes volunteer to store encrypted, hashed shards
- Random sector hash challenges verify honest storage
- Reputation reward for reliable storage
- Data is encrypted — hosts can't read what they store

### Verification

- Nodes verify claims made by proposals
- Nodes verify storage integrity of peers
- Verification itself is verifiable (recursive)
- Reputation reward for verification work

### Inference (Future)

- Nodes offer compute for inference
- Quality verified statistically — multiple nodes run the same query, results compared
- Don't need to prove the model — prove the output quality
- High reputation reward (expensive contribution)
- Proposed and ratified via the protocol when the network is ready

## Bootstrap

1. Protocol ships with the proposal loop only
2. First nodes join, can propose and vote
3. Storage, verification, inference get proposed as network services
4. The network decides when to adopt each capability
5. The protocol evolves through its own mechanism

## Trust Model

```
Node A sees:
  - Direct experience with Node B (proposals, votes, storage)
  - Node C's assessment of Node B (weighted by A's trust in C)
  - Node D's assessment of Node B (weighted by A's trust in D)
  → A computes local reputation score for B
```

No global reputation. No central authority. Each node's view is its own, informed by the network.

## What This Replaces

Valence (the monolith) tried to be:
- Knowledge substrate
- Trust system
- Reputation engine
- Federation protocol
- Session tracking
- Pattern recognition

Valence Network is the protocol layer. The knowledge substrate, trust, reputation — those are emergent properties of a network that can propose, refine, and adopt its own infrastructure. The concerns decompose naturally because each one can be proposed and evolved independently.
