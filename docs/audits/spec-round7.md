# Valence Network v0 Protocol — Adversarial Review Round 7 Findings

**Reviewer:** Subagent (round7-audit)
**Date:** 2025-02-18
**Scope:** Final review. Focus on novel attack vectors, cross-section consistency after 4 rounds of edits, implementability, game-theoretic completeness, economic model balance, message lifecycle completeness, and scale edge cases.

---

## Assessment Summary

The spec is in strong shape. Six rounds of adversarial review have hardened the core mechanisms substantially. The identity system, governance tiers, storage economics, and reputation model are internally consistent and well-specified. **No critical vulnerabilities were found.** The remaining issues are implementation ambiguities, edge cases, and one high-severity game-theoretic concern. This spec is ready for reference implementation work with the caveats noted below.

---

## HIGH

### R7-HIGH-01: Reputation formula non-determinism — vote-time reputation reconstruction diverges across nodes
**Severity:** HIGH
**Sections:** §8 Votes (vote_time_reputation), §9 Reputation Propagation
**Description:** §8 specifies that votes are weighted by the voter's reputation "at the time the vote was created," reconstructed from "the most recent REPUTATION_GOSSIP they hold with an `assessment_timestamp` prior to the vote's timestamp." §9 specifies that reputation is locally computed using α-weighted direct observations and peer assessments. These interact to create non-deterministic vote weighting.

**The problem:** Two nodes A and B evaluating the same vote from voter V will reconstruct different `vote_time_reputation` values for V because:
1. They hold different sets of REPUTATION_GOSSIP messages (gossip is probabilistic, not guaranteed delivery)
2. Their α values differ (different observation counts of V)
3. Their trust(P) values differ for every assessor P

§8 acknowledges this: "nodes with different REPUTATION_GOSSIP histories will still reconstruct slightly different weights." §15 lists it as an acknowledged limitation. But the spec provides no bound on this divergence and no convergence mechanism.

**Concrete scenario:**
1. Proposal P has 10 endorse votes and 8 reject votes. Close margin.
2. Node A computes weighted_endorse/weighted_total = 0.68 (just above 0.67 threshold). Ratified.
3. Node B computes 0.66. Not ratified.
4. Node A broadcasts ADOPT. Node B does not.
5. This divergence is permanent — there's no reconciliation mechanism for close-margin proposals.

**Why this is HIGH, not MEDIUM:** The spec explicitly relies on "persistent disagreement is expected to be rare and self-correcting as reputation data converges" (§8). But reputation data convergence doesn't help for already-concluded proposals — once a voting deadline passes, the vote weights are frozen at historical values that nodes may never agree on.

**Impact:** Split adoption of close-margin proposals. For content proposals this is tolerable (§12 says contradictory ratifications can coexist). For protocol change proposals at 0.90 threshold, the risk is lower (harder to be near the boundary). But it's a real interoperability concern.

**Suggested fix:** Add a deterministic fallback: when a proposal's voting deadline passes and its margin is within ±0.02 of the threshold, nodes MUST enter a 7-day "confirmation period" where they request and integrate all available REPUTATION_GOSSIP for voters in that proposal before making a final determination. This doesn't guarantee convergence but narrows the window significantly. Alternatively, acknowledge this explicitly in §8 as a protocol property with a worked example, so implementers know to expect it.

---

### R7-HIGH-02: REPLICATE_ACCEPT + SHARD_ASSIGNMENT creates a trust-the-uploader bottleneck
**Severity:** HIGH
**Sections:** §6 Content (Replicated), REPLICATE_ACCEPT, SHARD_ASSIGNMENT
**Description:** The replication flow is entirely uploader-driven: the uploader collects REPLICATE_ACCEPT messages, selects providers, distributes shards, and broadcasts SHARD_ASSIGNMENT. The uploader is a single point of failure and trust for the entire replication process.

**Attack scenario (phantom replication):**
1. Attacker at rep 0.3 broadcasts REPLICATE_REQUEST.
2. Providers broadcast REPLICATE_ACCEPT.
3. Attacker broadcasts SHARD_ASSIGNMENT claiming shards were distributed to providers P1-P8.
4. Attacker never actually sends the shard data.
5. The network believes content is replicated (SHARD_ASSIGNMENT says so).
6. First storage challenge cycle: P1-P8 fail challenges because they never received shards.
7. P1-P8 each receive -0.002 accept-and-abandon penalty (spec says "first challenge fail within 7 days").

**The problem:** Innocent providers are penalized for the uploader's failure to distribute shards. The accept-and-abandon penalty assumes the provider is at fault, but the uploader may be the one who never delivered.

**Worse:** Before the first challenge cycle (up to 30 days), the network has no way to know the replication didn't actually happen. The uploader could claim the content is "replicated" to satisfy PROPOSE requirements (§7: "Content MUST be replicated before it can be proposed") without it actually being stored anywhere.

**Suggested fix:**
1. SHARD_ASSIGNMENT should require **provider co-signatures** or provider-broadcast confirmation messages acknowledging receipt. A provider who never received data should not appear in a SHARD_ASSIGNMENT.
2. OR: The accept-and-abandon penalty (-0.002) should only apply when the provider broadcast REPLICATE_ACCEPT and the uploader confirms delivery (via SHARD_ASSIGNMENT listing the provider), AND the provider subsequently fails challenges. If the provider was listed in SHARD_ASSIGNMENT but never received data, the penalty should fall on the uploader (uploader penalized -0.002 per phantom assignment).
3. OR: Providers must broadcast a SHARD_RECEIVED confirmation before SHARD_ASSIGNMENT is considered valid.

---

## MEDIUM

### R7-MED-01: README.md lists "+0.001 per challenge" for storage challenges — contradicts spec
**Severity:** MEDIUM
**Sections:** README.md Participation Economy table, §9 Reputation
**Description:** The README's Participation Economy table includes:

> | Storage challenge passed | +0.001 per challenge | Adjacency proof |

But §9 of v0.md explicitly states that storage challenges are in the "Transferred" category, not "Created." Storage providers earn from rent transfers, verified via challenges — challenges don't independently create reputation. The README implies storage challenges are a reputation creation source (+0.001 each), which would be inflationary and contradicts the spec's economic model.

**Impact:** An implementer reading only the README would build an inflationary reputation system. The README is labeled as non-authoritative, but it's the first thing most people read.

**Suggested fix:** Remove the "Storage challenge passed" row from the README table, or replace it with a note about storage rent transfers: "Storage rent earned | market-priced (§6) | Transferred from uploaders, not created."

---

### R7-MED-02: Observation count growth rate (max 6/hour) creates a 100-minute window of pure peer-informed reputation
**Severity:** MEDIUM
**Sections:** §9 Reputation Propagation (α calculation)
**Description:** α = min(0.6, observation_count / 10), with observations deduplicated at 1 per message type per hour per identity (6 types). Reaching α = 0.6 requires 10 observations, which takes at minimum ceil(10/6) = 2 hours.

During these first 2 hours for any new peer relationship, the evaluating node relies entirely (α = 0) or predominantly on peer assessments. The spec caps peer-informed reputation at 0.2 until 3 assessors from distinct ASNs weigh in. But once 3 ASN-diverse assessors provide data, the cap lifts and α is still near 0.

**Attack scenario (reputation injection window):**
1. Attacker controls 3 nodes across 3 ASNs (trivial, ~$15/month).
2. All three submit REPUTATION_GOSSIP assessments inflating target node T's reputation to 0.8.
3. Victim node V has 0 direct observations of T. α = 0. The 3-assessor ASN-diverse cap lifts.
4. V now sees T at reputation ~0.8 (pure peer-informed, weighted by V's trust in the 3 attackers).
5. If the 3 attackers are relatively new (trust = 0.2 each), the weighted assessment is still 0.8 (all agree).
6. V treats T as a high-reputation node for up to 2 hours until direct observations shift α.

**Impact:** Time-limited reputation inflation. For 2 hours, a target node appears more reputable than warranted. During this window, votes from T carry inflated weight on V's local evaluation. Limited by: (a) only affects V's local view, (b) the 3 attacker nodes have low trust on V, reducing the weight of their assessments.

**Realistic impact:** Low in practice because trust(P) for fresh attacker nodes = ~0.2, making the weighted average much lower than 0.8. With all assessors at trust 0.2 assessing 0.8: peer_avg = (0.2×0.8 + 0.2×0.8 + 0.2×0.8) / (0.2+0.2+0.2) = 0.8. The trust weighting doesn't help when all assessors agree.

**Suggested fix:** Extend the 0.2 cap to require 3 assessors from distinct ASNs AND α ≥ 0.1 (at least 1 direct observation). This ensures the evaluating node has at least some direct interaction before accepting peer-elevated reputation. Alternatively, document this as a known property — the 2-hour window is bounded and the real-world impact is limited.

---

### R7-MED-03: Scarcity multiplier computation is undefined for networks with zero allocated storage
**Severity:** MEDIUM
**Sections:** §6 Storage Economics
**Description:** The scarcity formula is:
```
network_utilization = 1 - (total_available / total_allocated)
scarcity_multiplier = 1 + 99 × network_utilization^4
```

When `total_allocated = 0` (no nodes report storage, or all nodes have 0 allocated bytes), `total_available / total_allocated` is a division by zero.

This is a realistic scenario during early network bootstrap — the first 16 nodes may not all offer storage. If zero nodes have `"store"` capability, the formula is undefined.

**Suggested fix:** Add: "When `total_allocated = 0`, `scarcity_multiplier = 1.0` (no scarcity signal)." This is the natural interpretation (no storage supply = no pricing signal) but should be specified for deterministic implementation.

---

### R7-MED-04: Fixed-point arithmetic truncation order for rent convergence creates micro-divergence
**Severity:** MEDIUM
**Sections:** §2 Numeric Types, §6 Storage Economics (rent convergence)
**Description:** §2 says "multiply by 10,000 and truncate" for all numeric fields, with "truncation only on the final result." The rent convergence formula is:

```
effective_multiplier = locked + (current - locked) × min(1, cycles_elapsed / 5)
```

With accelerated convergence when divergence >10×:
```
cycles_elapsed / 3.33
```

The value `3.33` is not representable exactly in fixed-point (×10,000 → 33300). Is this truncated to 33300 or rounded to 33330? The spec says "truncate" but 1/3.33 ≈ 0.3003003... — the truncation of `cycles_elapsed / 3.33` depends on whether you compute `cycles_elapsed × 10000 / 33300` or `cycles_elapsed / 3.33 × 10000`.

**Impact:** Micro-divergence in rent calculations across implementations using different evaluation orders. Over many billing cycles, this compounds.

**Suggested fix:** Replace `3.33` with an exact rational: `10/3` (cycles_elapsed × 3 / 10). Or specify: "accelerated convergence uses `min(1, cycles_elapsed × 3 / 10)`." This is exact in integer arithmetic.

---

### R7-MED-05: Partition merge identity rule (DID_REVOKE wins) can be weaponized during intentional partition
**Severity:** MEDIUM
**Sections:** §12 Partition Detection (Identity state merge), §1 DID_REVOKE
**Description:** §12 says "When conflicting states exist (a key linked on one side, revoked on the other), DID_REVOKE takes precedence (revocation is irreversible)." This is correct for the general case but creates a grief vector:

1. Identity R has child C. Both are active on a healthy network.
2. Attacker compromises R's root key (or R is the attacker).
3. Attacker deliberately creates a network partition (e.g., by disrupting connectivity for R's peers).
4. On the attacker's side of the partition, R broadcasts DID_REVOKE for C.
5. On C's side, C continues operating normally, unaware of revocation.
6. Partitions merge. DID_REVOKE propagates. C is retroactively deauthorized.
7. All of C's votes during the partition are invalidated (§12: "Votes cast by a key that was revoked... before the vote's timestamp are invalidated on merge").

**The issue:** The attacker can retroactively invalidate votes by engineering a partition + revocation. C had no way to know or prevent this.

**This is inherent to the "revocation wins" design** — it's the correct security choice (a compromised key must be revocable). But the interaction with partition merge vote invalidation creates a timing attack on governance. An attacker who compromises a root key can invalidate all child votes retroactively by timing the revocation during a partition.

**Mitigation already in spec:** KEY_ROTATE from the root changes the root key, preventing further revocations from the compromised old root. But if the attacker already has the root key, the damage is done before rotation.

**Suggested fix:** This is arguably working as designed — root key compromise is catastrophic by nature, and the spec can't fully mitigate it. Add a note: "Root key compromise allows retroactive vote invalidation via DID_REVOKE during partitions. Operators MUST treat root key security as critical. Hardware key storage is strongly recommended."

---

### R7-MED-06: COMMENT rate limit "3 per identity per proposal per day" — "day" is undefined
**Severity:** MEDIUM
**Sections:** §7 COMMENT
**Description:** The per-proposal comment limit is "3 per identity per proposal per day." The spec doesn't define what "day" means: UTC day? Rolling 24-hour window? Local time?

If it's a UTC day boundary, a node can post 3 comments at 23:59 UTC and 3 more at 00:01 UTC (6 comments in 2 minutes). If it's a rolling 24-hour window, the timing game doesn't work but implementations need to track timestamps per (identity, proposal) pair.

**Suggested fix:** Specify: "3 per identity per proposal per rolling 24-hour window." Consistent with the rolling window pattern used for proposal rate limits.

---

## LOW

### R7-LOW-01: Constants table lists rent convergence as "20%/cycle" but the formula gives different convergence rates depending on reading
**Severity:** LOW
**Sections:** Constants Summary, §6 Storage Economics
**Description:** Constants table says: "Rent price locking | at replication time, converges 20%/cycle | §6". The formula in §6 is: `effective_multiplier = locked + (current - locked) × min(1, cycles_elapsed / 5)`.

At cycle 1: convergence factor = 1/5 = 0.20. At cycle 2: factor = 2/5 = 0.40. The constant "20%/cycle" describes the first cycle's convergence, but by cycle 2 you're 40% converged (not 20% + 20% = 40% — it's the same formula, just coincidentally additive here because it's linear). The constants table entry is accurate but could be clearer.

**Suggested fix:** Change to: "Rent multiplier convergence | linear, full market rate at 5 cycles (20% per cycle additive) | §6". The "(20% per cycle additive)" clarifies it's linear interpolation.

---

### R7-LOW-02: VDF verification segment selection is "randomly selected" — no PRNG specification
**Severity:** LOW
**Sections:** §10 Sybil Resistance (VDF verification)
**Description:** "Verifiers MUST verify at least 5 randomly selected segments." The randomness source is unspecified. Implementations might use different PRNGs seeded differently, leading to different verification patterns.

For v0 this is fine — the verification is local and doesn't affect consensus. But an implementation that uses predictable randomness (e.g., seeded from the VDF proof itself) would allow an attacker to predict which segments will be checked and forge only those.

**Suggested fix:** Add: "The segment selection SHOULD use a CSPRNG. The randomness MUST NOT be derived from the VDF proof being verified (to prevent targeted forgery)."

---

### R7-LOW-03: PROPOSE `claims` array has no size limit
**Severity:** LOW
**Sections:** §7 Proposals, PROPOSE schema
**Description:** The `claims` array in PROPOSE has no specified maximum length. A proposal with 10,000 claims, each with a 10 KiB evidence URL, would be within the 8 MiB message limit but would be expensive for nodes to process and verify.

**Suggested fix:** Add: "The `claims` array MUST NOT exceed 50 entries. Each claim's `statement` + `evidence` MUST NOT exceed 2 KiB combined."

---

### R7-LOW-04: PEER_ANNOUNCE `capabilities` array is open-ended with no defined vocabulary
**Severity:** LOW
**Sections:** §4 Peer Discovery, PEER_ANNOUNCE schema
**Description:** PEER_ANNOUNCE includes `"capabilities": ["propose", "vote", "store"]`. The spec references `"store"` capability in §6 (shard eligibility) but never defines the full vocabulary or whether unknown capabilities should be accepted.

§14 says unknown FIELDS should be accepted, but capabilities are values within a known field. An implementation that gates behavior on capabilities (e.g., only sending REPLICATE_ACCEPT to nodes with "store") needs to know the canonical capability strings.

**Suggested fix:** Add a capabilities vocabulary: `"propose"` (can author proposals), `"vote"` (can vote), `"store"` (can hold shards). Unknown capabilities SHOULD be accepted and ignored (forward compatibility for future capability strings proposed via governance).

---

### R7-LOW-05: §12 Proposal Retention archival timings reference "§7" but they're defined in §12
**Severity:** LOW
**Sections:** §12 Partition Detection, Constants Summary
**Description:** The constants table lists archival timings with "§7" references (e.g., "Expired proposal archive | 7 days | §7"). These are actually defined in §12 (Partition Detection → Proposal Retention). The content of proposals is in §7, but the archival rules are in §12.

**Suggested fix:** Update constants table references for archival timings from "§7" to "§12".

---

## CROSS-SECTION CONSISTENCY AUDIT

Systematically checked all 15 sections + constants table for agreement after 4 rounds of edits:

| Check | Status | Notes |
|-------|--------|-------|
| §1 gain dampening formula ↔ Constants table | ✓ | Both: `raw_gain / N^0.75` |
| §1 60-day voting cooldown ↔ Constants | ✓ | Both: 60 days |
| §1 identity message re-broadcast ↔ Constants | ✓ | Both: 30 days |
| §2 message type inventory ↔ all sections | ✓ | All 30 message types present with correct section refs |
| §2 fixed-point ×10,000 ↔ §9 formula | ✓ | §9 provides evaluation order |
| §3 GossipSub params ↔ Constants | ✓ | D=6, D_low=4, D_high=12 |
| §4 ASN diversity ↔ §6 validator ASN diversity | ✓ | Both enforce ASN requirements |
| §5 24-hour gossip age ↔ §1 identity message exemption | ✓ | Identity messages explicitly exempt |
| §6 rent convergence ↔ Constants | ✓ | Both: 20%/cycle, accelerate at >10× |
| §6 80/20 provider/validator split ↔ Constants | ✓ | |
| §6 grace period 7/3/0 ↔ Constants | ✓ | |
| §6 provenance 70/30 ↔ §9 earning table | ✓ | §9 explicitly references §6 |
| §6 provenance 0.002 cap below 0.3 ↔ Constants | ✓ | |
| §6 proposed content storage provider share ↔ §9 Transferred table | ✓ | "Carved from author's +0.05, NOT additional creation" |
| §7 proposal rate limit 3/7d ↔ §1 identity linking rule ↔ Constants | ✓ | Identity-wide limit |
| §8 min rep to vote 0.3 ↔ §9 capability ramp ↔ Constants | ✓ | §8 explicitly states 0.3 |
| §8 cold start period ↔ Constants | ✓ | 30 days, 50% headcount, 60-day expiry |
| §8 constitutional threshold ↔ §13 Evolution ↔ Constants | ✓ | 0.90, 30% quorum, 90-day min, 30-day cooling |
| §9 velocity limits ↔ Constants | ✓ | 0.02/day, 0.08/week |
| §9 α formula ↔ Constants | ✓ | min(0.6, count/10) |
| §10 VDF difficulty ↔ Constants | ✓ | 1,000,000 iterations |
| §11 tenure penalty ↔ Constants | ✓ | 6 cycles onset, 0.95× per cycle |
| §12 archival timings ↔ Constants | ⚠️ | Correct values but section references say "§7" not "§12" (R7-LOW-05) |
| §12 partition merge identity rules ↔ §1 DID_REVOKE | ✓ | Consistent, revoke wins |
| §14 unknown-type rate limit ↔ Constants | ✓ | 10/sender/hour |
| README participation table ↔ §9 | ⚠️ | README lists storage challenge +0.001 as creation (R7-MED-01) |
| protocol.md message types ↔ §2 inventory | ✓ | protocol.md updated with all 30 types |

**Result:** 2 inconsistencies found (R7-MED-01, R7-LOW-05). All others consistent.

---

## ECONOMIC MODEL TRACE

### Reputation Creation Sources (per §9)
| Source | Max rate | Annual max per identity |
|--------|----------|------------------------|
| Proposal adoption | +0.05/proposal, 3/week | 7.8 (velocity-capped to ~2.5) |
| Claim verified true | +0.001 | Unbounded but tiny |
| Claim found false | +0.005 | Bounded by false claims existing |
| First contradiction | +0.01 | Bounded by false claims existing |
| Uptime | +0.001/day, 30-day cap | 0.36 |
| **Effective max (velocity-capped)** | **0.02/day** | **~7.3** |

### Reputation Destruction Sinks
| Sink | Rate | Bounded? |
|------|------|----------|
| False claims | -0.003 | No cap on losses |
| Failed storage | -0.01 | No cap |
| Collusion | -0.05 | No cap |
| Inactivity | -0.02/month | Floor 0.1 |
| Flag upheld | -0.05 | No cap |
| False flag | -0.005 to -0.07 | No cap |
| Ban | Total forfeiture | One-time |

### Transfers (zero-sum)
| Flow | Rate |
|------|------|
| Storage rent: uploader → providers (80%) + validators (20%) | Market-priced, ongoing |
| Proposed content adoption: author → providers (30% of adoption cap) | Per-adoption event |

### Inflation/Deflation Assessment
- **Creation** is velocity-capped at 0.02/day per identity. With N identities, max daily system creation = 0.02N.
- **Destruction** is uncapped and increases with network activity (more proposals → more potential false claims, more storage → more potential failures).
- **At steady state**, the system trends deflationary because: (a) inactivity decay hits all nodes continuously, (b) penalties are uncapped while gains are capped, (c) the floor at 0.1 doesn't stop decay — it just prevents exclusion.
- **Potential issue:** At very small network sizes (16-50 nodes), if most nodes are contributing honestly, creation (0.02 × N/day) may exceed destruction (inactivity only hits inactive nodes). System inflates slowly. This is acceptable during bootstrap — reputation should grow as the network proves itself. At scale, destruction mechanisms (false claims, failed storage, collusion detection) activate more frequently, creating balance.

**Assessment: The economic model is sound.** No hidden inflation spirals. The deflationary bias is appropriate — reputation should be hard to earn and easy to lose.

---

## IMPLEMENTABILITY ASSESSMENT

Could a competent engineer build a correct, interoperable implementation from this spec alone?

### Clear and Sufficient
- ✓ Message format, canonicalization (JCS), signing, verification
- ✓ Identity lifecycle (DID_LINK, DID_REVOKE, KEY_ROTATE, KEY_CONFLICT)
- ✓ Transport (libp2p, GossipSub topics, stream protocols)
- ✓ Peer discovery and anti-fragmentation
- ✓ Gossip push/pull, deduplication, sync pagination
- ✓ Proposal and vote lifecycle
- ✓ Erasure coding parameters
- ✓ Storage challenge protocol
- ✓ VDF computation and verification
- ✓ Fixed-point arithmetic (×10,000, evaluation order specified)
- ✓ Partition detection (Merkle tree construction specified)
- ✓ Error handling

### Ambiguous (would cause interop issues without clarification)
- ⚠️ Vote-time reputation reconstruction (R7-HIGH-01) — which REPUTATION_GOSSIP messages to use
- ⚠️ Rent convergence `3.33` not exactly representable (R7-MED-04)
- ⚠️ COMMENT "per day" undefined (R7-MED-06)
- ⚠️ Zero-allocated-storage scarcity multiplier (R7-MED-03)
- ⚠️ VDF segment selection randomness source (R7-LOW-02)

### Missing but not blocking
- Conformance test suite (acknowledged in §15)
- Capability string vocabulary (R7-LOW-04)
- SHARD_ASSIGNMENT provider confirmation (R7-HIGH-02 — functional without it, but trust assumption on uploader)

**Overall: 90% implementable.** The 5 ambiguities above should be resolved before publishing a conformance test suite, but wouldn't prevent a motivated team from building a working implementation. The spec is remarkably complete for a protocol of this complexity.

---

## GAME-THEORETIC COMPLETENESS

### Rational strategies examined:

1. **Sybil farming:** VDF (30s) + capability ramp (100 days to 0.3) + gain dampening + velocity caps = expensive. Cost per useful Sybil: ~$50 (hosting) + 100 days. Not economically viable at scale.

2. **Free-riding storage:** Fixed by min 10 challenges from ≥2 validators per cycle. Lazy providers get prorated rent. ✓

3. **Validator spam:** Fixed by 30-challenge-per-content cap and per-pair weighting. ✓

4. **Vote trading for rent exemption:** Fixed by requiring ≥3 *endorse* votes. ✓

5. **Provenance farming:** Capped at 0.002/proposal below 0.3. To go from 0.2 to 0.3 via provenance alone: 50 proposals. At 3/week: ~17 weeks. Slow enough to not be exploitable. ✓

6. **Tenure evasion:** 3 consecutive skips cost 90 days inactivity + -0.06 reputation. More expensive than the 5% vote weight reduction. ✓

7. **Reputation laundering via storage rent:** Self-storage is zero-sum within identity. Cross-identity transfers require actual storage service. ✓

8. **Governance capture via cold start:** Cold start requires >67% of headcount. Attacker deploying 20 nodes against 16 honest: needs 67% of 36 = 24 endorses. Has 20. Not enough. ✓

9. **Land-grab storage:** Rent converges to market rate in 5 cycles (5 months). Tolerable early-mover advantage, bounded. ✓

**No unexploited rational strategies found.** The incentive structure is well-aligned.

---

## SCALE EDGE CASES

### 10 nodes
- Governance frozen (< 16). Content sharing only. No issues.
- Scarcity multiplier may have low sample quality (few PEER_ANNOUNCE reports). Acceptable.

### 100 nodes
- Standard governance active. Cold start applies.
- Reputation gossip: 100 nodes × 1 msg/15min × 10 assessments = ~11 messages/minute. Trivial.
- Storage challenges: depends on content volume. At 100 pieces × 8 shards × 1 challenge/day = 800/day. Manageable.

### 1,000 nodes
- Approaching constitutional threshold (1,024).
- Reputation gossip: ~111 messages/minute. Noted in §15. Manageable with current parameters.
- SHARE re-broadcast: 1,000 nodes × 10 items × 1 msg/30min = ~5.5 messages/second. Moderate but sustainable.

### 10,000 nodes
- Reputation gossip: ~111 messages/second. **This is the primary scaling concern.** §15 acknowledges it and proposes mitigations (adaptive frequency, delta-only gossip, hierarchical).
- SHARE re-broadcast: ~55 messages/second of just SHARE traffic. This becomes significant GossipSub overhead. The spec acknowledges this implicitly (SHARE was flagged in round 3) but the current design doesn't scale. A DHT-based content discovery mechanism would be needed.
- SHARD_QUERY at 10 per sender per minute × 10,000 senders = potential 100,000/minute. Rate limiting per-sender helps but aggregate load on popular content could be high.
- **Nothing structurally breaks** at 10,000 nodes, but gossip-based discovery (SHARE, REPUTATION_GOSSIP) becomes bandwidth-intensive. The protocol explicitly defers this to governance-proposed improvements, which is reasonable for v0.

---

## SUMMARY

| Severity | Count | IDs |
|----------|-------|-----|
| CRITICAL | 0 | — |
| HIGH | 2 | R7-HIGH-01, R7-HIGH-02 |
| MEDIUM | 6 | R7-MED-01 through R7-MED-06 |
| LOW | 5 | R7-LOW-01 through R7-LOW-05 |
| **Total** | **13** | |

## PRIORITY ORDER FOR FIXES

### Before reference implementation
1. **R7-HIGH-01** — Clarify vote-time reputation reconstruction (interop-critical)
2. **R7-MED-04** — Fix `3.33` to exact rational `10/3` (determinism)
3. **R7-MED-03** — Define scarcity multiplier when total_allocated = 0 (division by zero)
4. **R7-MED-06** — Define "day" as rolling 24-hour window

### Before conformance tests
5. **R7-HIGH-02** — Add provider confirmation to SHARD_ASSIGNMENT flow
6. **R7-LOW-02** — Specify VDF PRNG requirements
7. **R7-LOW-04** — Define capability string vocabulary

### Documentation
8. **R7-MED-01** — Fix README participation table
9. **R7-LOW-05** — Fix constants table section references
10. **R7-LOW-01** — Clarify convergence rate description

---

## FINAL ASSESSMENT

**The spec is ready for implementation.** After 7 rounds of review, no critical vulnerabilities remain. The two HIGH issues are about implementation determinism and trust assumptions, not protocol-breaking attacks. The economic model balances. The game theory holds. Cross-section consistency is strong (2 minor issues across 15 sections + constants). The acknowledged limitations in §15 are honest and appropriate.

The spec has successfully navigated the hardest design tensions: local reputation with approximate consensus, transfer-based storage economics without inflation, Sybil resistance without proof-of-stake, and self-governance from bootstrap. These are real achievements.

**Recommendation:** Fix the 4 "before reference implementation" items above, then proceed to implementation and conformance test development. The remaining issues can be addressed in parallel.
