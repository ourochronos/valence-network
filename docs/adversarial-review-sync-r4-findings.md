# Adversarial Review: §5 State Reconciliation Protocol (Round 4)

**Reviewer:** Automated adversarial audit (Claude, subagent)
**Date:** 2026-02-18
**Spec version:** v0 (post-round-3 fixes)
**Scope:** §5 State Reconciliation, re-audit after R1 (22 findings), R2 (16 findings), and R3 (13 findings) addressed

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 0 |
| HIGH     | 0 |
| MEDIUM   | 4 |
| LOW      | 3 |
| INFO     | 2 |
| **Total** | **9** |

**Overall assessment:** §5 is ready for implementation and conformance test generation. All 13 R3 findings have been properly resolved. Three rounds of adversarial hardening have addressed every critical and high-severity issue. The remaining findings are implementation ambiguities that could cause interoperability divergence between honest implementations — they are spec clarification issues, not security vulnerabilities. No finding in this round would allow an attacker to corrupt state, eclipse a node, or subvert governance.

---

## Round 3 Finding Status

### All Resolved (13 of 13)

| Finding | Fix | Assessment |
|---------|-----|------------|
| SYNC-R3-01 (REPUTATION_CURRENT min exploitable) | Trimmed minimum: discard highest and lowest, min of remaining 3. IQR >0.1 triggers full replay. | **Resolved.** Tolerates 1 Byzantine peer on each extreme. 2 colluding deflators trigger IQR fallback — expensive but safe. |
| SYNC-R3-02 (Non-publisher cross-check unachievable) | "Publisher" narrowed to nodes whose specific snapshot was used; cross-check peers may be additional connections; fallback after 10 attempts. | **Resolved.** Practical in small networks. |
| SYNC-R3-03 (Cross-phase retroactive invalidation) | DID_REVOKE application triggers scan of committed state; buffered gossip checked at both insertion and drain time. | **Resolved.** Explicit retroactive invalidation specified. |
| SYNC-R3-04 (Snapshot verification one-directional) | Merkle tree narrowing for divergence; unsigned/unknown-key messages trigger discard; signed verifiable messages retained. | **Resolved.** Narrows fallback to suspicious cases only. |
| SYNC-R3-05 (Deterministic revocation gap) | Jitter: every 3rd–5th cycle (random). Event-triggered on DID_REVOKE in previous gossip. | **Resolved.** Non-deterministic and responsive. |
| SYNC-R3-06 (Snapshot rep hash not cross-checked) | Post-REPUTATION_CURRENT, node SHOULD compute and compare hash against snapshot. | **Resolved.** Advisory cross-check with appropriate severity. |
| SYNC-R3-07 (Buffer overflow loses important messages) | Priority-based eviction per phase. Oldest-dropped timestamp tracked for follow-up sync. | **Resolved.** Critical messages retained longest. |
| SYNC-R3-08 (Degraded 50% weight + gossip lag) | VOTE includes voter's sync_status. Other nodes apply min(self_declared, observed). | **Resolved.** Handles both gossip lag directions. |
| SYNC-R3-09 (Snapshot publish frequency) | "At least once per 12h, at most once per 6h" for eligible nodes. | **Resolved.** |
| SYNC-R3-10 (Sync serving local-only tracking) | Acknowledged as known limitation; targeted refusal noted as future work. | **Resolved** (documented). |
| SYNC-R3-11 (last_assessment_timestamp ambiguous) | Defined as assessor's most recent REPUTATION_GOSSIP assessment_timestamp for the subject. MAY use for staleness. | **Resolved.** |
| SYNC-R3-12 (α=0 cap + snapshot bootstrap paradox) | Documented explicitly: REPUTATION_CURRENT values are upper bounds realized as observations accumulate. | **Resolved** (documented). |
| SYNC-R3-13 (Snapshot publisher KEY_ROTATE) | Snapshot signature verification must account for KEY_ROTATE chain. | **Resolved** (noted). |

### Previously Open Items

| Finding | Status | Notes |
|---------|--------|-------|
| SYNC-20 (R1, sync versioning) | **Still open.** | No sync_version field. Acceptable for v0. |
| SYNC-R2-14 (Reputation Merkle tree) | **Deferred to §15.** | Acknowledged gap. Acceptable for v0. |

---

## New Findings

---

### SYNC-R4-01
- **Severity:** MEDIUM
- **Title:** REPUTATION_CURRENT trimmed minimum is underspecified for sparse subject coverage across sync peers
- **Location:** §5, REPUTATION_CURRENT
- **Description:** The trimmed minimum aggregation says: "for each subject, discard the highest and lowest reported values, then use the minimum of the remaining 3." This assumes all 5 sync peers report values for every subject. In practice, sync peers have different views of the network — Peer A may know about nodes 1–100 while Peer B knows about nodes 50–150. For a subject known to only 3 of 5 peers, discarding 2 values leaves only 1. For a subject known to only 2 peers, the algorithm is undefined.

  Two honest implementers could diverge: one might skip trimming when <5 values exist (use minimum of all available), another might require ≥4 values and fall back to full REPUTATION_GOSSIP replay for subjects with fewer reports.

- **Recommended fix:** Specify behavior for sparse coverage: "When fewer than 5 sync peers report a value for a subject: with 4 reports, discard only the highest (trim 1); with 3 reports, use the minimum directly (no trim); with ≤2 reports, fall back to full REPUTATION_GOSSIP replay for that subject." This maintains conservative posture while gracefully degrading.

---

### SYNC-R4-02
- **Severity:** MEDIUM
- **Title:** Graduated degraded exit lacks weight granularity — phases 1–3 complete nodes penalized same as phases 1–2 only
- **Location:** §5, Degraded mode
- **Description:** The spec defines two graduated exit tiers: (a) phases 1–2 complete → MAY vote at 50% weight, and (b) phases 1–3 complete → MAY vote AND propose, but MUST NOT flag or participate in storage decisions. Both use `sync_status: "degraded"`, so other nodes apply the same 50% weight reduction to both.

  A node with complete identity, reputation, AND governance state (phases 1–3) is penalized identically to a node missing governance state entirely (phases 1–2 only). The phase-1-3 node has full context for informed voting but receives the same weight penalty as one that doesn't know what proposals exist.

- **Impact:** This is conservative (safe) but may discourage nodes from participating in governance when they have sufficient context. Not a security issue.
- **Recommended fix:** Consider a finer-grained sync_status: `"degraded-partial"` (phases 1–2) vs `"degraded-governance"` (phases 1–3). The latter could carry 75% weight instead of 50%. Alternatively, accept the 50% uniform penalty as the simpler, safer design — it avoids a proliferation of sync states and their interaction effects.

---

### SYNC-R4-03
- **Severity:** MEDIUM
- **Title:** Merkle tree exchange protocol for divergence narrowing is unspecified
- **Location:** §5, Merkle-Based Divergence Detection
- **Description:** The spec says: "Exchange Merkle trees at increasing depth to identify divergent subtrees. Request the specific message IDs that differ." This is referenced both in the general sync divergence path and in the new snapshot post-verification narrowing. However, no wire protocol is defined for this exchange. The existing SYNC_REQUEST/SYNC_RESPONSE messages don't have fields for Merkle tree depth, subtree indices, or partial tree exchange.

  Two implementers could produce incompatible narrowing protocols — one might add custom fields to SYNC_REQUEST, another might use a sequence of SYNC_REQUESTs with progressively narrower type/timestamp filters as an approximation.

- **Recommended fix:** Define a Merkle narrowing extension to SYNC_REQUEST: `"merkle_tree": "identity" | "proposal"`, `"depth": <integer>`, `"subtree_path": [<0|1>, ...]` (binary path from root). SYNC_RESPONSE includes `"merkle_nodes": [{"path": [...], "hash": "<hex>"}]`. This enables efficient binary search for divergent leaves. Alternatively, acknowledge this as implementation-defined for v0 and specify the wire format in a future revision.

---

### SYNC-R4-04
- **Severity:** MEDIUM
- **Title:** Oldest-dropped-message tracking for follow-up sync doesn't specify per-phase granularity
- **Location:** §5, Gossip buffering during sync
- **Description:** "The node MUST track the timestamp of the oldest dropped message and use it as `since_timestamp` for a follow-up incremental sync pass after the phase completes." Since gossip buffers are per-phase, the oldest-dropped timestamp is implicitly per-phase. But the follow-up sync pass isn't explicitly scoped to the affected phase's message types. An implementer might perform a full incremental sync (all phases) from the oldest dropped timestamp, wasting bandwidth on phases that didn't overflow.

- **Recommended fix:** Clarify: "The follow-up sync pass MUST use the `types` filter for the phase whose buffer overflowed, with `since_timestamp` set to the oldest dropped message's timestamp for that phase." Minor clarification that prevents unnecessary sync traffic.

---

### SYNC-R4-05
- **Severity:** LOW
- **Title:** IQR definition in trimmed minimum is ambiguous for 5-element sets
- **Location:** §5, REPUTATION_CURRENT
- **Description:** "If the interquartile range (difference between the 2nd-highest and 2nd-lowest values) exceeds 0.1 for any subject, the node MUST fall back to full REPUTATION_GOSSIP replay." For a 5-element sorted set [a, b, c, d, e], the "2nd-highest" is d and "2nd-lowest" is b. The spec's parenthetical definition matches this interpretation. However, "interquartile range" formally means Q3−Q1, and for n=5 there are multiple conventions for computing quartiles (exclusive vs. inclusive methods give different results).

  The parenthetical "(difference between the 2nd-highest and 2nd-lowest values)" is unambiguous and should be the authoritative definition. But calling it "interquartile range" may confuse implementers who look up the formal definition and use a library function that computes it differently.

- **Recommended fix:** Drop the term "interquartile range" and use the parenthetical directly: "If the difference between the 2nd-highest and 2nd-lowest reported values exceeds 0.1 for any subject..."

---

### SYNC-R4-06
- **Severity:** LOW
- **Title:** Cold start headcount + degraded 50% weight interaction still has an edge case
- **Location:** §5, Degraded mode + §8, Cold Start
- **Description:** The spec now clarifies that degraded nodes count toward headcount and carry 50% weight in endorsement. And: "cold start uses headcount (no reputation weighting), but the 50% degraded-mode weight is sync-status-based, not reputation-based, and MUST still apply."

  This is well-specified for the endorsement calculation. The edge case: during cold start, the threshold is ">67% endorse" by headcount. If degraded votes carry 50% weight but the threshold is headcount-based (not weight-based), how do fractional weights interact with a headcount threshold? E.g., 10 nodes vote: 7 synced endorse (weight 7.0), 3 degraded endorse (weight 1.5). Total endorsement weight = 8.5. But by headcount, 10/10 endorsed (100%). The 50% weight only matters if the threshold is weight-based, but cold start is headcount-based.

  The spec appears to want weight-based evaluation even during cold start (the endorsement "calculation" uses weights), but this contradicts "No reputation weighting" during cold start.

- **Recommended fix:** Clarify: during cold start, degraded nodes' votes count as 0.5 votes for headcount purposes (not full votes at reduced weight). So 7 synced + 3 degraded = 8.5 effective votes out of 10 headcount. Or: during cold start, degraded weight does not apply (since cold start is headcount-only). Either interpretation is defensible — pick one and state it explicitly.

---

### SYNC-R4-07
- **Severity:** LOW
- **Title:** Snapshot publishing interval ("at least once per 12h, at most once per 6h") creates a gap when nodes go offline
- **Location:** §5, State Snapshots
- **Description:** If all eligible snapshot publishers (rep ≥0.7) go offline simultaneously for >24 hours, no fresh snapshots exist when they return (snapshots must be <24h old). Syncing nodes fall back to full phased sync. This is the correct behavior — snapshot unavailability should never prevent sync, just slow it.

  The spec handles this gracefully. The only note is that the "at least once per 12h" is SHOULD, not MUST. In practice, snapshot availability depends entirely on voluntary behavior by high-rep nodes.

- **Recommended fix:** No fix needed. Document this as expected behavior: snapshot-based bootstrap is an optimization, not a requirement. Full phased sync is always the fallback.

---

### SYNC-R4-08
- **Severity:** INFO
- **Title:** Memory footprint during sync is substantial but bounded
- **Location:** §5, Gossip buffering during sync
- **Description:** Per-phase gossip buffers are capped at 100,000 messages or 100 MiB each. With 5 phases, maximum buffer memory is 500 MiB. Combined with Merkle trees (identity + proposal), dedup cache (100,000 entries), and the sync protocol's own message processing, a syncing node needs ~600–800 MiB of memory dedicated to sync state. This is non-trivial for resource-constrained devices.

  This is bounded and documented — not a vulnerability. Resource-constrained nodes may need to reduce buffer sizes (the spec sets maximums, not minimums). Implementations could use smaller buffers at the cost of more follow-up sync passes.

- **Recommended fix:** Consider noting a RECOMMENDED minimum buffer size (e.g., 10,000 messages or 10 MiB per phase) below which sync reliability degrades significantly. This gives implementers guidance for constrained environments.

---

### SYNC-R4-09
- **Severity:** INFO
- **Title:** The spec has reached a stable equilibrium — diminishing returns on further adversarial review
- **Location:** §5 (entire section)
- **Description:** Four rounds of adversarial review have progressed from 3 criticals + 5 highs (R1) → 2 criticals + 4 highs (R2) → 1 critical + 3 highs (R3) → 0 criticals + 0 highs (R4). The remaining findings are implementation ambiguities, not security vulnerabilities. The attack surface is well-defended:

  - **Eclipse resistance:** 5 sync peers from ≥3 ASNs, bootstrap node requirement for first-join.
  - **Byzantine tolerance:** Trimmed minimum for reputation, 3-of-5 Merkle agreement, 5-publisher snapshot requirement with non-publisher cross-check.
  - **DoS resistance:** Bounded buffers, bandwidth caps, rate limits, backpressure with partition-aware jitter.
  - **Consistency:** Identity + proposal Merkle trees, retroactive invalidation on DID_REVOKE, priority-based buffering.
  - **Incentive alignment:** Sync serving earns uptime credit (with anti-farming), degraded mode prevents uninformed governance.
  - **Graceful degradation:** Graduated degraded exit, snapshot fallback to full sync, per-phase failure recovery.

  The complexity is justified — each mechanism addresses a specific attack or scalability concern identified in previous rounds. A reference implementation would be the most productive next step.

---

## Cross-Section Interaction Analysis

### All Previously Identified Interactions: Resolved

The R3 cross-cutting concerns have been addressed:

1. **REPUTATION_CURRENT × snapshot × full replay loop:** Trimmed minimum makes the optimization robust to 1 Byzantine peer. With 2+ Byzantine peers, IQR triggers fallback — this is expensive but correct. The pathological loop (snapshot expiry during replay) is bounded by the 24h snapshot freshness window and mid-sync failure recovery (re-run phase 1 if >1h stale).

2. **DID_REVOKE immediate application × buffer drain:** The spec now requires revocation checks at both buffer-insertion and buffer-drain time. Retroactive invalidation of committed state is explicit. The interaction is fully specified.

3. **Degraded 50% × cold start headcount:** The spec clarifies that sync-status-based weight applies even during cold start. The remaining ambiguity (SYNC-R4-06) is about the mechanical interaction of fractional weights with headcount thresholds — a spec clarification issue, not a security gap.

### New Interaction: Snapshot Merkle Narrowing × KEY_ROTATE During Sync

When a syncing node performs post-snapshot Merkle narrowing against cross-check peers and discovers messages present in the snapshot but absent from cross-check peers, it checks whether those messages are "signed by unknown keys." A KEY_ROTATE during the sync window could make a formerly-known key appear unknown if the node hasn't processed the KEY_ROTATE yet. The spec's handling is correct: "If signed and verifiable, they are retained." A message signed by a key with a valid KEY_ROTATE chain is verifiable, so it's retained. No action needed.

---

## Implementation Readiness Verdict

**§5 is ready for implementation and conformance test generation**, with the following caveats:

1. **Four MEDIUM-severity ambiguities** (SYNC-R4-01 through SYNC-R4-04) should be resolved before conformance tests are written. They would cause interoperability failures between honest implementations that interpret the spec differently. All four are addressable with 1–2 sentence clarifications.

2. **The two deferred items** (SYNC-20: sync versioning, SYNC-R2-14: reputation Merkle tree) are acknowledged gaps suitable for post-v0 evolution. Neither affects v0 correctness.

3. **A reference implementation** (or detailed pseudocode for the sync state machine) would significantly reduce implementation risk. The spec is complex but the complexity is necessary — the concern is not the design but the probability of implementation bugs in the interaction between 5 phased sync, 2 Merkle trees, snapshot bootstrap, buffered gossip, degraded mode graduation, and incremental sync with rotating identity phases.

**Security posture:** An internet-facing v0 node implementing §5 as written has no known critical or high-severity attack vectors for state corruption, eclipse, or governance subversion via the sync protocol. The attack surface is well-bounded by the layered defenses added across four rounds of review.
