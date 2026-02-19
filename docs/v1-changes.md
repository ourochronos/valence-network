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

## 2. 24-Hour Payout Cycles with Flat Rate

**Section:** §6 (Content)

**Rationale:** 30-day cycles with price locking + convergence were over-engineered. At small scale (20-50 nodes), capacity won't be scarce. Daily cycles keep the market responsive without complex pricing.

**Changes:**
- Rent payout cycle: 30 days → 24 hours
- **Flat rate** per GB per day (set at genesis, adjustable via governance)
- No price locking, no convergence math, no scarcity multiplier

**Eliminates:**
- Price locking at replication time
- Convergence rate math (20%/cycle, 30% accelerated)
- `cycles_elapsed × 3 / 10` rational arithmetic
- Scarcity multiplier `1 + 99 × utilization⁴`
- All price-lock and convergence conformance tests

**Future:** Scarcity pricing and moving average smoothing can be introduced via governance when the network is large enough for capacity to matter.

---

## 3. Full Copies Instead of Erasure Coding

**Section:** §6 (Content)

**Rationale:** Erasure coding (Reed-Solomon, shard coordination, bucket math) is complex infrastructure for a small network. At 20-50 nodes, full copies are simpler, easier to verify, and sufficient for redundancy.

**Change:**
- Content is stored as full copies, not erasure-coded shards
- Consumer specifies number of copies in REPLICATE_REQUEST (1-5, default 3)
- **100MB file size cap** (can be raised via governance)
- Storage challenges verify the full copy (hash of content, not shard proofs)
- No shard assignment, no shard coordination, no encoding/decoding

**Future:** Erasure coding can be introduced when files get bigger and storage gets expensive. The consumer-chosen parts/buckets design (2-32 parts, K-of-N reconstruction) is documented but deferred.

---

## 4. Storage Obligation Transfer

**Section:** §6 (Content) — new message types

**Rationale:** Providers whose capacity drops should be able to hand off gracefully rather than failing challenges.

**New message types:**
- `RENT_TRANSFER` — provider initiates transfer of storage obligation to another provider
- `RENT_TRANSFER_ACK` — receiving provider confirms acceptance

**Economics:**
- Transfer fee: small rep cost to initiator (TBD — needs economics pass)
- Transfer fee << failure penalty (strong incentive to transfer rather than fail)
- Content must be transmitted to new provider before transfer is valid

---

## 5. QoS by Reputation

**Section:** §3 (Transport) or new subsection

**Rationale:** A full bandwidth economy is an unsolved problem in P2P. Instead, use reputation as a natural QoS signal.

**Change:**
- Peers prioritize serving higher-rep nodes (bandwidth, latency, connection slots)
- No bandwidth metering, no per-peer accounting, no transit fees
- Local peer behavior, not protocol-level enforcement

---

## 6. Stacking Storage Penalties

**Section:** §6 (Content), §9 (Reputation)

**Rationale:** Flat -0.05 per failed challenge is catastrophic for honest providers. A power outage shouldn't cost months of earned reputation. Penalties should start small and escalate — honest failures are cheap, abandonment is expensive.

**Change:** Penalties stack within a **7-day rolling window:**

| Failure # (in window) | Penalty |
|----------------------|---------|
| 1st | small |
| 2nd | larger |
| 3rd | significant |
| 4th+ | harsh |

Specific values TBD — need to model real scenarios (home user missing 1 challenge/week, provider down 4 hours, etc.) before picking numbers.

- Window resets as challenges are passed
- Timeout (node offline) vs wrong proof (data loss) may warrant different treatment

---

## 7. Daily-Adjusted Recurring Rep Values

**Section:** §9 (Reputation)

**Rationale:** With 24h cycles replacing 30-day cycles, recurring rep values divide by ~30 to maintain the same monthly effect.

| Value | v0 (monthly) | v1 (daily) | Monthly equivalent |
|-------|-------------|-----------|-------------------|
| Inactivity penalty | -0.02/month | -0.000667/day | -0.02/month |
| Uptime reward | +0.01/cycle | +0.000333/day | +0.01/month |

Discrete event values (adoption reward, flagging penalties) unchanged.

---

## 8. Content Metadata and Search

**Section:** §7 (Proposals) — PROPOSE schema, new subsection on search conventions

**Rationale:** Discovery requires metadata. Minimum viable: structured fields in PROPOSE, local search indexes.

**Schema additions to PROPOSE payload:**
```json
{
  "tags": ["string"],          // max 10 tags, max 64 chars each, lowercase
  "description": "<string>"   // max 1 KiB, plain text or markdown
}
```

Both fields optional. Nodes index PROPOSE metadata they receive via gossip. Search is local — you search your view of the network, weighted by proposer reputation.

**Tag conventions (guidance, not enforced):**
- Lowercase, hyphen-separated: `machine-learning`, `rust-crate`, `fine-tune`
- Specificity over breadth: `llama3-8b` better than `ai`
- Include content format when useful: `safetensors`, `csv`, `markdown`
- Domain prefix for namespacing: `ml/llama3`, `lang/rust`, `data/benchmark`

**Description conventions (guidance, not enforced):**
- First line: what it is (one sentence)
- Then: why someone would want it, what it's compatible with, known limitations
- Include version numbers, model names, dataset sizes — concrete details

**Future path:** Metadata standards are proposable content. Good standards get adopted through stigmergy. Bad ones fade.

---

## Deferred (post-MVP)

- **Erasure coding** — consumer-chosen parts/buckets (2-32 parts, K-of-N). When files get bigger and storage gets expensive.
- **Scarcity pricing** — `1 + 99 × utilization⁴` with moving average smoothing. When capacity actually matters.
- **Prepaid rent escrow** — lock rep upfront, evict on shortfall. When anonymous bad actors are realistic.
- **Availability tiers** — casual/reliable/dedicated with calibrated challenges. Let tiers emerge from uptime data instead.
- **Identity-group challenge routing** — any device answers challenges. With small stacking penalties, a missed challenge is trivial. Not worth the complexity.
- **Mutable content (CONTENT_REF)** — named pointers, version chains, encrypted volume sync. Too complex for MVP.
- **Bandwidth economy** — per-peer metering, transit compensation. Replaced by QoS-by-rep.
- **Routing incentives** — proof-of-relay. Open research problem.
- **PRODUCT.md parity with spec** — multiple values diverge (starting rep 0.5 vs 0.2, capability thresholds, rep subcategories). Needs systematic audit.

---

## Open: Economics Pass Needed

Before implementing, need to model:
- **Flat rent rate**: What's the right rep-per-GB-per-day at genesis?
- **Penalty curve**: Specific stacking values grounded in real scenarios
- **Transfer fee**: Small enough to incentivize transfer over failure, large enough to prevent churn farming
- **Daily rep values**: Verify monthly equivalents produce the right capability ramp timeline (how long from 0.2 to 0.3? to 0.5?)

---

## Implementation Notes

- All changes target `valence-bond` (production fork), not `valence-network-rs` (frozen ref impl)
- Spec amendments should be written against `valence-network/docs/v0.md` as a new version section or a `v1.md`
- Conformance tests for changed behaviors need rewriting
- FixedPoint type needs scale change (10,000 → 1,000,000)
- Erasure coding code stays in bond but is unused until post-MVP
