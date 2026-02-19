# Adversarial Review: §5 State Reconciliation Protocol (Round 2)

**Reviewer:** Automated adversarial audit (Claude, subagent)
**Date:** 2026-02-18
**Spec version:** v0 (post-round-1 fixes)
**Scope:** §5 State Reconciliation, re-audit after 22 findings addressed

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 2 |
| HIGH     | 4 |
| MEDIUM   | 5 |
| LOW      | 3 |
| INFO     | 2 |
| **Total** | **16** |

**Overall assessment:** The round 1 fixes are substantive and well-integrated. All 22 original findings have been addressed — 21 are properly resolved, 1 remains open (SYNC-20, sync protocol versioning, acknowledged as a design observation). However, the fixes introduce significant new attack surface, particularly around state snapshots and the graduated degraded mode exit. The protocol's complexity has increased materially, and several new mechanisms interact in ways that create exploitable gaps.

---

## Round 1 Finding Status

### Properly Resolved (21 of 22)

| Finding | Fix | Assessment |
|---------|-----|------------|
| SYNC-01 (2-of-3 Merkle) | Upgraded to 5 peers from ≥3 ASNs, 3-of-5 agreement | **Resolved.** Significantly harder to eclipse. |
| SYNC-02 (No sync peer auth) | First-join MUST include bootstrap node | **Resolved.** Bootstrap nodes anchor trust. |
| SYNC-03 (Permanent degraded) | Graduated exit, mandatory peer rotation, 50% weight after 7 days | **Resolved.** But introduces new attack surface (see SYNC-R2-03). |
| SYNC-04 (No identity Merkle) | Identity Merkle tree added | **Resolved.** Clean implementation. |
| SYNC-05 (Live gossip race) | Buffered gossip during sync phases | **Resolved.** But introduces new concerns (see SYNC-R2-06). |
| SYNC-06 (Unbounded sync time) | State snapshots, REPUTATION_CURRENT, elevated sync rate | **Resolved.** Snapshots introduce new attack surface (see SYNC-R2-01, SYNC-R2-02). |
| SYNC-07 (Incremental skips identity) | Every 4th cycle includes identity phase | **Resolved.** Good balance of cost vs. coverage. |
| SYNC-08 (Free-rider sync serving) | Sync serving counts as uptime activity | **Resolved.** But farmable (see SYNC-R2-05). |
| SYNC-09 (Bandwidth cap SHOULD) | Upgraded to MUST, added server-side cap | **Resolved.** |
| SYNC-10 (Merkle only proposals) | Identity Merkle tree added | **Resolved.** |
| SYNC-11 (First-valid Byzantine) | Sync responses MUST include all votes for divergent proposals | **Resolved.** |
| SYNC-12 (No parallel phases) | Phases 4 and 5 MAY parallel after phase 3 | **Resolved.** |
| SYNC-13 (Jitter insufficient) | Partition-aware jitter (0-300s), offline-scaled | **Resolved.** |
| SYNC-14 (Mid-sync failure) | Committed phases, stale check, peer rotation | **Resolved.** |
| SYNC-15 (Rep batch gameable) | Sync-received rep uses α=0 cap | **Resolved.** |
| SYNC-16 (30 min window) | Adaptive window using tracked timestamps | **Resolved.** |
| SYNC-17 (Corrupted timestamp) | Future timestamp → fallback now-24h, MUST persist | **Resolved.** |
| SYNC-18 (Phase 5 non-storage) | Non-store nodes MUST skip phase 5 | **Resolved.** |
| SYNC-19 (Staggered delay) | Specified as uniform random | **Resolved.** |
| SYNC-21 (§12 interaction) | sync_status exempts syncing nodes from partition detection | **Resolved.** |
| SYNC-22 (No sync status) | sync_status field in PEER_ANNOUNCE | **Resolved.** |

### Still Open (1 of 22)

| Finding | Status | Notes |
|---------|--------|-------|
| SYNC-20 (No sync versioning) | **Open.** | No sync_version field added. Still relies on full protocol version bump for sync changes. Low severity — acceptable for v0 but will cause friction when sync needs to evolve independently. |

---

## New Findings

---

### SYNC-R2-01
- **Severity:** CRITICAL
- **Title:** State snapshots can be forged to eclipse new nodes via colluding snapshot publishers
- **Location:** §5, State Snapshots
- **Description:** Snapshot bootstrap requires "same snapshot content (matching Merkle roots) from ≥3 snapshot publishers across ≥3 ASNs." The reputation threshold for publishing is ≥0.5. An attacker with 3 nodes across 3 ASNs, each at rep ≥0.5, can publish coordinated poisoned snapshots. A syncing node receiving these 3 matching snapshots from 3 ASNs would accept the poisoned state. The snapshot mechanism actually *weakens* security compared to the full sync path, because full sync requires 3-of-5 agreement from sync peers, while snapshots only need 3 agreeing publishers with no minimum total publisher count.
- **Exploit scenario:** Attacker operates 3 nodes (rep 0.5+) across 3 ASNs. They publish STATE_SNAPSHOT messages with identical but poisoned Merkle roots — omitting certain identities, inflating reputation, or including phantom proposals. A new node bootstrapping from snapshots accepts this as canonical. After snapshot bootstrap, incremental sync from the snapshot timestamp catches only *new* messages, not the historical omissions baked into the snapshot. The new node operates with a corrupted worldview indefinitely.
- **Additional concern:** The spec says snapshot publishing "earns no direct reputation reward" and is optional, but the ≥0.5 threshold is relatively low. An attacker who has legitimately earned 0.5 reputation (feasible in weeks via uptime + storage) can then abuse snapshot publishing.
- **Recommended fix:** (1) Require matching snapshots from ≥5 publishers across ≥3 ASNs (consistent with sync peer requirements). (2) After snapshot bootstrap, the node MUST still perform full Merkle verification against non-snapshot sync peers — compare identity and proposal Merkle roots against at least 2 peers that were NOT snapshot publishers. (3) Consider raising the snapshot publishing threshold to 0.7 or 0.8.

---

### SYNC-R2-02
- **Severity:** CRITICAL
- **Title:** REPUTATION_CURRENT sync type lacks integrity verification — enables reputation injection
- **Location:** §5, State Snapshots (REPUTATION_CURRENT)
- **Description:** The new `REPUTATION_CURRENT` sync type returns `(node_id, reputation, last_assessment_timestamp)` tuples. The spec says "significant divergence (>0.1 for the same subject across peers) is flagged and the lower value used." But this only protects against *divergent* peers — if all sync peers agree on an inflated reputation (e.g., all are colluding), the syncing node accepts the inflated values. Unlike replaying historical REPUTATION_GOSSIP (which allows the node to independently compute reputation from signed assessments), REPUTATION_CURRENT is an opaque summary with no cryptographic proof. There is no Merkle tree over reputation state, and the `reputation_summary_hash` in snapshots is a SHA-256 of sorted tuples — but the syncing node has no way to verify the tuples match the hash without receiving *all* tuples.
- **Exploit scenario:** Attacker controls 3-of-5 sync peers. During REPUTATION_CURRENT sync, they serve inflated reputation for colluding nodes (e.g., claiming a fresh node has rep 0.8). The divergence check only triggers when sync peers disagree — if 3 agree on the inflated value and 2 serve the honest value, the majority wins. The syncing node then weights the colluding nodes' votes highly, potentially swinging governance decisions.
- **Recommended fix:** (1) REPUTATION_CURRENT responses MUST include the `reputation_summary_hash` and the responding peer's own reputation Merkle root. (2) On divergence, use the *minimum* value across all peers (spec partially addresses this: "the lower value used" — but only for >0.1 divergence). (3) Consider making REPUTATION_CURRENT a MAY optimization that falls back to full REPUTATION_GOSSIP replay when any divergence is detected.

---

### SYNC-R2-03
- **Severity:** HIGH
- **Title:** Graduated degraded mode exit (50% vote weight after 7 days) enables governance dilution attacks
- **Location:** §5, Sync Completeness — Degraded mode
- **Description:** After 7 days in full degraded mode, a node "MAY participate with 50% vote weight." This creates a new class of nodes: partially-synced voters with unreliable state. An attacker can intentionally keep nodes in degraded mode (by disrupting sync), wait 7 days, then use those nodes to vote with 50% weight on proposals the attacker wants to influence. The degraded nodes have incomplete identity and reputation data, so they can't properly evaluate whether they *should* vote — but they're now allowed to.
- **Exploit scenario:** Attacker controls enough network position to disrupt sync for a set of target nodes (selective DDoS on their sync peers). After 7 days, these nodes enter 50%-weight voting. The attacker then submits proposals and coordinates with the degraded nodes (who have stale/incomplete state and may not see counter-arguments or relevant context). The 50% weight dilutes honest votes. With 10 degraded nodes at 50% weight, the attacker adds 5 effective votes to governance.
- **Additional concern:** The spec says the node "halves its own vote's vote_time_reputation field" — but this is self-reported. A malicious node could simply not halve it and claim to be synced (though other nodes could detect the mismatch with sync_status).
- **Recommended fix:** (1) Nodes in degraded mode MUST complete at least phases 1-2 (identity + reputation) before the 50% weight activates — voting without identity state is meaningless. (2) Votes from degraded nodes MUST be tagged (the sync_status already provides this) and other nodes SHOULD apply the 50% reduction based on the voter's sync_status, not trust the voter to self-apply it. (3) Consider requiring degraded nodes to have completed phase 3 (proposals) before voting — they need to know what they're voting on.

---

### SYNC-R2-04
- **Severity:** HIGH
- **Title:** Identity Merkle tree interaction with §12 partition detection creates false positive amplification
- **Location:** §5, Identity Merkle Tree + §12, Partition Detection
- **Description:** The spec adds an identity Merkle tree but §12's partition severity classification only mentions "% proposal set difference." The identity Merkle root is compared during sync (§5) but there's no severity classification for identity divergence. If two nodes have matching proposal Merkle roots but divergent identity Merkle roots, what severity is this? The spec doesn't say. More critically: identity divergence is arguably *more severe* than proposal divergence (revoked keys still appearing valid), but the §12 severity table doesn't account for it.
- **Exploit scenario:** During a partition, identity state diverges (DID_REVOKE on one side, not the other). Post-partition, proposal Merkle roots may match (same proposals exist on both sides) but identity roots diverge. If nodes only classify partition severity by proposal divergence, they treat identity divergence as a non-event. Alternatively, if nodes treat any Merkle divergence as a partition event, the introduction of the identity Merkle tree doubles the surface area for false positive partition alerts — any missed DID_LINK or KEY_ROTATE message triggers identity divergence.
- **Recommended fix:** (1) Add explicit severity classification for identity Merkle divergence in §12 (even 1 identity divergence involving a DID_REVOKE should be HIGH severity). (2) Distinguish between additive divergence (missing DID_LINK — low severity, will converge) and subtractive divergence (missing DID_REVOKE — high severity, security-critical). (3) Specify that identity Merkle divergence MUST NOT be used as a partition detection signal if the divergent node has sync_status "syncing."

---

### SYNC-R2-05
- **Severity:** HIGH
- **Title:** Sync serving incentive is farmable via reciprocal fake sync requests
- **Location:** §5, Sync Serving Incentive
- **Description:** "Each sync response served counts as a peer interaction for the purposes of uptime reward calculation." Two colluding nodes can continuously exchange trivial SYNC_REQUEST/SYNC_RESPONSE messages to farm uptime interactions. The sync rate limit is 10 per peer per minute (or 50 for syncing nodes), but even at 10/minute, two nodes generate 14,400 "peer interactions" per day between them. Normal sync is ~4 requests per 15-minute cycle = 384/day. The colluders earn ~37× the uptime signal of a legitimate node.
- **Exploit scenario:** Attacker runs 2 nodes. They exchange empty SYNC_REQUEST (since_timestamp = now, so no messages returned) every 6 seconds. Each response counts as an uptime interaction. The nodes accumulate uptime reputation at accelerated rates. Since uptime reward is +0.001/day capped at 30 days, the raw reputation gain is small — but the interaction count inflates the nodes' apparent availability, making them preferred as sync peers by other nodes, amplifying their position for other attacks.
- **Recommended fix:** (1) Cap sync interactions that count toward uptime at a reasonable frequency (e.g., max 1 counted interaction per peer per 15 minutes, aligned with incremental sync). (2) Only count sync responses that actually return messages (non-empty `messages` array). (3) Require sync interactions to come from ≥3 distinct peers to count toward uptime.

---

### SYNC-R2-06
- **Severity:** HIGH
- **Title:** Buffered gossip during sync has no size bound — enables memory exhaustion DoS
- **Location:** §5, Sync Phases
- **Description:** "The node MUST buffer live gossip messages received during each phase and apply them only after the corresponding phase's sync data has been processed." There is no specified maximum buffer size. During a long sync (especially first-join), gossip accumulates continuously. A node syncing for hours or days must buffer all incoming gossip messages for each phase. In an active network (§15: ~111 reputation messages/second alone), buffering even 1 hour of gossip requires storing ~400K messages. The spec says buffered messages are applied "in timestamp order after each phase completes" — this requires sorting potentially millions of messages.
- **Exploit scenario:** An attacker floods the network with valid but high-volume gossip (e.g., PEER_ANNOUNCE every 5 minutes from many nodes, REPUTATION_GOSSIP every 15 minutes). A syncing node must buffer all of this. If the buffer grows unbounded, the node runs out of memory and crashes. On restart, it re-enters sync and the cycle repeats. This is especially effective against resource-constrained nodes.
- **Recommended fix:** (1) Define a maximum buffer size per phase (e.g., 100,000 messages or 100 MiB). (2) When the buffer fills, drop oldest messages (they'll be caught by the next incremental sync). (3) Alternatively, allow a "discard and re-sync" strategy: if the buffer exceeds capacity, skip buffering and perform an additional incremental sync pass after the phase completes.

---

### SYNC-R2-07
- **Severity:** MEDIUM
- **Title:** Snapshot timestamp manipulation enables selective history erasure
- **Location:** §5, State Snapshots
- **Description:** "After snapshot bootstrap, the node performs incremental sync from the snapshot's timestamp to catch up on messages published since the snapshot." If colluding snapshot publishers backdate the snapshot timestamp, the syncing node will perform incremental sync from the (false) recent timestamp, missing legitimate messages between the true snapshot creation and the false timestamp. Conversely, if they future-date it, the node starts incremental sync from the future, receiving nothing and believing it's current.
- **Exploit scenario:** Attackers publish snapshots with `snapshot_timestamp` set to 2 hours ago, but the actual Merkle roots reflect state from 24 hours ago. The syncing node bootstraps 24-hour-old state but only incrementally syncs the last 2 hours, missing 22 hours of messages. Those 22 hours might contain critical DID_REVOKE messages, proposal votes, or content flags.
- **Recommended fix:** (1) After snapshot bootstrap, the node MUST verify that the `snapshot_timestamp` is consistent with the sync peers' view — request the Merkle root at the claimed timestamp from non-snapshot peers. (2) Alternatively, perform incremental sync from `snapshot_timestamp - 1 hour` (safety margin). (3) Validate that snapshot_timestamp is recent (e.g., within 24 hours of current time).

---

### SYNC-R2-08
- **Severity:** MEDIUM
- **Title:** Graduated degraded exit interacts poorly with cold start headcount voting
- **Location:** §5 Degraded Mode + §8 Cold Start
- **Description:** During cold start, voting uses headcount mode (majority of known nodes must vote, >67% endorse). Degraded nodes at 50% weight participate in headcount counts but with reduced weight. The spec doesn't clarify how degraded 50%-weight nodes interact with cold start headcount voting. Does a degraded node count as a full participant for the "50% of active nodes have cast a vote" requirement? If yes, degraded nodes can satisfy the participation quorum while having impaired judgment. If no, degraded nodes during cold start are effectively disenfranchised.
- **Recommended fix:** Specify that during cold start, degraded nodes count toward the headcount quorum (they are active nodes) but their votes carry 50% weight in the endorsement calculation. This is the most consistent interpretation but should be explicit.

---

### SYNC-R2-09
- **Severity:** MEDIUM
- **Title:** STATE_SNAPSHOT has no expiry or freshness requirement
- **Location:** §5, State Snapshots
- **Description:** The STATE_SNAPSHOT message includes `snapshot_timestamp` but the spec doesn't specify a maximum age for snapshots that can be used for bootstrap. A syncing node could receive 3 matching snapshots that are all 30 days old. Bootstrapping from month-old state and then incrementally syncing 30 days of messages partially defeats the purpose of snapshots.
- **Recommended fix:** Syncing nodes MUST reject snapshots with `snapshot_timestamp` older than 24 hours. Snapshot publishers SHOULD publish at least once per 6 hours.

---

### SYNC-R2-10
- **Severity:** MEDIUM
- **Title:** Identity Merkle tree ordering creates deterministic DoS amplification
- **Location:** §5, Identity Merkle Tree
- **Description:** "Leaves sorted lexicographically by message id." Message IDs are SHA-256 hashes of the signing body. An attacker who can generate many DID_LINK messages (link many child keys) can force large Merkle tree recomputations across all nodes. Each DID_LINK adds a leaf, and sorted insertion into a Merkle tree requires reconstruction of O(log n) intermediate nodes. At scale, an attacker with 1,000 child keys forces 1,000 leaf insertions and tree rebalancing on every node in the network.
- **Exploit scenario:** Attacker generates 1,000 Ed25519 keypairs, computes VDFs for each (30,000 seconds = ~8 hours with parallelism), and links them all to one root via DID_LINK. Every node must add 1,000 leaves to their identity Merkle tree. Each incremental sync cycle must process these additions. The tree recomputation itself is cheap per-operation but the attacker repeats this from multiple root identities.
- **Mitigating factors:** VDF cost (~30s per key) bounds the rate. The gain dampening formula penalizes large key sets. But the Merkle tree computation cost is borne by *all* nodes, not just the attacker.
- **Recommended fix:** (1) Cap the number of authorized keys per identity for Merkle tree purposes (e.g., max 100 DID_LINK leaves per root). (2) Batch Merkle tree recomputation (recompute at most once per minute, not per message).

---

### SYNC-R2-11
- **Severity:** MEDIUM
- **Title:** Buffered gossip ordering across phases can create inconsistent application
- **Location:** §5, Sync Phases
- **Description:** "Buffered messages are applied in timestamp order after each phase completes." But gossip messages don't respect phase boundaries — a VOTE message (phase 3) may arrive during phase 1 sync. Is it buffered for phase 3? If so, the node must classify incoming gossip by phase and maintain separate buffers. The spec doesn't specify how gossip messages that span phases are classified and buffered. A DID_LINK (phase 1) arriving during phase 3 sync — is it applied immediately (phase 1 is complete) or buffered?
- **Recommended fix:** Specify: (1) Gossip messages for already-completed phases are applied immediately (they can be validated against committed state). (2) Gossip messages for the current or future phases are buffered and applied when that phase completes. (3) Clarify the classification: message type → phase mapping is already defined in the phase table.

---

### SYNC-R2-12
- **Severity:** LOW
- **Title:** Snapshot `active_node_count` and `identity_count` are unverifiable metadata
- **Location:** §5, STATE_SNAPSHOT payload
- **Description:** The snapshot includes `active_node_count`, `identity_count`, and `proposal_count` as plain integers. These are not covered by the Merkle roots and cannot be verified. A poisoned snapshot could claim 10,000 active nodes when only 100 exist, influencing the syncing node's governance calculations (quorum requirements scale with network size).
- **Recommended fix:** (1) Mark these fields as advisory/informational only. (2) After snapshot bootstrap, the node MUST compute these values from its own state (count identities in the identity Merkle tree, count proposals in the proposal Merkle tree) rather than trusting the snapshot's claims.

---

### SYNC-R2-13
- **Severity:** LOW
- **Title:** Sync serving penalty threshold ("refuse >50%") is ambiguous and gameable
- **Location:** §5, Sync Serving Incentive
- **Description:** "Nodes that refuse sync requests from >50% of requesting peers (measured over a rolling 24-hour window) SHOULD be penalized." This is (a) a SHOULD, (b) measured how? — a node doesn't know how many peers requested sync from it vs. other nodes, (c) a node can serve all requests with empty responses (technically not refusing), and (d) the 50% threshold is generous — a node can refuse half of all requests without consequence.
- **Recommended fix:** (1) Define "refuse" explicitly (timeout, error response, or empty response when the requester's Merkle root diverges). (2) Make this peer-local: each node tracks which peers responded to its sync requests and deprioritizes non-responsive peers. This is simpler and doesn't require global measurement.

---

### SYNC-R2-14
- **Severity:** LOW
- **Title:** Reputation Merkle tree conspicuously absent despite identity and proposal trees
- **Location:** §5, State Snapshots / Merkle-Based Divergence Detection
- **Description:** The spec now has Merkle trees for identity state and proposal state, plus a `reputation_summary_hash` in snapshots. But there's no reputation Merkle tree for incremental divergence detection. This means reputation divergence is only detectable via the coarse snapshot hash or by comparing individual REPUTATION_CURRENT tuples — there's no efficient Merkle-based narrowing for reputation state. This is the last major state category without efficient divergence detection.
- **Recommended fix:** Consider adding a reputation Merkle tree in a future revision. For v0, document this as an acknowledged gap. The REPUTATION_CURRENT mechanism is a reasonable interim solution but won't scale to networks with 10K+ nodes.

---

### SYNC-R2-15
- **Severity:** INFO
- **Title:** The sync protocol's complexity has reached a concerning threshold
- **Location:** §5 (entire section)
- **Description:** The sync protocol now includes: 5 phased sync with ordering constraints, 2 Merkle trees (identity + proposal), state snapshots with multi-publisher verification, REPUTATION_CURRENT shortcut, buffered gossip with per-phase classification, graduated degraded mode with time-based weight restoration, sync_status advertisement, backpressure with partition-aware jitter, mid-sync failure recovery with stale phase checks, sync serving incentives, elevated rate limits for initial sync, and incremental sync with rotating identity phases. This is a substantial amount of interacting machinery for a "bootstrap protocol." The interaction surface between these mechanisms is large enough that implementation bugs are near-certain. Each mechanism individually is reasonable; the aggregate is daunting.
- **Recommended fix:** (1) Consider whether snapshots + REPUTATION_CURRENT could replace the phased sync entirely for first-join (snapshot bootstrap → incremental sync only, skip full phase replay). (2) Provide a reference implementation or detailed pseudocode — the prose spec leaves too many interaction edge cases to implementer interpretation. (3) Consider splitting §5 into a separate sync specification document with its own versioning (this also addresses the open SYNC-20).

---

### SYNC-R2-16
- **Severity:** INFO
- **Title:** STATE_SNAPSHOT on GossipSub `/valence/peers` — volume concern at scale
- **Location:** §5, STATE_SNAPSHOT message type (§2 message inventory)
- **Description:** STATE_SNAPSHOT is broadcast on `/valence/peers` (same topic as PEER_ANNOUNCE, REPUTATION_GOSSIP, etc.). If many nodes with rep ≥0.5 publish snapshots (the spec says "SHOULD periodically publish"), this adds substantial traffic to an already busy topic. No publishing frequency is specified, so eager nodes might publish every 15 minutes. In a 1,000-node network where 40% have rep ≥0.5, that's 400 snapshots per interval.
- **Recommended fix:** (1) Specify a minimum interval between snapshot publications (e.g., every 6 hours). (2) Consider a separate GossipSub topic for snapshots, or make snapshots pull-only (requested via sync protocol, not broadcast).

---

## Cross-Cutting Concerns

### Interaction: Snapshots × Partition Detection × Identity Merkle Tree

The three new mechanisms (snapshots, identity Merkle tree, and partition detection exemptions for syncing nodes) create a complex interaction triangle. Consider: a partition heals, nodes on both sides have divergent identity Merkle roots. Node A is "synced" on partition-1, Node B is "synced" on partition-2. They connect and detect identity Merkle divergence. Is this a partition event (§12) or a sync event (§5)? Both nodes are "synced" in their own view. The identity Merkle divergence triggers partition detection (both are sync_status: "synced"), but the resolution mechanism is sync-based (fetch missing messages). The spec should clarify: does identity Merkle divergence between two synced nodes follow the partition detection path (§12) or the sync divergence path (§5)?

### Interaction: Degraded Mode Voting × Reputation Assessment

A degraded node at 50% weight votes on proposals. Other nodes must assess this voter's reputation to weight the vote. But the degraded node's reputation may itself be based on incomplete data (it hasn't completed all sync phases). When Node A evaluates degraded Node B's vote, does A use its own reputation assessment of B (which may not account for B's degraded state), or does the 50% weight already capture this? The spec says the node "halves its own vote's vote_time_reputation field" — so the weight reduction is in the vote message, not in the assessment. This means the voter self-reports its degradation. A malicious node could simply not halve the field. Other nodes should independently verify by checking the voter's sync_status.
