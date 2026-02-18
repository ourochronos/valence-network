# Protocol Quick Reference

> This is a quick reference. The authoritative specification is [v0.md](v0.md).

## Message Types

| Type | Purpose | Section |
|---|---|---|
| `PEER_ANNOUNCE` | Announce presence, capabilities, addresses | §4 |
| `AUTH_CHALLENGE` | Authentication handshake (initiator sends nonce) | §3 |
| `AUTH_RESPONSE` | Authentication handshake (responder proves identity + VDF) | §3 |
| `SYNC_REQUEST` | Pull missed messages from a peer | §5 |
| `SYNC_RESPONSE` | Return requested messages + Merkle checkpoint | §5 |
| `PEER_LIST_REQUEST` | Request known peers (paginated) | §4 |
| `PEER_LIST_RESPONSE` | Return known peers | §4 |
| `REQUEST` | Broadcast a need | §7 |
| `PROPOSE` | Submit a solution (optionally answering a REQUEST) | §7 |
| `WITHDRAW` | Author withdraws a proposal | §7 |
| `ADOPT` | Node reports local adoption of a proposal (success/failure) | §7 |
| `VOTE` | Endorse, reject, or abstain on a proposal | §8 |
| `REPUTATION_GOSSIP` | Share signed reputation assessments of peers | §9 |
| `STORAGE_CHALLENGE` | Adjacency-proof challenge to a shard holder | §6 |
| `STORAGE_PROOF` | Response to a storage challenge | §6 |
| `SHARD_QUERY` | Ask a peer for available shards by content hash | §6 |
| `SHARD_QUERY_RESPONSE` | Return available shard indices and hashes | §6 |
| `KEY_ROTATE` | Rotate identity keypair (signed by both old and new keys) | §1 |
| `KEY_CONFLICT` | Broadcast conflicting KEY_ROTATE discovered during partition heal | §1 |

## Proposal Lifecycle

Each node tracks its own view — there is no global state machine.

| State | Meaning |
|---|---|
| **Open** | Proposal exists, votes accumulating, deadline not reached |
| **Converging** | Weighted endorsement exceeds local threshold |
| **Ratified** | Node considers the proposal accepted by the network |
| **Rejected** | Weighted rejection exceeds threshold, or deadline passed without ratification |
| **Adopted** | This node has applied the proposal locally |
| **Expired** | Voting deadline passed |
| **Withdrawn** | Author withdrew |

Edits are expressed as new proposals with `supersedes` pointing to the original.

## Governance Tiers

| Tier | Threshold | Quorum | Voting Period | Notes |
|---|---|---|---|---|
| **Standard** | 0.67 | max(active_nodes × 0.1, total_rep × 0.10) | 14d default, 90d max | Skills, configs, docs, code |
| **Constitutional** | 0.90 | 30% of total network reputation | 90d minimum | Protocol core: rep floor, Sybil resistance, voting rules, thresholds, identity, meta-governance |

Constitutional proposals also require a **30-day cooling period** between ratification and activation.

## Network Phases

| Active Nodes | Phase |
|---|---|
| < 16 | **Frozen** — content proposals only, no governance |
| ≥ 16 (sustained 3–7 days) | **Standard governance** activates (Tier 1). First 30 days use headcount voting. |
| ≥ 1,024 (sustained 30 days) | **Constitutional governance** activates (Tier 2). All v0 parameters become Tier 2 protected. |
