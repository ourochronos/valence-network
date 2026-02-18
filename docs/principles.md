# Principles

These are non-negotiable. They define what the Valence Network is and constrain what it can become. They are the spirit behind the constitutional tier — even when the rules can change, the principles explain *why* the rules exist.

Adapted from the [Valence knowledge substrate](https://github.com/ourochronos/valence) which provides the foundational primitives.

---

## 1. Privacy by Default

Your data is yours. Period.

- Content shared to the network is public. Content on your node is private.
- The network MUST NOT infer private information from public activity patterns
- Sharing is explicit, granular, and revocable
- Encrypted storage means hosts can't read what they store
- "Default private" means exactly that

**Test:** Can a node participate in the network (vote, propose, store) without revealing anything about its operator beyond its public key? If not, the protocol is broken.

## 2. Reputation from Rigor

Credibility comes from the quality of your contributions, not the size of your influence.

- Reputation reflects accuracy, consistency, and honest participation
- Popularity, volume, and social proof are not inputs to reputation
- A single well-crafted, widely-adopted proposal outweighs a hundred noise proposals
- Reputation is domain-specific — skill expertise doesn't transfer to config expertise
- Gaming reputation is treated as a system integrity threat (§10 Anti-Gaming)
- Finding errors is more valuable than confirming truths (asymmetric rewards)

**Test:** Can a new node with zero history establish credibility purely through useful contributions? If not, the incentives are wrong.

## 3. Exit Rights

You can always leave, and you can take your reputation history with you.

- Nodes can disconnect at any time with no penalty beyond inactivity decay
- Reputation history is signed and portable — your track record is yours
- No lock-in through network effects that trap participation
- Leaving does not destroy your contribution history
- Rejoining with the same key restores your identity

**Test:** Can a node leave, join a competing network, and point to its Valence Network history as proof of credibility? If not, we've built a roach motel.

## 4. No Central Authority

No single entity — including the protocol designers — can unilaterally control the network.

- The network governs itself through proposals and consensus
- Content moderation happens through local trust decisions, not central authority
- Each node controls its own filters and adoption decisions
- Disputes are resolved through transparent mechanisms, not executive fiat
- The constitutional tier protects against governance capture, not against change

**Test:** Can the protocol designers suppress a proposal that criticizes the protocol? If yes, the architecture is wrong.

## 5. Transparency

The protocol's behavior must be understandable and auditable.

- The protocol specification is public and complete
- Reference implementations are open source
- Reputation calculations are deterministic and reproducible from public data
- Governance decisions are documented with rationale (proposal body + vote record)
- "Trust the network" is never an acceptable answer to "how does this work?"

**Test:** Can an outsider read the spec and predict exactly how a given message will be processed? If not, the spec is incomplete.

## 6. Emergence Over Design

The network builds itself. We design the primitives; the network designs the system.

- v0 provides the minimum viable mechanism: identity, gossip, proposals, votes, reputation
- Everything else — storage, inference, verification, advanced trust — gets proposed through the protocol
- Structure emerges from use, not from upfront specification
- The network at 10,000 nodes will exhibit patterns you can't design at 10 nodes. That's the point.
- Start generous with what's possible; let usage prune what's unnecessary

**Test:** Can the network develop capabilities that the protocol designers never anticipated? If not, it's a product, not a protocol.

---

## Tensions and Tradeoffs

These principles sometimes conflict:

- **Privacy vs. Transparency:** Your data is private, but the protocol is transparent. The line is: *protocol behavior* is transparent, *node content* is private.
- **Exit Rights vs. Reputation Integrity:** You can leave, but you can't erase your signed history. The network remembers what you published.
- **No Central Authority vs. Bootstrap:** Someone has to write v0. The constitutional tier and critical mass thresholds are how we transition from "someone designed this" to "the network owns this."
- **Emergence vs. Safety:** The network can evolve into anything — including something harmful. The constitutional tier is the safety net, not a cage.

When principles conflict:

1. Name the tension explicitly
2. Prefer the principle that protects individual agency
3. Document the tradeoff and the reasoning
4. Revisit as the network evolves
