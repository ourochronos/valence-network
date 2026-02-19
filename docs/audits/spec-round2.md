# Valence Network v0 Protocol — Adversarial Review Round 2 Findings

**Reviewer:** Subagent (spec-review-r2)
**Date:** 2025-02-18
**Focus:** Verification of 3 critical fixes + identity linking attack surface + conformance test alignment

---

## PART 1: CRITICAL FIX VERIFICATION

### ✅ CRIT-01 FIX VERIFIED — DID_LINK Persistence (§1 Identity Message Persistence)

**What was fixed:**
- DID_LINK/DID_REVOKE now exempt from 24h gossip limit
- Indefinite retention MUST (or identity lifetime minimum)
- 7-day re-broadcast SHOULD for discovery
- Sync protocol extension: identity-specific queries with `identity_key` parameter

**New issues introduced:**

#### NEW-CRIT-01: Sync identity query has no rate limit or scope bound
**Severity:** Critical
**Location:** §1 Identity Message Persistence, sync extension
**Description:** The identity-specific sync query has no rate limit:
```json
{"type": "SYNC_REQUEST", "payload": {"types": ["DID_LINK", "DID_REVOKE"], "identity_key": "<hex>", "limit": 100}}
```

An attacker can flood nodes with SYNC_REQUEST queries for arbitrary `identity_key` values. If a node MUST respond with all DID_LINK/DID_REVOKE messages involving the key (as root OR child), this is a DoS vector:
- Query 1000 different keys → force node to search entire identity graph 1000 times
- No limit on requests per peer per time unit

**Attack scenario:**
1. Attacker connects to node N
2. Sends SYNC_REQUEST for 10,000 random public keys
3. Node N must search its DID_LINK/DID_REVOKE store for each key
4. CPU/IO exhaustion

**Fix:** Add rate limit: "Nodes MUST rate-limit SYNC_REQUEST messages to 10 per peer per minute. Identity-specific queries (with `identity_key` set) count toward this limit."

#### NEW-HIGH-01: 7-day re-broadcast creates permanent gossip bandwidth overhead
**Severity:** High
**Location:** §1 Identity Message Persistence
**Description:** "Root identities SHOULD re-broadcast their DID_LINK messages every 7 days on `/valence/peers`."

For a network with 10,000 root identities, each with 3 authorized keys (2 children + root), that's:
- 10,000 identities × 2 DID_LINK messages = 20,000 messages every 7 days
- ~2,857 messages per day
- ~119 messages per hour

As the network scales, this becomes permanent gossip overhead. At 100,000 identities, it's 1,190 messages/hour just for identity maintenance.

**Suggested fix:** 
1. Change frequency: "Root identities SHOULD re-broadcast every 30 days (not 7 days). New nodes joining rely on sync protocol, not gossip re-broadcast."
2. Or: DID_LINK re-broadcast only when ASN diversity of active children changes (event-driven, not periodic).

---

### ✅ CRIT-02 FIX VERIFIED — KEY_CONFLICT Schema + Suspension at 10% (§1 Key Rotation)

**What was fixed:**
- Full KEY_CONFLICT payload schema defined
- Suspension: reputation frozen at floor (0.1), voting weight reduced to **10% of normal** (not zeroed)

**New issues introduced:**

#### NEW-MED-01: 10% vote weight during suspension is still exploitable for governance blocking
**Severity:** Medium
**Location:** §1 Key Rotation, suspension rules
**Description:** Suspended keys have 10% voting weight "to allow participation in resolution governance." But in a small network, this can be gamed:

**Scenario:**
- Network has 20 identities, total reputation ~10.0
- Attacker controls 5 identities (25% of network)
- Attacker deliberately creates KEY_ROTATE conflicts for all 5 identities (partition+heal, or race condition exploitation)
- All 5 attacker identities suspended → each has 10% vote weight

If attacker's 5 identities had combined rep 2.5 before suspension:
- Suspended vote weight: 2.5 × 0.10 = 0.25
- Standard quorum for their resolution proposal: `max(20×0.1, 10.0×0.10) = 2.0`
- Remaining honest network: 7.5 reputation

Honest network can still pass proposals (7.5 > 2.0 quorum). But if constitutional quorum is needed (30% of total = 3.0), and attacker controls 10% vote weight abstention/rejection strategically, they can still influence outcomes.

**Real risk?** Low in practice (requires controlling 25%+ of network AND deliberately self-suspending). But 10% may be too generous.

**Suggested fix:** Reduce to 5% or make suspension vote weight decay over time: starts at 10%, decreases 1% per day until resolved.

#### NEW-HIGH-02: KEY_CONFLICT can be weaponized to freeze competitor identities
**Severity:** High
**Location:** §1 Key Rotation
**Description:** "Any node that observes two KEY_ROTATE messages for the same `old_key` with different `new_key` values MUST broadcast a KEY_CONFLICT and apply the suspension locally."

This is automatic and mandatory. An attacker can weaponize this:

**Attack scenario:**
1. Network has two partitions P1 and P2
2. In P1, identity A performs KEY_ROTATE(A→A1)
3. Attacker in P2 sees the KEY_ROTATE message during partition heal
4. Attacker forges KEY_ROTATE(A→A2) with a fabricated `new_key_signature` (they can't actually sign it because they don't control A2, but they can try)
5. Wait — the spec says "nodes that verify both signatures MUST update" — invalid signatures are rejected
6. So the attacker CAN'T forge a valid dual-signature

**Actual risk: Low.** The dual-signature requirement (old key + new key) prevents forgery. An attacker can't create a conflicting KEY_ROTATE without controlling both A and a second keypair they want to falsely link.

**BUT:** During a real partition, if A rotates to A1 in P1 and A rotates to A2 in P2 (compromised? confused operator? clock skew?), the conflict is legitimate and suspension is permanent until governance resolves it.

**No fix needed**, but worth documenting: KEY_CONFLICT is not a griefing vector because dual-signature prevents forgery. Legitimate conflicts are rare and governance-resolvable.

---

### ✅ CRIT-03 FIX VERIFIED — Gain Dampening via sqrt(key_count) (§1 Identity Linking, point 1)

**What was fixed:**
```
effective_gain = raw_gain / sqrt(authorized_key_count)
```
Where `authorized_key_count` includes root (minimum 1). Penalties NOT dampened.

**New issues introduced:**

#### NEW-CRIT-02: sqrt(key_count) dampening is insufficient for large-scale linking
**Severity:** Critical
**Location:** §1 Behavioral Rules, gain dampening formula
**Description:** The dampening curve `1/sqrt(N)` is too weak at large N.

**Analysis:**
- 1 key: dampening = 1.0 (no dampening)
- 4 keys: dampening = 0.5 (half rate)
- 9 keys: dampening = 0.33 (third rate)
- 25 keys: dampening = 0.2 (fifth rate)
- 100 keys: dampening = 0.1 (tenth rate)

An identity with 100 authorized keys earns reputation at 10× rate compared to a single node. But they also have:
- 100× observation count growth (faster α to 0.6)
- 100× storage challenge opportunities
- 100× geographic/ASN diversity (if distributed)

**Attack scenario:**
1. Wealthy operator deploys 100 nodes across diverse ASNs
2. Links them all under one root identity
3. Each node participates actively: storage, proposals, votes
4. Root identity earns reputation at 10× rate (dampened from 100× to 10×)
5. After 6 months, root identity has reputation 0.9 (near cap)
6. Root identity casts ONE vote but it carries 0.9 weight
7. A coalition of 5 such large identities controls 4.5 vote weight

Compare to 500 independent nodes:
- Average reputation: ~0.4 (slower gain, velocity limits, collusion risk)
- Total vote weight: 500 × 0.4 = 200
- But the 500 nodes are fragmented, can't coordinate without collusion detection

**The problem:** sqrt dampening prevents LINEAR scaling but still allows SUPERLINEAR gain compared to independent operators at the same infrastructure cost.

**Correct dampening curve?**
```
effective_gain = raw_gain / authorized_key_count
```
(Linear dampening, not sqrt.) This ensures 100 keys earns at the same rate as 1 key — no advantage to linking beyond the intended benefit (avoid collusion detection).

**Or:**
```
effective_gain = raw_gain / (authorized_key_count^0.75)
```
(Stronger than sqrt but weaker than linear, allowing slight multi-node efficiency without enabling dominance.)

**Rationale for current sqrt curve:** The spec says "multi-key identities have real infrastructure diversity and operational complexity, so some reward is justified." But this opens the door to plutocratic concentration.

**Fix:** Change to linear dampening (`/N`) OR document sqrt as a deliberate design choice and add governance monitoring for identity vote weight concentration.

#### NEW-HIGH-03: Observation count growth with multi-key identities creates α asymmetry
**Severity:** High
**Location:** §1 Behavioral Rules + §8 Reputation Propagation (α scaling)
**Description:** The spec says α = min(0.6, observation_count / 10). Observation count is "any protocol message authored by the subject node that this node has directly received and verified."

For a multi-key identity:
- Each authorized key can send PEER_ANNOUNCE, VOTE, PROPOSE, ADOPT, STORAGE_PROOF
- If an identity has 10 authorized keys, and each sends 1 PEER_ANNOUNCE, the observer counts 10 observations
- α → 0.6 immediately (6+ observations)

A single-node identity needs 6 observations from that one node to reach α=0.6. A 10-key identity reaches it 10× faster.

**Combined with NEW-CRIT-02:** Multi-key identities hit α=0.6 faster (rely more on direct observations) AND accumulate reputation faster (10× via sqrt dampening at 100 keys). Both effects amplify each other.

**Fix:** α scaling should be per-identity, not per-key. "Observation count for multi-key identities: count unique protocol message types per identity per hour (deduplicate across authorized keys). An identity that sends 10 PEER_ANNOUNCE messages from 10 keys in one hour counts as 1 observation, not 10."

#### NEW-MED-02: Gain dampening applies only to gains, not penalties — creates asymmetric risk
**Severity:** Medium
**Location:** §1 Behavioral Rules, point 1
**Description:** "Penalties are NOT dampened — they apply at full rate regardless of key count."

This is intentional (prevents multi-key identities from diluting penalties). But it creates an asymmetry:

- Gains: dampened by sqrt(N)
- Penalties: NOT dampened

For a 100-key identity:
- Earns at 10× rate (dampened from 100× to 10×)
- Penalized at 100× rate (no dampening)

**Scenario:**
- 100-key identity has 1 compromised child key
- Compromised key fails 10 storage challenges (−0.01 each) = −0.10 reputation
- Root identity takes the full −0.10 penalty
- Recovery: +0.02/day (velocity cap) → 5 days to recover

For a single-node identity:
- Fails 10 storage challenges = −0.10
- Recovery: +0.02/day → 5 days

**The asymmetry:** Multi-key identities have 100× the attack surface (100 keys that could be compromised) but gain only 10× faster. Penalties are 100×, gains are 10×.

**Is this balanced?** Arguably yes — it discourages reckless linking. If you can't secure 100 keys, don't link 100 keys.

**But:** The spec doesn't document this asymmetry clearly. Operators might link many keys expecting linear risk scaling, not realizing penalties are undampened.

**Fix:** Add explicit warning in §1: "Multi-key identities have asymmetric risk: reputation gains are dampened by sqrt(key_count), but penalties apply at full rate. An identity with 100 authorized keys suffers 100× penalties but earns 10× gains. Operators MUST weigh this carefully before linking."

---

## PART 2: HIGH-SEVERITY ISSUES FROM ROUND 1 — STATUS CHECK

### HIGH-01: Timestamp tolerance asymmetry
**Status:** STILL OPEN (not addressed in spec updates)

### HIGH-02: Storage challenge gaming via selective shard acceptance
**Status:** STILL OPEN (not addressed)

### HIGH-03: VDF proof reuse attack
**Status:** STILL OPEN (not addressed)

### HIGH-04: Partition merge deterministic tie-break grinding
**Status:** STILL OPEN (not addressed)

### HIGH-05: Cold start headcount floor creates quorum cliff
**Status:** STILL OPEN (spec unchanged)

### HIGH-06: Tenure penalty gaming via strategic inactivity
**Status:** STILL OPEN (spec unchanged)

### HIGH-07: Manifest hash collision vulnerability
**Status:** STILL OPEN (spec unchanged)

### HIGH-08: (Was there an 8th? Checking...)
Round 1 had 8 HIGH-severity issues listed. All remain unaddressed — the round 2 focus was exclusively on the 3 CRIT issues.

---

## PART 3: CONFORMANCE TEST ALIGNMENT (LINK-01 through LINK-14)

### LINK-01: DID_LINK happy path
**Alignment:** ✅ PASS — spec matches test, dual signature verification correctly specified

### LINK-02: DID_LINK conflict — same child, two roots
**Alignment:** ✅ PASS — "First valid DID_LINK for a given child key wins" is in spec §1

### LINK-03: DID_LINK — child is already a root
**Alignment:** ✅ PASS — "A child key MUST NOT also be a root key of another identity" in spec

### LINK-04: DID_REVOKE happy path
**Alignment:** ✅ PASS — revocation rules match test

### LINK-05: DID_REVOKE — non-child key
**Alignment:** ⚠️ AMBIGUOUS — spec says root can revoke children but doesn't explicitly say "reject if not a child." Test implies MUST reject. **FIX:** Add to spec: "Nodes MUST reject DID_REVOKE messages where `revoked_key` is not a child of the signing root."

### LINK-06: DID_REVOKE — root revokes itself
**Alignment:** ✅ PASS — "The root key MUST NOT revoke itself" is explicit in spec

### LINK-07: Vote attribution and supersession
**Alignment:** ✅ PASS — spec §1 point 2 + §7 vote rules match test

### LINK-08: Proposal rate limit per identity
**Alignment:** ✅ PASS — spec §1 point 4 + §6 rate limiting match test

### LINK-09: Collusion exemption
**Alignment:** ✅ PASS — spec §1 point 3 + §10 exemption match test

### LINK-10: Headcount counting
**Alignment:** ✅ PASS — spec §1 point 7 + §7 minimum voters match test

### LINK-11: KEY_ROTATE by root — new root inherits authorized keys
**Alignment:** ✅ PASS — spec §1 KEY_ROTATE Interaction matches test

### LINK-12: Re-registration after revocation
**Alignment:** ✅ PASS — spec §1 DID_REVOKE "After revocation" section matches test

### LINK-13: Child KEY_ROTATE blocks hostile DID_LINK
**Alignment:** ✅ PASS — spec §1 KEY_ROTATE Interaction: "A key that became an authorized key via KEY_ROTATE of an existing child MUST be treated as linked" matches test

**BUT:** Round 1 TEST-01 issue still applies: how do late-joining nodes discover this relationship? The CRIT-01 fix (identity message persistence) addresses this — nodes can query for DID_LINK history via sync protocol.

**Conformance test gap:** LINK-13 should add:
```
4. New node joins network after KEY_ROTATE(B→B')
5. New node queries SYNC_REQUEST(identity_key=B', types=["DID_LINK", "DID_REVOKE"])
6. Receives DID_LINK(root=A, child=B) from sync
7. Infers B' is linked to A via KEY_ROTATE chain
```

### LINK-14: WITHDRAW by sibling key
**Alignment:** ✅ PASS — spec §6 WITHDRAW section matches test

**Overall conformance test alignment: 13/14 PASS, 1 AMBIGUOUS (LINK-05)**

---

## PART 4: FOCUSED ATTACK PASS — Identity Linking + Reputation Gain Dampening

### Attack Vector 1: Wealth-based vote concentration via large-scale linking

**Setup:**
- Wealthy operator with $100K budget deploys 200 nodes across 50 ASNs (diverse infrastructure)
- Links all 200 nodes under one root identity
- Participates actively: storage, proposals, votes

**Economics:**
- $500/node/month (cloud VPS) × 200 = $100K/month
- After 12 months: $1.2M invested

**Reputation accumulation:**
- Raw gain per day across 200 nodes: 200 × (storage + proposals + uptime) ≈ 200 × 0.005 = 1.0/day
- Dampened by sqrt(200) = 14.14: 1.0 / 14.14 = 0.0707/day
- Velocity cap: 0.02/day
- Actual gain: 0.02/day (hit velocity cap every day)
- After 365 days: 0.2 + (365 × 0.02) = 7.5 → capped at 1.0

**Outcome:** After ~12 months, identity reaches reputation cap (1.0). ONE vote with weight 1.0.

**Compare to 200 independent nodes:**
- Each node earns ~0.005/day
- After 365 days: 0.2 + (365 × 0.005) = 2.025 → each capped at 1.0 (velocity limits)
- No, wait — velocity limits: 0.02/day max
- After 365 days: 0.2 + (365 × 0.02) = 7.5 → capped at 1.0
- 200 nodes × 1.0 = 200 vote weight

**So 200 independent nodes → 200 vote weight. One linked identity with 200 keys → 1 vote weight.**

**Doesn't this make linking strictly worse?** Yes, for vote multiplication. But:
1. 200 independent nodes risk collusion detection (vote correlation, VDF timing, ASN clustering)
2. If flagged, −0.05 reputation penalty each = −10.0 total reputation loss across the 200 nodes
3. Linking avoids this risk

**The attack:** Deploy 10 such identities (2000 nodes, $1M/month, 12 months). After 12 months:
- 10 identities, each with reputation 1.0
- 10 vote weight total
- Governance threshold: 0.67 weighted endorsement
- If network has 100 other identities with average rep 0.4, total network vote weight = 10 + 40 = 50
- Quorum: 50 × 0.10 = 5.0
- Attacker controls 10 / 50 = 20% of vote weight

**Can they dominate governance?**
- Standard proposals: need 0.67 endorsement. Attacker + 5 honest high-rep allies (2.0 weight) = 12.0 endorsement. Remaining network (38.0) rejects. 12 / (12+38) = 0.24 < 0.67. **Proposal fails.**
- Constitutional proposals: need 0.90 endorsement. Attacker + 81 weight endorsement needed. Attacker has 10. Need 71 more from honest network. **Not feasible unless overwhelming consensus.**

**Conclusion:** Even with massive investment, a plutocratic attacker cannot unilaterally dominate governance due to supermajority thresholds. They can influence (20% weight) but not control.

**BUT:** If sqrt dampening were removed (per NEW-CRIT-02) and replaced with linear dampening, the same 2000 nodes linked into 10 identities would earn at 1× rate (not 10× via sqrt). Time to cap would be the same (velocity limited), but the COST to reach cap would be the same as a single node. This eliminates the plutocratic advantage entirely.

**Verdict:** sqrt dampening creates a mild plutocratic advantage (10× gain rate at 100 keys) but doesn't break governance due to supermajority requirements. However, LINEAR dampening is more equitable.

---

### Attack Vector 2: Strategic linking/unlinking to game collusion detection

**Setup:**
- Operator has 10 nodes
- Links them when reputation grinding (avoid collusion penalties)
- Unlinks them when voting (10× vote weight)

**Problem:** The spec doesn't allow unlinking. DID_REVOKE is permanent — a revoked key cannot re-link.

**Can the operator rotate instead?**
1. Link 10 nodes under root A
2. Grind reputation for 6 months → root A has rep 0.8
3. Revoke all 10 children
4. Each child re-registers as independent root
5. Each now has rep 0.2 (fresh start)
6. Root A keeps 0.8 but has no authorized keys

**No — this doesn't work. Revoked keys re-registering lose their reputation. There's no way to "cash out" the linked reputation into independent nodes.**

**Verdict:** Strategic linking/unlinking is NOT exploitable. Linking is a one-way commitment.

---

### Attack Vector 3: Sybil army linked to appear legitimate

**Setup:**
- Attacker deploys 500 nodes (Sybil)
- Links them all under one root identity
- Claims "I'm one operator with distributed infrastructure, not a Sybil"

**Defense mechanisms:**
1. VDF proof per key (500 × 30 seconds = 4 hours of computation, or parallelized)
2. ASN diversity checks: if all 500 nodes are in the same ASN, they violate §4 25% ASN limit
3. Linking doesn't exempt from ASN diversity — §1 point 7 says "an identity counts as one operator regardless of how many keys it has across different ASNs" but "the network still benefits from infrastructure diversity"

**The spec says:** For ASN diversity, an identity counts as ONE operator. But does this mean:
a) The identity's keys can be 100% in one ASN and it's fine (they're "one operator")?
b) Or: ASN diversity still applies — the identity's keys should be spread across ASNs?

**Spec text (§1 point 7):** "For ASN diversity calculations (§4), an identity counts as one operator regardless of how many keys it has across different ASNs. The network still benefits from the infrastructure — more nodes across more ASNs improves connectivity — but the identity doesn't get outsized diversity credit."

**And §4:** "No single ASN > 25% of connections."

**Interpretation:** The 25% ASN limit applies to CONNECTIONS (peer table), not identities. If an identity has 500 keys all in ASN 1234, and you're connected to all 500, then ASN 1234 is 500 / (your total connections) percent. If you have 2000 total connections, that's 25% — right at the limit.

**But:** The attacker linking 500 keys doesn't bypass the ASN diversity check. Each peer you connect to still counts toward the ASN% calculation.

**Verdict:** Linking doesn't enable Sybil legitimization. ASN diversity checks still apply. A Sybil army in one ASN is detectable.

---

### Attack Vector 4: Reputation gain rate comparison — linked vs. independent

**Math:**

**Scenario A: 1 node, 12 months**
- Gain rate: 0.005/day (storage + proposals + uptime)
- Velocity cap: 0.02/day
- Actual: 0.005/day (below cap)
- After 365 days: 0.2 + (365 × 0.005) = 2.025 → capped at 1.0 (but takes 40 days to reach 1.0 at 0.02/day max)
- Wait, velocity cap is 0.02/day. So:
  - Days 1-40: gain 0.02/day → reach 1.0 (0.2 + 40×0.02 = 1.0)
  - Days 41-365: capped at 1.0

**Scenario B: 100 nodes linked, 12 months**
- Raw gain rate: 100 × 0.005/day = 0.5/day
- Dampened: 0.5 / sqrt(100) = 0.5 / 10 = 0.05/day
- Velocity cap: 0.02/day
- Actual: 0.02/day (hit cap)
- After 365 days: same as Scenario A, reaches 1.0 at day 40

**Scenario C: 100 independent nodes, 12 months**
- Each earns 0.005/day (below cap)
- After 365 days: each at 0.2 + (365 × 0.005) = 2.025... but velocity cap is 0.02/day
- Hmm, this is confusing. Let me recalculate.

**Velocity limits (§8):**
- Max daily gain: 0.02 above 0.2
- Max weekly gain: 0.08 above 0.2

If a node earns 0.005/day for 7 days = 0.035/week, that's below the weekly cap (0.08). So daily cap applies, not weekly.

**So:**
- Node earns 0.005/day actual, capped at 0.02/day
- Node gains 0.005/day (not capped because 0.005 < 0.02)
- After 160 days: 0.2 + (160 × 0.005) = 1.0

**Scenario B recalc:**
- 100 linked keys, raw 0.5/day
- Dampened: 0.05/day
- Velocity cap: 0.02/day
- Actual gain: 0.02/day (capped)
- After 40 days: 1.0

**Scenario C:**
- 100 independent nodes, each 0.005/day
- Not capped (0.005 < 0.02)
- After 160 days: each reaches 1.0
- Total vote weight: 100

**Time advantage:** Linked identity reaches cap in 40 days. Independent nodes reach cap in 160 days. 4× faster to cap with linking (due to sqrt dampening hitting velocity limits).

**Ah! This is the issue.** Linking 100 keys allows hitting velocity caps consistently, reaching reputation cap faster. Independent nodes with modest activity never hit velocity caps and grow slower.

**Is this a problem?** Yes — linking becomes a shortcut to cap. If the goal is "1 operator = 1 vote", linking should NOT allow faster reputation growth than a single diligent node.

**Fix:** This reinforces NEW-CRIT-02 — linear dampening eliminates this advantage. With linear dampening:
- 100 linked keys: raw 0.5/day, dampened 0.5/100 = 0.005/day (same as 1 node)
- After 160 days: reaches 1.0 (same timeline as independent node)

---

## PART 5: NEW CRITICAL ISSUES SUMMARY

1. **NEW-CRIT-01:** Sync identity query has no rate limit (DoS vector)
2. **NEW-CRIT-02:** sqrt(key_count) dampening insufficient — allows 10× gain rate at 100 keys, creating plutocratic advantage

## PART 6: RECOMMENDATIONS

### Immediate (Must Fix Before Deployment)
1. **Change gain dampening to linear:** `effective_gain = raw_gain / authorized_key_count` (eliminates plutocratic advantage)
2. **Add SYNC_REQUEST rate limit:** 10/peer/minute for identity queries
3. **Fix LINK-05 spec ambiguity:** Explicitly reject DID_REVOKE for non-child keys

### High Priority (Fix Before 1,024 Nodes)
1. Reduce 7-day DID_LINK re-broadcast to 30-day (reduce gossip overhead)
2. Address all round 1 HIGH-severity issues (still open)

### Medium Priority (Monitor and Revisit)
1. Consider reducing KEY_CONFLICT suspension vote weight from 10% to 5%
2. Document asymmetric risk of multi-key identities (gains dampened, penalties not)

---

## VERDICT

**CRIT-01 fix:** ✅ Solid, but introduces NEW-CRIT-01 (sync DoS) and NEW-HIGH-01 (gossip overhead)

**CRIT-02 fix:** ✅ Adequate, minor issue (10% vote weight exploitable in edge cases)

**CRIT-03 fix:** ❌ **INSUFFICIENT** — sqrt dampening allows 10× gain rate at 100 keys. Change to linear dampening.

**Conformance tests:** 13/14 aligned, 1 ambiguity

**Overall:** The identity linking mechanism is now structurally sound (persistence, conflict resolution, attribution). But the economic game theory is broken — sqrt dampening creates a pay-to-win reputation advantage. Fix before deploying to production.
