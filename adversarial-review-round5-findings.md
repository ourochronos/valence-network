# Valence Network v0 Protocol — Adversarial Review Round 5 Findings

**Reviewer:** Subagent (round5-audit)
**Date:** 2025-02-18
**Scope:** Post-round-4 fixes. Focus on fix interactions, second-order game theory, remaining spec gaps, economic coherence, message flow completeness, edge cases in new messages.

---

## CRITICAL

### R5-CRIT-01: Rent price locking + scarcity multiplier creates unbounded storage subsidy attack
**Severity:** CRITICAL
**Sections:** §6 Storage Economics (rent price locking, scarcity multiplier)
**Description:** Round 4 introduced rent price locking: "The scarcity multiplier used for rent calculation is locked at the time of replication." The quartic scarcity curve caps at 100×. These two fixes interact destructively.

**Attack scenario:**
1. Network is at 20% utilization. Scarcity multiplier ≈ 1.0.
2. Attacker replicates maximum content at multiplier 1.0. Monthly rent for 10 MiB = 10 × 0.001 × 1.0 = 0.01/month. Trivially cheap.
3. Network grows organically to 80% utilization. New replication prices at multiplier ~41.6×. But attacker's rent stays locked at 1.0×.
4. Attacker is paying 0.01/month for storage that should now cost 0.416/month — a 41.6× subsidy.
5. Attacker has NO incentive to withdraw. Their storage occupies real capacity, contributing to the 80% utilization that prices out legitimate new uploaders.
6. At scale: early adopters lock in cheap rent, fill capacity, and the scarcity multiplier punishes only newcomers. This is a first-mover storage land grab.

**Numbers:**
- 100 early identities at rep 0.3, each replicating 22.5 MiB at multiplier 1.0.
- Total: 2,250 MiB at 0.001/MiB/month = 2.25 reputation/month total (trivial, 0.0225/identity).
- If this fills 80% of network capacity, new uploaders face multiplier 41.6× while early lockers pay 1.0×.
- The locked content never churns because rent is cheap. Network storage becomes permanently occupied at below-market rates.

**The interaction:** Rent price locking was added (R4-CRIT-02) to prevent a death spiral from rising scarcity. But it creates the opposite problem: permanently subsidized storage for early movers that prevents efficient allocation.

**Suggested fix:**
1. Lock the multiplier for the first billing cycle only. Subsequent cycles use a blended rate: `locked_multiplier × 0.5 + current_multiplier × 0.5`. This dampens volatility while allowing price convergence over time.
2. OR: Lock rent for 6 billing cycles (6 months), then transition to current market rate with a maximum 2× increase per cycle (graduated price adjustment).
3. OR: Locked multiplier decays toward current: `effective_multiplier = locked + (current - locked) × min(1, cycles_elapsed / 12)`. Full convergence after 1 year.

---

### R5-CRIT-02: RENT_PAYMENT message is self-attested — no consensus on payment validity
**Severity:** CRITICAL
**Sections:** §6 Rent Payments, §9 Reputation
**Description:** RENT_PAYMENT is broadcast by the content uploader. The spec says "Storage providers verify the amount against the locked multiplier and their shard count." But there's no protocol mechanism for providers to accept or reject. There's also no mechanism for third parties to verify that the uploader's reputation was actually deducted.

**The fundamental problem:** Reputation is locally computed (§9). There is no global ledger. When an uploader broadcasts RENT_PAYMENT claiming to pay 0.01 to provider X:
- How does provider X know the uploader's reputation was actually reduced by 0.01?
- How do third-party nodes know this wasn't a fabricated payment?
- What if the uploader broadcasts RENT_PAYMENT but their locally-computed reputation was never deducted (because deduction is... also local)?

**Attack scenario:**
1. Uploader U broadcasts RENT_PAYMENT with correct amounts.
2. U never actually reduces their own reputation score.
3. Storage providers see the RENT_PAYMENT, increase their local assessment of their own reputation.
4. Net effect: reputation was created from nothing. Provider gained, uploader didn't lose.
5. Third-party nodes have no way to verify — they see the RENT_PAYMENT message, and their local reputation calculations for U and the providers will diverge based on whether they trust the payment.

**This is the core economic integrity problem:** In a system with locally-computed reputation and no global state, a signed "I paid" message is not proof of payment. It's an assertion.

**Suggested fix:**
1. Rent payment must be verified by the challenge system: providers who pass challenges are credited rent by EACH node's local reputation computation. The RENT_PAYMENT message becomes informational (the uploader's assertion of what they owe), and each node independently deducts from the uploader and credits providers in their local reputation view. No message is "proof of payment" — each node's local model accounts for the rent obligation based on: (a) the REPLICATE_REQUEST's locked multiplier, (b) the content's shard distribution, (c) which providers passed challenges.
2. Add explicit language: "Each node MUST independently compute rent obligations and adjustments to reputation scores. The RENT_PAYMENT message is a coordination signal, not an authoritative transfer. Nodes that observe divergence between a RENT_PAYMENT and their own calculation SHOULD trust their own calculation."

---

## HIGH

### R5-HIGH-01: Provenance gate (0.3) + capability ramp interaction creates dead-letter provenance credits
**Severity:** HIGH
**Sections:** §6 Content Provenance, §9 Capability Ramp
**Description:** Round 4 fix (R4-CRIT-01) added: "Provenance credit recipients MUST have reputation ≥ 0.3 to receive their share of adoption rewards. Below 0.3, the proposer receives 100%."

This interacts with the stated purpose of provenance: "Provenance credit is the on-ramp for content creators who haven't reached the 0.3 reputation threshold to propose."

**The contradiction:** Provenance was designed to help sub-0.3 nodes earn reputation. The fix gates provenance credit at 0.3. Now sub-0.3 nodes cannot receive provenance credit — the exact population the feature was designed to serve. The on-ramp has been walled off.

**Result:** Provenance credit is now only useful between already-established nodes (both ≥ 0.3). It serves no bootstrapping function. The spec still describes it as an on-ramp, which is misleading.

**Suggested fix:**
1. Set the provenance gate at 0.25 (halfway between starting 0.2 and proposal threshold 0.3). This still prevents fresh-identity farming (need some uptime/contribution) while preserving the on-ramp function.
2. OR: Gate at 0.3 but with a carve-out: provenance credit below 0.3 is capped at 0.002 per proposal (1/10th of normal max). Slow but functional on-ramp.
3. AND: Update the spec prose to accurately describe the current behavior. Remove or rewrite the "on-ramp for content creators who haven't reached 0.3" paragraph if the gate stays at 0.3.

---

### R5-HIGH-02: CONTENT_WITHDRAW + active proposal creates timing ambiguity
**Severity:** HIGH
**Sections:** §6 Content Withdrawal, §7 Proposals
**Description:** The spec says: "Content referenced by an active proposal (voting period not expired) cannot be withdrawn — the CONTENT_WITHDRAW MUST be rejected until the proposal concludes."

**Edge cases:**
1. **Racing messages:** Uploader broadcasts CONTENT_WITHDRAW. Simultaneously (within gossip propagation delay), another identity broadcasts PROPOSE referencing the same content_hash. Due to gossip ordering, some nodes receive WITHDRAW first (and accept it), others receive PROPOSE first (blocking future WITHDRAW). Network state diverges permanently.

2. **Multiple proposals:** Content C is referenced by proposals P1 (deadline T+30d) and P2 (deadline T+60d). CONTENT_WITHDRAW is submitted at T+35d after P1 concludes. Is P2 still blocking? The spec says "an active proposal" (singular) — ambiguous whether it means any or all.

3. **Proposal on withdrawn content:** If CONTENT_WITHDRAW propagates to node A but not yet to node B, and B creates a PROPOSE referencing the withdrawn content, node A rejects the proposal (content withdrawn) while node B accepts it. Permanent divergence on whether the proposal is valid.

**Suggested fix:**
1. CONTENT_WITHDRAW MUST include a `effective_after` timestamp (suggested: 24 hours from broadcast) to allow gossip convergence before taking effect. Any PROPOSE received before `effective_after` blocks the withdrawal.
2. Clarify: "Content cannot be withdrawn while referenced by ANY active proposal (any proposal whose `voting_deadline` has not yet passed)."
3. PROPOSE MUST be rejected if the content has a pending CONTENT_WITHDRAW (even if not yet effective). This prevents the race.

---

### R5-HIGH-03: Combined economic flows create net reputation inflation beyond stated creation sources
**Severity:** HIGH
**Sections:** §6 Content Lifecycle, §9 Reputation
**Description:** §9 claims the ONLY sources of new reputation are: proposal adoption, claim verification, contradiction finding, and uptime. But §6 says: "Storage providers who held shards during the voting period share the remaining 30% [of adoption rewards]... No additional reputation is created — this is a redistribution of the existing adoption reward pool."

**The math doesn't add up.** Adoption reward per ADOPT is +0.005, capped at +0.05 per proposal. The spec says the author gets 70% (+0.035) and storage providers get 30% (+0.015). But then it says: "this is independent of the §6 Provenance split — if both apply, provenance splits the author's 70% further into 70/30 with the original host."

**Full flow with both provenance AND storage provider split:**
- 10 ADOPTs → +0.05 total created
- 70% to "author side" = 0.035
  - If provenance applies: 70% of 0.035 = 0.0245 to proposer, 30% = 0.0105 to original host
- 30% to storage providers = 0.015

Total distributed: 0.0245 + 0.0105 + 0.015 = 0.05. ✓ This checks out.

**BUT:** The "Transferred" table in §9 says: "Replication of proposed content | Network (adoption pool) | Storage providers | Share of proposal adoption rewards." The word "Network (adoption pool)" as a source implies NEW creation, not a split of existing adoption rewards. The table format (separate row from proposal adoption) suggests this is an ADDITIONAL transfer.

**The inconsistency:** Is the storage provider 30% carved FROM the +0.05 adoption pool, or ADDED to it? The §6 text says "carved from." The §9 table structure implies "additional." A correct implementation can't tell which to implement.

**Suggested fix:** Rewrite the §9 "Transferred" table entry for "Replication of proposed content" to explicitly say: "30% of the per-proposal adoption reward cap (+0.05) is redistributed to storage providers. This is NOT additional reputation creation — it comes from the same +0.05 pool." Remove "Network (adoption pool)" as a source and replace with "Proposal author's adoption rewards."

---

### R5-HIGH-04: Storage challenge timing + rent billing cycle creates a free-riding window
**Severity:** HIGH
**Sections:** §6 Storage Challenges, §6 Storage Provider Revenue
**Description:** The spec says: "Providers who pass at least one challenge during that billing cycle" receive rent. Challenge frequency is "suggested: once per shard per day." Billing cycles are 30 days.

**Attack scenario (lazy provider):**
1. Storage provider P accepts shards for content C.
2. P holds shards for the first day of the billing cycle, passes a challenge.
3. P deletes the shards on day 2 (frees disk space).
4. P has passed "at least one challenge during that billing cycle."
5. P collects full rent for the 30-day cycle despite holding shards for 1 day.
6. P repeats next cycle: hold shards for day 1, get challenged, delete, collect rent.

**Impact:** Storage providers can commit 1/30th of the resources and earn full rent. Content durability is severely compromised — shards exist only on challenge days.

**Why this works:** Challenges are "suggested frequency: once per shard per day" (SHOULD, not MUST), and the spec only requires passing "at least one challenge during that billing cycle." The minimum viable strategy is to hold shards briefly, pass one challenge, then free the space.

**Suggested fix:**
1. Require passing challenges on ≥ N distinct days per billing cycle (e.g., ≥ 20 of 30 days) to receive rent. A single pass is insufficient.
2. OR: Challenges are mandatory (MUST, not SHOULD) and must occur at random intervals unknown to the provider. Rent is pro-rated by challenges passed: if a provider passes 15 of 30 challenges, they receive 50% of rent.
3. OR: Rent is paid per-challenge-passed, not per-cycle. Each successful challenge earns `monthly_rent / expected_monthly_challenges`.

---

### R5-HIGH-05: 60-day unlinked key voting cooldown is unenforceable across partitions
**Severity:** HIGH
**Sections:** §1 Identity Linking (DID_REVOKE), §12 Partition Detection
**Description:** Round 4 fix added a 60-day voting cooldown for revoked keys that re-register as independent identities. But in a partition scenario:

1. Identity R revokes child C on Partition A at time T.
2. Partition B never sees the DID_REVOKE.
3. C continues operating as part of R's identity on Partition B.
4. Partitions merge at T+30d.
5. Partition A nodes see C as "revoked, 60-day cooldown, can't vote independently until T+60."
6. Partition B nodes see C as "still linked to R, votes attributed to R."

**On merge:** Is C linked or revoked? §12 says flag state and content state merge via union. DID_REVOKE is not covered. If DID_REVOKE propagates (union), C is revoked everywhere — but C's votes on Partition B between T and T+30 were attributed to R. Were they valid? Should they be re-evaluated?

**The spec's partition merge rules (§12) don't cover identity state changes.**

**Suggested fix:** Add to §12 Content and Storage State merge rules: "Identity state (DID_LINK, DID_REVOKE) merges via union with earliest-timestamp priority. DID_REVOKE messages propagate from either partition. Votes cast by a revoked key between the revocation timestamp and the merge point are re-evaluated: if the key was revoked before the vote's timestamp, the vote is invalidated. The 60-day cooldown starts from the DID_REVOKE timestamp, not the merge point."

---

## MEDIUM

### R5-MED-01: RENT_PAYMENT `billing_cycle` field creates ambiguity for late payments
**Severity:** MEDIUM
**Sections:** §6 Rent Payments
**Description:** RENT_PAYMENT includes `"billing_cycle": <integer — cycle number from replication timestamp>`. But the spec doesn't define:

1. What happens if an uploader skips a cycle and pays for cycle 3 without having paid cycle 2?
2. Can an uploader pay for future cycles (prepay)?
3. What if two RENT_PAYMENT messages arrive for the same cycle (duplicate or conflicting amounts)?
4. Is there a deadline for cycle N's payment? (Start of cycle N? End of cycle N? Start of cycle N+1?)

**Impact:** Different implementations will handle these differently, causing reputation divergence.

**Suggested fix:** Add: "RENT_PAYMENT for cycle N MUST be broadcast within the first 7 days of cycle N. Late payments are treated as missed (grace period triggers). Skipped cycles cannot be retroactively paid. Duplicate RENT_PAYMENT for the same (content_hash, billing_cycle) pair: first-seen wins, subsequent are rejected. Prepayment is not supported — each cycle's payment is broadcast during that cycle."

---

### R5-MED-02: Scarcity multiplier 24-hour moving average + rent price locking creates observation window attack
**Severity:** MEDIUM
**Sections:** §6 Storage Economics
**Description:** The scarcity multiplier uses a "24-hour moving average of utilization" (R3 fix). Rent price is "locked at the time of replication" (R4 fix). An attacker can manipulate the 24-hour window to lock in a favorable rate:

1. Attacker controls several nodes with large `available_bytes` claims (reputation-weighted, but attacker has 0.3+ rep).
2. Attacker announces large available capacity, depressing the 24-hour average utilization.
3. In the same window, attacker replicates content, locking in the low multiplier.
4. Attacker then removes the fake capacity announcements. Multiplier recovers.
5. Attacker has permanently locked in a below-market rate.

**Mitigation already in spec:** Capacity claims are weighted by reputation, and 24-hour moving average dampens sudden changes. But the attack still works over 24 hours if the attacker has enough reputation-weighted capacity to meaningfully shift the average.

**Suggested fix:** Lock the multiplier as the average over the FIRST billing cycle (30 days), not a snapshot at replication time. This makes momentary manipulation ineffective. OR: Use a 7-day moving average instead of 24-hour for the lock-in calculation specifically.

---

### R5-MED-03: No mechanism to verify CONTENT_WITHDRAW authenticity when original uploader used a since-revoked child key
**Severity:** MEDIUM
**Sections:** §6 Content Withdrawal, §1 Identity Linking
**Description:** CONTENT_WITHDRAW says "Only the identity that submitted the REPLICATE_REQUEST may withdraw." If the REPLICATE_REQUEST was submitted by child key C (attributed to root R), and C is later revoked:

1. R wants to withdraw content. R broadcasts CONTENT_WITHDRAW.
2. Nodes check: was the REPLICATE_REQUEST from R's identity? Yes (C was linked to R at the time).
3. But C is now revoked. Does R still "own" the content for withdrawal purposes?

The spec says content remains attributed to the root (R4-MED-02 fix). So R can withdraw. But:

4. C re-registers as an independent identity. C broadcasts CONTENT_WITHDRAW for the same content.
5. Nodes check: was the REPLICATE_REQUEST from C? The `from` field is C's key.
6. But C is no longer part of R's identity. Should C be able to withdraw R's content?

**Suggested fix:** Add: "Content ownership follows the identity, not the key. After DID_REVOKE, the revoked key has no authority over content attributed to the root identity. CONTENT_WITHDRAW MUST be signed by a key currently authorized under the identity that owns the content."

---

### R5-MED-04: Reputation velocity limits interact poorly with rent obligations at high reputation
**Severity:** MEDIUM
**Sections:** §9 Velocity Limits, §6 Storage Economics
**Description:** Max daily gain is 0.02. Max weekly gain is 0.08. There are no velocity limits on LOSSES (penalties are uncapped).

A high-reputation node (0.8) with significant storage obligations:
- Monthly rent at moderate utilization (locked multiplier 5.0): 62.5 MiB × 0.001 × 5.0 = 0.3125/month
- Monthly earnings cap: 0.08/week × 4.3 = ~0.34/month
- Net: barely positive (0.03/month surplus)

If the node receives ANY penalty (failed challenge -0.01, false claim -0.003), the rent can become unsustainable for that month. The penalty triggers immediate reputation loss (uncapped), but recovery is capped at 0.02/day. A cascade of small penalties can make rent unpayable.

**This isn't a protocol-breaking issue** but creates fragility for high-reputation, high-storage nodes. The asymmetry between uncapped losses and capped gains means established nodes are more fragile than the economics suggest at first glance.

**Suggested fix:** Document this interaction explicitly in §6 or §9: "Node operators should maintain a reputation buffer above their minimum rent obligations. Reputation penalties are not velocity-limited and can cause temporary inability to pay rent." This is an operational consideration, not a protocol fix.

---

### R5-MED-05: Message flow gap — no REPLICATE_ACCEPT or shard assignment protocol
**Severity:** MEDIUM
**Sections:** §6 Content (Replicated)
**Description:** The spec describes REPLICATE_REQUEST but never specifies how storage providers respond. The flow has a gap:

1. Uploader broadcasts REPLICATE_REQUEST on `/valence/proposals`.
2. ??? — How do storage providers volunteer? How does the uploader know who will store shards?
3. Uploader "distributes shards to willing peers."

There's no REPLICATE_ACCEPT, SHARD_OFFER, or SHARD_ASSIGNMENT message. A correct implementation cannot determine:
- How a provider signals willingness to store shards
- How shards are assigned to specific providers
- How the uploader discovers willing providers
- What happens if not enough providers volunteer

**Suggested fix:** Add a REPLICATE_ACCEPT message:
```json
{
  "type": "REPLICATE_ACCEPT",
  "payload": {
    "content_hash": "<sha256_hex>",
    "shard_indices": [<integer — which shards this provider will hold>],
    "capacity_available": <bytes>
  }
}
```
Broadcast on `/valence/proposals`. Uploader collects acceptances, assigns shards, and distributes via direct stream. If fewer than `data_shards + parity_shards` providers volunteer within 48 hours, the replication fails and the uploader is notified.

---

### R5-MED-06: Tenure penalty + gain dampening compound unfairly against multi-key long-running operators
**Severity:** MEDIUM
**Sections:** §1 Identity Linking, §11 Anti-Gaming
**Description:** A legitimate operator running 4 keys for 7+ months faces:
- Gain dampening: `raw_gain / 4^0.75` = 35.4% effective gain
- Tenure penalty: 5% vote weight reduction per cycle after 6th (they're at cycle 7+)

These compound. At cycle 8 (2 cycles of penalty):
- Vote weight: reputation × 0.95² = 0.9025×
- Gain rate: 35.4% of raw
- Penalty rate: 100% (undampened)

A single-key node at the same tenure:
- Vote weight: reputation × 0.9025×
- Gain rate: 100% of raw
- Penalty rate: 100%

**The multi-key operator earns reputation ~3× slower and loses it at the same rate.** The spec acknowledges the asymmetric risk (§1) but the combination with tenure penalty creates a triple disadvantage: slower gains, same losses, AND reduced vote weight. This may discourage legitimate multi-device operations (home node + cloud relay).

**Suggested fix:** This is arguably working as designed (linking is a constraint, not a benefit). But consider: tenure penalty should apply to the identity's vote weight, not compound with gain dampening on earning. These are independent mechanisms penalizing different things (entrenchment vs. multi-key scaling). Document that they're intended to compound, or make tenure penalty apply to vote weight only (not to earning rate).

---

## LOW

### R5-LOW-01: RENT_PAYMENT and CONTENT_WITHDRAW are broadcast on `/valence/proposals` — topic overloading
**Severity:** LOW
**Sections:** §2 Message Type Inventory, §6
**Description:** RENT_PAYMENT and CONTENT_WITHDRAW are broadcast on `/valence/proposals`, the same topic as PROPOSE, VOTE, ADOPT, etc. At scale, rent payments will vastly outnumber proposals:
- 1,000 pieces of replicated content × 1 RENT_PAYMENT/month = 1,000 messages/month (33/day)
- Network with 100 active proposals = ~200 governance messages/day

Rent payments could represent 15%+ of proposals topic traffic. They're economically important but semantically different from governance.

**Suggested fix:** Consider a dedicated `/valence/storage` topic for RENT_PAYMENT, CONTENT_WITHDRAW, REPLICATE_REQUEST, and CHALLENGE_RESULT. Not critical for v0, but document the scaling concern in §15.

---

### R5-LOW-02: CONTENT_WITHDRAW `reason` field has no size limit
**Severity:** LOW
**Sections:** §6 Content Withdrawal
**Description:** Unlike FLAG (which has a 10 KiB limit on `details`), CONTENT_WITHDRAW's `reason` field has no specified size limit. Could be used for message bloat (up to 8 MiB payload limit).

**Suggested fix:** Add: "The `reason` field MUST NOT exceed 1 KiB."

---

### R5-LOW-03: Conformance gap — no test vectors for RENT_PAYMENT or CONTENT_WITHDRAW
**Severity:** LOW
**Sections:** §6 (new message types)
**Description:** These are round 4 additions with economic consequences but no conformance test coverage. Particularly important: RENT_PAYMENT amount calculation (locked multiplier × content size × base rate) and CONTENT_WITHDRAW rejection when proposal is active.

**Suggested fix:** Add test vectors for:
- RENT_PAYMENT amount calculation with locked vs current multiplier
- CONTENT_WITHDRAW rejection during active proposal voting period
- CONTENT_WITHDRAW acceptance after proposal concludes
- Duplicate RENT_PAYMENT handling

---

### R5-LOW-04: `locked_multiplier` in RENT_PAYMENT is redundant but creates a consistency check burden
**Severity:** LOW
**Sections:** §6 Rent Payments
**Description:** RENT_PAYMENT includes `"locked_multiplier"` — the scarcity multiplier at replication time. But this should be derivable from the original REPLICATE_REQUEST and the network state at that time. Including it in RENT_PAYMENT creates a consistency check: what if the uploader claims a lower locked_multiplier than what was actually in effect?

Every node must independently verify the locked multiplier against the original replication timestamp. The field is either redundant (nodes already know it) or a potential source of divergence (nodes disagree on what the multiplier was at replication time, since it's locally computed).

**Suggested fix:** Remove `locked_multiplier` from RENT_PAYMENT. Each node computes it independently from the REPLICATE_REQUEST timestamp and their historical scarcity data. OR: Include it as informational but specify "nodes MUST verify against their own records and reject RENT_PAYMENT messages where the claimed locked_multiplier diverges by more than 10% from their local calculation."

---

## SPECIFICATION GAPS (Not re-reported from R4)

### R5-GAP-01: No protocol for shard redistribution when a storage provider leaves
When a storage provider goes offline permanently, their shards are lost. §6 "Shard Health" says nodes "SHOULD periodically verify" and "SHOULD re-distribute shards to willing peers." But there's no message type for re-distribution requests, no protocol for who initiates re-distribution, and no incentive structure (who pays for re-replication?).

### R5-GAP-02: No mechanism to discover content's current rent status
Nodes can discover shards (SHARD_QUERY) and content metadata (via SHARE/REPLICATE_REQUEST in gossip). But there's no query for: "Is content C currently in grace period? When is rent due? Who are the current storage providers?" This information is essential for nodes deciding whether to store shards or propose content.

### R5-GAP-03: Storage challenge initiation after shard reassignment
When shards are redistributed to new providers, the new provider hasn't been challenged yet. The spec says "providers who pass at least one challenge during that billing cycle" get rent. But a new provider receiving shards mid-cycle may not be challenged before the cycle ends. Do they get prorated rent? No rent? The spec doesn't say.

---

## ECONOMIC MODEL COHERENCE CHECK

### Reputation Sources (Creation)
| Source | Rate | Bounded? |
|--------|------|----------|
| Proposal adoption | +0.005/ADOPT, max +0.05/proposal | Yes (3 proposals/week, velocity cap) |
| Claim verified true | +0.001 | Yes (requires verifiable claims) |
| Claim found false | +0.005 | Yes (requires false claims to find) |
| First contradiction | +0.01 | Yes (one per false claim) |
| Uptime | +0.001/day, caps at 30 days | Yes (0.03/month max) |

### Reputation Sinks (Destruction)
| Sink | Rate | Bounded? |
|------|------|----------|
| False claims | -0.003/claim | No cap on losses |
| Failed storage challenge | -0.01 | No cap |
| Collusion | -0.05 | No cap |
| Inactivity | -0.02/month | Bounded by floor (0.1) |
| Content flagged | -0.05 | No cap |
| Severe ban | -all | One-time |
| False flag | -0.005 to -0.07 | No cap |

### Reputation Transfers (Zero-sum)
| Transfer | From | To |
|----------|------|----|
| Storage rent | Uploader | Providers |

### Maximum creation rate per identity
- Uptime: 0.03/month
- 3 proposals/week × max +0.05 each = 0.60/month (but velocity-capped at 0.08/week = 0.34/month)
- Effective max: ~0.34/month (velocity cap is binding)

### Assessment
The economic model is approximately coherent. Creation is velocity-capped. Destruction is uncapped (by design — bad behavior should be punished swiftly). Transfer is zero-sum. The one remaining inconsistency (R5-HIGH-03) about whether storage providers for proposed content earn from the adoption pool or in addition to it needs explicit resolution. Otherwise, **the economic model is sound**.

---

## MESSAGE FLOW COMPLETENESS CHECK

### Content Lifecycle: Hosted → Replicated → Proposed
1. **Host content:** SHARE → `/valence/peers` ✓
2. **Download hosted:** CONTENT_REQUEST → CONTENT_RESPONSE (stream) ✓
3. **Request replication:** REPLICATE_REQUEST → `/valence/proposals` ✓
4. **Accept replication:** ❌ **NO MESSAGE** (R5-MED-05)
5. **Distribute shards:** Not specified (direct transfer, presumably stream) ⚠️
6. **Propose content:** PROPOSE → `/valence/proposals` ✓
7. **Vote:** VOTE → `/valence/votes` ✓
8. **Adopt:** ADOPT → `/valence/proposals` ✓
9. **Pay rent:** RENT_PAYMENT → `/valence/proposals` ✓
10. **Challenge storage:** STORAGE_CHALLENGE → STORAGE_PROOF (stream) ✓
11. **Report challenge:** CHALLENGE_RESULT → `/valence/proposals` ✓
12. **Withdraw content:** CONTENT_WITHDRAW → `/valence/proposals` ✓
13. **Query shards:** SHARD_QUERY → SHARD_QUERY_RESPONSE (stream) ✓

**Gap:** Step 4 (replication acceptance) has no message. Step 5 (shard distribution) uses unspecified direct transfer.

### Identity Lifecycle
1. **Register:** VDF computation → AUTH_CHALLENGE/RESPONSE ✓
2. **Announce:** PEER_ANNOUNCE → `/valence/peers` ✓
3. **Link keys:** DID_LINK → `/valence/peers` ✓
4. **Revoke keys:** DID_REVOKE → `/valence/peers` ✓
5. **Rotate keys:** KEY_ROTATE → `/valence/peers` ✓
6. **Conflict detection:** KEY_CONFLICT → `/valence/peers` ✓

**Complete.** All identity operations have corresponding messages.

### Proposal Lifecycle
1. **Request:** REQUEST → `/valence/proposals` ✓
2. **Propose:** PROPOSE → `/valence/proposals` ✓
3. **Comment:** COMMENT → `/valence/proposals` ✓
4. **Vote:** VOTE → `/valence/votes` ✓
5. **Withdraw:** WITHDRAW → `/valence/proposals` ✓
6. **Adopt:** ADOPT → `/valence/proposals` ✓
7. **Supersede:** new PROPOSE with `supersedes` field ✓

**Complete.**

### Storage Challenge Lifecycle
1. **Issue challenge:** STORAGE_CHALLENGE (stream) ✓
2. **Respond:** STORAGE_PROOF (stream) ✓
3. **Report result:** CHALLENGE_RESULT → `/valence/proposals` ✓
4. **Rent payment:** RENT_PAYMENT → `/valence/proposals` ✓

**Complete** (assuming RENT_PAYMENT doubles as rent authorization).

---

## SUMMARY

| Severity | Count | IDs |
|----------|-------|-----|
| CRITICAL | 2 | R5-CRIT-01, R5-CRIT-02 |
| HIGH | 5 | R5-HIGH-01 through R5-HIGH-05 |
| MEDIUM | 6 | R5-MED-01 through R5-MED-06 |
| LOW | 4 | R5-LOW-01 through R5-LOW-04 |
| Spec gaps | 3 | R5-GAP-01 through R5-GAP-03 |
| **Total** | **20** | |

## TOP ATTACK PRIORITIES (if I were an adversary)

1. **R5-CRIT-01** — Lock in cheap storage early, then squat on network capacity forever. Zero risk, pure gain.
2. **R5-CRIT-02** — Fabricate RENT_PAYMENT messages. Since reputation is local, "payment" is just an assertion with no enforcement.
3. **R5-HIGH-04** — Store shards for 1 day per month, collect full rent. Free-ride on storage economics.
4. **R5-HIGH-02** — Race CONTENT_WITHDRAW against PROPOSE to create permanent state divergence.

## WHAT'S SOLID (Round 4 fixes that hold up)

✓ **Quartic scarcity curve with 100× cap** — well-bounded, no infinities
✓ **Grace period reduction to 7/3/0 days** — limits rent evasion effectively
✓ **Rent exemption requiring ≥3 votes** — harder to game than blanket exemption
✓ **60-day unlinked key cooldown** — addresses vote multiplication (though partition interaction needs work)
✓ **Partition merge rules for content state** — union semantics are clean and deterministic
✓ **Capacity claim reputation-weighting** — good defense against false capacity reports

## OVERALL ASSESSMENT

The spec has matured significantly through 4 rounds. The core governance mechanisms (identity, voting, proposals) are robust. The economic model is approximately coherent with one clarification needed (R5-HIGH-03). The two critical findings are both about the **rent/payment mechanism** — which is the newest and least battle-tested part of the spec. R5-CRIT-01 (rent price locking land grab) is the most impactful real-world vulnerability. R5-CRIT-02 (unverifiable rent payments) is a fundamental design question about how locally-computed reputation handles explicit transfer messages. Addressing these two would make the storage economics substantially more robust.
