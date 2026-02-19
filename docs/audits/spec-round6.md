# Valence Network v0 Protocol — Adversarial Review Round 6 Findings

**Reviewer:** Subagent (round6-audit)
**Date:** 2025-02-18
**Scope:** Post-round-5 fixes. Focus on storage validator economy, rent convergence, REPLICATE_ACCEPT, content opacity, cross-section consistency, economic lifecycle completeness, second-order fix interactions.

---

## CRITICAL

*None found.*

The spec has matured through 5 rounds. The remaining issues are HIGH or below.

---

## HIGH

### R6-HIGH-01: Validator economy — collusion between validator and provider to split fraudulent rent
**Severity:** HIGH
**Sections:** §6 Storage Provider and Validator Revenue, §6 Storage Challenges
**Description:** The 80/20 provider/validator split creates a new collusion vector. A storage provider and a validator can collude:

1. Provider P claims to store shards but doesn't.
2. Validator V issues "challenges" and reports `result: pass` via CHALLENGE_RESULT.
3. P receives 80% of rent. V receives their share of the 20% validator pool.
4. Neither actually holds the shard data.

**Detection gap:** The spec says "Third-party nodes track challenge success rates per (challenger, challenged) pair. Suspiciously high pass rates between the same pair (>20 consecutive passes) SHOULD trigger collusion review." But:
- The minimum is 10 challenges from ≥2 validators. A colluding pair (P + V1 + V2, where V1 and V2 are the same operator) satisfies the "2 distinct validators" requirement if V1 and V2 are different identities.
- The >20 consecutive pass threshold is per-pair. If V1 and V2 alternate, neither pair reaches 20. With 2 colluding validators doing 5 challenges each per cycle, neither triggers the flag.
- The collusion detection in §11 looks for voting correlation, not challenge-result correlation. Storage challenge collusion has no dedicated detection mechanism.

**Impact:** Rent is extracted from uploaders without actual storage occurring. Content durability is silently compromised — the network believes shards are stored when they're not. Discovery happens only when legitimate retrieval fails.

**Suggested fix:**
1. Add challenge-result correlation detection: flag validator-provider pairs where one validator accounts for >50% of a provider's passed challenges over 3+ billing cycles.
2. Third-party spot-check: any node holding the shard MAY independently challenge a provider. If a provider passes all challenges from "friendly" validators but fails independent challenges, both the provider and friendly validators are penalized.
3. Require that the ≥2 distinct validators be from ≥2 distinct ASNs (raises collusion cost).

---

### R6-HIGH-02: Validator spam — minimum 10 challenges creates perverse incentive to flood challenges
**Severity:** HIGH
**Sections:** §6 Storage Provider and Validator Revenue, §6 Storage Challenges
**Description:** Validators earn from the 20% pool proportional to "challenges issued and verified during the billing cycle." The spec caps challenges at 10 per peer per day. But:

- A validator with the shard can challenge EVERY provider holding that shard, across ALL content they validate.
- For popular content with 8 shard holders, a single validator can issue 10 challenges/day × 8 providers = 80 challenges/day for one piece of content.
- Over a 30-day cycle: 2,400 challenges from one validator for one content item.
- The 20% pool is split proportional to challenges. A spam-validator issuing 2,400 challenges drowns out honest validators issuing the minimum 10.

**The perverse incentive:** Validators are rewarded per-challenge, not per-unique-insight. Issuing more challenges earns more of the 20% pool. The economically rational strategy is to issue the maximum 10 challenges per peer per day, every day.

**Result:** The validator pool is captured by high-frequency challengers. Honest validators who issue a reasonable number of challenges earn nearly nothing. The validation economy becomes a race to the bottom on challenge volume, not challenge quality.

**Suggested fix:**
1. Cap validator earnings at challenges from the first N challenges per content per validator per cycle (e.g., N=30 — roughly one per day). Excess challenges still verify storage but don't earn additional validator share.
2. OR: Weight validator earnings by unique (validator, provider) pairs challenged, not total challenges. Each pair counts once per cycle.
3. AND: Validators who discover failures earn double share (already in spec) — this correctly rewards quality over quantity. But the baseline reward should not scale linearly with challenge count.

---

### R6-HIGH-03: REPLICATE_ACCEPT shard assignment conflicts — no coordination protocol
**Severity:** HIGH
**Sections:** §6 Content (Replicated), REPLICATE_ACCEPT
**Description:** REPLICATE_ACCEPT is broadcast on `/valence/proposals`. Multiple providers can claim the same `shard_indices`. The spec says "The uploader collects acceptances and distributes shards to providers via direct stream." But:

1. **Race condition:** Provider A and Provider B both broadcast REPLICATE_ACCEPT claiming shard index 3. The uploader sees A first and sends the shard to A. B never receives the shard but announced willingness. Other nodes saw B's acceptance and may expect B to hold shard 3.
2. **No assignment confirmation:** There's no message from the uploader saying "I assigned shard 3 to Provider A." Third-party nodes have no way to know the actual shard assignment. SHARD_QUERY discovers holders after the fact, but during the initial 48-hour acceptance window, assignment is opaque.
3. **Phantom providers:** A node broadcasts REPLICATE_ACCEPT, is selected by the uploader, but goes offline before receiving the shard. The uploader must retry with another acceptor, but there's no protocol for re-assignment.
4. **Incentive to accept then not store:** A provider broadcasts REPLICATE_ACCEPT to claim a slot, preventing other providers from claiming it, then never actually stores the shard. The provider pays no penalty for acceptance without follow-through — the worst case is not earning rent (no negative consequence).

**Suggested fix:**
1. Add a SHARD_ASSIGNMENT message broadcast by the uploader after distributing shards:
```json
{
  "type": "SHARD_ASSIGNMENT",
  "payload": {
    "content_hash": "<hex>",
    "assignments": [{"shard_index": 0, "provider": "<hex_node_id>"}, ...]
  }
}
```
2. Providers who broadcast REPLICATE_ACCEPT but are not assigned (or fail to respond to shard transfer within 24 hours) are free to be replaced.
3. If assigned providers fail the first storage challenge within 7 days of assignment, apply a small penalty (-0.002) to discourage accept-and-abandon.

---

### R6-HIGH-04: Rent convergence formula (10%/cycle) is too slow to prevent land grabs
**Severity:** HIGH
**Sections:** §6 Storage Economics (Rent price locking with convergence)
**Description:** The convergence formula is: `effective_multiplier = locked + (current - locked) × min(1, cycles_elapsed / 10)`. This means full convergence takes 10 billing cycles (10 months).

**Land grab scenario:**
1. Network at 20% utilization, multiplier ≈ 1.16.
2. Attacker replicates maximum content at locked multiplier 1.16.
3. Network grows to 70% utilization over 5 months. Current multiplier ≈ 24.
4. At month 5, attacker's effective multiplier: 1.16 + (24 - 1.16) × 0.5 = 12.6. Still 2× cheaper than new uploaders.
5. At month 10, convergence is complete. But for 10 months, the attacker occupied capacity at below-market rates.

**The problem isn't the final convergence but the duration.** 10 months of subsidized storage is enough for early movers to establish entrenched positions. The quartic scarcity curve means the difference between locked and current can be 40-100×, and even at 50% convergence (5 months), the effective rate is wildly below market.

**Counterpoint:** The spec's intent (preventing death spirals from sudden rent spikes on existing content) is valid. But 10 months is too generous.

**Suggested fix:**
1. Converge at 20%/cycle instead of 10%: full market rate at 5 months. `min(1, cycles_elapsed / 5)`.
2. OR: Use exponential convergence: `effective = locked × (1 - 0.2)^cycles + current × (1 - (1-0.2)^cycles)`. This front-loads adjustment (56% converged at 3 months) while still preventing sudden spikes.
3. AND: If current multiplier exceeds locked by >10×, accelerate convergence to 30%/cycle. This handles the extreme divergence case without affecting moderate price changes.

---

## MEDIUM

### R6-MED-01: Content opacity + flagging creates unfalsifiable flag defense for encrypted content
**Severity:** MEDIUM
**Sections:** §6 Content opacity, §6 Content Flagging
**Description:** The spec says "The network treats all content as opaque bytes. Encrypted content is valid at every layer." It also says "Automated detection SHOULD run on established nodes — hash matching operates on raw bytes."

**Implication:** Known-bad hash databases match against plaintext content. Encrypted content NEVER matches a known-bad hash (the ciphertext is random bytes). This means:

- CSAM encrypted before upload is invisible to automated detection.
- Manual flags require the flagger to have decrypted the content (possessing the decryption key and thus arguably possessing the content).
- Hash-match flags (automated, lower threshold for deletion) are structurally impossible for encrypted content.
- Only manual `severity: illegal` flags (requiring rep ≥ 0.5) can target encrypted content, and the flagger must explain how they know the content is illegal without the hash-match mechanism.

**This is arguably working as designed** — the spec explicitly says encrypted content is valid, and discovery relies on out-of-band key distribution. But the intersection of "encrypted content is valid" and "automated detection is the primary enforcement mechanism" creates a blind spot that the spec should acknowledge.

**Suggested fix:** Add a note in the Content opacity section: "Encrypted content is opaque to automated hash-match detection. The network relies on out-of-band reporting (manual flags from nodes that have decrypted the content) for enforcement against encrypted illegal content. This is a known limitation — the protocol cannot inspect encrypted bytes, and this is by design. Nodes may choose not to store encrypted content from unestablished identities as a local policy."

---

### R6-MED-02: Rent convergence + grace period interaction — existing content never hits sudden death spiral, but partial convergence can still cause cascade
**Severity:** MEDIUM  
**Sections:** §6 Storage Economics (rent convergence, grace period)
**Description:** The convergence formula prevents *sudden* rent spikes. But consider:

1. Content replicated at multiplier 2.0. Monthly rent for 10 MiB: 0.02.
2. Network utilization climbs. At month 6, current multiplier is 50.0.
3. Effective multiplier at month 6: 2.0 + (50 - 2) × 0.6 = 30.8. Rent: 0.308/month.
4. Uploader's max monthly income (velocity cap): ~0.34/month.
5. Uploader is now spending 90% of their maximum income on rent for ONE piece of content.
6. Any penalty or second piece of content pushes them into grace period.

**The convergence prevents death from rent spike, but doesn't prevent death from gradual convergence toward an unsustainable rate.** The uploader committed to content when rent was cheap and has no escape route except CONTENT_WITHDRAW (losing the content) or continuing to pay escalating rent.

**This is arguably market dynamics working correctly** — if storage is expensive, old content should eventually pay market rate. But the spec doesn't warn uploaders about this risk.

**Suggested fix:** Document this in §6 or §15: "Uploaders should note that rent converges toward current market rates over 10 billing cycles. Content replicated during low-utilization periods will see increasing rent as the network grows. Uploaders should plan for rent at the current (not locked) multiplier when evaluating long-term storage costs."

---

### R6-MED-03: REPLICATE_ACCEPT — no mechanism for uploader to reject unsuitable providers
**Severity:** MEDIUM
**Sections:** §6 Content (Replicated), REPLICATE_ACCEPT
**Description:** Any node with `store` capability and sufficient `available_bytes` can broadcast REPLICATE_ACCEPT. The uploader "collects acceptances and distributes shards." But:

- What if all acceptors are from the same ASN? (Poor shard diversity)
- What if acceptors have low reputation? (Higher risk of storage failure)
- What if more providers accept than there are shards? (Selection criteria?)

The spec gives the uploader complete discretion on shard distribution but provides no guidance on selection criteria and no protocol for rejecting unsuitable providers.

**Suggested fix:** Add guidance: "Uploaders SHOULD distribute shards across providers from diverse ASNs (minimum 3 distinct ASNs when possible). Uploaders SHOULD prefer providers with higher reputation. When more providers accept than shards exist, the uploader selects providers at their discretion. Unselected providers are not notified — they simply never receive shards."

---

### R6-MED-04: Validator "double share for failures" creates incentive to cause failures
**Severity:** MEDIUM
**Sections:** §6 Storage Provider and Validator Revenue
**Description:** "Validators who discover failures earn double their share for that challenge." This incentivizes honest detection but also:

1. Validator V issues a challenge to Provider P.
2. V controls the network path between itself and P (or is on a congested route).
3. V's challenge arrives at P, but V drops/ignores P's STORAGE_PROOF response.
4. V reports `result: fail` — earning double share.
5. P loses rent for the cycle AND pays -0.01 penalty.

**The spec says:** STORAGE_PROOF is sent via stream `/valence/sync/1.0.0`. The proof is point-to-point between challenger and provider. There's no third-party witness.

**Impact:** A malicious validator can selectively "fail" honest providers to earn double share and suppress competitors in the rent pool.

**Mitigation already in spec:** "Suspiciously high pass rates" are flagged, but there's no corresponding flag for suspiciously high FAIL rates from a single validator. A validator who reports failures disproportionately (vs. other validators challenging the same provider) should be treated as suspect.

**Suggested fix:**
1. Add: "Nodes SHOULD flag validators whose failure discovery rate is >3× the network average for the same providers. Suspiciously high failure rates from a single validator SHOULD be weighted less in rent calculations."
2. OR: Failed challenges require corroboration — a second, independent challenge from a different validator within 24 hours. If the second challenge passes, the original failure report is discounted.

---

### R6-MED-05: Economic lifecycle gap — reputation flow for content that enters grace period and recovers
**Severity:** MEDIUM
**Sections:** §6 Storage Economics, §9 Reputation
**Description:** Tracing the full lifecycle reveals an under-specified recovery path:

1. Content C replicated. Uploader U pays rent cycle 1. Providers earn 80%, validators 20%. ✓
2. U fails to pay cycle 2. Grace period (7 days). "Storage providers are not paid but SHOULD continue holding shards." ✓
3. U recovers reputation, pays cycle 3 before grace expires.

**Question:** When U pays cycle 3, do providers receive back-pay for the grace period? Or is the grace period simply a gap in provider earnings?

The spec says: "RENT_PAYMENT for cycle N MUST be broadcast within the first 7 days of cycle N. Late payments are treated as missed (grace period triggers). Skipped cycles cannot be retroactively paid."

**So providers lose the grace period earnings entirely.** They held shards for free during grace. This is stated but worth confirming: providers have a rational incentive to drop shards immediately on grace period entry (they're working for free with uncertain recovery). The 7-day grace period assumes provider altruism.

**Suggested fix:** Either:
1. Accept this as designed and add explicit guidance: "Storage providers who continue holding shards during grace periods do so without compensation. Providers may rationally choose to drop shards at grace period entry. Uploaders should not rely on shard survival through grace periods."
2. OR: On recovery, the first rent payment after a grace period includes a 50% surcharge distributed to providers who demonstrably held shards (passed a challenge) during the grace period. This incentivizes holding.

---

## LOW

### R6-LOW-01: Cross-section inconsistency — constants table lists "Rent multiplier convergence | 10% per cycle" but §6 formula uses `cycles_elapsed / 10`
**Severity:** LOW
**Sections:** §6, Constants Summary
**Description:** The convergence formula `min(1, cycles_elapsed / 10)` is a linear ramp, not "10% per cycle." At cycle 1, the effective convergence is 10%. At cycle 5, it's 50%. The constants table saying "10% per cycle" suggests additive 10% per cycle, which matches the formula — but "10% per cycle" could also be read as multiplicative (compound 10%). The formula is unambiguous; the constants table entry is slightly ambiguous.

**Suggested fix:** Change constants table to: "Rent multiplier convergence | linear, full market rate at 10 cycles | §6"

---

### R6-LOW-02: protocol.md still uses old section numbers
**Severity:** LOW
**Sections:** docs/protocol.md
**Description:** protocol.md references §4 for Peer Discovery (correct), §7 for Proposals (correct), §8 for Votes (correct). But it doesn't include the newer message types (SHARE, REPLICATE_REQUEST, REPLICATE_ACCEPT, RENT_PAYMENT, CONTENT_WITHDRAW, FLAG, CHALLENGE_RESULT, COMMENT, DID_LINK, DID_REVOKE). The quick reference is stale relative to the full spec.

**Suggested fix:** Update protocol.md to include all message types from the §2 Message Type Inventory.

---

## CROSS-SECTION CONSISTENCY AUDIT

After 3 rounds of edits, checking for internal agreement:

| Check | Status | Notes |
|-------|--------|-------|
| §6 validator 80/20 split ↔ Constants table | ✓ CONSISTENT | Both say 80/20 |
| §6 min 10 challenges ↔ Constants table | ✓ CONSISTENT | Both say 10 from ≥2 validators |
| §6 rent convergence formula ↔ Constants | ⚠️ MINOR | Formula is clear, constant entry slightly ambiguous (R6-LOW-01) |
| §6 REPLICATE_ACCEPT ↔ §2 Message Type Inventory | ✓ CONSISTENT | Listed in inventory |
| §6 content opacity ↔ §6 flagging | ✓ CONSISTENT | No contradiction, but blind spot acknowledged (R6-MED-01) |
| §6 grace period (7/3/0 days) ↔ Constants | ✓ CONSISTENT | |
| §9 "Transferred" table ↔ §6 storage provider revenue | ✓ CONSISTENT | §9 explicitly says "carved from author's +0.05 pool, NOT additional creation" |
| §8 min rep to vote (0.3) ↔ §9 Capability Ramp | ✓ CONSISTENT | §8 now explicitly states the 0.3 requirement |
| §6 RENT_PAYMENT as coordination signal ↔ §9 locally computed reputation | ✓ CONSISTENT | Each node independently computes, RENT_PAYMENT is informational |
| §6 provenance credit cap below 0.3 (0.002) ↔ Constants | ✓ CONSISTENT | |
| §12 partition merge identity state ↔ §1 DID_REVOKE | ✓ CONSISTENT | Revoke takes precedence |
| §6 CONTENT_WITHDRAW effective_after ↔ §6 active proposal blocking | ✓ CONSISTENT | 24h delay, proposals block withdrawal |

**Overall:** Cross-section consistency is strong. The 5 rounds of edits have been well-integrated. Only one minor ambiguity found (R6-LOW-01).

---

## ECONOMIC LIFECYCLE TRACE

Full lifecycle of content from SHARE through CONTENT_WITHDRAW, every reputation flow:

### Phase 1: Hosted
- **SHARE** broadcast → no reputation cost ✓
- Content available only while host is online ✓
- Can be flagged (attribution to host) ✓

### Phase 2: Replicated
- **REPLICATE_REQUEST** broadcast (requires rep ≥ 0.3) → `reputation_stake` signals affordability ✓
- **REPLICATE_ACCEPT** from providers → shard assignment ✓ (but no SHARD_ASSIGNMENT confirmation — R6-HIGH-03)
- Shards distributed via `/valence/content/1.0.0` ✓
- **Billing cycle 1:** Uploader rep deducted (rent = size × base_rate × locked_multiplier). 80% to providers who passed ≥10 challenges from ≥2 validators. 20% to validators proportional to challenges. ✓
- **Subsequent cycles:** locked_multiplier converges at 10%/cycle toward current. ✓
- **Grace period entry:** Unpaid rent → 7 days (3 days 2nd, 0 days 3rd within 90 days). Providers unpaid. ✓
- **Grace period recovery:** Uploader pays next cycle; skipped cycle is lost to providers. ⚠️ (R6-MED-05)
- **Decayed:** GC eligible, providers MAY drop shards. ✓

### Phase 3: Proposed
- **PROPOSE** referencing replicated content_hash ✓
- Rent exempt during voting period IF ≥3 endorse votes ✓
- **Adoption:** +0.005/ADOPT, max +0.05. 70% to author, 30% to storage providers (carved from same pool). ✓
- **With provenance:** Author's 70% further split 70/30 with original host. Below-0.3 hosts capped at 0.002. ✓
- Full flow with both: 0.05 total → 0.015 to providers, 0.0245 to proposer, 0.0105 to host = 0.05. ✓ Conservative.

### Phase 4: Withdrawal
- **CONTENT_WITHDRAW** broadcast, effective_after ≥ 24h. ✓
- Blocked by active proposals (voting_deadline not passed). ✓
- After effective_after: providers drop shards, no further rent. ✓

### Phase 5: Flagging (alternate path)
- **FLAG** (dispute): -0.005 stake from flagger. Accumulation → deprioritization → governance. ✓
- **FLAG** (illegal): -0.02 stake. ≥5 flags from ≥3 ASNs → shard deletion. ✓
- Upheld: -0.05 to content source. Stake returned to flagger. ✓
- Rejected: stake forfeited + -0.05 for frivolous severe flags. ✓

**Assessment:** The lifecycle is complete and the reputation flows are internally consistent. The only gap is provider compensation during grace period recovery (R6-MED-05).

---

## SECOND-ORDER INTERACTIONS BETWEEN ROUNDS 3-5 FIXES

### Interaction 1: Validator economy (R5) + Challenge-per-cycle minimum (R5) + Lazy provider fix (R5)
**Status:** The min 10 challenges from ≥2 validators addresses the lazy provider attack (R5-HIGH-04). The validator 20% share incentivizes challenge issuance. These interact well EXCEPT for the validator spam incentive (R6-HIGH-02) — the per-challenge reward creates a volume race.

### Interaction 2: Rent convergence (R5) + Scarcity curve cap at 100× (R3) + Storage land grab (R5-CRIT-01)
**Status:** The convergence formula was the fix for R5-CRIT-01 (replacing permanent lock-in). It's better than permanent locking but still allows 10 months of below-market rates (R6-HIGH-04). The quartic cap at 100× bounds the maximum divergence. These interact correctly but the convergence speed is the remaining tuning question.

### Interaction 3: Provenance cap below 0.3 (R4/R5) + On-ramp language
**Status:** R5-HIGH-01 was addressed by adding the 0.002/proposal cap. The spec now correctly describes this as a functional but slow on-ramp. The language is consistent. ✓

### Interaction 4: RENT_PAYMENT as coordination signal (R5) + Locally computed reputation (§9)
**Status:** R5-CRIT-02 was addressed by making RENT_PAYMENT informational with independent computation. Each node computes rent from first principles. This is the correct architecture. ✓

### Interaction 5: CONTENT_WITHDRAW with 24h delay (R5) + Proposal blocking + Partition merge
**Status:** The 24h effective_after delay addresses the race condition (R5-HIGH-02). Partition merge for content state is union semantics. A CONTENT_WITHDRAW seen on one partition but not the other propagates on merge. If a PROPOSE was created on the other partition before effective_after, it blocks the withdrawal on merge. This is deterministic and correct. ✓

### Interaction 6: Grace period escalation (7/3/0 days, R3) + Rent convergence (R5) + Velocity caps (§9)
**Status:** An uploader whose locked rent converges toward an unsustainable rate will hit grace period. The escalating grace periods (7 → 3 → 0) mean they get fewer chances to recover. Combined with velocity-capped gains, a node in this situation must either CONTENT_WITHDRAW or accept content loss. This is harsh but functional — it prevents zombie content from occupying capacity indefinitely. The interaction is correct. ✓

---

## SUMMARY

| Severity | Count | IDs |
|----------|-------|-----|
| CRITICAL | 0 | — |
| HIGH | 4 | R6-HIGH-01 through R6-HIGH-04 |
| MEDIUM | 5 | R6-MED-01 through R6-MED-05 |
| LOW | 2 | R6-LOW-01, R6-LOW-02 |
| **Total** | **11** | |

## OVERALL ASSESSMENT

The spec is in strong shape after 5 rounds. No critical vulnerabilities remain. The core governance, identity, and economic systems are internally consistent and well-specified. The storage validator economy is the newest subsystem and carries the most remaining risk — specifically around validator incentive alignment (spam vs. quality) and provider-validator collusion. The REPLICATE_ACCEPT flow needs a shard assignment confirmation mechanism to prevent silent state divergence. The rent convergence formula works but could converge faster (5-month full convergence would be more appropriate than 10).

**If deploying today, the highest-priority fixes are:**
1. **R6-HIGH-02** (validator spam incentive) — cap per-validator earnings per content per cycle
2. **R6-HIGH-03** (REPLICATE_ACCEPT coordination gap) — add SHARD_ASSIGNMENT message
3. **R6-HIGH-01** (validator-provider collusion) — require ≥2 ASN-diverse validators

The rest are documentation improvements, operational guidance, and tuning parameters. The protocol's fundamental architecture is sound.
