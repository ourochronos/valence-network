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

## Open Questions

- How do nodes find each other initially? (Bootstrap nodes? DHT? DNS seeds?)
- What prevents sybil attacks? (Reputation cost of entry? Proof of storage? Vouching?)
- How large can proposals be? (Inline for small, sharded for large?)
- Conflict resolution for competing proposals addressing the same request?
- Offline nodes — how do they catch up on proposals and votes?
- Garbage collection — when do rejected/expired proposals get pruned?
