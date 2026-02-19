# Adversarial Review: §5 State Reconciliation Protocol

**Reviewer:** Automated adversarial audit (Claude, subagent)
**Date:** 2026-02-18
**Spec version:** v0
**Scope:** §5 State Reconciliation subsection, evaluated against full spec (§1–§15)

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 3 |
| HIGH     | 5 |
| MEDIUM   | 7 |
| LOW      | 4 |
| INFO     | 3 |
| **Total** | **22** |

**Overall assessment:** The State Reconciliation protocol is well-structured with sensible phased sync and Merkle-based divergence detection. However, it has several critical gaps around eclipse attacks during sync, lack of Byzantine tolerance in the 2-of-3 Merkle agreement, and an exploitable degraded mode that can be weaponized. The protocol would benefit from stronger guarantees around sync peer authentication, anti-censorship during sync, and bounded sync completion times.

---

## Findings

---

### SYNC-01
- **Severity:** CRITICAL
- **Title:** 2-of-3 Merkle agreement allows a single malicious peer to force acceptance of poisoned state
- **Location:** §5, Sync Completeness, bullet 2
- **Description:** Sync-complete requires the node's proposal Merkle root to match "at least 2 of 3 sync peers." If an attacker controls 2 of the 3 sync peers (possible if ASN diversity only requires ≥2 ASNs, and the attacker controls nodes across 2 ASNs), the syncing node will accept the attacker's state as canonical. The spec requires ≥2 ASNs but doesn't require ≥3 ASNs for the 3 sync peers, so 2 peers from ASN-A and 1 from ASN-B satisfies the requirement while giving ASN-A's operator a 2-of-3 majority.
- **Exploit scenario:** Attacker runs nodes across 2 ASNs. A new node connecting during sync selects 2 attacker peers and 1 honest peer (satisfying ≥2 ASN requirement). Attacker peers serve a consistent but poisoned proposal set (e.g., omitting certain proposals, or including fabricated ones). The 2-of-3 agreement passes. The new node enters normal operation with corrupted state — missing proposals, wrong vote tallies, or phantom proposals that don't exist on the honest network.
- **Recommended fix:** Require sync peers from ≥3 distinct ASNs (not just ≥2). Increase minimum sync peers to 5 with agreement threshold of 3-of-5 from ≥3 ASNs. Alternatively, require Merkle root agreement from ALL sync peers (unanimous), and if disagreement occurs, expand the peer set until a supermajority emerges.

---

### SYNC-02
- **Severity:** CRITICAL
- **Title:** No authentication of sync peer reputation before trusting their state
- **Location:** §5, Sync Peer Selection
- **Description:** During initial sync, a node has no reputation data (it hasn't completed phase 2 yet). It selects 3 peers for sync but has no basis to evaluate their trustworthiness — reputation data arrives in phase 2, but the identity data from phase 1 is already accepted from these unvetted peers. A node performing first-join sync (`since_timestamp = 0`) is maximally vulnerable because it has zero prior state.
- **Exploit scenario:** An attacker positions low-cost nodes (30-second VDF each) as responsive bootstrap-adjacent peers. New nodes preferentially connect to them (they're always available, fast responders). During phase 1, the attacker serves crafted DID_REVOKE messages for legitimate high-reputation nodes, or omits DID_LINK messages to fragment identity graphs. The syncing node then enters phase 2 with a corrupted identity view, causing all subsequent phases to build on poisoned foundations.
- **Recommended fix:** Add a pre-sync reputation bootstrap: before full state reconciliation, the node should request REPUTATION_GOSSIP from bootstrap nodes (which are presumably well-known and trusted) to get a rough reputation landscape. Use this to prefer high-reputation sync peers. Alternatively, require that sync peers for first-join MUST include at least one bootstrap node.

---

### SYNC-03
- **Severity:** CRITICAL
- **Title:** Degraded mode can be weaponized for permanent denial of governance participation
- **Location:** §5, Sync Completeness, "Partial sync tolerance"
- **Description:** A node in degraded mode "MUST NOT vote or propose" but "MAY relay messages, serve storage challenges, and accumulate reputation via uptime." There is no specified mechanism to exit degraded mode other than completing full sync. If an attacker can prevent a node from ever completing sync (e.g., by selectively timing DDoS on sync peers, or by exploiting the rate limit of 10 SYNC_REQUEST per peer per minute to slow sync below the rate of new state generation), the node is permanently locked out of governance while still contributing infrastructure resources.
- **Exploit scenario:** Attacker targets a specific node's sync peers with intermittent disruption. The victim node repeatedly fails mid-sync, re-enters degraded mode, and retries. With each retry, the network state has advanced, requiring more sync work. At scale (millions of messages), the sync-to-completion time may exceed the interval between disruptions. The victim earns reputation through uptime and storage but can never vote or propose — taxation without representation.
- **Recommended fix:** (1) Define a maximum time in degraded mode (e.g., 7 days) after which the node can participate with reduced vote weight (e.g., 50%) rather than zero. (2) Allow incremental exit from degraded mode — if phases 1-3 complete, allow voting even if phases 4-5 haven't. (3) Specify that degraded mode nodes SHOULD attempt sync from different peers on each retry, with mandatory peer rotation.

---

### SYNC-04
- **Severity:** HIGH
- **Title:** Selective omission during sync is undetectable for non-proposal state
- **Location:** §5, Merkle-Based Divergence Detection
- **Description:** Merkle-based divergence detection only covers the "proposal Merkle root" (§12). There is no Merkle tree or integrity check for identity state (phase 1), reputation state (phase 2), content metadata (phase 4), or storage state (phase 5). A malicious sync peer can selectively omit DID_REVOKE messages, REPUTATION_GOSSIP assessments, FLAG messages, or SHARD_ASSIGNMENT messages, and the syncing node has no mechanism to detect the omission.
- **Exploit scenario:** Attacker sync peer omits DID_REVOKE messages for a compromised key. The syncing node treats the compromised key as still authorized. Later, the attacker uses the compromised key to cast votes or submit proposals that the victim node accepts but other nodes reject (because they have the revocation). This creates a persistent state divergence that only surfaces when the victim's Merkle root diverges on subsequent incremental sync — by which time the damage is done.
- **Recommended fix:** Extend Merkle tree coverage to identity state (at minimum) and reputation state. Define a separate identity Merkle tree over all DID_LINK and DID_REVOKE message IDs. Require identity Merkle root agreement during phase 1 completion, before proceeding to phase 2.

---

### SYNC-05
- **Severity:** HIGH
- **Title:** Race condition between live gossip processing and phased sync
- **Location:** §5, Sync Phases — "It MAY relay gossip messages it receives and SHOULD process them into local state"
- **Description:** During sync, the node processes live gossip into local state while simultaneously receiving historical state from sync peers. The spec says the node "SHOULD process [live gossip] into local state" but doesn't specify how to reconcile conflicts between live gossip and sync data. For example: a node in phase 1 sync receives a live DID_LINK message for a key, then receives a DID_REVOKE for the same key from sync (which has an earlier timestamp). The processing order may cause the node to incorrectly treat the key as linked (if it processes live gossip first and the DID_REVOKE from sync doesn't override it).
- **Exploit scenario:** An attacker with a revoked key broadcasts a new DID_LINK during the victim's sync window. The victim processes the live DID_LINK before receiving the historical DID_REVOKE from sync. If the implementation doesn't properly reconcile, the attacker's key appears authorized on the victim node.
- **Recommended fix:** Specify that during sync phases, live gossip messages MUST be buffered and only applied after the corresponding sync phase completes. Alternatively, specify that sync state always takes precedence over live gossip for messages with earlier timestamps, and that DID_REVOKE always overrides DID_LINK regardless of processing order.

---

### SYNC-06
- **Severity:** HIGH
- **Title:** No bound on sync completion time enables state drift during long syncs
- **Location:** §5, Sync Phases (general)
- **Description:** There is no specified maximum time for sync completion. For a node offline for 30 days, the sync must process: all identity changes, reputation gossip (every 15 min × 30 days × 10 peers = ~28,800 REPUTATION_GOSSIP messages), all proposals/votes/comments, all content metadata, and all storage state. At the rate limit of 10 SYNC_REQUEST per peer per minute, syncing from 3 peers gives 30 requests/minute. With 100 messages per response, that's 3,000 messages/minute. For a network with 10k nodes producing ~111 reputation messages/second (per §15), 30 days generates ~288M reputation messages alone. At 3,000/minute sync rate, this takes ~66 days to sync — during which another 66 days of state accumulates.
- **Exploit scenario:** Not an attack per se, but a fundamental liveness issue. Large networks make full sync from `since_timestamp = 0` practically impossible. Even reconnection after 30 days may take longer than 30 days to sync, creating an infinite sync loop.
- **Recommended fix:** (1) Define state snapshots — periodic signed checkpoints that summarize the current state, allowing nodes to sync from a recent snapshot rather than replaying all history. (2) For reputation, allow syncing the *current* reputation state rather than all historical REPUTATION_GOSSIP messages. (3) Increase sync rate limits for nodes in initial sync (e.g., 50 SYNC_REQUEST per peer per minute during state reconciliation). (4) Explicitly address the "sync takes longer than the offline period" scenario.

---

### SYNC-07
- **Severity:** HIGH
- **Title:** Incremental sync skips phase 1 (identity), allowing revocation gaps
- **Location:** §5, Incremental Sync
- **Description:** "Only phases 2–4 are needed for incremental sync — identity changes propagate via live gossip and the 30-day re-broadcast mechanism (§1)." This assumes live gossip is reliable for identity changes. But GossipSub message delivery is best-effort. If a node misses a DID_REVOKE during the 15-minute incremental sync interval, and the next re-broadcast is up to 30 days away, the node operates with a stale identity graph for up to 30 days.
- **Exploit scenario:** A compromised child key is revoked via DID_REVOKE. A targeted node misses the gossip message (network glitch, brief disconnection). The 30-day re-broadcast hasn't happened yet. Incremental sync doesn't cover identity messages. For up to 30 days, the victim node accepts messages from the revoked key as legitimate, including votes and proposals attributed to the parent identity.
- **Recommended fix:** Include identity messages (DID_LINK, DID_REVOKE, KEY_ROTATE) in incremental sync, at least at a reduced frequency (e.g., every 4th incremental sync cycle, or once per hour). The cost is minimal — identity messages are rare compared to reputation gossip.

---

### SYNC-08
- **Severity:** HIGH
- **Title:** Free-rider problem in serving sync requests
- **Location:** §5 (general — no incentive structure for sync serving)
- **Description:** Serving sync requests costs bandwidth and CPU but earns nothing. There is no reputation reward for serving sync, unlike storage (which earns rent). Rational nodes may deprioritize or refuse sync requests, especially under load. As the network grows, the cost of serving sync increases (more state to serve) while the incentive remains zero.
- **Exploit scenario:** At scale, most nodes implement aggressive rate limiting on sync responses. New nodes struggle to find willing sync peers. The few nodes that do serve sync become bottlenecks, making them high-value DDoS targets. The network becomes increasingly hostile to new entrants, calcifying the existing node set — antithetical to the protocol's values.
- **Recommended fix:** Add a small reputation reward for serving sync requests (e.g., counted as an "observation" for uptime rewards). Alternatively, make sync serving a soft requirement for reputation maintenance (nodes that refuse sync from >50% of requesters get flagged). Or tie sync serving to the `store` capability — nodes that want to earn storage rent must also serve sync.

---

### SYNC-09
- **Severity:** MEDIUM
- **Title:** Sync bandwidth cap of 50% is a SHOULD, not a MUST
- **Location:** §5, Backpressure
- **Description:** "Nodes SHOULD limit sync traffic to 50% of available bandwidth." As a SHOULD, nodes MAY ignore this. A syncing node with aggressive sync behavior could saturate its own connection and its peers' connections, degrading live gossip delivery for those peers.
- **Exploit scenario:** An attacker runs a node that perpetually syncs at maximum bandwidth from many peers simultaneously, acting as a selective DDoS on sync-serving nodes. Since the cap is SHOULD, the attacker isn't violating protocol.
- **Recommended fix:** Make the bandwidth cap a MUST for the requesting side. Add a MUST for sync-serving nodes to cap sync responses per peer (e.g., max 10 MiB/minute per peer for sync traffic).

---

### SYNC-10
- **Severity:** MEDIUM
- **Title:** Merkle root comparison after phases 1-3 but tree only covers proposals
- **Location:** §5, Merkle-Based Divergence Detection
- **Description:** The spec says "After completing phases 1–3, the node MUST compare its proposal Merkle root." But phase 1 is identity, phase 2 is reputation, and phase 3 is proposals. The Merkle root only validates phase 3 data. Phases 1 and 2 are accepted without integrity verification. This is inconsistent — why require phases 1-2 to complete first if you can't verify they completed correctly?
- **Exploit scenario:** See SYNC-04. The phases 1-2 data is unverified, making the phase 3 Merkle check a false sense of security — the proposal data might match but the identity/reputation context for evaluating those proposals may be corrupted.
- **Recommended fix:** Define Merkle trees for identity state and optionally reputation state. Verify all three after phase 3 completes.

---

### SYNC-11
- **Severity:** MEDIUM
- **Title:** "First valid instance" rule for duplicate sync messages doesn't handle Byzantine disagreement
- **Location:** §5, Sync Peer Selection
- **Description:** "When the same message is received from multiple sync peers, the node uses the first valid instance." This is fine for identical messages but doesn't address what happens when two sync peers serve messages that are individually valid but collectively inconsistent — e.g., two different VOTE messages from the same identity for the same proposal (before §8 supersession is applied). The spec mentions that "conflicting REPUTATION_GOSSIP assessments are all ingested" but doesn't generalize this to other message types.
- **Exploit scenario:** Two sync peers provide different VOTE messages from the same voter for the same proposal (both signed, both valid, different timestamps). If the syncing node receives the older vote first and the superseding vote second, it correctly supersedes. If it receives only one (from a peer that omitted the other), it has stale vote state. With no integrity check on vote completeness, this is undetectable.
- **Recommended fix:** For proposals that appear in the Merkle tree, require that sync responses include ALL votes for those proposals. Define vote completeness as part of the Merkle check — include vote message IDs in the Merkle tree or a separate vote-per-proposal subtree.

---

### SYNC-12
- **Severity:** MEDIUM
- **Title:** Sync phase ordering prevents parallel sync, increasing total sync time
- **Location:** §5, Sync Phases — "Each phase MUST complete before the next begins"
- **Description:** Strict sequential phases mean total sync time is the sum of all phase times. For a large network, this compounds the performance issues in SYNC-06. Phases 4 and 5 (content metadata, storage state) are largely independent of each other and could run in parallel.
- **Exploit scenario:** Not an attack, but a design issue. The strict ordering is justified for phases 1→2→3 (identity before reputation before proposals), but phases 4 and 5 could safely parallel with phase 3 completion. The MUST language prevents this optimization.
- **Recommended fix:** Allow phases 4 and 5 to run in parallel after phase 3 completes. Reword: "Phases 1, 2, and 3 MUST complete in order. Phases 4 and 5 MAY proceed in parallel after phase 3."

---

### SYNC-13
- **Severity:** MEDIUM
- **Title:** Jitter range (0-30s) insufficient for large partition heals
- **Location:** §5, Backpressure
- **Description:** After a network partition heals, potentially thousands of nodes reconnect simultaneously. A 0-30 second jitter spreads sync initiation across 30 seconds. With 1,000 nodes at 10 SYNC_REQUEST/min each, that's still ~333 requests/second hitting the network within the first minute. Combined with the 1-5 second inter-phase jitter, this creates a sync storm.
- **Exploit scenario:** After a natural partition heals, the sync storm degrades network performance for all nodes, including those that were online. Live gossip latency increases, potentially causing nodes to miss messages and triggering further sync needs — a cascading failure.
- **Recommended fix:** Scale jitter with estimated partition size. If a node detects it was in a partition (Merkle divergence from >20% of peers), increase jitter to 0-300 seconds. Add progressive sync: nodes that were offline longer wait longer before syncing (e.g., jitter = min(300, offline_seconds / 100)).

---

### SYNC-14
- **Severity:** MEDIUM
- **Title:** No specification for mid-sync failure recovery
- **Location:** §5, Sync Phases (general)
- **Description:** The spec doesn't address what happens when sync fails mid-phase. If a node completes phases 1-2 and fails during phase 3, must it restart from phase 1? Can it resume phase 3? Is the phase 1-2 state committed or rolled back? The "Partial sync tolerance" section mentions degraded mode but doesn't specify whether partial phase completion is persisted.
- **Exploit scenario:** A node fails mid-phase-3 after processing 90% of proposals. It enters degraded mode and retries. If it restarts from phase 1, it re-fetches identity and reputation data (wasted work). If it resumes phase 3, it may have stale identity data if phase 1 state changed during the failed attempt.
- **Recommended fix:** Specify that completed phases are persisted (phase 1-2 state is committed). On retry, resume from the last incomplete phase with a fresh `since_timestamp` for that phase. If more than 1 hour has elapsed since phase 1 completion, re-run phase 1 before resuming.

---

### SYNC-15
- **Severity:** MEDIUM
- **Title:** REPUTATION_GOSSIP batch requirement for sync-complete is gameable
- **Location:** §5, Sync Completeness, bullet 3
- **Description:** Sync-complete requires "at least one REPUTATION_GOSSIP batch from each sync peer." A malicious sync peer can satisfy this trivially by sending a REPUTATION_GOSSIP with fabricated assessments for 10 random nodes. The syncing node has no way to verify these assessments are honest during initial sync. Combined with the α=0 cap (§9), this limits damage somewhat, but the sync-complete gate provides false assurance.
- **Exploit scenario:** Attacker sync peer sends a REPUTATION_GOSSIP batch inflating the reputation of colluding nodes. The syncing node, having just completed sync, begins normal operation with inflated trust in the attacker's collaborators. While the α=0 cap limits individual reputation to 0.2, the attacker can ensure their collaborators appear at exactly 0.2 (rather than penalized below), hiding penalty history.
- **Recommended fix:** Require REPUTATION_GOSSIP batches from sync peers to be cross-checked: assessments for the same subject from different sync peers should be compared, and significant divergence (>0.1 difference) should be flagged. Additionally, specify that sync-received REPUTATION_GOSSIP is treated with the same α=0 cap as any unverified peer assessment.

---

### SYNC-16
- **Severity:** LOW
- **Title:** Incremental sync window (30 min) may miss messages in degraded gossip conditions
- **Location:** §5, Incremental Sync
- **Description:** Incremental sync uses `since_timestamp = now - 30 minutes`. In degraded network conditions, message propagation may exceed 30 minutes. The spec acknowledges gossip convergence is "~15 minutes" in a healthy network (§1), but doesn't account for degraded conditions in the incremental sync window.
- **Exploit scenario:** During high network load, gossip latency reaches 45 minutes. Incremental sync with a 30-minute window misses messages from the 30-45 minute ago range. These messages are never synced incrementally and only caught on the next full sync (if ever).
- **Recommended fix:** Make the incremental sync window adaptive: `max(30 minutes, 2 × observed_gossip_latency)`. Alternatively, track the highest timestamp received from the last sync and use that as the cursor instead of a fixed lookback.

---

### SYNC-17
- **Severity:** LOW
- **Title:** No mechanism to detect or recover from corrupted `last known good timestamp`
- **Location:** §5, Sync Phases — "since_timestamp SHOULD be the node's last known good timestamp (persisted to disk on clean shutdown)"
- **Description:** If the persisted timestamp is corrupted (disk error, crash during write), the node may use a future timestamp (missing all recent messages) or a very old timestamp (unnecessary full resync). The SHOULD language means implementations may not persist it at all.
- **Exploit scenario:** A node crashes during timestamp persistence, corrupting the value to a future date. On restart, it sends SYNC_REQUEST with a future `since_timestamp`, receives no messages, considers itself synced, and operates with stale state — potentially for a long time before anyone notices.
- **Recommended fix:** (1) Upgrade to MUST for timestamp persistence. (2) Validate the persisted timestamp against current time — reject if it's in the future. (3) On any persistence failure, fall back to `since_timestamp = now - 24 hours` rather than 0 (avoids full resync while catching recent messages).

---

### SYNC-18
- **Severity:** LOW
- **Title:** Phase 5 (storage state) sync may overwhelm nodes not participating in storage
- **Location:** §5, Sync Phases, phase 5
- **Description:** Phase 5 includes "REPLICATE_REQUEST, REPLICATE_ACCEPT, SHARD_ASSIGNMENT, SHARD_RECEIVED" which is described as "Only relevant if the node participates in storage." However, the phased sync is specified as mandatory ("Reconciliation proceeds in strict priority order. Each phase MUST complete before the next begins"), with no carve-out for skipping phase 5 for non-storage nodes.
- **Exploit scenario:** A lightweight node (no storage capability) is forced to sync all storage state before reaching sync-complete, wasting bandwidth and delaying its entry into normal operation. At scale, storage messages may dominate the total message volume.
- **Recommended fix:** Allow nodes without the `store` capability to skip phase 5. Modify sync-complete criteria to require only phases 1-4 for non-storage nodes.

---

### SYNC-19
- **Severity:** LOW
- **Title:** Staggered phase delay (1-5s) is underspecified
- **Location:** §5, Backpressure
- **Description:** "Nodes SHOULD wait 1–5 seconds (random) between completing one sync phase and starting the next." This is a SHOULD, and the range is small. It also doesn't specify the distribution (uniform? weighted toward higher values?).
- **Exploit scenario:** Not directly exploitable, but insufficient specification leads to implementation divergence that could be fingerprinted (identifying implementation by timing behavior).
- **Recommended fix:** Minor: specify uniform random distribution. Consider whether this should be MUST for consistency.

---

### SYNC-20
- **Severity:** INFO
- **Title:** No versioning on the sync protocol sub-messages
- **Location:** §5, Sync Phases
- **Description:** The SYNC_REQUEST and SYNC_RESPONSE messages use the same format for all phases. There's no version field specific to the sync sub-protocol, so if sync needs to evolve (e.g., adding Merkle trees for identity state per SYNC-04), it requires a full protocol version bump.
- **Exploit scenario:** N/A — design observation.
- **Recommended fix:** Consider adding an optional `sync_version` field to SYNC_REQUEST/SYNC_RESPONSE for forward compatibility, allowing sync protocol improvements without full v0→v1 transition.

---

### SYNC-21
- **Severity:** INFO
- **Title:** Interaction with §12 partition detection during sync is underspecified
- **Location:** §5 State Reconciliation + §12 Partition Detection
- **Description:** §12 defines partition severity by "% proposal set difference." A syncing node by definition has a large proposal set difference (it's missing proposals). The spec doesn't clarify whether a syncing node's Merkle divergence should trigger partition alerts in other nodes. If it does, a wave of new nodes joining could cause false partition alarms.
- **Exploit scenario:** An attacker brings 100 new nodes online simultaneously. Each triggers Merkle divergence with existing peers. If peers interpret this as a partition event, they may initiate unnecessary partition healing procedures, wasting resources.
- **Recommended fix:** Specify that nodes MUST NOT trigger partition detection based on Merkle divergence with peers that are currently in sync mode. Peers should be able to identify syncing nodes (e.g., via a flag in SYNC_REQUEST or by tracking that a peer has recently initiated multi-phase sync).

---

### SYNC-22
- **Severity:** INFO
- **Title:** The sync protocol has no mechanism for a node to advertise its sync status to peers
- **Location:** §5 (general)
- **Description:** Peers cannot distinguish between a fully-synced node and a node in degraded mode. This means peers may rely on a degraded-mode node for sync (circular dependency) or route governance-critical messages to it expecting full participation.
- **Exploit scenario:** A node stuck in degraded mode is selected as a sync peer by another new node. The degraded node serves incomplete state, causing the new node to also have incomplete state. This can cascade in a network where many nodes are syncing simultaneously.
- **Recommended fix:** Add a `sync_status` field to PEER_ANNOUNCE (`syncing | degraded | synced`). Nodes in `syncing` or `degraded` status MUST NOT be selected as sync peers. This is simple, backward-compatible (unknown fields are ignored per §14), and prevents sync-from-stale-peer cascades.
