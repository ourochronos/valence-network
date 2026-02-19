# Adversarial Review: §5 State Reconciliation Protocol (Round 3)

**Reviewer:** Automated adversarial audit (Claude, subagent)
**Date:** 2026-02-18
**Spec version:** v0 (post-round-2 fixes)
**Scope:** §5 State Reconciliation, re-audit after R1 (22 findings) and R2 (16 findings) addressed

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 1 |
| HIGH     | 3 |
| MEDIUM   | 4 |
| LOW      | 3 |
| INFO     | 2 |
| **Total** | **13** |

**Overall assessment:** The R2 fixes are well-integrated. All 16 R2 findings have been addressed — 15 properly resolved, 1 acknowledged as an open question (SYNC-R2-14, reputation Merkle tree, deferred to §15). The spec is now substantially more robust. However, the REPUTATION_CURRENT minimum-across-all-peers rule has a critical flaw that allows a single malicious sync peer to force expensive full replays, defeating the optimization entirely. The post-snapshot non-publisher cross-check has an achievability gap in small/high-reputation networks. And several interaction effects between the accumulated mechanisms create subtle implementation pitfalls.

---

## Round 2 Finding Status

### Properly Resolved (15 of 16)

| Finding | Fix | Assessment |
|---------|-----|------------|
| SYNC-R2-01 (Snapshot poisoning) | Raised to ≥5 publishers from ≥3 ASNs, rep threshold to 0.7, added 2 non-publisher cross-check | **Resolved.** Significantly stronger. But cross-check has achievability concerns (see SYNC-R3-02). |
| SYNC-R2-02 (REPUTATION_CURRENT integrity) | Minimum across all 5 peers, fallback to full replay on >0.1 divergence | **Resolved in intent.** But the minimum rule is exploitable (see SYNC-R3-01). |
| SYNC-R2-03 (Graduated degraded mode) | Requires phases 1-2 complete for 50% weight; other nodes enforce via sync_status | **Resolved.** Clean separation of self-report vs. external enforcement. |
| SYNC-R2-04 (Identity Merkle + partition) | Severity classification added: additive=info, DID_REVOKE=critical | **Resolved.** |
| SYNC-R2-05 (Sync farming) | Capped at 1 counted interaction/peer/15min, non-empty only, ≥3 distinct peers/day | **Resolved.** Farming cost now exceeds legitimate earning rate. |
| SYNC-R2-06 (Buffered gossip unbounded) | 100,000 messages or 100 MiB cap per phase, oldest dropped | **Resolved.** |
| SYNC-R2-07 (Snapshot timestamp manipulation) | 24h max age, incremental sync from snapshot_timestamp − 1 hour | **Resolved.** 1-hour safety margin closes the backdating gap. |
| SYNC-R2-08 (Degraded + cold start) | Degraded nodes count toward headcount, 50% weight in endorsement calculation | **Resolved.** Explicit interaction specified. |
| SYNC-R2-09 (Snapshot expiry) | 24-hour freshness requirement | **Resolved.** |
| SYNC-R2-10 (Identity Merkle DoS) | Batched recomputation at most once per minute | **Resolved.** Bounds amplification. |
| SYNC-R2-11 (Buffered gossip ordering) | Completed-phase gossip applied immediately; current/future phases buffered | **Resolved.** Clear classification rule. |
| SYNC-R2-12 (Snapshot unverifiable metadata) | Fields marked advisory; node MUST compute from own state | **Resolved.** |
| SYNC-R2-13 (Sync serving penalty) | Replaced with peer-local tracking of non-responsive peers | **Resolved.** Simpler and more robust. |
| SYNC-R2-15 (Complexity) | Acknowledged. No structural simplification but no further increase. | **Resolved** (acknowledged). |
| SYNC-R2-16 (Snapshot on GossipSub) | Snapshots requested via sync protocol, not broadcast | **Resolved.** |

### Acknowledged / Deferred (1 of 16)

| Finding | Status | Notes |
|---------|--------|-------|
| SYNC-R2-14 (Reputation Merkle tree absent) | **Deferred to §15.** | Documented as an acknowledged gap. REPUTATION_CURRENT is the interim solution. Acceptable for v0. |

---

## New Findings

---

### SYNC-R3-01
- **Severity:** CRITICAL
- **Title:** A single malicious sync peer can force full REPUTATION_GOSSIP replay for all subjects, defeating the REPUTATION_CURRENT optimization
- **Location:** §5, REPUTATION_CURRENT
- **Description:** The spec states: "The syncing node MUST request [REPUTATION_CURRENT] from all 5 sync peers and use the **minimum** value for each subject across all responses. If any sync peer's response diverges by more than 0.1 for any subject from the minimum, the node MUST fall back to full REPUTATION_GOSSIP replay for that subject."

  A single malicious sync peer can report `reputation = 0.1` (the floor) for every subject. The minimum across all 5 peers becomes 0.1 for everyone. The other 4 honest peers reporting values like 0.5–0.8 diverge from the minimum by >0.1, triggering mandatory full REPUTATION_GOSSIP replay for **every single subject in the network**. This completely defeats the purpose of REPUTATION_CURRENT as a sync optimization — the attacker forces the most expensive sync path with zero cost to themselves.

  The spec acknowledges "colluding peers can only deflate reputation (which hurts their collaborators), not inflate it." But deflating *everyone* to trigger universal full replay is a DoS, not a reputation attack. The attacker doesn't need collaborators — they're not trying to change reputation, they're trying to make sync expensive.

- **Exploit scenario:** Attacker operates 1 node among the syncing node's 5 sync peers. During REPUTATION_CURRENT, the attacker reports floor values for all subjects. Result: the syncing node falls back to full REPUTATION_GOSSIP replay for every subject. In a 1,000-node network, this means replaying potentially millions of historical REPUTATION_GOSSIP messages instead of receiving 1,000 tuples. The REPUTATION_CURRENT optimization provides zero benefit whenever any single sync peer is malicious.

- **Recommended fix:** Change the aggregation from "minimum across all 5" to "minimum across the **middle 3 of 5**" (trimmed minimum — discard the highest and lowest, then take the minimum of the remaining 3). This tolerates 1 outlier on each end while remaining conservative. Alternatively, use median instead of minimum, and trigger full replay only when the **interquartile range** exceeds 0.1 (not when any single peer diverges from the minimum). The goal is to make the optimization robust to a single Byzantine peer among the 5.

---

### SYNC-R3-02
- **Severity:** HIGH
- **Title:** Post-snapshot non-publisher cross-check may be unachievable in small or high-reputation networks
- **Location:** §5, State Snapshots — post-snapshot verification
- **Description:** The spec requires: "After snapshot bootstrap, the node MUST perform Merkle root verification against at least **2 sync peers that were NOT snapshot publishers**." Sync peers must have `sync_status: "synced"` and are selected from the node's connected peers. Snapshot publishers are nodes with reputation ≥ 0.7.

  In a mature but small network (e.g., 30–80 nodes), most synced, well-connected nodes may have reputation ≥ 0.7. If these nodes publish snapshots (as they SHOULD per the spec), finding 2 synced peers who are NOT snapshot publishers becomes difficult. The spec says publishing is optional ("SHOULD"), but it's the only soft constraint preventing this deadlock.

  More subtly: the syncing node cannot easily verify which nodes are snapshot publishers before completing sync. It discovers publishers by requesting snapshots. But it needs non-publishers for cross-checking, and it doesn't have a complete picture of who has published until it's already deep into the snapshot bootstrap process.

- **Exploit scenario:** In a 50-node network where 30 nodes have rep ≥ 0.7 and all publish snapshots, a syncing node selects 5 sync peers — statistically, most or all will be publishers. The node cannot satisfy the 2-non-publisher requirement and must fall back to full phased sync, making snapshot bootstrap useless in practice for mature small networks.

- **Recommended fix:** (1) Clarify that the 2 non-publisher cross-check peers need not be from the original 5 sync peers — the node MAY connect to additional peers specifically for cross-checking. (2) Define "snapshot publisher" narrowly: a node is a publisher for this sync only if the syncing node actually received and used a snapshot from that specific node. Nodes that *could* publish but didn't aren't publishers. (3) Add a fallback: if the node cannot find 2 non-publisher cross-check peers after attempting N additional connections (e.g., 10), it MAY proceed with cross-checking against any 2 sync peers whose snapshots were NOT among the 5 used for bootstrap (even if those peers have published other snapshots).

---

### SYNC-R3-03
- **Severity:** HIGH
- **Title:** Gossip buffer phase classification creates a validation gap for cross-phase messages
- **Location:** §5, Gossip buffering during sync
- **Description:** The spec classifies gossip by message type → phase and states: "Messages for already-completed phases are applied immediately (they can be validated against committed state)." But some messages require cross-phase context for proper validation:

  - A VOTE (phase 3) arriving after phase 3 completes can be applied immediately, but its weight depends on the voter's reputation (phase 2). If the voter's reputation was updated by a REPUTATION_GOSSIP that arrived during phase 3 and was applied immediately (phase 2 was complete), the weight calculation is correct. But if the VOTE references a voter whose reputation was updated by a REPUTATION_GOSSIP that arrived during phase 4 (applied immediately since phase 2 is complete), the vote weight may shift retroactively.

  More critically: a DID_REVOKE (phase 1) arriving after phase 1 completes is applied immediately. This may invalidate votes (phase 3) that were already committed during phase 3 sync — votes from the now-revoked key. The spec doesn't address retroactive invalidation of already-committed phase data when a later-arriving identity message changes the foundation.

- **Exploit scenario:** During sync, phases 1-3 complete. A DID_REVOKE arrives via live gossip (applied immediately — phase 1 is complete). The revoked key had cast votes during phase 3 sync that were already accepted. These votes are now invalid but already committed. The node's state includes votes from a revoked key. Without explicit retroactive validation, this inconsistency persists.

- **Recommended fix:** Specify that when a DID_REVOKE is applied (whether during sync or normal operation), the node MUST scan committed state for messages from the revoked key with timestamps after the revocation's `effective_from` and invalidate them. This is already implied by "messages from the revoked key MUST be rejected going forward" (§1) but the retroactive case for already-committed sync data needs explicit treatment.

---

### SYNC-R3-04
- **Severity:** HIGH
- **Title:** Snapshot Merkle root verification is one-directional — catches omissions but not insertions
- **Location:** §5, State Snapshots + Merkle-Based Divergence Detection
- **Description:** Post-snapshot, the node compares its identity and proposal Merkle roots against 2 non-publisher cross-check peers. If roots diverge, the node discards the snapshot and falls back to full sync. But this only catches cases where the snapshot **omitted** data (the cross-check peers have messages the snapshot didn't include). It does NOT catch cases where the snapshot **inserted** phantom data — proposals or identity messages that don't exist on the honest network.

  If 5 colluding snapshot publishers include a phantom proposal in their snapshot, the syncing node's Merkle root will include this phantom. When cross-checking against 2 honest peers, the roots diverge. Per the spec, the node "MUST discard the snapshot and fall back to full phased sync." So the insertion IS caught — but only by triggering a full fallback, which is expensive. More importantly, the node cannot distinguish "snapshot had extra data" (poisoning) from "cross-check peers are missing data" (they're slightly behind). The spec says divergence → discard, which is safe but may cause unnecessary fallbacks in normal operation where cross-check peers are slightly stale.

- **Recommended fix:** When Merkle roots diverge post-snapshot, the node SHOULD perform Merkle tree narrowing (as in the existing divergence detection protocol) to identify the specific differing messages. If the differences are messages present in the snapshot but absent from cross-check peers, AND those messages cannot be independently verified (no valid signatures, or signed by unknown keys), discard the snapshot. If the differences are messages present on cross-check peers but absent from the snapshot, fetch and integrate them. This narrows the fallback to only the suspicious case.

---

### SYNC-R3-05
- **Severity:** MEDIUM
- **Title:** Incremental sync identity phase (every 4th cycle) creates a deterministic 45-minute worst-case revocation gap
- **Location:** §5, Incremental Sync
- **Description:** Identity messages are included in every 4th incremental sync cycle (~1 hour). Combined with the 15-minute sync interval, the worst case for a missed DID_REVOKE is: gossip delivery fails, then identity sync doesn't run for up to 45 more minutes (3 non-identity sync cycles). During this window, the node accepts messages from a revoked key.

  The spec improved this from 30 days (R1 finding SYNC-07) to ~1 hour, which is good. But the pattern is **deterministic** — if an attacker knows a node's sync schedule (observable from sync request patterns), they can time a revoked-key exploit to land in the gap between identity sync cycles.

- **Recommended fix:** Add jitter to which cycle includes identity sync (e.g., "every 3rd to 5th cycle, randomly selected" rather than deterministic "every 4th"). Alternatively, include identity sync whenever the previous cycle's gossip contained any DID_REVOKE messages (event-triggered rather than purely periodic).

---

### SYNC-R3-06
- **Severity:** MEDIUM
- **Title:** Snapshot `reputation_summary_hash` is not cross-checked against REPUTATION_CURRENT results
- **Location:** §5, State Snapshots + REPUTATION_CURRENT
- **Description:** The STATE_SNAPSHOT includes a `reputation_summary_hash` (SHA-256 of sorted (node_id, reputation) pairs). The REPUTATION_CURRENT sync returns individual (node_id, reputation, last_assessment_timestamp) tuples. The spec doesn't require the syncing node to verify that the REPUTATION_CURRENT results are consistent with the snapshot's `reputation_summary_hash`. These are two representations of the same state that could diverge if snapshot publishers and sync peers disagree about reputation, but no cross-validation is specified.

- **Recommended fix:** After completing REPUTATION_CURRENT sync, the node SHOULD compute a reputation summary hash from the received tuples and compare it against the snapshot's `reputation_summary_hash`. Divergence indicates either snapshot staleness or reputation manipulation and SHOULD trigger a warning log. This is advisory (not a hard gate) since reputation is locally computed and some divergence is expected.

---

### SYNC-R3-07
- **Severity:** MEDIUM
- **Title:** Gossip buffer overflow (oldest dropped) can systematically lose DID_REVOKE messages
- **Location:** §5, Gossip buffering during sync
- **Description:** The gossip buffer is capped at 100,000 messages or 100 MiB per phase, with "oldest messages dropped" when full. During a long sync, phase 1 (identity) completes first, so identity gossip is applied immediately. But if phase 3 sync takes a long time and the phase 3 buffer fills, older VOTE messages are dropped. Among these dropped messages could be votes that supersede earlier votes — the syncing node ends up with stale vote state.

  More concerning: the spec says dropped messages "will be caught by a follow-up incremental sync pass after the phase completes." But if the buffer overflow systematically drops messages from a specific time range (the oldest), and that time range contains critical state transitions, the follow-up sync may not prioritize those messages.

- **Recommended fix:** Priority-based buffering: within each phase's buffer, messages should be retained by type priority (DID_REVOKE > VOTE > PROPOSE > COMMENT for phase 3). When the buffer fills, drop lower-priority messages first. Alternatively, track the timestamp of the oldest dropped message and use it as `since_timestamp` for the follow-up incremental sync pass (ensuring the dropped range is re-fetched).

---

### SYNC-R3-08
- **Severity:** MEDIUM
- **Title:** Degraded mode graduated exit (phases 1-2 complete) + 50% weight creates an inconsistent quorum calculation across nodes
- **Location:** §5, Degraded mode + §8, Voting
- **Description:** A degraded node with phases 1-2 complete votes at 50% weight. Other nodes "MUST independently apply the 50% weight reduction based on the voter's `sync_status: 'degraded'`." But the voter's `sync_status` is learned via PEER_ANNOUNCE gossip. If some nodes have received the voter's latest PEER_ANNOUNCE (showing "degraded") and others haven't (or received an older one showing "synced" from before the node entered degraded mode), they'll apply different weights to the same vote.

  This is a specific instance of the general gossip-lag problem, but it's worse here because the weight difference is 2× (100% vs 50%), which is larger than typical reputation-driven weight variations.

- **Recommended fix:** The VOTE message itself should include the voter's `sync_status` at vote time (self-declared). Other nodes use `min(self_declared_weight, weight_from_observed_sync_status)` — they apply the 50% reduction if EITHER the vote self-declares degraded OR they independently observe the voter as degraded. This handles both directions of gossip lag.

---

### SYNC-R3-09
- **Severity:** LOW
- **Title:** "At most once per 6 hours" snapshot publishing has no minimum — publishers could publish once and never again
- **Location:** §5, State Snapshots
- **Description:** The spec says snapshot publishers "SHOULD publish signed state snapshots at most once per **6 hours**." This is an upper bound on frequency but no lower bound. A node could publish one snapshot and never publish again, making stale snapshots the only ones available. Combined with the 24-hour freshness requirement, this means snapshots become unavailable if no publisher publishes within 24 hours.

  In practice, the SHOULD language and the indirect incentive (healthy network benefits all) may be insufficient to ensure consistent snapshot availability.

- **Recommended fix:** Minor: add "Nodes meeting the snapshot publishing threshold SHOULD publish at least once per **12 hours** when they are synced and online." This is still SHOULD but sets an expectation for both bounds.

---

### SYNC-R3-10
- **Severity:** LOW
- **Title:** Sync serving quality tracking is purely local with no protocol-level consequence
- **Location:** §5, Sync Serving Incentive
- **Description:** "Peers that fail to respond to sync requests are deprioritized in future peer selection — both for sync and for general gossip routing." This is peer-local tracking only. A node that refuses sync to Node A but serves Node B has no network-wide consequence. The spec correctly identifies this as "simpler and more robust than global measurement" (fixing R2-13), but it means a targeted sync refusal attack — where an attacker serves everyone except specific victim nodes — is invisible to the network.

- **Recommended fix:** Acknowledge this as a known limitation. The local-only tracking is the right design choice for v0 (global measurement is complex and gameable), but document that targeted sync refusal is a gap that network-level reputation reporting could address in future versions.

---

### SYNC-R3-11
- **Severity:** LOW
- **Title:** REPUTATION_CURRENT `last_assessment_timestamp` is unused and ambiguous
- **Location:** §5, REPUTATION_CURRENT
- **Description:** REPUTATION_CURRENT responses include `(node_id, reputation, last_assessment_timestamp)` tuples. The `last_assessment_timestamp` is not referenced in any validation rule. It's unclear whether this is the timestamp of the responding peer's last assessment of the subject, or the timestamp of the most recent REPUTATION_GOSSIP the responding peer received about the subject. This ambiguity would cause divergent implementations.

- **Recommended fix:** Define `last_assessment_timestamp` as "the `assessment_timestamp` of the most recent REPUTATION_GOSSIP assessment this peer has issued for the subject." Specify its use: syncing nodes MAY use it to estimate staleness — if a peer's `last_assessment_timestamp` is significantly older than others', that peer's value may be stale and SHOULD be weighted lower in the aggregation.

---

### SYNC-R3-12
- **Severity:** INFO
- **Title:** The interaction between snapshot bootstrap and the α=0 cap creates a trust bootstrapping paradox
- **Location:** §5, State Snapshots + §9, Reputation Propagation
- **Description:** After snapshot bootstrap, the syncing node has reputation values for all peers (from REPUTATION_CURRENT). But per §9, α=0 (no direct observations) means peer-informed reputation is capped at 0.2 until 3 independent assessors from distinct ASNs are received. The snapshot doesn't count as direct observation. So immediately post-sync, the node effectively treats everyone as having rep ≤ 0.2, regardless of the REPUTATION_CURRENT values received.

  This is actually the correct security behavior — the node shouldn't trust claimed reputation without direct observation. But it means the REPUTATION_CURRENT values are effectively unused until the node accumulates direct observations, which takes hours. During this bootstrapping period, all peers appear equally untrustworthy (0.2), which degrades the node's ability to make governance decisions.

- **Recommended fix:** This is working as designed (the α=0 cap is a safety feature). Document this interaction explicitly: "After snapshot bootstrap, REPUTATION_CURRENT values serve as an upper bound that will be realized as the node accumulates direct observations. Initial governance participation quality is limited until sufficient observations are gathered."

---

### SYNC-R3-13
- **Severity:** INFO
- **Title:** No explicit handling of snapshot publishers that KEY_ROTATE during the syncing node's bootstrap window
- **Location:** §5, State Snapshots + §1, Key Rotation
- **Description:** A syncing node collects 5 matching snapshots from publishers. While performing post-snapshot incremental sync, one of the publishers KEY_ROTATEs. The node then encounters the KEY_ROTATE during sync. The snapshot's `from` field references the old key. If the node needs to re-verify the snapshot (e.g., after a cross-check failure), it must trace the KEY_ROTATE chain to validate the old signature. This works per the existing KEY_ROTATE rules but isn't explicitly addressed in the snapshot verification flow.

- **Recommended fix:** Minor: note that snapshot signature verification must account for KEY_ROTATE — a snapshot signed by a key that has since rotated is still valid if the KEY_ROTATE occurred after the snapshot's timestamp and the key chain is intact.

---

## Cross-Cutting Analysis

### Interaction: REPUTATION_CURRENT Minimum Rule × Snapshot Cross-Check × Full Replay Fallback

The REPUTATION_CURRENT minimum rule (SYNC-R3-01) interacts poorly with the snapshot cross-check. If a single malicious sync peer forces full REPUTATION_GOSSIP replay by reporting floor values, and the full replay takes significant time, the snapshot's 24-hour freshness window may expire during the replay. The node then needs a new snapshot, but its sync state is partially committed. The spec handles mid-sync failure (re-run phase 1 if stale, resume from last incomplete phase), but the interaction between "REPUTATION_CURRENT fallback to full replay" and "snapshot freshness expiry during replay" creates a pathological loop for nodes syncing into large networks with even one malicious sync peer.

### Interaction: Gossip Buffer Phase Classification × DID_REVOKE Immediate Application × Vote Retroactive Invalidation

When a DID_REVOKE arrives after phase 1 completes and is applied immediately, it must retroactively invalidate any committed votes from the revoked key (SYNC-R3-03). But the gossip buffer for phase 3 may contain additional votes from the revoked key that haven't been applied yet. The classification rule says these buffered phase 3 messages will be applied when phase 3 completes — but they should be discarded since the key is now revoked. The spec says "messages from the revoked key MUST be rejected going forward" (§1), which covers this case, but the interaction between "apply immediately" (for phase 1 messages) and "buffer and apply later" (for phase 3 messages) means implementations must check revocation status at both buffer-insertion time AND buffer-drain time.

### Interaction: Degraded Mode 50% Weight × Cold Start Headcount × Small Network

In a network that just crossed 16 nodes and activated governance, with several nodes in degraded mode, the cold start headcount rule requires "majority of known nodes must vote, >67% endorse." If 5 of 16 nodes are degraded (50% weight), they count toward headcount but dilute endorsement. A proposal needs 9 votes (majority of 16), with >67% endorsement by weight. If 5 degraded nodes vote endorse (2.5 effective weight) and 4 synced nodes vote endorse (4.0 effective weight), total endorsement = 6.5 out of 9 headcount votes with total weight 6.5 + 0 = 6.5. This interacts with the cold start rule "no reputation weighting" — during cold start, are degraded nodes at 50% weight or full weight (since cold start uses headcount, not reputation)? The spec says "No reputation weighting" during cold start, but the 50% weight for degraded nodes isn't reputation-based — it's sync-status-based. This needs clarification.

---

## Overall Assessment: Is §5 Now Implementable Without Ambiguity?

**Almost.** The section has improved dramatically across three rounds. The major structural issues (eclipse attacks, unbounded sync, missing Merkle trees, no incentives) are resolved. The remaining issues are:

1. **One critical flaw** (SYNC-R3-01) that makes REPUTATION_CURRENT useless against even a single Byzantine peer. This needs fixing before implementation — the optimization is important for scalability and it currently has no Byzantine tolerance.

2. **A few implementation ambiguities** that would cause divergent implementations:
   - REPUTATION_CURRENT `last_assessment_timestamp` semantics (SYNC-R3-11)
   - Degraded 50% weight interaction with cold start headcount mode
   - Retroactive invalidation of committed state when DID_REVOKE arrives post-phase-1 (SYNC-R3-03)

3. **Achievability concerns** with the 2-non-publisher cross-check in small networks (SYNC-R3-02), which would cause implementations to frequently fall back to full sync, making snapshots a dead feature in practice for networks under ~100 nodes.

**The section is implementable** with the caveats above. An implementer following the spec literally would produce a correct system, but one where REPUTATION_CURRENT provides no benefit (always falls back to full replay) and snapshot bootstrap fails in small networks. The remaining findings are fixable without structural changes to the protocol.

**Complexity verdict:** §5 is complex but the complexity is justified — each mechanism addresses a real attack or scalability concern. The interaction effects are manageable with careful implementation. A reference implementation or detailed pseudocode (as suggested in R2) would significantly reduce implementation risk.
