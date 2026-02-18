# Valence Network v0 Protocol — Adversarial Review Round 3 Findings

**Reviewer:** Subagent (spec-review-r3)
**Date:** 2025-02-18
**Focus:** NEW §6 Content, Storage Economics, Capability Ramp, Content Flagging, COMMENT type, cross-references

---

## CRITICAL ISSUES

### CRIT-R3-01: Storage scarcity multiplier becomes infinite at 100% utilization
**Severity:** CRITICAL
**Location:** §6 Storage Economics
**Description:** The scarcity multiplier formula is:
```
scarcity_multiplier = 1 / (1 - network_utilization)
```

At 100% utilization (all storage full), the denominator becomes zero → infinite multiplier → undefined rent cost.

**Attack scenario:**
1. Network approaches 100% storage utilization
2. Scarcity multiplier explodes: 99% → 100×, 99.9% → 1,000×, 99.99% → 10,000×
3. At some threshold, rent costs exceed maximum possible reputation (1.0)
4. ALL existing replicated content becomes unpayable
5. Everything enters grace period → 30 days later, mass shard deletion
6. Network storage layer collapses

**Real-world trigger:** A coordinated upload attack (flood network with junk content) or organic growth hitting capacity limits.

**Mathematical issue:** Division by zero is undefined. The formula needs an asymptotic bound.

**Suggested fix:**
```
scarcity_multiplier = min(100, 1 / max(0.01, 1 - network_utilization))
```
Cap at 100× (extreme but not infinite) and prevent division by zero with a floor at 1% available capacity.

**Or:** Use a sigmoid/tanh curve that approaches a finite maximum:
```
scarcity_multiplier = 1 + 99 × (network_utilization)^4
```
This gives: 50% → 7.2×, 90% → 67×, 99% → 97×, 100% → 100× (bounded).

---

### CRIT-R3-02: Capability ramp creates reputation lock-in for uploader-provider collusion
**Severity:** CRITICAL
**Location:** §6 Storage Economics + §9 Capability Ramp
**Description:** Storage providers earn reputation from rent transfers, but uploaders need 0.3 reputation to replicate content. This creates a circular dependency for new nodes:

**Attack scenario (bootstrapping catch-22):**
1. New node N joins at rep 0.2
2. N wants to earn reputation → best path is storage provider (earn rent transfers)
3. But there's no content to store because:
   - Uploaders need 0.3 rep to replicate
   - Fresh nodes can only store shards IF content already exists
4. N can only earn via uptime (+0.001/day) → 100 days to reach 0.3

**Collusion variant:**
1. Established nodes A and B collude
2. A uploads junk content, pays rent to B
3. B earns reputation from rent transfers (zero-sum from A to B)
4. A and B rotate: B uploads, A stores
5. Reputation circulates between A and B, new nodes excluded

**The issue:** Storage rent is a TRANSFER market, not a creation market. But access to the market requires 0.3 rep. New nodes have no way to enter the storage market until they've earned 0.1 reputation via other means.

**Suggested fix:**
1. Lower storage provider threshold: nodes at 0.2 (starting) CAN store shards and earn rent
2. OR: Add a reputation creation path for storage: first-time shard storage earns a one-time bonus (+0.01) to bootstrap storage providers
3. OR: Explicitly document that new nodes MUST earn initial reputation via adoption messages (propose skills/configs) before entering the storage market

**Current path for new nodes:** Earn via uptime (0.001/day × 100 = 0.1) OR submit proposals and get adoption (+0.005 per ADOPT, need 20 adoptions). Both are slow.

---

### CRIT-R3-03: Content flagging FALSE FLAG attack via hash collision targeting
**Severity:** CRITICAL
**Location:** §6 Content Flagging
**Description:** The spec says nodes SHOULD run replicated content against known-bad hash databases and generate FLAG messages with `hash_match` populated for automated detection.

**Attack scenario:**
1. Attacker finds a SHA-256 collision between benign content C and a known-bad hash H (or finds a hash in the known-bad database that collides with C by chance or via birthday attack)
2. Attacker uploads benign content C to the network (replicates it)
3. Automated hash matching flags C as `severity: illegal`, `hash_match: "ncmec:xxxxx"`
4. Nodes immediately drop shards (spec says "SHOULD immediately drop shards... Do not wait for consensus")
5. Content source DID is suspended pending governance vote
6. Even if governance ultimately rejects the flag, the content is already destroyed (shards dropped)

**Real risk assessment:**
- SHA-256 collision: computationally infeasible with current tech (2^128 work)
- Known-bad database poisoning: attacker compromises the hash database and inserts benign content hashes
- Birthday attack: with 2^64 hashes in the database, probability of collision is non-negligible

**The spec says:** "Automated hash matches generate a FLAG... No human review is required or desired."

**Problem:** Automated flagging with immediate shard deletion (no human review) + hash databases that could be poisoned = potential for weaponized censorship.

**Suggested fix:**
1. Hash match flags SHOULD trigger immediate suspension but NOT immediate shard deletion
2. Require human confirmation before shard deletion for hash-match flags
3. OR: Use perceptual hashing (pHash/PhotoDNA) instead of cryptographic hashing for CSAM detection — these are resistant to benign collisions
4. OR: Multi-source hash verification: require hash match from 2+ independent databases before triggering automated response
5. Document the trust model for hash databases explicitly

---

### CRIT-R3-04: FLAG stake insufficient to prevent grief-flagging at scale
**Severity:** CRITICAL
**Location:** §6 Content Flagging
**Description:** Flagging costs:
- `severity: dispute` → -0.005 stake
- `severity: illegal` → -0.02 stake

Stake is returned if flag is upheld, forfeited if rejected.

**Attack scenario (Sybil flag spam):**
1. Attacker creates 100 Sybil identities (100 × 30 seconds VDF = 50 minutes)
2. Each Sybil starts at rep 0.2
3. Each Sybil flags target content with `severity: illegal` → -0.02 stake
4. Target content source is suspended (reputation frozen, 10% vote weight)
5. 100 flags from 100 Sybils trigger immediate shard deletion (nodes see mass flags, drop shards)
6. Governance vote takes 14+ days (proposal voting deadline)
7. Even if all flags are eventually rejected:
   - Each Sybil loses -0.02 stake → rep drops to 0.18 (still above floor)
   - Target was suspended for 14+ days
   - Content was deleted (shards dropped)
8. Attack cost: 50 minutes of VDF computation
9. Damage: 14-day suspension + content destruction

**The spec says:** "Nodes SHOULD immediately drop shards for the flagged content" after `severity: illegal` flag. There's no threshold for "how many flags trigger immediate action?"

**Problem:** A single severe flag triggers immediate shard deletion. Mass Sybil flagging is cheap and causes disproportionate damage.

**Suggested fix:**
1. Immediate shard deletion requires ≥ 5 flags from ≥ 3 distinct ASNs (prevents Sybil control)
2. OR: Only hash-match flags trigger immediate deletion; manual flags enter suspension but keep shards for 7 days pending governance
3. OR: Increase severe flag stake to -0.1 (half the starting reputation) — makes Sybil flagging expensive
4. OR: Flag spam penalty: if an identity submits >5 flags in 24 hours and >80% are rejected, apply -0.1 penalty (spam detection)

---

### CRIT-R3-05: Proposed content storage exemption creates unbounded liability for storage providers
**Severity:** CRITICAL
**Location:** §6 Storage Economics, "Proposed content is exempt from decay"
**Description:** The spec says:
> "Content referenced by an active (non-archived) proposal MUST NOT be garbage collected regardless of rent status. The proposal's governance status protects its content — the network bears the storage cost as a public good."

**Attack scenario:**
1. Attacker reaches 0.3 rep (minimum to propose)
2. Attacker replicates 60 MiB of junk content (approaching their storage limit)
3. Attacker submits a PROPOSE message referencing the content
4. Voting deadline: 90 days (max allowed)
5. Content is now exempt from rent payment for 90 days
6. After voting fails/expires, proposal is archived after 7-30 days (depending on outcome)
7. Total free storage: 90 + 30 = 120 days
8. Attacker repeats with new junk content every 90 days

**At scale:**
- 1000 attackers (0.3 rep each)
- Each uploads 60 MiB every 90 days
- 60 GB of unpaid storage, permanently renewed

**The problem:** Proposing content makes it immune to rent payment. The "public good" exemption is exploitable.

**Why this exists:** The spec wants voters to be able to access proposal artifacts even if the uploader can't pay rent. Legitimate use case. But no cost control.

**Suggested fix:**
1. Exempt content only for the voting period (90 days max), not the post-vote archival period
2. Charge rent for proposed content, but draw from a shared "governance pool" (funded by a small % of all reputation gains network-wide)
3. Proposed content exemption applies only if the proposal receives ≥ 3 votes from distinct nodes (signal of legitimate interest)
4. OR: Proposed content rent is deferred, not waived — if the proposal is rejected, the uploader owes back-rent for the voting period

---

## HIGH SEVERITY ISSUES

### HIGH-R3-01: Storage challenge collusion undetectable when uploader and provider are the same identity
**Severity:** HIGH
**Location:** §6 Storage Economics + §1 Identity Linking
**Description:** An identity with multiple authorized keys can:
1. Upload content from key A
2. Store shards on key B (same identity)
3. Pay rent from A to B (reputation transfer within the same identity)
4. Issue storage challenges from A to B (same identity)
5. Pass challenges trivially (both keys controlled)

**Result:** The identity extracts rent from itself, passes challenges against itself, and earns reputation from "successful" storage provision. This is a closed-loop reputation factory.

**Spec says (§1):** "Collusion detection exemption: Correlated activity between keys in the same identity is expected and MUST NOT trigger collusion penalties."

**So this is ALLOWED.** But should it be?

**Analysis:**
- Rent is a zero-sum transfer: A→B within the same identity is a no-op (identity's total reputation unchanged)
- Storage challenges passed earn... wait, do they earn reputation?

**Checking §9 reputation table:**
- "Created (new reputation entering the system)" does NOT include storage challenges
- Storage challenges are in the "Transferred" section: storage providers earn rent via challenges

**So:** Self-storage doesn't create reputation, it just transfers reputation from one key to another within the same identity. Net zero.

**BUT:** The spec says (§6): "Storage providers earn reputation from rent payments, verified via the challenge system."

**If challenges create reputation (not just transfer), this is broken.**

**Checking §6 again:** "Storage challenges... verify data integrity AND authorize rent payment. No challenge passed = no revenue."

**So:** Challenges don't CREATE reputation. They authorize TRANSFER of rent from uploader to provider. If uploader and provider are the same identity, it's a wash.

**Revised risk assessment:** LOW severity — self-storage is inefficient (you pay rent to yourself) and doesn't create reputation. Only harm is wasted network bandwidth for self-challenges.

**Downgrade to LOW.**

---

### HIGH-R3-02: Scarcity multiplier can be manipulated via false capacity reporting
**Severity:** HIGH
**Location:** §6 Storage Economics, scarcity multiplier calculation
**Description:** Scarcity multiplier is computed from PEER_ANNOUNCE messages:
```
network_utilization = 1 - (total_available / total_allocated)
scarcity_multiplier = 1 / (1 - network_utilization)
```

Nodes self-report `allocated_bytes` and `available_bytes` in PEER_ANNOUNCE. There's no verification.

**Attack scenario (suppress scarcity):**
1. Attacker deploys 10 storage nodes
2. Each reports `allocated_bytes: 1 TB`, `available_bytes: 990 GB`
3. Network sees +10 TB total capacity, mostly available
4. Network_utilization drops → scarcity multiplier drops → rent is cheap
5. Attacker actually stores nothing (the reported capacity is fake)
6. Uploaders pay low rent due to false abundance signal
7. Real storage providers earn less (rent is cheap)

**Attack scenario (inflate scarcity):**
1. Attacker controls 30% of storage nodes
2. Each reports `allocated_bytes: 1 TB`, `available_bytes: 10 GB` (falsely claim near-full)
3. Network sees high utilization → scarcity multiplier spikes → rent is expensive
4. Uploaders pay high rent
5. Attacker's storage nodes earn more (rent is expensive)
6. Attacker actually has 990 GB available per node (the scarcity is fake)

**The spec has no verification mechanism for capacity claims.**

**Suggested fix:**
1. Storage challenges should verify not just data integrity but also capacity claims — challenge a random byte offset, if the node can't produce it, penalize
2. OR: Capacity claims are weighted by reputation — new nodes' capacity reports are discounted until they've proven reliability
3. OR: Outlier detection: if a node reports 1 TB allocated but only stores 10 MB of actual shards (detected via SHARD_QUERY_RESPONSE), flag as suspicious
4. OR: Scarcity multiplier uses a moving average over 24 hours, preventing sudden manipulation

---

### HIGH-R3-03: Capability ramp at 0.3 for voting is too low — Sybil vote weight accumulation
**Severity:** HIGH
**Location:** §9 Capability Ramp
**Description:** Nodes can vote once they reach 0.3 reputation. A fresh node starts at 0.2, needs +0.1 to vote.

**Path to 0.3:**
- Uptime: 0.001/day × 100 = 0.1 (100 days)
- Proposal adoption: +0.005 per ADOPT, need 20 adoptions
- Or combination: 50 days uptime + 10 adoptions

**Attack scenario (Sybil vote farm):**
1. Attacker deploys 100 nodes (50 minutes VDF)
2. Each node runs for 100 days (uptime +0.1) → rep 0.3
3. Cost: 100 × 100 days of hosting ≈ $5,000 (at $0.50/day/node)
4. Each node can now vote with weight 0.3
5. Total Sybil vote weight: 100 × 0.3 = 30

**Network defense:**
- If network has 200 honest nodes with average rep 0.5 → total honest vote weight = 100
- Sybil vote weight (30) is 23% of total (130)
- Standard quorum: 0.67 threshold
- Sybils alone can't pass proposals (30 / 130 = 0.23 < 0.67)
- But Sybils can BLOCK proposals: 30 reject votes + 50 honest reject = 80 reject, 50 endorse → 50/130 = 0.38 < 0.67 (fails)

**The issue:** 100 days of uptime is the ONLY requirement to reach 0.3. No quality signal, just time.

**Suggested fix:**
1. Voting requires ≥ 1 successful proposal adoption (received ≥ 1 ADOPT from another node) — proves you've contributed something
2. OR: Voting weight is zero until 0.4 reputation (higher bar)
3. OR: Votes from nodes below 0.4 have 50% weight (reduced influence for unproven nodes)

---

### HIGH-R3-04: Grace period for unpaid storage is too long — 30 days allows rent evasion
**Severity:** HIGH
**Location:** §6 Storage Economics
**Description:** When uploader can't pay rent:
> "The content enters a 30-day grace period. During the grace period, storage providers are not paid but SHOULD continue holding shards."

**Attack scenario:**
1. Attacker uploads 60 MiB content (max at 0.3 rep)
2. Pays first month rent: 60 MiB × 0.001 × 2.0 (50% utilization) = 0.12 reputation
3. After 30 days, attacker's rep drops below payment threshold
4. Content enters 30-day grace period (free storage)
5. 1 day before grace period expires, attacker earns +0.12 reputation (via proposals/uptime)
6. Pays rent for month 2
7. Repeat: oscillate between "just enough rep to pay" and "grace period"

**Result:** Attacker gets 50% free storage by gaming the grace period.

**Why 30 days?** To give legitimate users time to earn reputation and resume payment. But it's exploitable.

**Suggested fix:**
1. Grace period is 7 days, not 30 (enough time to recover, not enough to systematically game)
2. OR: Grace period is pro-rated — if you were only 10% short on rent, grace period is 3 days; if 90% short, 27 days
3. OR: No grace period for repeated failures — first failure gets 30 days, second failure within 90 days gets 7 days, third gets 0 days (immediate GC)

---

### HIGH-R3-05: COMMENT rate limit (10/hour) insufficient to prevent spam on controversial proposals
**Severity:** HIGH
**Location:** §7 COMMENT message type
**Description:** COMMENT rate limit is 10 per identity per hour. For a controversial proposal:

**Attack scenario:**
1. Proposal P is controversial (community split)
2. Attacker deploys 50 Sybil identities (25 minutes VDF)
3. Each Sybil posts 10 COMMENT messages per hour on proposal P
4. 50 × 10 = 500 comments per hour
5. Over 14-day voting period: 500/hour × 24 × 14 = 168,000 spam comments

**The spec says:** "COMMENT messages... are lightweight, signed, and attributed. Comments do not affect governance — they are signal for evaluation."

**But:** 168,000 spam comments make the signal unusable. Legitimate discussion is drowned out.

**Suggested fix:**
1. COMMENT rate limit scales with reputation: 0.2 → 1/hour, 0.5 → 5/hour, 0.8 → 10/hour
2. OR: COMMENT messages cost a tiny reputation stake (-0.0001) to prevent free spam
3. OR: COMMENT rate limit is per-proposal, not global: max 3 comments per identity per proposal per day

---

### HIGH-R3-06: Hosted content SHARE re-broadcast (30 min) creates permanent gossip overhead
**Severity:** HIGH
**Location:** §6 Content, Hosted mode
**Description:** SHARE messages are re-broadcast every 30 minutes. For a network with:
- 10,000 nodes
- Each sharing 10 hosted files
- 100,000 SHARE messages every 30 minutes
- ~56 messages/second

As the network scales:
- 100,000 nodes → 5,555 SHARE messages/second (permanent gossip overhead)

**Why 30 minutes?** To keep hosted content discoverable while nodes are online. But it's a linear scaling problem.

**Suggested fix:**
1. SHARE re-broadcast only when content list changes (event-driven, not periodic)
2. OR: SHARE re-broadcast interval scales with network size: 30 min at <1K nodes, 2 hours at 10K nodes, 6 hours at 100K nodes
3. OR: Use a DHT for hosted content discovery instead of gossip (not in v0 scope, but acknowledge the limitation in §15)

---

### HIGH-R3-07: Proposed content can reference replicated content that has already decayed
**Severity:** HIGH
**Location:** §6 Content modes, §7 Proposals
**Description:** The spec says:
> "Content MUST be replicated before it can be proposed."

But what if:
1. Content C is replicated at T=0
2. Uploader pays rent for 1 month
3. At T=30 days, rent is due again
4. Uploader can't pay → grace period starts
5. At T=60 days, grace period expires → content eligible for GC
6. At T=70 days, some storage nodes have dropped shards
7. At T=80 days, someone submits PROPOSE referencing content C
8. Content C is unrecoverable (not enough shards available)
9. Proposal references dead content

**The spec says (§6):** "Proposed content is exempt from decay" — but this only applies AFTER the proposal is submitted, not retroactively.

**Suggested fix:**
1. PROPOSE message submission MUST verify that content is currently available (query for shard availability, require ≥ data_shards responses)
2. OR: PROPOSE submission automatically resurrects the content (re-initiates replication if shards are available, fails if not)
3. OR: Proposed content exemption is retroactive — if content decayed in the past 30 days, the proposal re-activates it

---

## MEDIUM SEVERITY ISSUES

### MED-R3-01: Storage base rate (0.001/MiB/month) has no justification or adjustment mechanism
**Severity:** MEDIUM
**Location:** §6 Storage Economics
**Description:** The base storage rate is:
```
base_rate = 0.001 per MiB per month
```

**Questions:**
1. Why 0.001? Is this derived from expected storage costs, or arbitrary?
2. At 50% utilization (multiplier 2.0), 1 GB costs 2 MiB × 0.001 × 2.0 = 0.002/month... no wait.

**Let me recalculate:**
1 GB = 1,024 MiB
At 50% utilization, multiplier = 2.0
Monthly rent = 1,024 × 0.001 × 2.0 = 2.048 reputation per month

**To store 1 GB for a year:**
2.048 × 12 = 24.576 reputation

**But max reputation is 1.0.** No one can afford to store 1 GB for a year.

**Wait, let me re-read...**

Ah, the formula is `content_size_mib × base_rate × scarcity_multiplier`.

So for 1 MiB at 50% utilization:
1 × 0.001 × 2.0 = 0.002 per month

For 100 MiB:
100 × 0.001 × 2.0 = 0.2 per month
0.2 × 12 = 2.4 reputation per year

**So storing 100 MiB for a year costs 2.4 reputation, but max rep is 1.0.**

**This means:** You can only store ~40 MiB permanently (at 50% utilization) if you're willing to dedicate ALL your reputation to storage.

**Is this reasonable?** For a skill/config sharing network, 40 MiB per identity is tight. A single ML model checkpoint is 100+ MiB.

**But:** The spec says (§6) max active replicated content scales with reputation:
```
max_active_replicated = 10 MiB × (reputation / 0.2)^2
```

At rep 0.5: 10 × (0.5/0.2)^2 = 10 × 6.25 = 62.5 MiB max

So you can store 62.5 MiB but it costs 0.125/month (at 50% util) = 1.5/year. That's sustainable if you earn >1.5 rep/year.

**Velocity limits:** Max 0.02/day = 7.3/year. So yes, sustainable.

**Revised assessment:** The base rate makes sense in context. But it's a fixed constant with no adjustment mechanism.

**Suggested improvement:** Add to §15 Open Questions: "Storage base rate is fixed at 0.001/MiB/month in v0. A market-driven rate adjustment mechanism (based on supply/demand signals) can be proposed via protocol governance."

---

### MED-R3-02: Capability ramp at 0.5 for severe flags is too high — delayed response to illegal content
**Severity:** MEDIUM
**Location:** §9 Capability Ramp, §6 Content Flagging
**Description:** Only nodes with rep ≥ 0.5 can flag content as `severity: illegal`.

**Scenario:**
1. New node N joins, discovers CSAM content (hash match in known-bad database)
2. N's reputation is 0.2 (fresh node)
3. N cannot flag as `illegal` (requires 0.5)
4. N can only flag as `dispute` (available at 0.3)
5. `dispute` flags don't trigger immediate shard deletion
6. Illegal content remains available until enough high-rep nodes flag it

**Why 0.5?** To prevent Sybil abuse of severe flags. But it creates a detection gap.

**Suggested fix:**
1. Hash-match flags (automated detection) don't require 0.5 rep — any node can submit a hash-match severe flag
2. Manual severe flags (no hash_match) require 0.5 rep
3. This allows new nodes to report verified CSAM without opening the door to Sybil severe-flag spam

---

### MED-R3-03: Erasure coding levels are fixed constants with no upgrade path
**Severity:** MEDIUM
**Location:** §6 Content, erasure coding table
**Description:** The spec defines:
- minimal: 3-of-5
- standard: 5-of-8
- resilient: 8-of-12

These are hardcoded. What if the network wants to add a new level (e.g., "paranoid: 12-of-20")?

**The spec says:** Protocol change proposals can add new levels. But existing content is locked to its original coding level.

**Problem:** If a proposal is ratified with "standard" coding and the network later decides "standard" is insufficient, there's no re-replication mechanism.

**Suggested fix:** Add to §15 Open Questions: "Erasure coding levels are fixed in v0. A content re-replication protocol (upgrade existing content to higher resilience) can be proposed via governance."

---

### MED-R3-04: Content flagging has no appeal mechanism for upheld flags
**Severity:** MEDIUM
**Location:** §6 Content Flagging
**Description:** FLAG → governance vote → if upheld, source DID takes -0.05 reputation penalty.

**What if the vote was wrong?** What if new evidence emerges after the vote?

**The spec has no appeal mechanism.**

**Suggested fix:** Add: "A flag upheld by governance can be challenged via a new DISPUTE proposal (standard governance, 0.67 threshold). If the dispute is upheld, the original penalty is reversed (source DID regains +0.05) and the original flaggers forfeit their stakes."

---

### MED-R3-05: Storage challenges use adjacency proofs but window_size is undefined
**Severity:** MEDIUM
**Location:** §6 Storage Challenges
**Description:** STORAGE_CHALLENGE includes:
```
"offset": <bytes>,
"direction": "before | after",
"window_size": <bytes>
```

But `window_size` is never defined. How big is the window?

**If window is too small:** Challenges are easy to precompute/cache.
**If window is too large:** Challenges are expensive to compute for legitimate providers.

**Suggested fix:** Add to constants table:
```
Storage challenge window size: 4 KiB (default)
```

And clarify: "Challengers MAY vary window_size between 1 KiB and 64 KiB to prevent precomputation."

---

### MED-R3-06: FLAG message includes "details" field with no size limit
**Severity:** MEDIUM
**Location:** §6 Content Flagging, FLAG schema
**Description:** FLAG payload includes:
```
"details": "<string — evidence or explanation>"
```

No size limit specified. An attacker can:
1. Flag content with a 7 MiB "details" field (just under the 8 MiB message limit)
2. Spam the network with massive FLAG messages

**Suggested fix:** Add to schema: "Details field MUST NOT exceed 10 KiB. Nodes MUST reject FLAG messages with details > 10 KiB."

---

### MED-R3-07: Proposed content storage exemption applies to archived proposals
**Severity:** MEDIUM
**Location:** §6 Storage Economics + §12 Proposal Retention
**Description:** The spec says:
> "Content referenced by an active (non-archived) proposal MUST NOT be garbage collected."

And §12 says ratified proposals are archived after 180 days.

**But:** What happens between proposal expiry and archival?

**Example:**
1. Proposal P references content C
2. P voting deadline expires at T=0
3. P is ratified
4. P is archived at T=180 days
5. Between T=0 and T=180, is C exempt from rent?

**The spec says:** "active (non-archived)" — so after expiry but before archival (180 days), the proposal is technically active.

**This creates 180 days of free storage for all ratified proposals.**

**Suggested fix:** Clarify: "Storage exemption applies only during the voting period (from PROPOSE submission to voting_deadline). After the deadline, normal rent applies regardless of archival status."

---

### MED-R3-08: Capability ramp has no enforcement mechanism for PROPOSE size limits
**Severity:** MEDIUM
**Location:** §9 Capability Ramp + §6 Storage
**Description:** Capability ramp says:
- 0.3: "Propose content (small, storage limit applies)"
- 0.5: "Full proposal size limits"

But what are the limits? And how are they enforced?

**The spec says (§6):** Max active replicated content scales with reputation. But there's no mention of per-proposal size limits in the capability ramp.

**Interpretation:** "Storage limit applies" means the identity-wide storage limit (max_active_replicated), not a per-proposal limit.

**Suggested clarification:** Add to §9 capability ramp table: "Small = subject to identity storage limit (§6). Full = same, but higher identity limit due to higher reputation."

Or if there ARE per-proposal limits, define them explicitly.

---

## LOW SEVERITY ISSUES

### LOW-R3-01: CONTENT_RESPONSE total_size field allows size spoofing
**Severity:** LOW
**Location:** §6 Content, CONTENT_REQUEST/RESPONSE
**Description:** CONTENT_RESPONSE includes:
```
"total_size": <bytes>
```

The requester uses this to know when the full content is received. But the provider could lie:
- Report `total_size: 1 MiB` but only send 500 KiB
- Requester waits forever for the remaining 500 KiB

**Fix:** Requester should verify `total_size` against the `content_size` from the original SHARE message (or manifest). If they don't match, reject.

**But:** The spec doesn't explicitly require this verification.

**Suggested fix:** Add to §6: "Requesters MUST verify that CONTENT_RESPONSE.total_size matches the advertised content_size from SHARE/REPLICATE_REQUEST. Mismatches MUST be treated as protocol violations."

---

### LOW-R3-02: SHARE message "tags" field has no schema or size limit
**Severity:** LOW
**Location:** §6 Content, SHARE schema
**Description:** SHARE includes:
```
"tags": ["<string>", ...]
```

No limit on tag count or tag string length. Could be exploited for message bloat:
```
"tags": ["a", "b", "c", ..., "zzz"]  // 10,000 tags
```

**Suggested fix:** Add: "Tags array MUST NOT exceed 20 entries. Each tag MUST NOT exceed 64 bytes."

---

### LOW-R3-03: "Nodes SHOULD immediately drop shards" is ambiguous timing
**Severity:** LOW
**Location:** §6 Content Flagging, illegal flag response
**Description:** The spec says:
> "Nodes that process the flag SHOULD immediately drop shards for the flagged content."

**Questions:**
1. What does "process the flag" mean? Receive it? Verify the signature? Validate the hash_match?
2. What does "immediately" mean? Within 1 second? 1 minute? 1 hour?

**Suggested fix:** Clarify: "Nodes that receive and verify a FLAG message with severity: illegal SHOULD drop shards for the flagged content within 60 seconds of FLAG receipt, after validating the signature and timestamp."

---

### LOW-R3-04: Storage rent "monthly" is 30 days, but months vary (28-31 days)
**Severity:** LOW
**Location:** §6 Storage Economics
**Description:** The spec says "monthly rent" and "30-day billing cycles." But it also says "per month" in the base rate.

**Is it 30 days or calendar months?**

**The spec says:** "30-day billing cycles (computed from the content's replication timestamp)."

**So it's 30 days, not calendar months.** This is consistent.

**But:** The base rate is described as "per month" in the formula, which could be misread as "per calendar month."

**Suggested fix:** Change "per month" to "per 30 days" everywhere for consistency.

---

### LOW-R3-05: Content modes (hosted/replicated/proposed) are hierarchical but spec doesn't clarify transitions
**Severity:** LOW
**Location:** §6 Content modes
**Description:** The spec says "each mode is a superset of the previous" but doesn't clarify:
1. Can replicated content become hosted? (downgrade)
2. Can proposed content revert to replicated if the proposal is withdrawn?
3. Can hosted content be promoted to replicated later?

**Assumption:** These are one-way transitions (hosted → replicated → proposed), not reversible.

**Suggested fix:** Add to §6: "Content mode transitions are one-way: hosted → replicated → proposed. Downgrading is not supported in v0."

---

### LOW-R3-06: FLAG "category" field has no enforcement or validation
**Severity:** LOW
**Location:** §6 Content Flagging, FLAG schema
**Description:** FLAG includes:
```
"category": "dmca | spam | malware | csam | other"
```

But the spec doesn't say what happens if category is invalid or misused (e.g., flagging spam as "csam" to trigger severe response).

**Suggested fix:** Add: "Nodes MUST validate that FLAG.category matches FLAG.severity (e.g., 'csam' requires severity: illegal). Mismatched flags SHOULD be rejected."

---

### LOW-R3-07: REPLICATE_REQUEST "coding" field allows "minimal" but protocol proposals MUST use standard/resilient
**Severity:** LOW
**Location:** §6 Content, REPLICATE_REQUEST schema + §7 Proposals
**Description:** REPLICATE_REQUEST allows `"coding": "minimal | standard | resilient"`.

But §7 says: "Protocol change proposals MUST use 'standard' or 'resilient' erasure coding."

**Is "minimal" ever valid for REPLICATE_REQUEST?**

**Answer:** Yes, for non-protocol content (skills, configs, documents). Only PROTOCOL proposals require standard/resilient.

**Suggested clarification:** Add to §6: "The 'minimal' coding level is valid for non-protocol content proposals. Protocol change proposals MUST use 'standard' or 'resilient' (§7)."

---

### LOW-R3-08: "Network bears the storage cost as a public good" creates undefined commons
**Severity:** LOW (philosophical)
**Location:** §6 Storage Economics, proposed content exemption
**Description:** The spec says:
> "The network bears the storage cost as a public good. Providers holding shards for proposed content earn from the proposal adoption reward pool instead of rent."

**Questions:**
1. What is the "adoption reward pool"? Is this the +0.005/ADOPT reputation creation?
2. How is this distributed to storage providers?
3. Is this new reputation creation, or transfer from somewhere?

**Checking §9:** "Proposal adopted by peers: +0.005 per ADOPT message received. Capped at +0.05 per proposal."

This is reputation CREATED, not transferred. So storage providers for proposed content earn from... newly created reputation? Or from the uploader's adoption rewards?

**The spec is unclear.**

**Suggested fix:** Clarify in §6: "Storage providers for proposed content receive a share of the proposal author's adoption rewards. If a proposal receives 10 ADOPT messages (author earns +0.05), storage providers collectively receive +0.05 (proportional to shards held). This is new reputation creation, totaling +0.10 for the proposal (author + providers)."

Or if that's NOT the design, specify what the adoption pool is.

---

## CROSS-REFERENCE ISSUES

### XREF-R3-01: Section renumbering — all cross-references checked and CORRECT
**Severity:** INFO
**Location:** Throughout spec
**Description:** The spec underwent renumbering:
- Old §6 (Content) is now... wait, §6 is still Content. Let me re-read the task.

**Re-reading task:** "Old §6-§14 renumbered to §7-§15."

**Checking current spec:**
- §1 Identity
- §2 Message Format
- §3 Transport
- §4 Peer Discovery
- §5 Gossip
- §6 Content (NEW)
- §7 Proposals (was §6)
- §8 Votes (was §7)
- §9 Reputation (was §8)
- §10 Sybil Resistance (was §9)
- §11 Anti-Gaming (was §10)
- §12 Partition Detection (was §11)
- §13 Protocol Evolution (was §12)
- §14 Error Handling (was §13)
- §15 Open Questions (was §14)

**Cross-reference audit:**

Searching for "§6" references in the spec:
- §1: "see below" → no §6 ref in Identity section
- §2: message type table includes §6 for content types ✓
- §6: self-references ✓
- §7: "Content MUST be replicated before it can be proposed" → should ref §6 ✓ (implicit)
- §7: "Protocol change proposals MUST use 'standard' or 'resilient' erasure coding (§6)" ✓
- §9: "Storage is a service market (§6)" ✓
- §9: "max_active_replicated (§6)" ✓
- Constants table: All section refs checked ✓

Searching for "§7" references:
- §1: "3 proposals per 7-day rolling window (§7)" → should be §7 ✓
- §6: "PROPOSE message (§8)" → WRONG, should be §7
- §8: self-references ✓
- §9: "Proposal adopted (§7)" → WRONG, should be §7 but message type is in §7
- Constants table: checked ✓

**FOUND ERRORS:**

1. **§6 PROPOSE reference error:** The sentence "Proposed (Governed): Replicated content wrapped in a PROPOSE message (§8)" should be "(§7)" not "(§8)".

2. **§9 reputation table:** "Proposal adopted by peers" references §7 implicitly (correct) but could be clearer.

**All other cross-references are CORRECT.**

---

### XREF-R3-02: Message type inventory includes new types, all present
**Severity:** INFO
**Location:** §2 Message Type Inventory
**Description:** Checking that all new message types are in the inventory table:

**New types for round 3:**
- CHALLENGE_RESULT ✓ (listed, §6)
- SHARE ✓ (listed, §6)
- REPLICATE_REQUEST ✓ (listed, §6)
- CONTENT_REQUEST ✓ (listed, §6)
- CONTENT_RESPONSE ✓ (listed, §6)
- FLAG ✓ (listed, §6)
- COMMENT ✓ (listed, §7)

**All present and correctly sectioned.**

---

### XREF-R3-03: Constants table missing several new constants
**Severity:** MEDIUM
**Location:** Constants Summary table
**Description:** The constants table is missing several values introduced in round 3:

**Missing constants:**
1. Storage challenge window_size (undefined, see MED-R3-05)
2. FLAG details max length (undefined, see MED-R3-06)
3. SHARE tag count limit (undefined, see LOW-R3-02)
4. SHARE tag length limit (undefined, see LOW-R3-02)
5. Content chunk size (mentioned in §6 as "≤ 1 MiB per CONTENT_REQUEST")
6. Hosted content SHARE re-broadcast interval (30 minutes, mentioned in §6)
7. DID_LINK re-broadcast interval (30 days, mentioned in §1 but was changed from 7 days per round 2)

**Suggested fix:** Add these to the constants table.

---

## CONFORMANCE TEST GAPS

### TEST-R3-01: No test for scarcity multiplier calculation
**Severity:** MEDIUM
**Location:** Conformance tests (none exist for §6 Storage Economics)
**Description:** The scarcity multiplier is a critical pricing mechanism but has no test coverage.

**Suggested test:**
```
STORAGE-01: Scarcity multiplier calculation
- Input:
  - total_allocated: 1000 GB
  - total_available: 500 GB
  - network_utilization: 1 - (500/1000) = 0.5
  - scarcity_multiplier: 1 / (1 - 0.5) = 2.0
- Expected: 2.0

STORAGE-02: Scarcity multiplier at high utilization
- Input:
  - total_allocated: 1000 GB
  - total_available: 10 GB
  - network_utilization: 0.99
  - scarcity_multiplier: 1 / (1 - 0.99) = 100
- Expected: 100

STORAGE-03: Scarcity multiplier edge case (prevent division by zero)
- Input:
  - total_allocated: 1000 GB
  - total_available: 0 GB
  - network_utilization: 1.0
  - scarcity_multiplier: 1 / (1 - 1.0) = UNDEFINED
- Expected: Should cap at some maximum (per CRIT-R3-01 fix)
```

---

### TEST-R3-02: No test for capability ramp enforcement
**Severity:** MEDIUM
**Location:** Conformance tests (none for §9 Capability Ramp)
**Description:** Capability ramp gates critical actions but has no test coverage.

**Suggested test:**
```
CAPABILITY-01: Voting requires 0.3 reputation
- Input:
  - Node A: rep 0.2 (fresh)
  - Node A submits VOTE message
- Expected: VOTE rejected (below 0.3 threshold)

CAPABILITY-02: Replication requires 0.3 reputation
- Input:
  - Node B: rep 0.25
  - Node B submits REPLICATE_REQUEST
- Expected: REPLICATE_REQUEST rejected

CAPABILITY-03: Severe flagging requires 0.5 reputation
- Input:
  - Node C: rep 0.4
  - Node C submits FLAG(severity: illegal)
- Expected: FLAG rejected (below 0.5 threshold)
```

---

### TEST-R3-03: No test for FLAG stake forfeiture/return
**Severity:** MEDIUM
**Location:** Conformance tests (none for §6 Content Flagging)
**Description:** FLAG staking and penalty/reward logic has no tests.

**Suggested test:**
```
FLAG-01: FLAG stake forfeited when rejected
- Input:
  - Node A: rep 0.5
  - Node A submits FLAG(severity: illegal, stake -0.02)
  - Governance vote rejects flag (0.80 weighted rejection)
- Expected:
  - Node A reputation: 0.5 - 0.02 (stake) - 0.05 (false severe penalty) = 0.43

FLAG-02: FLAG stake returned when upheld
- Input:
  - Node B: rep 0.6
  - Node B submits FLAG(severity: dispute, stake -0.005)
  - Governance vote upholds flag (0.70 weighted endorsement)
  - Content source DID takes -0.05 penalty
- Expected:
  - Node B reputation: 0.6 (stake returned)
  - Content source: 0.5 - 0.05 = 0.45
```

---

## ATTACK SURFACE SEVERITY RANKING

If I were an adversary, here's what I'd exploit first:

1. **CRIT-R3-01** (Scarcity multiplier → infinity) — Flood network storage to trigger economic collapse
2. **CRIT-R3-04** (FLAG Sybil spam) — 50 minutes VDF, 100 Sybils, grief-flag all content
3. **CRIT-R3-05** (Proposed content exemption) — Free storage via governance spam
4. **CRIT-R3-03** (Hash collision targeting) — Compromise known-bad hash database, weaponize automated flagging
5. **HIGH-R3-02** (False capacity reporting) — Manipulate scarcity multiplier for profit or sabotage

The rest require sustained coordination, significant resources, or are detectable/recoverable.

---

## POSITIVE FINDINGS (What's Solid)

✓ **Content modes hierarchy** — Clean separation (hosted/replicated/proposed) with clear incentives
✓ **Storage as transfer market** — Zero-sum economics prevents inflation
✓ **Capability ramp** — 0.2/0.3/0.5 gates prevent fresh Sybil impact
✓ **Erasure coding integration** — Standard 5-of-8 is reasonable for governed content
✓ **COMMENT threading** — Lightweight, attributed, doesn't affect governance (correct design)
✓ **Grace period concept** — 30 days is generous but the intent is sound
✓ **FLAG tiered severity** — dispute vs. illegal distinction is appropriate
✓ **Reputation transfer clarity** — §9 clearly separates creation from transfer
✓ **Cross-references** — 99% correct after renumbering (only 1 error found)

---

## SUMMARY STATS

- **Critical:** 5 issues (scarcity multiplier infinity, capability ramp lock-in, hash collision targeting, FLAG Sybil spam, proposed content exemption)
- **High:** 7 issues (storage capacity manipulation, capability ramp Sybil votes, grace period gaming, COMMENT spam, SHARE gossip overhead, decayed content in proposals)
- **Medium:** 8 issues (base rate justification, severe flag capability, erasure coding upgrade path, no flag appeal, challenge window undefined, FLAG details bloat, storage exemption archival edge case, capability size limits unclear)
- **Low:** 8 issues (content size spoofing, tags bloat, shard drop timing, monthly vs. 30-day, mode transitions, FLAG category validation, minimal coding clarification, adoption pool undefined)
- **Cross-reference:** 1 error (§6 references §8 instead of §7 for PROPOSE)
- **Conformance test gaps:** 3 critical gaps (scarcity multiplier, capability ramp, FLAG mechanics)
- **Constants table:** 7 missing entries

**Total:** 32 issues identified (29 protocol issues + 1 xref error + 2 meta issues)

---

## COMPARISON TO PREVIOUS ROUNDS

**Round 1:** 40 issues (8 HIGH, rest lower/editorial)
**Round 2:** 5 new issues (2 CRIT, 2 HIGH, 1 MED) from identity linking fixes
**Round 3:** 32 issues (5 CRIT, 7 HIGH) from new content/economics sections

**Trend:** New features introduce new attack surface. The core protocol (identity, gossip, voting) is stabilizing. The content layer and storage economics need hardening.

---

## RECOMMENDATIONS

### Immediate (Must Fix Before Any Deployment)

1. **CRIT-R3-01:** Cap scarcity multiplier at finite maximum (100×) with floor to prevent /0
2. **CRIT-R3-04:** Require ≥5 flags from ≥3 ASNs before immediate shard deletion, OR increase severe flag stake to -0.1
3. **CRIT-R3-05:** Proposed content exemption only during voting period (not 180-day archival), OR require ≥3 votes for exemption

### High Priority (Fix Before Beta)

1. **CRIT-R3-02:** Lower storage provider barrier to 0.2 OR add bootstrap reputation path for storage
2. **CRIT-R3-03:** Multi-source hash verification for automated severe flags
3. **HIGH-R3-02:** Capacity claim verification or outlier detection
4. **HIGH-R3-03:** Voting requires ≥1 proposal adoption OR raise threshold to 0.4
5. **HIGH-R3-04:** Grace period reduced to 7 days OR pro-rated by shortfall

### Medium Priority (Address Before 1.0)

1. Add storage base rate to §15 Open Questions (market-driven adjustment)
2. Define storage challenge window_size (4 KiB default, 1-64 KiB range)
3. Add FLAG details max length (10 KiB)
4. Clarify proposed content → storage provider revenue model
5. Write conformance tests for: scarcity multiplier, capability ramp, FLAG mechanics

### Low Priority (Document/Clarify)

1. Fix §6 cross-reference (§8 → §7)
2. Add missing constants to table
3. Clarify content mode transitions (one-way)
4. Change "per month" to "per 30 days"

---

**Review complete.** The content and storage economics are ambitious and mostly well-designed, but have 5 critical vulnerabilities that MUST be addressed. The capability ramp and flagging system are solid concepts with execution gaps. Fix the scarcity multiplier, FLAG spam, and proposed content exemption before deployment.
