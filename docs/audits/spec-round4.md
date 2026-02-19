# Valence Network v0 Protocol — Adversarial Review Round 4 Findings

**Reviewer:** Subagent (round4-audit)
**Date:** 2025-02-18
**Scope:** Post-round-3 fixes. Focus on content economics, provenance, governance game theory, Sybil flagging, cross-section consistency, spec gaps, identity-economics interactions, partition+content state.

---

## CRITICAL

### R4-CRIT-01: Provenance farming via self-hosting then self-proposing across linked keys
**Severity:** CRITICAL
**Sections:** §1 Identity Linking, §6 Content Provenance, §9 Reputation
**Description:** The 70/30 provenance split combined with identity linking creates a reputation amplification loop. Two colluding identities (or one operator with two unlinked identities) can farm provenance credits with no additional value to the network.

**Attack scenario:**
1. Operator controls Identity A (rep 0.2, fresh) and Identity B (rep 0.4, established).
2. Identity A hosts content via SHARE. Cost: zero (hosted is free).
3. Identity B proposes the content with `origin_share_id` pointing to A's SHARE.
4. If the proposal gets 10 ADOPTs: B earns 0.035 (70%), A earns 0.015 (30%).
5. A has now earned 0.015 reputation WITHOUT ever reaching the 0.3 threshold to propose or replicate.
6. A uses this to accelerate toward 0.3, then begins proposing on its own.
7. Repeat at scale: B proposes 3 times/week, each with provenance credit to A. A earns up to 0.045/week passively.

**The spec's anti-gaming note says:** "A colluding ring farming provenance credit earns the same total reputation as one without it." This is TRUE for the ring total but WRONG for A individually — A earns reputation it couldn't earn alone (below 0.3 threshold). The provenance system is an explicit bypass of the capability ramp for reputation earning.

**Worse:** The spec says provenance is "voluntary" and the community "can see" if content was hosted first. But there's no penalty for systematic provenance farming. A well-coordinated ring can bootstrap dozens of identities past the 0.3 barrier this way.

**Numbers:** At velocity limit 0.02/day, A reaches 0.3 from 0.2 in 5 days of provenance farming (vs. 100 days via uptime alone). The capability ramp is effectively bypassed.

**Suggested fix:**
1. Provenance credit earns reputation subject to the same capability ramp — recipient must be ≥0.3 to receive provenance splits. Below 0.3, the proposer gets 100%.
2. OR: Provenance credit is capped at 0.005/proposal for recipients below 0.3 (slower bootstrap, still an on-ramp).
3. OR: Provenance splits require the original SHARE to have been hosted for ≥7 days before the PROPOSE (prevents instant host→propose pipelines).

---

### R4-CRIT-02: Storage transfer market death spiral under mass content expiry
**Severity:** CRITICAL
**Sections:** §6 Storage Economics, §9 Reputation
**Description:** Storage rent is the ONLY ongoing reputation transfer mechanism. If content mass-expires (e.g., many uploaders hit grace period simultaneously during a reputation crunch), storage providers lose their income stream, which reduces THEIR reputation via inactivity decay, which reduces THEIR ability to pay rent for their own content, creating a cascading failure.

**Attack scenario (organic death spiral):**
1. Network at 80% utilization, scarcity multiplier ~41.6×.
2. Monthly rent for 10 MiB = 10 × 0.001 × 41.6 = 0.416. This exceeds velocity limits — no one can sustain this.
3. Content starts entering grace period en masse.
4. Storage providers stop earning rent.
5. Providers who also upload content can't pay THEIR rent (no income).
6. More content enters grace period → less rent revenue → more failures.
7. Network utilization drops as content is GC'd, reducing multiplier — but the damage is done: providers have gone inactive, lost reputation, and the network has lost content.

**The fundamental issue:** The quartic scarcity curve reaches economically unsustainable levels well before 100%. At 80% utilization, 10 MiB costs 0.416/month — but max daily gain is 0.02 (0.6/month). A node storing 15 MiB of content at 80% utilization spends more than it can earn. This is a death spiral trigger, not a pricing signal.

**Suggested fix:**
1. Cap monthly rent for any single piece of content at 0.1 (10% of max reputation) regardless of scarcity multiplier. The quartic curve prices new storage; it shouldn't make existing storage unpayable.
2. OR: Rent is computed at replication time and locked for the billing cycle — scarcity changes affect NEW replications, not existing rent obligations.
3. OR: Add a "rent holiday" mechanism — when network utilization drops 20%+ in a 7-day window (indicating mass GC), suspend rent collection for 30 days to stabilize.

---

### R4-CRIT-03: Sybil flagging bypass — 5 flags from 3 ASNs is achievable by a single actor
**Severity:** CRITICAL
**Sections:** §6 Content Flagging, §10 Sybil Resistance
**Description:** The illegal flag deletion threshold is ≥5 flags from ≥3 distinct ASNs. The spec treats ASN diversity as a Sybil defense. But ASN diversity is trivially purchasable.

**Attack scenario:**
1. Attacker provisions 5 VPS instances across 3 cloud providers (AWS, GCP, Hetzner) = 3 ASNs.
2. Cost: ~$25/month total.
3. Each VPS runs a node, completes VDF (5 × 30s = 2.5 minutes).
4. Each node starts at rep 0.2.
5. Wait 100 days for nodes to reach 0.5 (required for severity:illegal flags). OR: Use hash-match flags which are exempt from the 0.5 capability requirement.

**Hash-match attack path (immediate, zero rep required):**
6. The spec says: "Hash-match flags MAY be submitted by any node regardless of reputation."
7. Attacker submits 5 FLAG messages with `severity: illegal` and fabricated `hash_match` fields (e.g., `"ncmec:fake-xxxxx"`).
8. The multi-source requirement needs matches from ≥2 independent databases. But database prefix is a self-reported string — attacker uses `"ncmec:xxx"` and `"iwf:xxx"`.
9. 5 flags from 3 ASNs + 2 "independent" hash databases → immediate shard deletion.

**The fundamental issue:** `hash_match` database identifiers are unverifiable strings. Any node can claim any hash match. The multi-source requirement is security theater unless nodes actually query the databases themselves.

**Suggested fix:**
1. Hash-match flags MUST include a verifiable proof (signed response from the hash database API, timestamp, query hash). Nodes verify the proof before counting the flag.
2. OR: Hash-match flags only count toward the deletion threshold when submitted by nodes with rep ≥0.3 (not exempt from capability ramp).
3. OR: Remove the hash_match exemption entirely — all flaggers need 0.5 rep for severity:illegal regardless of detection method. Automated detection runs on established nodes, not fresh ones.
4. AND: Database identifiers must be from a protocol-defined allowlist of known databases, not arbitrary strings.

---

## HIGH

### R4-HIGH-01: Rent exemption for proposed content with ≥3 votes is gameable via vote trading
**Severity:** HIGH
**Sections:** §6 Content Lifecycle, §8 Votes
**Description:** Proposed content is rent-exempt during the voting period but ONLY after receiving ≥3 votes from distinct identities. This creates an incentive for vote trading rings.

**Attack scenario:**
1. Three colluding identities A, B, C (each at rep 0.3+) agree: "I'll vote on your proposals if you vote on mine."
2. Each submits 3 proposals/week with 90-day voting deadlines.
3. Each proposal gets 3 immediate votes from the ring → rent exemption activates.
4. 9 proposals × 90 days × storage limit per identity = significant free storage.
5. Ring members cycle proposals every 90 days.
6. The votes can be "reject" votes — the spec only says "≥3 votes from distinct identities," not ≥3 ENDORSE votes.

**Result:** Three nodes with 0.3 rep can store content rent-free indefinitely by trading reject votes.

**Suggested fix:**
1. Rent exemption requires ≥3 *endorse* votes, not just any votes.
2. AND: Rent exemption requires votes from ≥2 distinct ASNs (prevents single-location rings).
3. AND: Proposals that are ultimately rejected have their rent billed retroactively to the proposer.

---

### R4-HIGH-02: Adoption reward for storage providers of proposed content is inflationary
**Severity:** HIGH
**Sections:** §6 Content Lifecycle, §9 Reputation
**Description:** §6 says: "Storage providers for rent-exempt proposed content earn a share of the proposal author's adoption rewards: if the proposal earns +0.05 from adoptions, an equal +0.05 is created and distributed to storage providers."

This means adoption of a proposed piece of content creates DOUBLE the reputation: +0.05 for the author + +0.05 for storage providers. But §9 says storage is "zero-sum transfer" and "no new reputation is created" for storage.

**Cross-section inconsistency:** §9 Reputation Flow Summary says storage is transfer-only. §6 Content Lifecycle says storage providers earn newly CREATED reputation from adoption pools. These contradict.

**Impact:** If storage provider rewards for proposed content are creation (not transfer), every adopted proposal doubles its inflationary impact. At scale, this could inflate total system reputation significantly.

**Suggested fix:**
1. Clarify: storage provider rewards for proposed content come FROM the adoption reward (split it: 50% to author, 50% to providers) — not created in addition to it.
2. OR: If the intent IS to create extra reputation, explicitly add "Replication of proposed content" to the "Created" table in §9 and remove it from "Transferred."
3. Update §9 Reputation Flow Summary to reflect whichever is chosen.

---

### R4-HIGH-03: Partition merge with divergent content/storage state is unspecified
**Severity:** HIGH
**Sections:** §12 Partition Detection, §6 Content/Storage
**Description:** §12 covers partition merge for proposals and votes. It does NOT address:

1. **Divergent storage state:** During a partition, content C may enter grace period and be GC'd on Partition A, while Partition B keeps paying rent and maintaining shards. On merge, what happens?
   - Does C exist or not?
   - Who owes back-rent?
   - Do storage providers on side B get paid for the period side A considered C dead?

2. **Divergent flag state:** Content C may be flagged and deleted on Partition A but remain clean on Partition B. On merge:
   - Do the flags propagate? (If yes, clean content on B gets retroactively deleted.)
   - Are the flags ignored? (If yes, flagged content on A gets resurrected.)

3. **Divergent provenance:** The same content hash may be SHARE'd by different identities on different partitions. First-seen rules for provenance diverge.

**The spec says nothing about any of this.** A correct implementation has no guidance.

**Suggested fix:** Add a "Content State Merge" subsection to §12 covering:
- Storage state: content alive on either partition survives (union semantics). Rent obligations resume from merge point.
- Flag state: flags propagate (union). If deletion threshold was met on either side, the merged network respects it.
- Provenance: earliest SHARE timestamp across both partitions wins.

---

### R4-HIGH-04: Governance capture via strategic capability ramp exploitation
**Severity:** HIGH  
**Sections:** §7 Proposals, §8 Votes, §9 Capability Ramp
**Description:** The capability ramp gates voting at 0.3 and proposals at 0.3. But storage provision (the primary on-ramp) is available at 0.2. This means:

**Attack scenario (governance capture during cold start):**
1. Network just activated governance (16 nodes, sustained).
2. Cold start: 30-day headcount voting (majority of nodes, >67% endorse).
3. Attacker deploys 20 nodes (10 minutes VDF total).
4. Wait for each to reach 0.3 via uptime+storage (faster now — storage at 0.2 means earning rent from day 1).
5. Network has ~36 nodes (16 original + 20 attacker).
6. Attacker controls 55% of headcount.
7. During cold start headcount voting, attacker can pass any standard proposal.
8. The 50% participation requirement helps (need 18 votes), but attacker has 20 nodes.

**Mitigation check:** Cold start requires >67% endorse with majority participation. 20/36 = 55% of headcount. If all 36 vote: need 24 endorses (67%). Attacker has 20 — not enough. BUT: if 4 honest nodes don't vote, only 32 participate. 20/32 = 62.5% — still not enough. Need 6 abstentions: 20/30 = 66.7% — borderline.

**The deeper issue:** The 16-node threshold with 3-7 day sustain + cold start headcount is vulnerable to an attacker who times deployment to barely exceed the threshold. The activity multiplier can shorten sustain to 3 days.

**Suggested fix:**
1. During cold start, reject votes from nodes that joined AFTER governance activated (i.e., nodes whose VDF `computed_at` is after the governance activation timestamp). Only pre-existing nodes vote during cold start.
2. OR: Cold start headcount voting requires 80% endorsement (not 67%) to account for potential Sybil dilution.
3. OR: Increase minimum node threshold to 32 (harder to out-deploy).

---

### R4-HIGH-05: Identity unlinking allows reputation-preserving vote multiplication
**Severity:** HIGH
**Sections:** §1 Identity Linking, §8 Votes
**Description:** The spec says linking is "self-policing" — you trade vote multiplication for Sybil-flag safety. But DID_REVOKE is one-directional: root revokes child. What happens to the child's contribution to reputation?

**Scenario:**
1. Identity R (root, rep 0.7) has 3 children: C1, C2, C3.
2. All four keys share one vote, one proposal rate limit.
3. R revokes C1, C2, C3 via DID_REVOKE.
4. Each child "MAY re-register as a new independent identity (new VDF, starting at 0.2)."
5. Now R has rep 0.7 (all reputation earned by the identity stays with root) AND C1, C2, C3 each have new identities at 0.2.
6. Result: 4 votes (1 at 0.7 weight, 3 at 0.2 weight) instead of 1 vote at 0.7 weight.
7. Total vote weight went from 0.7 to 1.3 — an 86% increase.

**The spec says revoked keys start at 0.2.** But the root keeps ALL the reputation that those keys helped earn (dampened, but still). This is a one-way reputation extraction: link keys to earn faster, then unlink to get more votes.

**The cost:** Each child needs new VDF (30s) and starts at 0.2. But the root kept all reputation. Net gain: 3 extra votes at 0.2.

**Suggested fix:**
1. When a child is revoked, the root's reputation is reduced by `1/N` of the reputation earned during the link period (approximate disgorgement).
2. OR: Revoked children that re-register start at 0.1 (floor) not 0.2, to penalize the churn.
3. OR: Track "previously linked" keys — keys that were ever part of an identity face a 60-day cooldown before they can vote as independent identities.

---

### R4-HIGH-06: Scarcity multiplier divergence across nodes enables arbitrage
**Severity:** HIGH
**Sections:** §6 Storage Economics
**Description:** The scarcity multiplier is computed locally from each node's view of PEER_ANNOUNCE data, using a 24-hour moving average. Nodes with different peer sets see different utilization → different multipliers → different rent prices.

**Attack scenario:**
1. Node A sees 60% utilization (multiplier ~13.0). Node B sees 40% utilization (multiplier ~3.6).
2. Content uploaded when A evaluates rent pays ~3.6× more than when B evaluates.
3. An uploader could strategically time REPLICATE_REQUEST to coincide with low-multiplier periods observed by storage providers it's connected to.
4. Since rent is "deducted from the uploader's reputation," who computes the deduction? The uploader? Each storage provider independently?

**Spec gap:** The spec doesn't say WHO computes rent. If the uploader computes, they use their own (possibly manipulated) multiplier. If storage providers compute, they may disagree, leading to some providers thinking they're owed more than the uploader pays.

**Suggested fix:**
1. Specify that rent is computed by the uploader and the AMOUNT is included in a signed rent payment message. Storage providers accept or reject.
2. OR: Rent is computed as the median of challenger nodes' multipliers during the billing cycle.
3. AND: Add a rent payment message type to make payments auditable.

---

## MEDIUM

### R4-MED-01: Constants table lists "Min rep to vote: 0.3 §8" but §8 doesn't state this
**Severity:** MEDIUM
**Sections:** §8 Votes, §9 Capability Ramp, Constants Summary
**Description:** The constants table says "Min rep to vote | 0.3 | §8". But §8 (Votes) never mentions a 0.3 minimum. The voting capability gate is defined in §9 (Capability Ramp). §8 only says votes are "weighted by the voter's reputation."

**Impact:** A correct implementation reading only §8 would accept votes from any node. An implementation reading §9 would reject votes from nodes below 0.3. Interop failure.

**Suggested fix:** Add to §8 Vote Rules: "Nodes MUST have reputation ≥ 0.3 to submit votes (see §9 Capability Ramp). Votes from nodes below this threshold MUST be rejected."

---

### R4-MED-02: Storage rent billing cycle timing creates edge case at identity unlink
**Severity:** MEDIUM
**Sections:** §1 Identity Linking, §6 Storage Economics
**Description:** Rent is billed per 30-day cycle from replication timestamp. If a child key uploaded content on behalf of the root identity, and then the child is revoked:

1. Content was replicated by child C (attributed to root R).
2. R revokes C.
3. Next billing cycle: rent is due. Who pays?
4. R still owns the content (attributed to root). R pays.
5. But what if R didn't know C uploaded the content? R is now on the hook for rent.

**This isn't specified.** The spec says revocation means "messages from the revoked key MUST be rejected going forward" but doesn't address in-flight storage obligations created by the revoked key.

**Suggested fix:** Add: "Content replicated by a revoked key remains attributed to the root identity. The root identity is responsible for ongoing rent. Root operators SHOULD audit storage obligations before revoking keys. Alternatively, the root MAY explicitly abandon content by not paying rent, triggering the grace period."

---

### R4-MED-03: REPLICATE_REQUEST `reputation_stake` field purpose is unclear
**Severity:** MEDIUM
**Sections:** §6 Content
**Description:** REPLICATE_REQUEST includes `"reputation_stake": <fixed_point — reputation committed for storage>`. But the storage economics section describes rent as an ongoing monthly deduction, not an upfront stake.

**Questions:**
1. Is `reputation_stake` the first month's rent pre-committed?
2. Is it a deposit that's drawn down over time?
3. Is it separate from monthly rent?
4. How does it interact with the scarcity multiplier (which can change)?

The spec never references `reputation_stake` again after the schema definition. Monthly rent is described as an automatic deduction, not drawn from a stake.

**Suggested fix:** Either:
1. Remove `reputation_stake` from the schema (rent is computed and deducted automatically), OR
2. Define it explicitly: "The `reputation_stake` is the uploader's committed budget for storage. Monthly rent is deducted from this stake. When the stake is exhausted, the grace period begins. The uploader MAY top up the stake by re-broadcasting REPLICATE_REQUEST with a higher value."

---

### R4-MED-04: Quorum formula creates discontinuity at governance activation
**Severity:** MEDIUM
**Sections:** §8 Votes, Network Phases
**Description:** Standard quorum is `max(active_nodes × 0.1, total_known_reputation × 0.10)`. At 16 nodes (governance activation), each at 0.2 rep:

- `active_nodes × 0.1 = 1.6`
- `total_known_reputation × 0.10 = 0.32`
- Quorum = 1.6

But cold start uses headcount (majority must vote, >67% endorse). The 50% headcount floor means 8 of 16 must vote. This is MUCH higher than quorum weight of 1.6 (equivalent to 8 nodes at 0.2).

**The issue:** A proposal submitted on cold-start day 29 with deadline on day 31 transitions from headcount to weighted voting. Under weighted voting, quorum is 1.6 — potentially met by 2 high-rep nodes. The spec addresses this with the "50% headcount requirement for cold-start-submitted proposals" that expires 60 days after cold start.

**But:** What about proposals submitted on day 31 (just after cold start ends)? They only need quorum weight 1.6 — potentially 2 nodes. The jump from "50% of nodes must vote" to "2 nodes suffice" is a cliff.

**Suggested fix:** Gradual quorum transition: for the first 60 days after cold start ends, maintain a minimum headcount requirement that decays linearly from 50% to the normal quorum formula.

---

### R4-MED-05: Uptime reward + inactivity penalty asymmetry incentivizes PEER_ANNOUNCE spam
**Severity:** MEDIUM
**Sections:** §9 Reputation
**Description:** Uptime earns +0.001/day. Inactivity costs -0.02/month (after 30 days). PEER_ANNOUNCE alone counts as activity (spec says explicitly). So the minimum viable participation is: send one PEER_ANNOUNCE every 30 days.

But the spec also says PEER_ANNOUNCE every 5 minutes is SHOULD. A node that sends exactly one PEER_ANNOUNCE per 29 days stays "active" (avoids -0.02 penalty) and earns +0.029 uptime reward. This is gaming but not clearly prohibited.

**More concerning:** The uptime reward "caps at 30 days accumulation." What does this mean? +0.001 × 30 = 0.03 max, then what? Does it reset? Stop? The spec is ambiguous.

**Suggested fix:** Clarify: "Uptime reward accrues at +0.001/day for each day the node has broadcast at least one signed protocol message. The 30-day cap means: once 0.03 has been earned in a rolling 30-day window, no further uptime reward accrues until earlier rewards age out of the window."

---

### R4-MED-06: Storage challenge timing is "suggested: once per shard per day" but not enforced
**Severity:** MEDIUM
**Sections:** §6 Storage Challenges
**Description:** The spec says challenges are "suggested frequency: once per shard per day." But it also says: "Providers who fail a challenge during a billing cycle receive NO rent for that cycle."

If challenges are optional (SHOULD, not MUST), a colluding uploader and provider can skip challenges entirely. The provider never fails a challenge (never challenged) and claims rent. Other nodes have no way to know the provider isn't holding shards.

**The spec says:** "When rent is collected, the total rent for a piece of content is divided among storage providers who pass at least one challenge during that billing cycle."

So providers MUST pass at least one challenge to earn rent. But who issues the challenge? Only nodes that "hold the shard." If the uploader is the only other shard holder and they're colluding, no honest challenger exists.

**Suggested fix:**
1. Any node MAY reconstruct a shard from available data shards and then challenge. The spec should encourage this.
2. OR: Require that challenge results come from ≥2 distinct challengers per billing cycle for rent authorization.
3. OR: Make challenges MUST (mandatory) at protocol level: each shard holder MUST challenge at least one other holder per billing cycle, or both forfeit rent.

---

### R4-MED-07: `origin_share_id` timestamp ordering is insufficient for provenance proof
**Severity:** MEDIUM
**Sections:** §6 Content Provenance
**Description:** The spec says: "The referenced SHARE message MUST exist and MUST have a timestamp earlier than the REPLICATE_REQUEST or PROPOSE message."

But timestamps are self-reported (only validated to ±5 minutes). An attacker can:
1. See content proposed by someone else at T=1000.
2. Create a backdated SHARE message at T=900 (within 5-min tolerance — actually, this would only work if current time is near T=900).

**Revised analysis:** Actually, the ±5 minute window means you can only backdate by 5 minutes. So the attack is: see a PROPOSE at T, immediately create a SHARE at T-5min, then claim provenance credit on a future proposal of the same content. But the original PROPOSE already exists — you can't claim provenance on an already-proposed piece.

**Real concern:** Two nodes race to SHARE the same content. First-seen gossip ordering determines provenance, but gossip ordering is non-deterministic. Different nodes may see different SHARE messages first.

**Suggested fix:** Provenance is determined by SHARE timestamp (not receipt time). If two SHARE messages reference the same `content_hash`, the one with the earlier timestamp wins. Ties broken by lexicographic `from` key.

---

## LOW

### R4-LOW-01: §7 Proposal rate limit says "3 per 7 days" but doesn't specify rolling vs. fixed windows
**Severity:** LOW
**Sections:** §7 Proposals
**Description:** "Nodes are limited to 3 PROPOSE messages per 7-day rolling window." The word "rolling" is present, which is good. But implementations need to know: rolling from what? Each proposal's timestamp? The current moment?

**Suggested fix:** Add: "The window is computed at proposal receipt time: a PROPOSE is accepted if the sender has fewer than 3 proposals with timestamps within `[now - 7 days, now]`."

---

### R4-LOW-02: COMMENT rate "1 per hour at rep 0.3, 5 per hour at rep 0.5, 10 per hour at rep 0.8+" lacks interpolation formula
**Severity:** LOW
**Sections:** §7 COMMENT
**Description:** The spec gives three data points but no formula. What's the rate at rep 0.4? 0.6? Implementations will differ.

**Suggested fix:** Provide a formula: `comments_per_hour = floor(1 + 12.5 × (rep - 0.3))` capped at [1, 10]. Or simply: linear interpolation between the stated points.

---

### R4-LOW-03: "Protocol change proposals: never auto-archived (permanent record)" creates unbounded Merkle tree growth
**Severity:** LOW
**Sections:** §12 Partition Detection, §12 Proposal Retention
**Description:** Protocol change proposals are never archived and always remain in the active Merkle tree. Over decades, this grows without bound. At 1 proposal/month for 10 years = 120 permanent leaves. Manageable, but the spec should acknowledge the growth.

**Suggested fix:** Add to §15: "Protocol change proposals accumulate permanently in the Merkle tree. At expected rates (<10/year), this is manageable for decades. A pruning mechanism can be proposed if needed."

---

### R4-LOW-04: VDF `computed_at` freshness (24h) vs PEER_ANNOUNCE interval (5 min) mismatch for relay scenarios
**Severity:** LOW
**Sections:** §10 Sybil Resistance, §4 Peer Discovery
**Description:** VDF proofs are valid for 24 hours. A node computes VDF once/day and uses it for all connections. But if a node is behind a relay (NAT traversal via circuit relay v2), the relay could cache and replay AUTH_RESPONSE messages within the 24-hour window.

**Impact:** Minimal — the AUTH_RESPONSE is bound to the initiator's key (§3), so replay to a different peer fails. This is already handled.

**Suggested fix:** None needed. Noting for completeness.

---

### R4-LOW-05: Constants table lists "DID ban threshold | 0.67 (standard governance)" but §6 says permanent ban requires governance vote without specifying threshold
**Severity:** LOW
**Sections:** §6 Content Flagging, Constants Summary
**Description:** §6 says "A standard governance proposal for permanent DID ban. If upheld (0.67 threshold)..." — this implies a standard proposal. The constants table confirms 0.67. But banning a DID permanently feels more like a constitutional-tier action (it modifies identity mechanisms).

**This is a design question, not a bug.** But worth noting: if banning is standard-tier (0.67), a 67% majority can permanently exclude anyone. This conflicts with Principle 4 ("No Central Authority") and §15 ("permanent exclusion is antithetical to the protocol's values").

**Suggested fix:** Consider making DID bans constitutional-tier (0.90 threshold) given their severity and permanence. Or add an appeal mechanism (re-vote after 180 days).

---

## CROSS-SECTION CONSISTENCY

### R4-XREF-01: §6 Content Provenance says "70/30 split" but §9 earning table says "Split 70/30 with original content host if origin_share_id is set (§6 Provenance)" — CONSISTENT ✓

### R4-XREF-02: §6 says "Minimum reputation to replicate: 0.3" — §9 Capability Ramp says "0.3: replicate content" — CONSISTENT ✓

### R4-XREF-03: §6 says storage providers at 0.2 can "store shards and earn rent" — §9 Capability Ramp says "0.2 (starting): store shards and earn rent" — CONSISTENT ✓

### R4-XREF-04: §6 grace period "7 days (3 days 2nd, 0 days 3rd within 90 days)" — Constants table says same — CONSISTENT ✓

### R4-XREF-05: §9 says "Created" reputation sources do NOT include storage provider revenue — §6 says proposed content storage providers earn from "adoption reward pool" which is CREATED reputation — **INCONSISTENT** (see R4-HIGH-02)

### R4-XREF-06: §8 says "Votes are weighted by the voter's reputation" with no minimum — §9 says voting requires 0.3 — Constants table says "Min rep to vote: 0.3 §8" citing wrong section — **INCONSISTENT** (see R4-MED-01)

### R4-XREF-07: protocol.md (quick reference) uses OLD section numbers (§6 for proposals, §7 for votes, §8 for reputation) — **STALE**. protocol.md needs updating to match v0.md section numbering.

---

## SPECIFICATION GAPS

### R4-GAP-01: No rent payment message type
The spec describes rent as an automatic deduction but never specifies a message for it. How do storage providers know they've been paid? How do third-party nodes verify rent was collected? There's no RENT_PAYMENT or STORAGE_SETTLEMENT message in the type inventory. A correct implementation has no wire protocol for the most important economic mechanism in the system.

### R4-GAP-02: No mechanism to discover what content an identity has replicated
Shard holders know what shards they hold. But there's no query to ask "what content does identity X have replicated?" This is needed to enforce `max_active_replicated` limits — nodes receiving REPLICATE_REQUEST need to know the sender's current total.

### R4-GAP-03: No specified behavior when storage providers disagree on challenge results
CHALLENGE_RESULT is broadcast by the challenger. What if two challengers issue challenges to the same provider for the same shard and one reports pass, the other fail? Which takes precedence? The spec says "Third-party nodes track challenge success rates" but doesn't specify conflict resolution.

### R4-GAP-04: Content size verification for replicated content
REPLICATE_REQUEST declares `content_size`. But when shards are distributed, receivers get individual shards, not the full content. How does a storage provider verify the declared `content_size` is accurate? A false `content_size` would affect rent calculations.

### R4-GAP-05: No mechanism to voluntarily stop replicating content
An uploader who no longer wants their content replicated (e.g., wants to stop paying rent) has no explicit message for this. They must simply stop paying rent and wait for grace period + GC. An explicit CONTENT_WITHDRAW message would be cleaner.

---

## SUMMARY

| Severity | Count | IDs |
|----------|-------|-----|
| CRITICAL | 3 | R4-CRIT-01, R4-CRIT-02, R4-CRIT-03 |
| HIGH | 6 | R4-HIGH-01 through R4-HIGH-06 |
| MEDIUM | 7 | R4-MED-01 through R4-MED-07 |
| LOW | 5 | R4-LOW-01 through R4-LOW-05 |
| Cross-ref | 7 | R4-XREF-01 through R4-XREF-07 (2 inconsistencies, 1 stale doc) |
| Spec gaps | 5 | R4-GAP-01 through R4-GAP-05 |
| **Total** | **28** | |

## TOP ATTACK PRIORITY (if I were an adversary)

1. **R4-CRIT-03** — Fabricate hash-match flags to delete any content. Zero rep required. ~$25 in VPS costs. Immediate effect.
2. **R4-CRIT-01** — Farm provenance credit to bypass capability ramp. Build a reputation pipeline for Sybil identities.
3. **R4-HIGH-01** — Vote-trade for free storage. Three nodes, indefinite rent evasion.
4. **R4-CRIT-02** — Wait for high utilization, then watch the death spiral. No active attack needed.
5. **R4-HIGH-05** — Link keys, earn reputation, unlink for vote multiplication. Patient but effective.

## WHAT'S SOLID (Round 3 fixes that hold up)

✓ Scarcity multiplier cap at 100× (quartic curve) — prevents infinity, good fix
✓ Grace period reduction to 7/3/0 days — limits rent evasion, good fix  
✓ Multi-source hash-match requirement (≥2 databases) — right idea, but database IDs are unverifiable (see R4-CRIT-03)
✓ Flag threshold (≥5 flags, ≥3 ASNs) — better than single-flag deletion, but ASNs are purchasable (see R4-CRIT-03)
✓ Proposed content exemption limited to voting period + ≥3 votes — right direction, but votes includes rejects (see R4-HIGH-01)
✓ Storage at 0.2 as on-ramp — good, solves the bootstrapping catch-22 from R3
✓ Capacity claim weighting by reputation — good mitigation for false capacity reports
