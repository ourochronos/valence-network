# Roadmap: From Valence to the Network

Valence (the knowledge substrate) contains rich primitives that should be proposed as network services once v0 is running. This document maps what exists in Valence to what the network can become.

## Already in v0

These Valence primitives are incorporated into the v0 spec:

| Valence Component | v0 Section | Notes |
|---|---|---|
| Ed25519 identity + signing | §1 Identity | Direct port |
| Reputation scoring | §8 Reputation | Simplified from Valence's full model |
| VDF sybil resistance | §9 Sybil Resistance | Iterated SHA-256 with checkpoints |
| libp2p + GossipSub transport | §3 Transport | Direct port |
| Merkle consistency | §11 Partition Detection | Simplified from merkle_sync |
| Erasure-coded storage | §6 Content Delivery | From our-storage/backup |
| Collusion detection | §10 Anti-Gaming | From our-consensus anti_gaming |
| ASN diversity | §4 Peer Discovery | From asn_diversity |
| Slashing constants | §8 Reputation penalties | Adapted from slashing.py |
| Adjacency storage challenges | §6 Storage | New, inspired by Valence's challenge model |

## Propose via v0 (Post-Bootstrap)

These Valence components should become network service proposals once governance activates:

### Confidence Vectors

**Valence source:** `spec/components/confidence-vectors/SPEC.md`

Single-number reputation is a compression of a richer signal. Valence has six-dimensional confidence vectors:

1. **Source reliability** — How trustworthy is the origin?
2. **Method quality** — How was this derived? (observation → hearsay → speculation)
3. **Internal consistency** — Does it contradict other known things?
4. **Temporal freshness** — Is it still valid?
5. **Corroboration** — How many independent sources agree?
6. **Domain applicability** — In what contexts does it hold?

**Network application:** Proposals could carry confidence vectors on their claims. Votes could weight by dimensional confidence, not just overall reputation. "I trust this node on skills but not on configs" becomes expressible.

**When:** After basic reputation is proven, probably 100+ nodes.

### Trust Graph

**Valence source:** `spec/components/trust-graph/SPEC.md`

v0 reputation is a single score per node. Valence has directional, domain-specific, evidence-based trust edges:

- A→B trust differs from B→A
- Trust is domain-scoped ("I trust you on Python, not on DevOps")
- Trust decays with disuse
- Transitive trust with damping (0.7× per hop, max 3 hops)

**Network application:** Vote weighting could use trust-graph edges instead of (or in addition to) global reputation. "I weight votes from nodes I personally trust higher than votes from strangers." This makes the network more resilient to sybil — fake nodes might have reputation but they won't have trust edges from real participants.

**When:** After the reputation system is running and its limitations become felt.

### Trust Layers (L1→L4)

**Valence source:** `spec/components/consensus-mechanism/SPEC.md`

Beliefs elevate through trust layers via independent corroboration:

- **L1 Personal:** I believe X
- **L2 Federated:** My trusted group believes X
- **L3 Domain:** Domain experts agree on X
- **L4 Communal:** The network has independently verified X

**Network application:** Proposals could have trust layers. A ratified proposal starts at L2 (community endorsed). If independently verified by nodes in different contexts, it elevates to L3/L4. This gives a richer signal than binary ratified/not-ratified.

**When:** After federations exist.

### Federations

**Valence source:** `spec/components/federation-layer/SPEC.md`

Bounded trust groups with:
- Shared encryption keys
- Membership governance
- Aggregated beliefs ("5 members believe X" without revealing which 5)
- Privacy-preserving computation within the group

**Network application:** Working groups. A federation of security-focused nodes could review proposals in their domain. Domain-specific reputation could be scoped to federation membership.

**When:** When the network is large enough that "everyone reviews everything" breaks down. Maybe 200+ nodes.

### Query Protocol

**Valence source:** `spec/components/query-protocol/SPEC.md`

Trust-weighted search across the network:
- Results ranked by trust, not just relevance
- Privacy-respecting (search doesn't reveal what you're looking for)
- Live subscriptions (not just snapshots)
- Explainable rankings

**Network application:** "Find me proposals related to X, weighted by my trust graph." Discovery that respects your trust relationships.

**When:** After the trust graph exists.

### Verification Protocol (Full)

**Valence source:** `spec/components/verification-protocol/SPEC.md`

v0 has basic claim verification. Valence has a full protocol:
- Commit-reveal voting (prevents herding)
- Staking on verifications
- Dispute resolution with appeals
- Calibration scoring (Brier scores — are your confidence claims accurate?)
- Bounties for finding contradictions

**Network application:** Formal verification of proposal claims. Staking reputation on votes. Calibration rewards for nodes whose predictions are well-calibrated.

**When:** After basic reputation needs strengthening. The asymmetric reward structure (5× for contradictions) is partially in v0 already.

### Commit-Reveal Voting

**Valence source:** `core/consensus/commit_reveal.py`

Two-phase voting: commit a hidden hash of your vote, then reveal. Prevents voters from being influenced by seeing how others voted.

**Network application:** For constitutional proposals where herding is especially dangerous. Standard proposals probably don't need it (too much friction).

**When:** When constitutional governance activates (1,024 nodes).

## Not From Valence (Network-Native)

These don't exist in Valence but should emerge from the network:

- **Inference services** — Nodes offering LLM compute to the network
- **Skill marketplace** — Standardized skill format with reviews and adoption tracking
- **Cross-network bridging** — A2A protocol integration for inter-network proposals
- **Reputation portability** — Carrying Valence Network reputation to other systems

These are by definition unpredictable — they'll be proposed by participants we haven't met yet, solving problems we haven't encountered. That's the point of Principle 6.
