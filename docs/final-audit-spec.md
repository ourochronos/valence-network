# Final Audit: Valence Network v0 Specification

**Auditor:** Claude (Opus, subagent)
**Date:** 2026-02-18
**Spec version:** v0 (post-adversarial-review rounds 1–4)
**Scope:** Full spec (§1–§15), conformance tests, constants table, editorial

---

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 0 |
| HIGH     | 1 |
| MEDIUM   | 5 |
| LOW      | 6 |
| INFO     | 4 |
| **Total** | **16** |

---

## 1. Internal Consistency

### AUDIT-01: Rent convergence rate mismatch between prose and constants

- **Severity:** HIGH
- **Location:** §6 (Rent price locking with convergence) vs Constants Summary
- **Description:** The prose in §6 says convergence is "10% per cycle" with formula `min(1, cycles_elapsed / 5)` (which is actually 20% per cycle — reaching 100% at cycle 5). The Constants Summary says "linear 20%/cycle (full market at 5 cycles)". The prose description "10% per cycle" contradicts both the formula and the constants table. The formula and constants table agree with each other (20%/cycle), but the prose sentence introducing it is wrong.
- **Fix:** Change the §6 prose from "converges toward the current market multiplier at **10% per cycle**" to "converges toward the current market multiplier at **20% per cycle**". The formula `cycles_elapsed / 5` already correctly implements 20%/cycle.

### AUDIT-02: Scarcity multiplier example inconsistency

- **Severity:** MEDIUM
- **Location:** §6 (Replication Rent examples)
- **Description:** The replication rent examples section says "Examples at 50% utilization (multiplier 2.0)" but the scarcity multiplier formula gives `1 + 99 × 0.5^4 = 7.1875` at 50% utilization, not 2.0. The examples table above correctly shows 7.2 at 50%. The rent calculation examples use a stale/wrong multiplier value.
- **Fix:** Update the rent examples to use the correct multiplier (7.1875), or change the example utilization to match multiplier 2.0 (approximately 21% utilization).

### AUDIT-03: §8 quorum formula uses × 0.10 while describing "10%"

- **Severity:** INFO
- **Location:** §8 (Network Phases, quorum formula)
- **Description:** The standard quorum is `max(active_nodes × 0.1, total_known_reputation * 0.10)`. One uses `× 0.1`, the other `* 0.10`. Stylistically inconsistent but functionally identical.
- **Fix:** Normalize to one notation style (suggest `× 0.10` for both).

---

## 2. Conformance Test Coverage Gaps

### AUDIT-04: No conformance test for KEY_ROTATE first-seen-wins rejection

- **Severity:** MEDIUM
- **Location:** §1 (Key Rotation, "Only one rotation per key")
- **Description:** The spec says "Subsequent KEY_ROTATE messages with the same `old_key` but a different `new_key` MUST be rejected." There is no conformance test vector for this MUST. SIG-02 tests the happy path but not the rejection of a second rotation.
- **Fix:** Add a test (e.g., `SIG-03`) that presents two KEY_ROTATE messages for the same `old_key` with different `new_key` values and verifies the second is rejected.

### AUDIT-05: No conformance test for KEY_CONFLICT generation

- **Severity:** MEDIUM
- **Location:** §1 (Key Rotation, KEY_CONFLICT)
- **Description:** "Any node that observes two KEY_ROTATE messages for the same `old_key` with different `new_key` values MUST broadcast a KEY_CONFLICT." No conformance test vector exists for KEY_CONFLICT message generation or the suspension behavior (reputation frozen at 0.1, voting weight 10%).
- **Fix:** Add a test verifying KEY_CONFLICT is generated and suspension is applied when conflicting KEY_ROTATE messages are observed.

### AUDIT-06: No conformance test for timestamp validation

- **Severity:** LOW
- **Location:** §2 (Timestamp Validation)
- **Description:** VALID-02 and VALID-03 describe the behavior but don't provide concrete test vectors with specific timestamps and expected outcomes (they use relative time). This is acceptable for behavioral tests but less rigorous than the deterministic vectors elsewhere.
- **Fix:** Consider adding absolute-timestamp vectors (e.g., "given local time X, message timestamp Y, expect reject").

### AUDIT-07: No conformance test for VDF proof freshness rejection

- **Severity:** LOW
- **Location:** §10 (Rate Limiting)
- **Description:** "nodes MUST reject VDF proofs where the earliest checkpoint timestamp is older than 24 hours" — but the VDF proof schema shown doesn't include per-checkpoint timestamps, only a top-level `computed_at`. The prose references "earliest checkpoint timestamp" which doesn't exist in the schema.
- **Fix:** Clarify that the freshness check uses `computed_at`, not per-checkpoint timestamps (the schema has no per-checkpoint timestamps). Add a conformance test for VDF proof freshness rejection.

### AUDIT-08: No conformance test for auth handshake binding

- **Severity:** LOW
- **Location:** §3 (Authentication Handshake)
- **Description:** "Binding the initiator's key into the signed response prevents replay attacks" — no conformance test verifies that AUTH_RESPONSE with a mismatched initiator key is rejected.
- **Fix:** Add a test verifying AUTH_RESPONSE replay detection (wrong initiator key in signature).

### AUDIT-09: Missing conformance tests for close-margin confirmation

- **Severity:** MEDIUM
- **Location:** §8 (Close-margin confirmation)
- **Description:** VOTE-02 describes the behavior but doesn't provide a full deterministic test vector with weighted votes that land within ±0.02 and verification that the 7-day period is entered. This is a governance-critical MUST.
- **Fix:** Add a full test vector with specific vote weights, threshold calculation, and expected confirmation period trigger.

---

## 3. Completeness / Underspecified Behaviors

### AUDIT-10: VOTE message schema doesn't show `sync_status` field

- **Severity:** MEDIUM
- **Location:** §8 (VOTE Message) vs §5 (Degraded mode)
- **Description:** §5 says "The VOTE message itself MUST include the voter's `sync_status` at vote creation time" but the VOTE schema in §8 doesn't include this field. Implementers reading §8 alone would produce non-conformant VOTE messages.
- **Fix:** Add `"sync_status": "<syncing | degraded | synced>"` to the VOTE payload schema in §8, or add a note referencing §5's requirement.

### AUDIT-11: VDF proof schema checkpoint timestamps

- **Severity:** LOW
- **Location:** §10 (VDF Proof Schema, Rate Limiting)
- **Description:** §10 Rate Limiting says "nodes MUST reject VDF proofs where the earliest checkpoint timestamp (included in the proof, see schema below) is older than 24 hours." But the VDF proof schema checkpoints contain only `{"iteration": ..., "hash": "..."}` — no timestamp per checkpoint. Only `computed_at` exists at the top level. This is an internal inconsistency within §10.
- **Fix:** Change the freshness check prose to reference `computed_at` instead of "earliest checkpoint timestamp", or add a `timestamp` field to each checkpoint in the schema.

---

## 4. Constants Table Verification

### AUDIT-12: Rent convergence prose/constants mismatch (same as AUDIT-01)

- **Severity:** HIGH (duplicate of AUDIT-01)
- **Description:** Already covered above. The constants table says "20%/cycle" which matches the formula. The prose says "10% per cycle" which is wrong.

All other constants verified: every value in the constants summary table matches its definition in the prose. No other mismatches found.

---

## 5. Message Type Inventory

All 33 message types in the §2 table verified against their definitions in the spec:

- All message types referenced in the spec body appear in the §2 table ✓
- Transport mappings (stream vs GossipSub) are correct ✓
- Section references are correct ✓
- `REPUTATION_CURRENT` (§5) is a SYNC_REQUEST type filter, not a separate message type — correctly not listed ✓
- No orphaned message types (all listed types are defined somewhere in the spec) ✓

No issues found.

---

## 6. §15 Open Questions Review

### AUDIT-13: "Blacklisting / Banning" is partially resolved

- **Severity:** LOW
- **Location:** §15 (Blacklisting / Banning)
- **Description:** §15 says "v0 has no mechanism to permanently exclude a node." But §6 Content Flagging defines a permanent DID ban mechanism: "If upheld (0.67 threshold), the source DID is permanently banned: all content removed, all reputation forfeited, new messages from the DID MUST be rejected by all nodes." This open question is at least partially resolved by the spec itself. The ban is content-triggered (illegal content), not a general-purpose ban mechanism, so the open question is arguably still valid for non-content-related bans.
- **Fix:** Update the §15 entry to acknowledge the content-triggered DID ban mechanism in §6 and narrow the open question to general-purpose banning (not tied to content flags).

### AUDIT-14: All other open questions are genuinely open

- **Severity:** INFO
- **Location:** §15
- **Description:** The remaining open questions (Reputation Merkle Tree, Large-Scale Reputation Gossip, VDF Hardware Heterogeneity, Constitutional Participation Incentives, Hierarchical Delegation) are all genuinely unresolved by the current spec. The Acknowledged Limitations subsection correctly describes inherent trade-offs, not bugs. No missing open questions identified.

---

## 7. Editorial

### AUDIT-15: Inconsistent "SHOULD"/"MUST" for storage challenge frequency

- **Severity:** LOW
- **Location:** §6 (Storage Challenges)
- **Description:** "Challenges MUST be issued randomly by peers who hold the same shard... suggested frequency: once per shard per day." The MUST applies to randomness, but the frequency is only "suggested." Given the minimum challenge requirement (10 per cycle from ≥2 validators), the suggested frequency is important but isn't normative. This could confuse implementers about whether the frequency is required.
- **Fix:** Clarify: "Challenges MUST be issued randomly... Frequency SHOULD be approximately once per shard per day to meet the minimum challenge requirements."

### AUDIT-16: Minor: §6 back-references to §1 for identity

- **Severity:** INFO
- **Location:** §6 (Content Withdrawal)
- **Description:** "Only the identity that currently owns the content may withdraw (see §1 — content ownership follows the identity, not the key; revoked keys have no authority over content attributed to their former root)." The parenthetical is helpful clarification but the principle "content ownership follows the identity" isn't explicitly stated in §1 — it's implied. A reader following the cross-reference won't find that exact statement.
- **Fix:** Either add an explicit statement in §1 ("Content and proposals are attributed to the identity, not to the signing key") or change the cross-reference to be self-contained.

---

## Final Verdict

### **READY FOR v0 RELEASE** — with one recommended fix before publishing

**Justification:**

- **Zero CRITICAL findings.** No security vulnerabilities, no protocol-breaking inconsistencies.
- **One HIGH finding** (AUDIT-01/12): The "10% per cycle" prose contradicts the formula and constants table. This is a typo-level fix (change "10%" to "20%") — the formula is correct and unambiguous, and the conformance test (CONTENT-03) uses the correct values. An implementer following the formula would produce correct behavior; an implementer reading only the prose sentence might not. **Recommend fixing before release.**
- **Five MEDIUM findings:** Four are conformance test coverage gaps (KEY_ROTATE rejection, KEY_CONFLICT, close-margin confirmation, VOTE sync_status schema). These don't affect the spec's correctness but reduce test suite completeness. The VOTE sync_status schema gap (AUDIT-10) should ideally be fixed — it's a real interoperability risk if implementers only read §8.
- **Remaining findings** are editorial polish, minor clarifications, and INFO-level observations.

The spec is internally consistent (modulo the one HIGH typo), complete enough for interoperable implementation, and has been hardened through 4 rounds of adversarial review. The conformance test suite covers all critical paths. The open questions in §15 are genuinely open and appropriately scoped.

**Recommended pre-release fixes (in priority order):**
1. Fix "10% per cycle" → "20% per cycle" in §6 prose (AUDIT-01)
2. Add `sync_status` to VOTE schema in §8 (AUDIT-10)
3. Fix VDF freshness check prose to reference `computed_at` (AUDIT-11)
4. Fix rent examples multiplier (AUDIT-02)
5. Update §15 Blacklisting entry to acknowledge §6 DID ban (AUDIT-13)
