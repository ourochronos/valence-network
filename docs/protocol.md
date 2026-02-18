# Protocol

## Message Types

### Discovery

- `ANNOUNCE` — node joins the network, shares identity and capabilities
- `DISCOVER` — request peer list from known nodes
- `PEERS` — response with known peers

### Proposals

- `REQUEST` — broadcast a need to the network
- `PROPOSE` — submit a solution (optionally in response to a request)
- `EDIT` — propose a modification to an existing proposal
- `WITHDRAW` — author withdraws a proposal

### Consensus

- `VOTE` — endorse or reject a proposal (weighted by reputation)
- `RATIFY` — proposal reaches consensus threshold (informational, not authoritative)

### Reputation

- `REPUTATION_GOSSIP` — share reputation observations with peers
- `REPUTATION_QUERY` — ask a peer for their view of a node's reputation

### Verification

- `CLAIM_VERIFY` — request verification of a proposal's claims
- `CLAIM_RESULT` — verification outcome
- `STORAGE_CHALLENGE` — random hash challenge to a storage provider
- `STORAGE_PROOF` — response to a storage challenge

### Data

- `SHARD_STORE` — request a node to store an encrypted shard
- `SHARD_RETRIEVE` — request a stored shard
- `SHARD_ACK` — acknowledge shard storage/retrieval

## Proposal Lifecycle

```
draft → open → voting → converging → ratified
                  ↓                      ↓
               rejected              adopted (per-node decision)
                  ↓
              withdrawn (by author at any stage)
```

## Consensus Threshold

TBD — needs design work. Questions:

- Fixed threshold vs. dynamic based on network size?
- Quorum requirements?
- Time-bounded voting periods?
- How to handle contentious proposals (close votes)?
- Supermajority for protocol-level changes vs. simple majority for content?

## Transport

TBD — candidates:

- libp2p (proven P2P stack, used by IPFS/Filecoin)
- Custom gossip over WebSocket/QUIC
- Hybrid: structured DHT for discovery, gossip for propagation

## Identity

TBD — candidates:

- Ed25519 keypairs (simple, proven)
- DIDs (interoperable, but complex)
- Something simpler that can bridge to DIDs later

## Anti-Fragmentation

- Nodes periodically reach out to random peers outside their usual neighborhood
- Hybrid push/pull gossip — nodes actively pull ("what have you seen since X?"), not just passively receive
- Any node that can reach any peer can eventually reach the whole network

## Protocol Versioning

Every message carries a protocol version. Versions have baked-in EOLs. See [evolution.md](evolution.md) for the full lifecycle.

Nodes must upgrade or they get dropped. Upgrading should be trivial — the proposal contains the new spec.

## Open Questions

- How large can proposals be? (Inline for small, sharded for large?)
- Conflict resolution for competing proposals addressing the same request?
- Garbage collection — when do rejected/expired proposals get pruned?
- v0 specification — what's the minimal viable protocol that can propose changes to itself?
