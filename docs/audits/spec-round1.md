# Valence Network v0 Protocol — Adversarial Review Findings

**Reviewer:** Subagent (spec-review)
**Date:** 2025-02-18
**Documents reviewed:** v0.md, architecture.md, principles.md, protocol.md, evolution.md, roadmap.md, conformance-tests.md

---

## CRITICAL ISSUES

### CRIT-01: DID_LINK persistence and discovery for late-joining nodes
**Severity:** Critical
**Location:** §1 Identity Linking, §5 Gossip
**Description:** When a child key B is linked to root A via DID_LINK, and later B performs KEY_ROTATE to B', the spec says B' is implicitly linked. But how does a node joining the network AFTER the KEY_ROTATE discover the A→B→B' relationship?

The DID_LINK(A→B) message was broadcast once when the link was created. If a node joins after that message has aged out of the 24-hour gossip window (§5), and only sees KEY_ROTATE(B→B'), it has no way to know that B was linked to A, and therefore cannot infer that B' inherits that link.

**Attack scenario:**
1. Attacker sees DID_LINK(A→B) and KEY_ROTATE(B→B') in gossip
2. Attacker submits DID_LINK(attacker_root→B') claiming B' for their own identity
3. New nodes joining after the original DID_LINK(A→B) has aged out see:
   - KEY_ROTATE(B→B') — proves B and B' are the same key lineage
   - DID_LINK(attacker→B') — appears valid if they haven't seen DID_LINK(A→B)
4. Network splits: old nodes know B'→A, new nodes think B'→attacker

**Root cause:** DID_LINK and DID_REVOKE messages have no persistence requirement beyond the 24-hour gossip window. Identity relationships are foundational but treated as ephemeral gossip.

**Suggested fix:**
1. **Identity relationship retention:** Nodes MUST retain all DID_LINK and DID_REVOKE messages indefinitely (or at least for the lifetime of the identity). They are not subject to the 24-hour gossip age limit.
2. **Sync protocol extension:** Add IDENTITY_SYNC_REQUEST / IDENTITY_SYNC_RESPONSE to `/valence/sync/1.0.0` allowing nodes to request the full identity linking history for a given root key.
3. **Periodic re-broadcast:** Root identities SHOULD re-broadcast their DID_LINK messages every 30 days (similar to PEER_ANNOUNCE frequency) to ensure new nodes can discover relationships.

Without this, identity linking is fundamentally broken for any network that runs longer than 24 hours with node churn.

### CRIT-02: KEY_CONFLICT message schema undefined
**Severity:** Critical
**Location:** §1, §2 message inventory
**Description:** The spec mentions KEY_CONFLICT in the message type inventory (§2 table) and describes when it's used (§1 partition merge), but provides no schema/payload definition. Implementers cannot construct or validate this message.
**Suggested fix:** Add KEY_CONFLICT schema to §1:
```json
{
  "type": "KEY_CONFLICT",
  "payload": {
    "old_key": "<hex_public_key>",
    "rotate_message_1_id": "<message_id>",
    "rotate_message_2_id": "<message_id>",
    "new_key_1": "<hex_public_key>",
    "new_key_2": "<hex_public_key>"
  }
}
```

### CRIT-02: Circular dependency in suspension resolution
**Severity:** Critical
**Location:** §1 KEY_ROTATE partition handling
**Description:** When KEY_ROTATE conflict is discovered, "both new keys MUST be suspended — reputation frozen at floor (0.1), voting weight zeroed — until the conflict is resolved via a standard governance proposal." But if voting weight is zeroed, the suspended identity cannot vote on the governance proposal to resolve its own suspension. If multiple identities are in conflict, quorum may become unreachable.
**Attack surface:** An attacker could deliberately create KEY_ROTATE conflicts during a partition to lock identities out of governance.
**Suggested fix:** Clarify that:
1. Suspended keys can still VOTE (votes have zero weight) to signal intent
2. The governance proposal to resolve the conflict is decided by non-suspended identities
3. Or: suspension only affects proposal authorship and storage, not voting (voting weight reduced but not zeroed)

### CRIT-03: Identity linking enables vote weight concentration attack
**Severity:** Critical (governance attack surface)
**Location:** §1 Identity Linking, §7 Votes
**Description:** An operator with N nodes can link them all as authorized keys under one root identity. The identity gets:
- N× observation count (faster α growth to 0.6 max direct weight)
- N× storage challenges passed (faster reputation gain)
- N× adoption messages sent/received (if nodes adopt proposals from different sources)
All reputation accrues to ONE identity that casts ONE vote, but the identity accumulates reputation N× faster than an independent node because it has N keys generating reputation events.

While linking sacrifices vote multiplication (N votes → 1 vote), the accelerated reputation growth means the single vote carries more weight. A wealthy operator could deploy 50 nodes, link them all, grind reputation aggressively, and achieve outsized voting power compared to 50 independent nodes that might be flagged for collusion.

**Attack scenario:**
1. Deploy 50 nodes (expensive but feasible for a determined adversary)
2. Link them all as authorized keys under one root
3. Have each node participate actively (storage, proposals, voting)
4. Root identity accumulates reputation from 50 nodes' worth of activity
5. Root identity's single vote now has massive weight, bypassing collusion detection

**Mitigation gap:** §10 collusion detection exempts linked keys, as intended. But there's no cap on reputation gain rate per identity regardless of how many authorized keys contribute.

**Suggested fix:** Add a reputation gain dampening factor for identities with multiple authorized keys:
```
effective_gain = raw_gain × dampening_factor
dampening_factor = 1.0 / sqrt(authorized_key_count)
```
This ensures linking doesn't amplify reputation accumulation beyond what a single node could achieve.

**Alternative fix:** Cap observation count growth per identity: α growth based on unique peer interactions, not raw message count. 50 authorized keys talking to the same 20 peers shouldn't give 50× α growth.

---

## HIGH SEVERITY ISSUES

### HIGH-01: Timestamp tolerance asymmetry under clock skew
**Severity:** High
**Location:** §2 Timestamp Validation
**Description:** The spec says "Two nodes both within ±5 minutes of true time can disagree by up to 10 minutes" and recommends NTP + warnings for >2 min skew. But there's no MUST requiring nodes to attempt clock synchronization, and no mechanism to detect or penalize nodes with bad clocks beyond local rejection.

A node with a clock 4:59 fast can send messages that a node with a clock 4:59 slow will reject (9:58 total skew). This creates asymmetric propagation: some peers accept, others reject.

**Attack surface:** An attacker can deliberately skew their clock to control which peers see their messages. Proposals and votes with selective visibility.

**Suggested fix:** 
1. Elevate "Nodes SHOULD use NTP" to "Nodes MUST attempt NTP synchronization on startup and log warnings for skew >1 minute"
2. Add a gossip-based clock offset estimation: nodes include a `sender_timestamp` in PEER_ANNOUNCE, receivers compare to their own clock and compute median offset from peers. Nodes with >3 min offset from the peer-estimated median SHOULD reduce reputation of the skewed node.

### HIGH-02: Storage challenge gaming via selective shard acceptance
**Severity:** High
**Location:** §6 Storage challenges
**Description:** The spec says "Challengers MUST be able to independently verify the proof, either by holding the shard directly or by reconstructing it from sufficient data shards." But there's no requirement that challengers actually possess shards before issuing challenges.

A colluding ring could:
1. Node A claims to store shard S
2. Node B (A's confederate) "challenges" A but doesn't actually possess shard S
3. A provides any random hash as the proof
4. B accepts it (no verification possible, B doesn't have the shard)
5. A earns +0.001 reputation per fake challenge passed

The "challenge frequency SHOULD be once per shard per day" is vague — should from whom? If challenges come only from colluding peers, the whole mechanism is defeated.

**Mitigation gap:** The spec assumes honest verification but doesn't enforce it.

**Suggested fix:**
1. Challenges MUST only be issued by nodes that provably hold the shard (the challenger includes their own copy of the challenged window in the challenge, encrypted/committed, and reveals after the response)
2. Or: Challenges are probabilistic — a challenger picks a shard they hold, randomly selects another node claiming to hold the same shard (via SHARD_QUERY_RESPONSE), and challenges them. The challenger's possession is verifiable because they can demonstrate the window bytes.
3. Add a challenge verification message: after receiving STORAGE_PROOF, the challenger broadcasts CHALLENGE_VERIFIED or CHALLENGE_FAILED (signed), allowing third parties to track challenge success rates and penalize nodes with suspiciously high pass rates from suspicious challengers.

### HIGH-03: VDF proof reuse attack
**Severity:** High
**Location:** §3 Auth Handshake, §9 VDF
**Description:** The spec says "VDF proofs MUST be fresh — verified during the auth handshake, not pre-shared." But the auth handshake verification happens AFTER the proof is submitted in AUTH_RESPONSE. There's no mechanism to ensure the proof was computed recently or ties to the specific connection.

An attacker could:
1. Compute one VDF proof (30 seconds)
2. Reuse it across 1000 AUTH_RESPONSE messages to different peers
3. Each peer accepts the proof because it verifies correctly for the public key

The "Maximum 50 VDF verifications per day" limit is per-verifier, not per-prover. A single attacker can flood the network with AUTH_RESPONSE messages containing the same valid VDF proof, forcing every node to verify it up to 50 times.

**Suggested fix:**
1. VDF `input_data` should include a connection-specific nonce: `input_data = pubkey_bytes || nonce_from_AUTH_CHALLENGE`. This binds the proof to the specific handshake.
2. Or: Add a VDF timestamp field and reject proofs older than 24 hours. Nodes must recompute VDF at least daily.
3. Clarify that "fresh" means "computed for this specific auth handshake" and update test vectors accordingly.

### HIGH-04: Partition merge deterministic tie-break can be gamed
**Severity:** High
**Location:** §11 Partition Detection, protocol change resolution
**Description:** For conflicting protocol change proposals during partition merge, the spec uses timestamp priority (earlier `voting_deadline` wins) with lexicographic `id` tie-break. Since `id` = SHA-256(signing_body) and signing_body includes `timestamp`, an attacker who controls proposal timing can grind for favorable `id` values.

If two partitions P1 and P2 each ratify a conflicting protocol change:
1. P1 proposal has `voting_deadline` = T
2. P2 proposal has `voting_deadline` = T (same)
3. Tie-break by `id` (lexicographic)

An attacker in P2 can grind `timestamp` values (millisecond precision) to generate thousands of candidate proposals, compute their `id`, and submit the one with the lowest lexicographic hash. This is cheap (SHA-256 grinding) and gives deterministic priority after a partition heal.

**Attack scenario:** During a partition, an adversary knows the "good" protocol change has `voting_deadline` = T. They grind proposals with the same deadline but malicious content, searching for an `id` that sorts lower. If they succeed, their malicious proposal wins the tie-break.

**Mitigation gap:** Deterministic tie-breaking based on grindable inputs.

**Suggested fix:**
1. Add an additional tie-break: count of ADOPT messages before the partition. The proposal with more widespread adoption pre-merge wins.
2. Or: Partition merge for protocol changes requires human intervention (no automatic tie-break). Both proposals are flagged, and the network must vote on a resolution proposal.
3. Or: Use `from` (proposer public key) as the tie-break — not grindable, since rotating keys costs reputation continuity.

### HIGH-05: Cold start headcount floor creates quorum cliff
**Severity:** High
**Location:** §7 Cold Start, quorum evaluation
**Description:** A proposal submitted during cold start (first 30 days after governance activation) whose `voting_deadline` falls after cold start ends MUST meet TWO conditions:
1. Weighted voting threshold (0.67+)
2. At least 50% of active nodes voted

At the cold start boundary (day 30), there's a sudden quorum cliff. A proposal submitted on day 29 with a 14-day deadline expires on day 43. If the network has 20 active nodes and only 9 voted (all high-rep), the proposal fails the 50% headcount floor despite potentially meeting weighted quorum.

**Problem:** The 50% headcount requirement continues indefinitely for cold-start-submitted proposals, even if they're evaluated years later when the network has 10,000 nodes. This creates a permanent disadvantage for early proposals.

**Suggested fix:**
1. Clarify that the 50% headcount floor applies only if the network is still below 100 active nodes when the deadline expires. Above 100 nodes, the normal weighted quorum takes precedence.
2. Or: The 50% headcount floor expires 60 days after cold start ends (grace period for transition).

### HIGH-06: Tenure penalty gaming via strategic inactivity
**Severity:** High
**Location:** §10 Anti-Gaming, tenure
**Description:** Tenure penalties kick in after 6 consecutive active cycles, with −5%/cycle vote weight reduction. Skipping a cycle decays tenure by 1. To go from tenure 6 to tenure 3, you skip 3 cycles (90 days).

During those 90 days, inactivity decay is −0.02/month = −0.06 total. If a node at reputation 0.8 skips 3 cycles:
- Tenure: 6 → 3 (reset by inactivity)
- Reputation: 0.8 → 0.74 (−0.06)
- Vote weight penalty: 5%×1 → 0% (tenure reset)

The reputation loss (−0.06) is recovered in 3 days under normal velocity limits (0.02/day). The 90-day inactivity cycle nets a gain: tenure reset with negligible reputation cost.

**Gaming strategy:** High-tenure nodes oscillate: 6 active cycles (accumulate power), 3 inactive cycles (reset tenure), repeat. The inactivity cost is trivial compared to the vote weight benefit.

**Suggested fix:**
1. Increase inactivity decay: −0.02 per month → −0.05 per month. 3-month absence = −0.15 reputation, taking 7.5 days to recover.
2. Or: Tenure decay is non-linear. Skipping 1 cycle decays tenure by 1, but skipping consecutive cycles decays faster (2nd cycle: −2, 3rd: −3). Full reset requires 6 months of inactivity at increasing cost.
3. Or: Tenure penalty doesn't reset to zero — it decays toward a floor. Once you've hit tenure 6, your vote weight penalty never goes below 2.5% even after inactivity.

### HIGH-07: Manifest hash collision vulnerability
**Severity:** High
**Location:** §6 Content Delivery, manifest_hash
**Description:** The `manifest_hash` construction is:
```
SHA-256(shard_hashes_sorted_concatenated || content_hash)
```
All hex strings as UTF-8 bytes, no delimiters. For N shards, this is:
```
SHA-256("hash1hash2...hashNcontent_hash")
```
Each hash is 64 hex chars (32 bytes). Total input: `64N + 64` bytes.

**Collision scenario:** Shards aren't fixed-length. If `shard_size` is manipulated or differs between shards, an attacker could craft:
- Set A: 5 shards, content_hash X → manifest M
- Set B: 5 different shards, content_hash Y → manifest M (collision)

Because the hash inputs are concatenated without delimiters, a boundary ambiguity exists: `["aaa", "bbb"]` concatenated = `"aaabbb"` = `["aa", "abbb"]` concatenated.

**Actual risk:** SHA-256 collisions are computationally infeasible, BUT the lack of delimiters creates ambiguity about which byte sequence corresponds to which shard. If shard count or shard hash length is variable, the manifest becomes ambiguous.

**Suggested fix:**
1. Include shard count in the manifest: `SHA-256(shard_count || shard_hashes || content_hash)` where shard_count is a 4-byte big-endian integer.
2. Or: Use structured hashing: `SHA-256(len(hash1) || hash1 || len(hash2) || hash2 || ... || content_hash)` (canonical encoding).
3. Or: Document that shard hashes are always 32 bytes (64 hex chars) and the concatenation is unambiguous because lengths are fixed.

**Conformance test gap:** MANIFEST-01 doesn't test for ambiguous cases (variable shard count or mixed shard sizes).

---

## MEDIUM SEVERITY ISSUES

### MED-01: α scaling creates information asymmetry amplification
**Severity:** Medium
**Location:** §8 Reputation Propagation, α computation
**Description:** α = min(0.6, observation_count / 10). With 6+ observations, a node relies 60% on direct experience, 40% on peer attestations.

For a well-connected node with 100 peers, accumulating 6 observations is trivial (6 PEER_ANNOUNCE messages). α → 0.6 quickly.

For an isolated node with 5 peers, accumulating 6 observations takes longer. Until then, α < 0.6 and the node relies more heavily on peer attestations, which may be stale or incomplete.

**Matthew effect:** Well-connected nodes build reputation faster (more observations → higher α → more weight on their own positive direct experience). Isolated nodes are slower to trust their own observations.

This is noted in §14 as an "acknowledged limitation," but it's more severe than implied. An attacker with strong connectivity can bootstrap reputation faster than honest but poorly-connected nodes.

**Suggested fix:**
1. Cap observation count growth per time unit: max 1 observation per peer per hour. This prevents a burst of PEER_ANNOUNCE messages from immediately maxing out α.
2. Or: α scaling includes a time component: α = min(0.6, (observation_count / 10) × (days_active / 30)). New nodes can't max α instantly.

### MED-02: SHARD_QUERY response not rate-limited
**Severity:** Medium
**Location:** §6 Content Delivery, shard discovery
**Description:** SHARD_QUERY / SHARD_QUERY_RESPONSE exchange is defined but has no rate limiting. An attacker can flood the network with SHARD_QUERY messages, forcing nodes to respond with SHARD_QUERY_RESPONSE.

Each response lists available shard indices + hashes (potentially large). If a node stores shards for 100 proposals, the response is 100 × (number of shards per proposal) × (shard hash size).

**DoS vector:** Query every node for every content_hash, exhaust bandwidth.

**Suggested fix:**
1. Add rate limit: max 10 SHARD_QUERY messages per sender per minute.
2. Paginate SHARD_QUERY_RESPONSE: if a node stores shards for >10 proposals matching the query, return the first 10 and require subsequent queries.

### MED-03: Abstain votes counted for quorum but not threshold creates strategic ambiguity
**Severity:** Medium
**Location:** §7 Votes, abstain handling
**Description:** Abstain votes count toward quorum but not toward endorse/reject ratio. This is intentional (allows "I showed up but no strong opinion"). But it creates a strategic ambiguity:

Standard quorum: `max(active_nodes × 0.1, total_rep × 0.10)`. If 30% of the network abstains, they contribute to quorum (participation) but don't affect threshold (outcome). A minority of high-rep endorsers can ratify a proposal if enough high-rep nodes abstain.

**Scenario:**
- Total network rep: 10.0
- Quorum: 1.0
- 5 nodes (rep 0.3 each) endorse → weighted_endorse = 1.5
- 10 nodes (rep 0.5 each) abstain → contribute to quorum but not threshold
- 0 nodes reject → weighted_reject = 0
- Threshold: 1.5 / 1.5 = 1.0 ≥ 0.67 ✓
- Quorum: 1.5 + 5.0 = 6.5 ≥ 1.0 ✓

Proposal ratifies with only 15% of reputation endorsing, because abstentions inflated the quorum without affecting the outcome.

This isn't necessarily broken — it's a design choice. But it should be acknowledged in §7 and potentially in Open Questions if it's considered a risk.

**Suggested fix:** Document this clearly as intended behavior. Or: abstentions count half-weight for quorum (participation credit) but still don't affect threshold.

### MED-04: Proposal supersession chain has no depth limit
**Severity:** Medium
**Location:** §6 Proposals, supersession
**Description:** Proposals can supersede previous proposals via the `supersedes` field. The spec says "edits are expressed as new proposals with `supersedes` pointing to the original proposal's ID." But there's no limit on chain depth.

An attacker could create a supersession chain of depth 1000:
```
P1 → P2 (supersedes P1) → P3 (supersedes P2) → ... → P1000 (supersedes P999)
```

When evaluating P1000, nodes must traverse the entire chain to understand its lineage. This is a DoS vector (computational cost) and a UX disaster (understanding provenance).

**Suggested fix:**
1. SHOULD limit: implementations SHOULD display only the last 5 supersessions in the chain. Deeper history is available but not shown by default.
2. Or: MUST limit: proposals MUST NOT supersede proposals that themselves supersede others (max depth 2). Express deeper edits by superseding the latest in the chain, not the root.
3. Add a `supersedes_depth` field to PROPOSE messages (required if supersedes is set), allowing nodes to reject proposals with excessive depth.

### MED-05: KEY_ROTATE grace period is global, not per-peer
**Severity:** Medium
**Location:** §1 Key Rotation
**Description:** "After rotation, messages signed by the old key MUST be rejected (after a 1-hour grace period to allow propagation)." The 1-hour timer starts when... what? When the KEY_ROTATE message is created (timestamp)? When a node receives it? When a node processes it?

If it's timestamp-based, a node that receives the KEY_ROTATE message 55 minutes after it was created has only 5 minutes to process messages from the old key.

If it's receipt-based per-node, different nodes have different grace period windows, causing asymmetric message propagation.

**Suggested fix:**
1. Clarify: grace period is `KEY_ROTATE.timestamp + 3600000 ms`. All nodes use the message timestamp, not local receipt time.
2. Extend grace period to 2 hours (gossip convergence worst-case is acknowledged as ~15 min; 1 hour is conservative but tight for degraded conditions).

### MED-06: Constitutional proposal cooling period enforcement undefined
**Severity:** Medium
**Location:** §7 Constitutional Proposals
**Description:** Constitutional proposals have a "30-day cooling period between ratification and activation." But the spec doesn't define:
1. When does the cooling period start? When the proposal first reaches threshold? When the voting_deadline expires? When a specific node locally ratifies it?
2. What does "activation" mean? When nodes adopt it? When it takes effect (EOL of previous version)?
3. How is the cooling period enforced? Nodes MUST NOT adopt until 30 days have passed since... what event?

**Suggested fix:**
1. Define cooling period start: `cooling_start = voting_deadline` (deterministic, all nodes agree)
2. Nodes MUST NOT broadcast ADOPT messages for constitutional proposals until `current_time >= cooling_start + 30 days`
3. Update constants table: "Constitutional cooling period: 30 days after voting_deadline"

### MED-07: Erasure coding shard health is non-mandatory
**Severity:** Medium
**Location:** §6 Content Delivery, shard health
**Description:** "Nodes that store shards SHOULD periodically verify that at least `data_shards + 1` copies of the full shard set remain available across known peers." This is a SHOULD, not a MUST.

If no nodes perform shard health checks, proposals with erasure-coded artifacts can become irretrievable when storage nodes churn. The network has no guarantee of shard availability.

**Suggested fix:**
1. Elevate to MUST for protocol change proposals: "Nodes storing shards for protocol change proposals MUST verify shard health at least once per 7 days."
2. For content proposals, leave as SHOULD but document the risk: "Content proposals with poor shard replication may become irretrievable. Nodes evaluating proposals SHOULD check shard availability before adoption."

### MED-08: Activity multiplier inputs are gameable
**Severity:** Medium
**Location:** §7 Network Phases, activity multiplier
**Description:** Activity multiplier reduces standard governance sustain period from 7 days (min activity) to 3 days (max activity). Inputs:
- `adopted_proposals` (≥3 ADOPT messages)
- `earned_rep_delta` (sum of rep gains, capped at 1.0)
- `challenge_pairs` (distinct challenger/challenged pairs)

All inputs are derived from publicly observable messages, which means they're forgeable by colluding nodes:
1. Colluders submit low-quality proposals and send each other ADOPT messages → inflates `adopted_proposals`
2. Colluders issue fake storage challenges to each other (per HIGH-02) → inflates `challenge_pairs`
3. Earned rep is harder to fake but can be amplified via the identity linking attack (CRIT-03)

**Attack scenario:** A colluding group wants to rush standard governance activation (bypass the 7-day sustain period). They:
1. Submit 3 garbage proposals
2. Each node in the group sends ADOPT(success=false) for all 3 → `adopted_proposals = 3`
3. Issue 10 fake storage challenges between pairs → `challenge_pairs = 10`
4. Activity multiplier → 1.0 + min(1.33, 3×0.33 + 0 + 10×0.1) = 1.0 + 1.33 = 2.33
5. Sustain period drops to 3 days instead of 7

**Mitigation gap:** Collusion detection penalties apply but don't prevent the governance acceleration.

**Suggested fix:**
1. Cap `adopted_proposals` count to proposals with ≥3 ADOPT messages from distinct ASNs (harder to fake with geographic diversity)
2. Weight `challenge_pairs` by challenger reputation: only challenges from nodes with rep >0.4 count (new Sybils can't inflate this)
3. Or: Remove activity acceleration entirely for the constitutional threshold (30 days fixed, no shortcuts)

---

## LOW SEVERITY / AMBIGUITY ISSUES

### LOW-01: "Active node" definition inconsistency
**Severity:** Low (ambiguity)
**Location:** §7 Network Phases, §8 Inactivity
**Description:** §7 defines `active_nodes` as "nodes with a PEER_ANNOUNCE received within the last 30 minutes." §8 inactivity decay says a node is "active if it has authored and broadcast at least one signed protocol message (VOTE, PROPOSE, ADOPT, STORAGE_PROOF, or PEER_ANNOUNCE) within the window."

These are different definitions:
- §7: active = received a PEER_ANNOUNCE (perspective of the observer)
- §8: active = authored a message (perspective of the subject)

For reputation inactivity decay, the definition is self-referential (a node is active if it broadcast something). For governance quorum, it's observational (a node is active if I've heard from it).

**Suggested fix:** Align terminology. Use "active_nodes" for governance (observed PEER_ANNOUNCE) and "participating node" for inactivity (authored a protocol message). Update references in §8 to clarify.

### LOW-02: Voting cycle definition vs. tenure measurement unclear
**Severity:** Low (ambiguity)
**Location:** §10 Anti-Gaming, tenure
**Description:** "A voting cycle is a rolling 30-day window. A node is 'active in a cycle' if it has voted on or had proposals ratified during that window."

"Had proposals ratified" is ambiguous:
1. Does it mean proposals authored by the node?
2. Does it mean proposals the node voted on that were ratified?
3. Does it mean proposals the node adopted?

**Suggested fix:** Clarify: "active in a cycle = the node authored at least one VOTE, PROPOSE, or ADOPT message during that 30-day window."

### LOW-03: Peer backoff maximum timing undefined
**Severity:** Low (ambiguity)
**Location:** §13 Error Handling, unresponsive peers
**Description:** "Nodes SHOULD implement exponential backoff for unresponsive peers, with a maximum retry interval of 10 minutes." But there's no spec for the backoff formula.

Is it 1s, 2s, 4s, 8s, ... → 10 min cap? Or 10s, 20s, 40s, ... → 10 min cap? Different implementations will have different behaviors.

**Suggested fix:** Provide a reference formula: "Initial retry: 5 seconds. Subsequent retries: previous × 2, capped at 10 minutes. Example: 5s, 10s, 20s, 40s, 80s, 160s, 320s, 600s (10 min), 600s, ..."

### LOW-04: GossipSub configuration parameters missing
**Severity:** Low (completeness)
**Location:** §3 Transport, §5 Gossip
**Description:** The spec says "v0 uses libp2p as the transport layer" and lists GossipSub topics, but doesn't specify critical GossipSub parameters:
- D (target peer count per topic)
- D_low / D_high (peer count bounds)
- fanout (peers to gossip to when not subscribed)
- heartbeat interval
- message cache size (for IHAVE/IWANT)

Different implementations will use different defaults, causing mesh instability.

**Suggested fix:** Add a GossipSub configuration section to §3 or §5:
```
GossipSub parameters (SHOULD use these values):
- D = 6
- D_low = 4, D_high = 12
- fanout = 6
- heartbeat = 1 second
- message cache TTL = 3 heartbeats
```

### LOW-05: Reputation gossip batch size "10 random peers" is per-message or total?
**Severity:** Low (ambiguity)
**Location:** §8 Reputation Propagation
**Description:** "Nodes SHOULD share reputation gossip every 15 minutes, covering 10 random peers per message."

Does "10 random peers per message" mean:
1. Each REPUTATION_GOSSIP message includes assessments for 10 peers (one message every 15 min)?
2. Or: Split assessments across multiple messages, 10 per message, send all messages every 15 min?

If a node has assessments for 100 peers, option 1 means it would take 10 × 15 min = 2.5 hours to share all assessments. Option 2 means 10 messages every 15 minutes.

**Suggested fix:** Clarify: "Each REPUTATION_GOSSIP message SHOULD include assessments for up to 10 peers. Nodes with more than 10 assessments SHOULD rotate which peers are included in each 15-minute interval to ensure full coverage over time."

### LOW-06: Proposal archival is deterministic but not monotonic
**Severity:** Low (ambiguity)
**Location:** §11 Partition Detection, proposal retention
**Description:** "A proposal is archived the instant it meets the retention criteria above." Archival is deterministic (all nodes agree when a proposal should be archived based on clock time). But what if a node's clock is wrong?

If Node A's clock is 1 hour fast, it archives a proposal 1 hour earlier than Node B. When they compute Merkle roots, they diverge. The spec says this is "deterministic" but only if all nodes have synchronized clocks (which is SHOULD, not MUST, per §2).

**Suggested fix:** Add: "Proposal archival is based on local clock time. Nodes with clock skew >2 minutes MAY observe transient Merkle root divergence during archival windows. This is expected and resolves once archival criteria are met on all nodes' local clocks."

### LOW-07: Vote supersession timing edge case
**Severity:** Low (edge case)
**Location:** §7 Votes
**Description:** "A second vote from the same `from` key supersedes the first." But what if two votes from the same key have the same `timestamp` (same millisecond)? How is the supersession order determined?

Given that `id` = SHA-256(signing_body) and signing_body includes `timestamp`, two votes at the same millisecond will have different `id` values (due to different `payload.stance`). But which one supersedes?

**Suggested fix:** Add tie-break rule: "If two votes from the same voter have the same `timestamp`, the vote with the lexicographically later `id` supersedes (latest by content hash). This scenario is expected to be extremely rare in practice."

### LOW-08: Inline content base64 size ambiguity
**Severity:** Low (ambiguity)
**Location:** §6 Proposals, content delivery
**Description:** "Small artifacts (< 1 MiB) MAY be inlined as base64 in `content_inline`." Base64 encoding has ~33% overhead. Is the 1 MiB limit on the original artifact or the base64-encoded string?

If original artifact = 1 MiB, base64 encoding → 1.33 MiB. Does this exceed the inline limit?

**Suggested fix:** Clarify: "The 1 MiB limit applies to the original artifact size before base64 encoding. The encoded `content_inline` string may exceed 1 MiB (up to ~1.33 MiB)."

---

## CONFORMANCE TEST ISSUES

### TEST-01: LINK-13 test assumes implicit linking but KEY_ROTATE spec is unclear
**Severity:** Medium
**Location:** conformance-tests.md LINK-13
**Description:** LINK-13 tests that when child B rotates to B', the new key B' is implicitly linked to root A. The spec (§1) says "A key that became an authorized key via KEY_ROTATE of an existing child MUST be treated as linked."

But HOW is this enforced? When B sends KEY_ROTATE(old=B, new=B'), who knows that B was an authorized child of A?

Nodes that saw the original DID_LINK(root=A, child=B) can infer B' inherits B's authorized status. But a new node joining after B's rotation has no record of the DID_LINK and only sees KEY_ROTATE(B→B'). How does it know B' belongs to identity A?

**Suggested fix:**
1. Nodes MUST retain DID_LINK history for all identities, even after KEY_ROTATE. The linking relationship is permanent and transitive across key rotations.
2. Or: DID_LINK messages must be re-broadcast periodically (like PEER_ANNOUNCE) so new nodes can discover identity relationships.
3. Update LINK-13 test to specify that nodes joining after the KEY_ROTATE can still discover B'→A relationship via historical DID_LINK message sync.

### TEST-02: MANIFEST-01 doesn't test delimiters or boundary cases
**Severity:** Medium
**Location:** conformance-tests.md MANIFEST-01, related to HIGH-07
**Description:** The manifest hash test uses 3 shards with identical-length hex strings. It doesn't test:
1. Variable shard count (1 shard vs. 10 shards)
2. Mixed shard hash lengths (not possible with SHA-256, but if the hash function ever changes?)
3. Boundary cases where concatenation ambiguity could arise

**Suggested fix:** Add test vectors:
- MANIFEST-02: Single shard
- MANIFEST-03: 10 shards (ensure concatenation is unambiguous at scale)
- MANIFEST-04: Verify that manifest hash differs when shard order changes (sorted requirement)

### TEST-03: REP-02 and REP-03 test α=0 cap but don't verify ASN distinctness enforcement
**Severity:** Low
**Location:** conformance-tests.md REP-02, REP-03
**Description:** REP-02 tests that with <3 ASN-distinct assessors, peer-informed rep is capped at 2000. REP-03 tests ≥3 assessors → uncapped. But the test inputs don't specify ASNs for the assessors.

An implementation could pass these tests without actually checking ASN distinctness (just count assessors, ignore ASN).

**Suggested fix:** Update test inputs to include ASN data:
```
REP-02 (capped):
  - Peer 1: ASN 1234
  - Peer 2: ASN 1234 (same as Peer 1)
  - Only 1 distinct ASN → capped at 2000

REP-03 (uncapped):
  - Peer 1: ASN 1234
  - Peer 2: ASN 5678
  - Peer 3: ASN 9012
  - 3 distinct ASNs → uncapped
```

### TEST-04: No test for KEY_ROTATE first-seen enforcement
**Severity:** Medium
**Location:** conformance-tests.md §1 tests
**Description:** The spec says "Nodes MUST accept only the first KEY_ROTATE message seen for a given `old_key`. Subsequent KEY_ROTATE messages with the same `old_key` but a different `new_key` MUST be rejected."

There's no conformance test for this. A broken implementation could accept multiple KEY_ROTATE messages for the same old key.

**Suggested fix:** Add test:
```
ROTATE-02: KEY_ROTATE conflict — same old_key, two new_keys
- Input:
  1. KEY_ROTATE(old=A, new=B, dual-signed) — received first
  2. KEY_ROTATE(old=A, new=C, dual-signed) — received second
- Expected:
  1. First KEY_ROTATE accepted
  2. Second KEY_ROTATE rejected
  3. Subsequent messages from A and C MUST be rejected; only B is valid
```

### TEST-05: No test for vote supersession tie-break (same timestamp)
**Severity:** Low
**Location:** conformance-tests.md §7 tests
**Description:** Per LOW-07, vote supersession with same timestamp has no defined tie-break. No test exists for this edge case.

**Suggested fix:** Add test:
```
VOTE-03: Vote supersession — same timestamp
- Input:
  1. VOTE(proposal=P, stance=endorse, timestamp=T)
  2. VOTE(proposal=P, stance=reject, timestamp=T, from same voter)
- Expected:
  - If tie-break rule is added (lexicographic id), test that it's enforced
  - If no tie-break rule, document that behavior is implementation-defined
```

### TEST-06: No test for proposal rate limit transfer across KEY_ROTATE
**Severity:** Medium
**Location:** conformance-tests.md RATE-04
**Description:** RATE-04 tests that proposal rate limits transfer across KEY_ROTATE, but the test description is incomplete. It says "2nd proposal from new key MUST be rejected" but doesn't specify timing or the rolling window behavior.

**Suggested fix:** Expand RATE-04:
```
RATE-04: Rate limits transfer across KEY_ROTATE
- Input:
  1. Old key submits PROPOSE at T=0
  2. Old key submits PROPOSE at T=1 day
  3. KEY_ROTATE at T=2 days (old→new)
  4. New key submits PROPOSE at T=3 days
  5. New key submits PROPOSE at T=4 days (4th proposal in 7-day window)
- Expected:
  - Proposals 1-3 accepted
  - Proposal 4 rejected (identity has submitted 3 proposals in rolling 7-day window: T=0, T=1d, T=3d)
  - New key inherits proposal history from old key
```

---

## EDITORIAL / FORMATTING ISSUES

### EDIT-01: Inconsistent use of "node" vs. "identity" vs. "peer"
**Severity:** Editorial
**Location:** Throughout v0.md
**Description:** The spec uses "node," "identity," and "peer" somewhat interchangeably. After §1 Identity Linking is introduced, there's a semantic distinction:
- Node = a running instance with a keypair
- Identity = a root key + authorized children (may span multiple nodes)
- Peer = another entity you're connected to

But many sections written before identity linking still say "node" when they mean "identity" (e.g., "one vote per node" → should be "one vote per identity").

**Suggested fix:** Audit the entire spec for node/identity/peer usage and align terminology. Add a terminology section to §1.

### EDIT-02: Constants table includes some derived values
**Severity:** Editorial
**Location:** Constants Summary table
**Description:** Some entries in the constants table are not actually constants but formulas:
- "Standard quorum: max(active_nodes × 0.1, total_rep × 0.10)"
- "Activity multiplier cap: 2.33 (sustain floor 3 days)"

These aren't single constants — they're computed values. Mixing constants and formulas in the same table is confusing.

**Suggested fix:** Split into two tables: "Fixed Constants" and "Computed Thresholds."

### EDIT-03: §2 signing body example uses old key order
**Severity:** Editorial
**Location:** §2 Signing Body
**Description:** The signing body schema shows keys in lexicographic order: `from`, `payload`, `timestamp`, `type`. But the prose description doesn't emphasize that this order is MANDATORY per JCS.

**Suggested fix:** Add: "Keys are in lexicographic order per JCS canonicalization (§2). The order `from, payload, timestamp, type` is not arbitrary — it's the required sort order."

### EDIT-04: "v0 is the bootstrap protocol" but there's no v1 spec
**Severity:** Editorial
**Location:** v0.md header
**Description:** The spec says "v0 is the bootstrap protocol. It is the only version designed by hand — everything after is proposed and ratified by the network through v0's own mechanisms." This implies there will be a v1. But §12 Protocol Evolution doesn't clarify whether v1 is planned, hypothetical, or TBD.

**Suggested fix:** Add to §12: "At the time of this spec, no v1 protocol change has been proposed. v1 will be defined by the network via the constitutional proposal process once the network reaches critical mass."

### EDIT-05: Markdown table alignment inconsistencies
**Severity:** Editorial
**Location:** Multiple tables throughout v0.md
**Description:** Some tables use `|---|---|---|` (3 dashes) and others use longer dash sequences. Inconsistent alignment makes diffs harder to read.

**Suggested fix:** Normalize all tables to 3-dash separators: `|---|---|---|`.

### EDIT-06: §6 "Protocol change proposals MUST use 'standard' or 'resilient' erasure coding" but constants table doesn't list erasure levels
**Severity:** Editorial
**Location:** §6 Content Delivery, Constants Summary
**Description:** Erasure coding levels (minimal, standard, resilient) are defined in a table in §6 but aren't in the Constants Summary. This makes them easy to miss.

**Suggested fix:** Add to Constants Summary:
```
| Erasure coding: minimal | 3-of-5 | §6 |
| Erasure coding: standard | 5-of-8 | §6 |
| Erasure coding: resilient | 8-of-12 | §6 |
| Protocol proposal min erasure | standard or resilient | §6 |
```

### EDIT-07: "Suggestions" vs. "requirements" language inconsistency
**Severity:** Editorial
**Location:** Throughout v0.md
**Description:** The spec mixes normative language (MUST/SHOULD/MAY per RFC 2119) with informal suggestions ("suggested: X"). This creates ambiguity about what's required for conformance.

Examples:
- "suggested default: 14 days" — is this a SHOULD?
- "suggested: every 10 minutes" — is this a SHOULD?

**Suggested fix:** Replace all "suggested:" with SHOULD or MAY, or document that "suggested" means "non-normative example."

---

## SUMMARY STATS

- **Critical:** 3 issues (KEY_CONFLICT schema, suspension circular dependency, identity linking vote weight concentration)
- **High:** 8 issues (timestamp asymmetry, storage challenge gaming, VDF reuse, partition tie-break grinding, quorum cliff, tenure gaming, manifest hash, shard query DoS)
- **Medium:** 8 issues (α amplification, activity multiplier gaming, abstain strategy, supersession depth, grace period ambiguity, cooling period, shard health, various ambiguities)
- **Low:** 8 issues (terminology inconsistencies, minor ambiguities, edge cases)
- **Conformance test gaps:** 6 issues
- **Editorial:** 7 issues

**Total:** 40 issues identified

---

## POSITIVE FINDINGS (What's Solid)

These areas passed adversarial review without issues:

✓ **JCS canonicalization** — RFC 8785 reference is correct, fixed-point integer encoding eliminates float divergence
✓ **Content addressing** — SHA-256(signing_body) with excluded `version` field is sound
✓ **Ed25519 signature scheme** — Standard, well-vetted
✓ **Merkle tree construction** — Left-biased binary tree is deterministic and correctly specified
✓ **Vote weighting at vote creation time** — Prevents retroactive reputation manipulation
✓ **Reputation floor/cap** — 0.1–1.0 bounds are sensible, prevent exclusion and runaway growth
✓ **Velocity limits** — Daily/weekly caps prevent sudden reputation spikes
✓ **Recovery below 0.2 uncapped** — Eliminates identity recycling incentive
✓ **Abstain votes** — Well-designed for constitutional quorum without forcing binary positions
✓ **Constitutional tier** — 0.90 threshold + 30% quorum + cooling period is appropriately conservative
✓ **Partition merge timestamp priority** — Despite gaming risk (HIGH-04), it's better than no tie-break
✓ **ASN diversity requirements** — 25% cap + 4 minimum ASNs is a strong eclipse resistance measure
✓ **Proposal archival determinism** — Time-based archival rules are clear
✓ **Inactivity decay** — −0.02/month is gradual and fair
✓ **Tenure penalty** — 5%/cycle after 6 cycles addresses entrenchment

The spec is remarkably thorough for a v0. Most issues found are edge cases, attack surfaces that require mitigation, or ambiguities that need clarification. The core mechanisms are sound.

---

## ATTACK SURFACE SEVERITY RANKING

If I were an adversary, here's what I'd exploit first:

1. **CRIT-03** (Identity linking vote concentration) — Deploy 50 nodes, link them, grind reputation, dominate governance
2. **HIGH-02** (Storage challenge collusion) — Sybil storage network, earn reputation without storing anything
3. **HIGH-04** (Partition merge grinding) — During partition, grind malicious protocol proposals for tie-break advantage
4. **CRIT-02** (Suspension circular dependency) — Create KEY_ROTATE conflicts to lock identities out of governance
5. **HIGH-03** (VDF proof reuse) — Compute one proof, flood network with auth attempts

The rest are either low-impact, detectable, or require sustained coordination that collusion detection would catch.

---

**Review complete.** 40 issues documented. Spec is strong overall — most issues are addressable without major redesign.
