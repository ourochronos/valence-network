# v1 Spec Changes

Proposed changes to the v0 protocol spec for v1, documented during design session Feb 18, 2026.

Status: **Draft** — design decisions captured, spec amendments not yet written.

---

## 1. Fixed-Point Precision: ×10,000 → ×1,000,000

**Section:** §2 (Message Format), §9 (Reputation), Constants table

**Rationale:** ×10,000 (4 decimal places, 10,001 distinct values) is too coarse for daily micro-payments, rent splits across many providers, and transfer fees. With daily rent cycles and multi-provider splits, values round to zero at the margin.

**Change:**
- All wire-format integers scale by 1,000,000 instead of 10,000
- Smallest unit: 0.000001 rep
- Intermediate arithmetic: i64 safe — √(i64_max) ≈ 3×10⁹, product of two ×1M values = 10¹², 3 orders of magnitude headroom
- Truncation rule unchanged (truncate on final result only)

**Impact:** Wire format breaking change. All fixed-point constants, examples, and conformance tests need updating.

---

## 2. Daily Rent Cycles with Spot Pricing

**Section:** §6 (Content)

**Rationale:** 30-day cycles with price locking + convergence were designed to protect against monthly price shocks. With ×1M precision, daily cycles are viable and make the storage market much more responsive. Price locking becomes unnecessary — a bad price only lasts 24 hours.

**Changes:**
- Rent payout cycle: 30 days → 24 hours
- **Spot pricing:** Rent computed daily at current market rate. No price locking at replication time.
- Scarcity multiplier uses **7-day moving average** of network utilization (smooths daily noise)
- Scarcity curve unchanged: `1 + 99 × utilization⁴`

**Eliminates:**
- Price locking at replication time
- Convergence rate math (20%/cycle, 30% accelerated at >10× divergence)
- `cycles_elapsed × 3 / 10` rational arithmetic
- All price-lock conformance tests

**Keeps:**
- Scarcity multiplier `1 + 99 × utilization⁴`
- Rent exemption during voting (with ≥3 endorsements)
- CONTENT_WITHDRAW with 24h delay

---

## 3. Prepaid Rent Escrow

**Section:** §6 (Content)

**Rationale:** Prevents the attack where someone commits storage, gets content replicated, then craters their rep — leaving providers holding shards with no rent income.

**Change:**
- Rep is **locked** for the next payout cycle upfront when committing content
- Each cycle, locked rep transfers to providers
- If available rep (total minus locked) drops below floor, can't commit more
- If total rep drops below locked amount, content evicted (lowest-adoption first)
- Grace period: 1 cycle before eviction (allows recovery or transfer)

---

## 4. Storage Obligation Transfer

**Section:** §6 (Content) — new message types

**Rationale:** Providers whose capacity drops should be able to hand off obligations gracefully rather than failing challenges. Consumers who want to leave should be able to transfer rather than withdraw.

**New message types:**
- `RENT_TRANSFER` — provider or consumer initiates transfer of storage obligation to another party
- `RENT_TRANSFER_ACK` — receiving party confirms acceptance

**Economics:**
- Transfer fee: small rep cost to initiator (TBD, order of 0.001)
- Transfer fee << failure penalty (strong incentive to transfer rather than fail)
- Shards must be transmitted to new provider before transfer is valid
- No fee if transfer is due to capacity below minimum threshold (graceful degradation)

---

## 5. QoS by Reputation (No Bandwidth Economy)

**Section:** §3 (Transport) or new subsection

**Rationale:** A full bandwidth market (metering, per-peer accounting, transit compensation) is an unsolved problem in P2P networks. Proof-of-relay is open research. Instead, use reputation as a natural QoS signal.

**Change:**
- Peers prioritize serving higher-rep nodes (bandwidth, latency, connection slots)
- No bandwidth metering, no per-peer accounting, no transit fees
- No new message types needed — this is local peer behavior, not protocol

**Deferred:**
- Bandwidth metering and accounting
- Transit/relay compensation
- Per-provider bandwidth pricing

---

## 6. Stacking Storage Penalties

**Section:** §6 (Content), §9 (Reputation)

**Rationale:** Flat -0.05 per failed storage challenge is catastrophic for honest providers with infrastructure issues. A provider with 21 storage obligations who has a power outage loses months of earned reputation in one event. The risk/reward asymmetry discourages participation.

**Change:** Penalties stack within a **7-day rolling window:**

| Failure # (in window) | Penalty |
|----------------------|---------|
| 1st | -0.000100 |
| 2nd | -0.000500 |
| 3rd | -0.002000 |
| 4th | -0.010000 |
| 5th+ | -0.050000 |

- Window resets as challenges are passed
- Distinguishes timeout (node offline) from wrong proof (data loss): timeout gets grace window + retry; wrong proof is immediate penalty
- Total penalty per 24h cycle is implicitly capped by the stacking curve

**Replaces:** Flat -0.05 per failed challenge, flat -0.002 accept-and-abandon penalty.

---

## 7. Availability Tiers

**Section:** §6 (Content) — new subsection

**Rationale:** The current spec treats all storage providers as equivalent. Real providers range from laptops that sleep to cloud instances with SLAs. The protocol should let providers advertise what they actually offer and let uploaders choose.

**New concept:** Providers declare an availability tier in REPLICATE_ACCEPT:

| Tier | Uptime Target | Rent Multiplier | Challenge Frequency | Grace Window | Typical Hardware |
|------|--------------|-----------------|--------------------:|-------------:|-----------------|
| `casual` | ~90% | 0.3× | Lower | Longer | Laptops, home machines |
| `reliable` | ~99% | 1.0× (baseline) | Standard | Standard | Home servers, desktops |
| `dedicated` | ~99.9% | 2.0× | Higher | Shorter | Cloud, dedicated servers |

- Uploaders specify desired tier in REPLICATE_REQUEST
- Challenge frequency and grace windows calibrated per tier
- Casual providers earn less but aren't punished for being normal computers
- Dedicated providers earn more but are held to stricter standards

---

## 8. Identity-Group Challenge Routing

**Section:** §1 (Identity), §6 (Content)

**Rationale:** Users with multiple devices (laptop + cloud instances) linked under one DID should benefit from combined availability. A challenge should pass if ANY device in the identity group can answer.

**Change:**
- Storage challenges for content held by an identity route to **all devices** in that identity group
- Any device holding the challenged shard can respond
- One valid response = challenge passed
- Only fails if NO device responds within the tier's grace window
- Identity group counts as **one logical provider** for diversity requirements (can't game ASN diversity with your own devices)

**Effect:** Multi-device users naturally have higher availability (union of device uptimes, not intersection). A home user with one cloud instance becomes a reliable provider even if their laptop sleeps.

---

## 9. Daily-Adjusted Recurring Rep Values

**Section:** §9 (Reputation)

**Rationale:** With 24h cycles replacing 30-day cycles, recurring rep values need adjustment to maintain the same monthly effect.

| Value | v0 (monthly) | v1 (daily) | Monthly equivalent |
|-------|-------------|-----------|-------------------|
| Inactivity penalty | -0.02/month | -0.000667/day | -0.02/month |
| Uptime reward | +0.01/cycle | +0.000333/day | +0.01/month |

Discrete event values unchanged:
- Adoption reward: +0.05
- Transfer fee: ~0.001 (new)
- Storage challenge failure: per stacking curve (see §6 above)

---

## Changes NOT included (deferred)

- **Content tagging / search** — `tags` field in PROPOSE, search index. Pinned for later discussion.
- **Bandwidth economy** — per-peer metering, transit compensation. Replaced by QoS-by-rep.
- **Routing incentives** — proof-of-relay. Open research problem.
- **PRODUCT.md parity with spec** — multiple values diverge (starting rep 0.5 vs 0.2, capability thresholds, rep subcategories). Needs systematic audit.

---

## Implementation Notes

- All changes target `valence-bond` (production fork), not `valence-network-rs` (frozen ref impl)
- Spec amendments should be written against `valence-network/docs/v0.md` as a new version section or a `v1.md`
- Conformance tests for changed behaviors need rewriting
- FixedPoint type in ref impl needs scale change (10,000 → 1,000,000)
