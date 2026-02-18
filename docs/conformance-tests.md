# Valence Network v0 — Conformance Test Suite

This document contains deterministic test vectors for every MUST in the v0 specification that produces a deterministic output. Implementers SHOULD use these vectors to validate their implementations.

## Test Keypairs

All tests use the following Ed25519 keypairs unless otherwise noted.

**Keypair A (primary):**
- Seed (private key): `0000000000000000000000000000000000000000000000000000000000000001`
- Public key: `4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29`

**Keypair B (secondary, used for KEY_ROTATE):**
- Seed (private key): `0000000000000000000000000000000000000000000000000000000000000002`
- Public key: `7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674`

---

## 1. Canonicalization (§2)

### CANON-01: Nested object key sorting

- **Description:** JCS recursively sorts object keys lexicographically by Unicode code point.
- **Input (logical):** `{"z": 1, "a": {"c": 3, "b": 2}}`
- **Expected output (bytes):** `{"a":{"b":2,"c":3},"z":1}`
- **Spec reference:** §2 — Canonicalization

### CANON-02: Unicode key ordering

- **Description:** Keys with non-ASCII characters sort by Unicode code point.
- **Input (logical):** `{"ä": "ö", "a": "b"}`
- **Expected output (bytes):** `{"a":"b","ä":"ö"}`
- **Spec reference:** §2 — Canonicalization

### CANON-03: Null values included

- **Description:** Null values MUST be included in canonical output, not omitted.
- **Input (logical):** `{"b": null, "a": 1}`
- **Expected output (bytes):** `{"a":1,"b":null}`
- **Spec reference:** §2 — Canonicalization

### CANON-04: Numeric types — integers only

- **Description:** All numeric fields in protocol messages MUST be integers. No floating-point on the wire.
- **Input (logical):** `{"float_as_int": 10000, "int": 1, "neg": -1, "zero": 0}`
- **Expected output (bytes):** `{"float_as_int":10000,"int":1,"neg":-1,"zero":0}`
- **Spec reference:** §2 — Canonicalization

### CANON-05: Fixed-point encoding — standard values

- **Description:** Floating-point values are encoded as integers: multiply by 10,000 and truncate.
- **Input → Expected output:**
  - `0.8` → `8000`
  - `0.67` → `6700`
  - `0.0` → `0`
  - `1.0` → `10000`
  - `0.1` → `1000`
- **Spec reference:** §2 — Canonicalization

### CANON-06: Fixed-point truncation (not rounding)

- **Description:** Fixed-point conversion truncates, does not round.
- **Input:** `0.67891`
- **Expected output:** `6789` (not `6790`)
- **Spec reference:** §2 — Canonicalization

---

## 2. Content Addressing (§2)

### ADDR-01: Signing body construction and content address

- **Description:** The signing body excludes `version`, `id`, and `signature`. Keys are in JCS lexicographic order: `from`, `payload`, `timestamp`, `type`.
- **Input envelope:**
  ```json
  {
    "version": 0,
    "type": "PROPOSE",
    "from": "4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29",
    "timestamp": 1700000000000,
    "payload": {"title": "Test Proposal", "body": "Hello world"}
  }
  ```
- **Expected signing body (JCS bytes):**
  ```
  {"from":"4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29","payload":{"body":"Hello world","title":"Test Proposal"},"timestamp":1700000000000,"type":"PROPOSE"}
  ```
- **Expected `id` (SHA-256 of signing body UTF-8 bytes):**
  ```
  9f827d6492e180166a78958594a000b88063ba7a4ab3474749732cca5d60fdb3
  ```
- **Spec reference:** §2 — Signing Body, Content Addressing

### ADDR-02: `version` excluded from signing body

- **Description:** Changing `version` does not change the signing body, `id`, or signature. The signing body for a `version: 0` and `version: 1` message with the same `from`, `type`, `timestamp`, and `payload` are identical.
- **Input:** Same envelope as ADDR-01 but with `"version": 1`
- **Expected `id`:** `9f827d6492e180166a78958594a000b88063ba7a4ab3474749732cca5d60fdb3` (same as ADDR-01)
- **Spec reference:** §2 — Signing Body

### ADDR-03: Different `from` produces different ID

- **Description:** Two messages with identical payloads but different senders have different IDs.
- **Input:** Same as ADDR-01 but `from` = `7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674` (Keypair B)
- **Expected `id`:** `c2e79409de028e00f6218509d955fdbbcd4aa747bffa730e4d994de85405f619`
- **Spec reference:** §2 — Content Addressing

### ADDR-04: Different `timestamp` produces different ID

- **Description:** Two messages with identical payloads and sender but different timestamps have different IDs.
- **Input:** Same as ADDR-01 but `timestamp` = `1700000001000`
- **Expected `id`:** `0cfa99228d8783a5fce1f68ae1614272fee4ad0c0028606efc82f4f7aa8cf396`
- **Spec reference:** §2 — Content Addressing

---

## 3. Signatures (§1, §2)

### SIG-01: Sign and verify a message

- **Description:** Sign the signing body with Ed25519 private key, verify with public key.
- **Keypair:** Keypair A
- **Signing body (UTF-8 bytes):**
  ```
  {"from":"4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29","payload":{"body":"Hello world","title":"Test Proposal"},"timestamp":1700000000000,"type":"PROPOSE"}
  ```
- **Expected signature (hex):**
  ```
  b2efdcfa9498823d9864207de34e7633b23e048ee100401498255f4acf51ca917a6797a8696b37cf630602c31529d4ff241280f9d5e91b6e46555204e1eb730d
  ```
- **Verification:** MUST return `true` with Keypair A's public key.
- **Spec reference:** §1, §2

### SIG-02: KEY_ROTATE — dual signature

- **Description:** A KEY_ROTATE message is signed by the old key (envelope signature) and contains `new_key_signature` (payload signed by new key).
- **Keypairs:** Old = Keypair A, New = Keypair B
- **Payload (before `new_key_signature`):**
  ```json
  {"new_key":"7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674","old_key":"4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29"}
  ```
- **`new_key_signature`** (Keypair B signs the JCS of the payload above):
  ```
  5444bf8b821a9d112ac12184a771afc550f66f85362eb0e83e1a28bc5b915ffa5be3b09f92af6d23070a9a0a87548d1b3838045dff0a0f5c786b1313cf1da705
  ```
- **Complete signing body:**
  ```
  {"from":"4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29","payload":{"new_key":"7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674","new_key_signature":"5444bf8b821a9d112ac12184a771afc550f66f85362eb0e83e1a28bc5b915ffa5be3b09f92af6d23070a9a0a87548d1b3838045dff0a0f5c786b1313cf1da705","old_key":"4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29"},"timestamp":1700000000000,"type":"KEY_ROTATE"}
  ```
- **Envelope signature** (Keypair A signs the signing body):
  ```
  baeed04ce99c7f1e8d205c5d5e8f6804bb7361e7a95e76cd08b83b70fa6a61952351296556a1bf605ec5ed5ac3da1ca896d456e14e1154ea62260f3bad7ff008
  ```
- **Message `id`:** `2c6d430b9da6bd6def88237efedaede13ab8423955f2c9fda4642afaaf531ce7`
- **Verification:**
  1. Verify envelope `signature` against signing body using Keypair A's public key → `true`
  2. Verify `new_key_signature` against JCS of `{"new_key":"...","old_key":"..."}` using Keypair B's public key → `true`
- **Spec reference:** §1 — Key Rotation

---

## 4. VDF (§9)

### VDF-01: Iteration semantics (difficulty=10)

- **Description:** VDF computes iterated SHA-256 over the public key bytes.
- **Input:** Public key bytes (Keypair A): `4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29`
- **Iterations:**
  ```
  h[0]  = SHA-256(pubkey_bytes) = 4a67330b803d5c88757afb9328615344a89c49839a07f1f76887ad62d06a1f57
  h[1]  = SHA-256(h[0])         = d111c750e50313ec88ef003a46b2e75bb439740c9145e7c8871e0a92f2e4f71e
  h[2]  = SHA-256(h[1])         = 36bf77dd9b62b1f6de5d69e36716c317290f3ade523de1ff07aaed38e541bd3f
  h[3]  = SHA-256(h[2])         = 258f310ec5d92fe424cf05afb208ffa9161575d51940af6dfa3fb876930418b5
  h[4]  = SHA-256(h[3])         = 18a99c7ec36bd99d88cf34e6efa39ba2ce6304e8e94375796c43a1882a181536
  h[5]  = SHA-256(h[4])         = a02a25c25b7ea445d9f8e64d42e96c9e8e18cfe59b6b70aac4ece4b6033929ca
  h[6]  = SHA-256(h[5])         = 1c7fc0d6447423192601d6013088f1b9eb0a090dad041805d142731f09d5362a
  h[7]  = SHA-256(h[6])         = 7f66ccd286f0650d5b3160352bd9befe35337ea21287deb4a5a98ee2e1220682
  h[8]  = SHA-256(h[7])         = fa8f34e4825e5855ee980e87f790ec2be08195ff116aa863ee637a54086764a0
  h[9]  = SHA-256(h[8])         = 47d8b22fe73482b8c2fc6bef4d3a878f0b42ed1cd874de3cd1b086d7bc89e495
  h[10] = SHA-256(h[9])         = 449bcdcf7459a1bb9b5ad418b3c2f96623fbf19080d30bcacccb664bab6abaa3
  ```
- **Output:** `h[10]` = `449bcdcf7459a1bb9b5ad418b3c2f96623fbf19080d30bcacccb664bab6abaa3`
- **Spec reference:** §9 — VDF iteration semantics

### VDF-02: Checkpoint extraction

- **Description:** Checkpoints are at every `difficulty / 10` iterations. For difficulty=10, checkpoints are at iterations 1, 2, …, 10. For production difficulty=1,000,000, checkpoints are at 100000, 200000, …, 1000000.
- **Input:** Same as VDF-01, difficulty=10 (checkpoint interval = 1)
- **Checkpoints:**
  ```
  checkpoint[1]  = h[1]  = d111c750e50313ec88ef003a46b2e75bb439740c9145e7c8871e0a92f2e4f71e
  checkpoint[2]  = h[2]  = 36bf77dd9b62b1f6de5d69e36716c317290f3ade523de1ff07aaed38e541bd3f
  checkpoint[3]  = h[3]  = 258f310ec5d92fe424cf05afb208ffa9161575d51940af6dfa3fb876930418b5
  checkpoint[4]  = h[4]  = 18a99c7ec36bd99d88cf34e6efa39ba2ce6304e8e94375796c43a1882a181536
  checkpoint[5]  = h[5]  = a02a25c25b7ea445d9f8e64d42e96c9e8e18cfe59b6b70aac4ece4b6033929ca
  checkpoint[6]  = h[6]  = 1c7fc0d6447423192601d6013088f1b9eb0a090dad041805d142731f09d5362a
  checkpoint[7]  = h[7]  = 7f66ccd286f0650d5b3160352bd9befe35337ea21287deb4a5a98ee2e1220682
  checkpoint[8]  = h[8]  = fa8f34e4825e5855ee980e87f790ec2be08195ff116aa863ee637a54086764a0
  checkpoint[9]  = h[9]  = 47d8b22fe73482b8c2fc6bef4d3a878f0b42ed1cd874de3cd1b086d7bc89e495
  checkpoint[10] = h[10] = 449bcdcf7459a1bb9b5ad418b3c2f96623fbf19080d30bcacccb664bab6abaa3
  ```
- **Spec reference:** §9 — Intermediate checkpoints

### VDF-03: Segment verification

- **Description:** To verify segment between checkpoint[k] and checkpoint[k+1], recompute SHA-256 from checkpoint[k] and compare to checkpoint[k+1].
- **Input:**
  - checkpoint[3] = `258f310ec5d92fe424cf05afb208ffa9161575d51940af6dfa3fb876930418b5`
  - checkpoint[4] = `18a99c7ec36bd99d88cf34e6efa39ba2ce6304e8e94375796c43a1882a181536`
  - Segment length: 1 iteration (for difficulty=10)
- **Verification:** SHA-256(checkpoint[3]) = `18a99c7ec36bd99d88cf34e6efa39ba2ce6304e8e94375796c43a1882a181536` = checkpoint[4] ✓
- **Spec reference:** §9 — Verifiers MUST verify segments

---

## 5. Merkle Tree (§11)

### MERKLE-01: Empty proposal set

- **Description:** Empty active proposal set → Merkle root is SHA-256 of empty byte string.
- **Input:** No proposals
- **Expected root:** `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`
- **Spec reference:** §11 — Merkle Consistency

### MERKLE-02: Single proposal

- **Description:** Single proposal → leaf hash = SHA-256(id), root = leaf.
- **Input:** Proposal ID = `a1b2c3d4e5f6`
- **Leaf hash:** SHA-256("a1b2c3d4e5f6") = `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
- **Expected root:** `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
- **Spec reference:** §11

### MERKLE-03: Two proposals

- **Description:** Two proposals, sorted lexicographically by ID, root = SHA-256(left || right).
- **Input:** Proposal IDs = `["a1b2c3d4e5f6", "b2c3d4e5f6a1"]` (already sorted)
- **Leaves:**
  - L1 = SHA-256("a1b2c3d4e5f6") = `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
  - L2 = SHA-256("b2c3d4e5f6a1") = `fabd7d26ffca35f755c4f17604bacd2ca74638b836e0ccc8f7c06cb5bc9a0e5d`
- **Expected root:** SHA-256(L1 || L2) = `cd4eacb7493443618c0a6325db660723d010873c4e188e62eda660021b0de9a0`
- **Spec reference:** §11

### MERKLE-04: Three proposals (left-biased, odd count)

- **Description:** Three proposals. Left-biased: pair from left, last leaf promoted to next level.
- **Input:** Proposal IDs = `["a1b2c3d4e5f6", "b2c3d4e5f6a1", "c3d4e5f6a1b2"]`
- **Leaves:**
  - L1 = `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
  - L2 = `fabd7d26ffca35f755c4f17604bacd2ca74638b836e0ccc8f7c06cb5bc9a0e5d`
  - L3 = `3c0d93689d4786f99c82252a81c1abccf8b19041fd78c147ce7c1becb7b522e3`
- **Tree:**
  ```
          root
         /    \
      N12      L3
     /   \
   L1     L2
  ```
  - N12 = SHA-256(L1 || L2) = `cd4eacb7493443618c0a6325db660723d010873c4e188e62eda660021b0de9a0`
  - root = SHA-256(N12 || L3) = `e0b65593156cd8bd67f8c000a8bbd71fe708ec30b05a903adc64273c2c81a70e`
- **Expected root:** `e0b65593156cd8bd67f8c000a8bbd71fe708ec30b05a903adc64273c2c81a70e`
- **Spec reference:** §11

### MERKLE-05: Five proposals (full tree)

- **Description:** Five proposals. Left-biased binary tree with promotion at each level.
- **Input:** Proposal IDs = `["a1b2c3d4e5f6", "b2c3d4e5f6a1", "c3d4e5f6a1b2", "d4e5f6a1b2c3", "e5f6a1b2c3d4"]`
- **Leaves:**
  - L1 = `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
  - L2 = `fabd7d26ffca35f755c4f17604bacd2ca74638b836e0ccc8f7c06cb5bc9a0e5d`
  - L3 = `3c0d93689d4786f99c82252a81c1abccf8b19041fd78c147ce7c1becb7b522e3`
  - L4 = `1ba9aa1d2cfca27804fb599c6776b4874ec32c331f97ee657a1e5fd3bc7c9c98`
  - L5 = `26a4d81c78017de998d2842f17dae058b0a166e01941283cb002293e7cfdd131`
- **Tree:**
  ```
              root
             /    \
         N1234     L5
         /    \
      N12      N34
     /   \    /   \
   L1    L2  L3   L4
  ```
  - N12 = SHA-256(L1 || L2) = `cd4eacb7493443618c0a6325db660723d010873c4e188e62eda660021b0de9a0`
  - N34 = SHA-256(L3 || L4) = `2573b4a608a5323c614cc6bfe6915c7db10684e3a7d54b804a64d64c88706bdb`
  - N1234 = SHA-256(N12 || N34) = `101f70ccb36f106682a8797bd397407a27a979071e29e28354655150394e0efe`
  - root = SHA-256(N1234 || L5) = `d15072ca65d39c1f12e1f402c49eb9d2760d7838aad99f52801d58ac3bb8398d`
- **Expected root:** `d15072ca65d39c1f12e1f402c49eb9d2760d7838aad99f52801d58ac3bb8398d`
- **Spec reference:** §11

---

## 6. Manifest Hash (§6)

### MANIFEST-01: Shard manifest hash computation

- **Description:** Manifest hash = SHA-256 of shard hashes sorted lexicographically, concatenated without delimiters, with content_hash appended — all as UTF-8 bytes.
- **Input:**
  - `shard_hashes`: `["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc", "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"]`
  - `content_hash`: `"dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"`
- **Sorted shard hashes:**
  1. `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
  2. `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`
  3. `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc`
- **Concatenated input (UTF-8 string, 256 chars):**
  ```
  aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
  ```
- **Expected manifest_hash:** `648c162914266d5db16db4b797e2bdc6afe56646ea6984085ccd705276d95be4`
- **Spec reference:** §6 — Content Delivery, manifest_hash

---

## 7. Fixed-Point Reputation Formula (§8)

### REP-01: Standard reputation computation

- **Description:** Compute reputation with α=0.3, direct observation=0.5, two peer assessments.
- **Input (fixed-point ×10,000):**
  - α = 3000 (0.3 — i.e., 3 direct observations / 10)
  - direct = 5000 (0.5)
  - Peer 1: trust = 7000 (0.7), assessment = 6000 (0.6)
  - Peer 2: trust = 4000 (0.4), assessment = 8000 (0.8)
- **Computation:**
  ```
  peer_sum    = (7000 × 6000) + (4000 × 8000) = 42,000,000 + 32,000,000 = 74,000,000
  peer_weight = 7000 + 4000 = 11,000
  peer_avg    = 74,000,000 ÷ 11,000 = 6727  (integer division, truncated)
  result      = (3000 × 5000 + 7000 × 6727) ÷ 10,000
              = (15,000,000 + 47,089,000) ÷ 10,000
              = 62,089,000 ÷ 10,000
              = 6208  (truncated)
  ```
- **Expected result:** `6208` (represents 0.6208)
- **Spec reference:** §8 — Reputation Propagation, Evaluation order

### REP-02: α=0 — pure peer assessment, capped

- **Description:** With 0 direct observations, α=0. Peer-informed reputation is capped at 0.2 (2000) until assessments from ≥3 distinct ASN assessors are received.
- **Input:** Same peers as REP-01, but α = 0, fewer than 3 ASN-distinct assessors.
- **Computation:**
  ```
  peer_avg = 6727 (same as REP-01)
  result   = (0 × direct + 10000 × 6727) ÷ 10000 = 6727
  ```
  But capped at `2000` because < 3 distinct ASN assessors.
- **Expected result:** `2000` (capped at starting reputation)
- **Spec reference:** §8 — When α = 0

### REP-03: α=0 — uncapped with ≥3 ASN-distinct assessors

- **Description:** Same as REP-02 but with ≥3 assessors from distinct ASNs.
- **Expected result:** `6727` (uncapped)
- **Spec reference:** §8

### REP-04: α=0.6 — maximum direct weight

- **Description:** At 6+ direct observations, α = min(0.6, 6/10) = 0.6.
- **Input:** α = 6000, direct = 5000, same peers as REP-01.
- **Computation:**
  ```
  result = (6000 × 5000 + 4000 × 6727) ÷ 10000
         = (30,000,000 + 26,908,000) ÷ 10,000
         = 56,908,000 ÷ 10,000
         = 5690  (truncated)
  ```
- **Expected result:** `5690` (represents 0.5690)
- **Spec reference:** §8

---

## 8. Quorum Evaluation (§7)

### QUORUM-01: Standard quorum — ratified

- **Description:** Evaluate a proposal with 20 active nodes, total_rep = 5.2.
- **Input:**
  - active_nodes = 20
  - total_known_reputation = 52000 (5.2 × 10,000)
  - Endorse votes (5 voters): rep weights 5000, 4000, 3000, 6000, 7000 → weighted_endorse = 25000
  - Reject votes (1 voter): rep weight 3000 → weighted_reject = 3000
  - Total distinct voters: 6 (≥ 3 minimum ✓)
- **Computation:**
  ```
  quorum = max(active_nodes × 0.1, total_rep × 0.10)
         = max(20 × 1000, 52000 × 1000 ÷ 10000)
         = max(20000, 5200)
         = 20000  (represents 2.0)
  
  total_voter_rep = 25000 + 3000 = 28000 ≥ 20000  ✓ quorum met
  
  threshold = weighted_endorse / (weighted_endorse + weighted_reject)
            = 25000 × 10000 / 28000
            = 8928  (represents 0.8928)
  
  8928 ≥ 6700 (0.67)  ✓ threshold met
  ```
- **Expected result:** **Ratified** (quorum met, threshold met, ≥ 3 voters)
- **Spec reference:** §7

### QUORUM-02: Cold start — headcount mode

- **Description:** During first 30 days after governance activation, voting uses headcount mode.
- **Input:**
  - active_nodes = 20
  - 15 nodes endorse, 3 reject, 2 abstain
  - Total voters: 20 (majority of 20 = 11 must vote ✓, all 20 voted)
  - Endorse ratio: 15 / (15 + 3) = 0.833... ≥ 0.67 ✓
- **Expected result:** **Ratified** (headcount majority voted, >67% endorse)
- **Spec reference:** §7 — Cold Start

### QUORUM-03: Constitutional quorum

- **Description:** Constitutional proposals require 30% of total known reputation.
- **Input:**
  - total_known_reputation = 52000 (5.2)
  - Constitutional quorum = 52000 × 3000 / 10000 = 15600 (represents 1.56)
  - Total voter reputation: 16000 (1.6) ≥ 15600 ✓
  - Threshold: 0.90 required
- **Expected:** Quorum met if voter rep ≥ 15600 AND endorsement ratio ≥ 9000 (0.90)
- **Spec reference:** §7 — Constitutional quorum

### QUORUM-04: Cold-start proposal under weighted voting

- **Description:** A proposal submitted during cold start whose deadline falls after cold start ends is evaluated under weighted voting, but MUST additionally require ≥50% of active nodes to have voted.
- **Input:**
  - active_nodes = 20
  - 8 nodes voted (< 50% of 20)
  - Weighted endorsement meets threshold
  - Quorum met by weight
- **Expected result:** **Not ratified** (50% headcount floor not met: 8 < 10)
- **Spec reference:** §7 — Cold Start

---

## 9. Activity Multiplier (§7)

### ACT-01: High activity

- **Description:** Compute activity multiplier and sustain period.
- **Input:**
  - adopted_proposals = 2
  - earned_rep_delta = 0.3 (3000 fixed-point, but used as raw 0.3 in formula)
  - challenge_pairs = 5
- **Computation:**
  ```
  raw = adopted_proposals × 0.33 + earned_rep_delta × 2.0 + challenge_pairs × 0.1
      = 2 × 0.33 + 0.3 × 2.0 + 5 × 0.1
      = 0.66 + 0.60 + 0.50
      = 1.76
  
  activity_multiplier = 1.0 + min(1.33, 1.76) = 1.0 + 1.33 = 2.33
  
  sustain_days = max(3, 7 / 2.33) = max(3, 3.00...) = 3
  ```
- **Expected multiplier:** 2.33
- **Expected sustain_days:** 3
- **Spec reference:** §7 — Network Phases

### ACT-02: Zero activity

- **Description:** No activity produces minimum multiplier.
- **Input:** adopted_proposals = 0, earned_rep_delta = 0.0, challenge_pairs = 0
- **Computation:**
  ```
  raw = 0
  activity_multiplier = 1.0 + min(1.33, 0) = 1.0
  sustain_days = max(3, 7 / 1.0) = max(3, 7) = 7
  ```
- **Expected multiplier:** 1.0
- **Expected sustain_days:** 7
- **Spec reference:** §7

---

## 10. Message Validation (§2, §5, §13)

### VALID-01: Valid message — accepted

- **Description:** A correctly formed, signed, timestamped message MUST be accepted and propagated.
- **Input:** Complete envelope with valid signature, timestamp within ±5 min, payload < 8 MiB, known type.
- **Expected:** Accept and propagate.
- **Spec reference:** §2, §5

### VALID-02: Timestamp > 5 minutes in the future — rejected

- **Description:** Messages with future timestamps beyond tolerance MUST be rejected.
- **Input:** `timestamp` = current_time + 301,000 ms (5 min + 1 sec)
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §2 — Timestamp Validation

### VALID-03: Timestamp > 5 minutes in the past — rejected

- **Description:** Messages with past timestamps beyond tolerance MUST be rejected.
- **Input:** `timestamp` = current_time - 301,000 ms (5 min + 1 sec)
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §2 — Timestamp Validation

### VALID-04: Payload > 8 MiB — rejected

- **Description:** Messages exceeding maximum payload size MUST be rejected.
- **Input:** Message with payload > 8,388,608 bytes
- **Expected:** MUST reject.
- **Spec reference:** §2 — Wire Format

### VALID-05: Unknown message type — propagate with rate limit

- **Description:** Messages with unrecognized type SHOULD be propagated for forward compatibility, subject to 10/sender/hour rate limit.
- **Input:** Message with `"type": "FUTURE_TYPE"`, valid signature
- **Expected:** Propagate (if under rate limit). Do not process.
- **Spec reference:** §13 — Unknown Message Types

### VALID-06: Unknown type — rate limited

- **Description:** 11th unknown-type message from same sender within an hour MUST be dropped.
- **Input:** 11th message with unknown type from same `from` key within 1 hour
- **Expected:** MUST NOT propagate (exceeds 10/sender/hour limit).
- **Spec reference:** §13

### VALID-07: Missing signature — rejected

- **Description:** Messages without a signature MUST be rejected.
- **Input:** Envelope with `signature` field missing or empty
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §13 — Malformed Messages

### VALID-08: Invalid signature — rejected

- **Description:** Messages with a signature that doesn't verify MUST be rejected.
- **Input:** Envelope with valid structure but signature from a different key
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §13 — Malformed Messages

### VALID-09: Gossip message > 24 hours old — rejected

- **Description:** Messages received via GossipSub with timestamps older than 24 hours MUST be rejected.
- **Input:** GossipSub message with `timestamp` = current_time - 86,400,001 ms (24h + 1ms)
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §5 — Time-based rejection

### VALID-10: Sync message > 24 hours old — accepted

- **Description:** The 24-hour rule does NOT apply to messages received via `/valence/sync/1.0.0`.
- **Input:** Sync response containing message with `timestamp` = current_time - 86,400,001 ms
- **Expected:** MUST accept (sync is for historical messages).
- **Spec reference:** §5 — Time-based rejection

---

## 11. Storage Challenge (§6)

### STORAGE-01: Adjacency proof computation

- **Description:** Compute a storage proof from shard data, nonce, offset, direction, and window size.
- **Input:**
  - Shard data (hex): `48656c6c6f2c20746869732069732074657374207368617264206461746120666f72207468652056616c656e6365204e6574776f726b20636f6e666f726d616e636520746573742073756974652e`
  - Shard data (ASCII): `Hello, this is test shard data for the Valence Network conformance test suite.`
  - challenge_nonce: `abababababababababababababababababababababababababababababababab`
  - offset: 10
  - direction: `after`
  - window_size: 16
- **Window extraction:** bytes[10..26] = `73206973207465737420736861726420` (ASCII: `s is test shard `)
- **Proof computation:**
  ```
  proof_hash = SHA-256(challenge_nonce_bytes || window_bytes)
  ```
  - challenge_nonce_bytes: 32 bytes (0xAB repeated)
  - window_bytes: 16 bytes
  - Input to SHA-256: 48 bytes total
- **Expected proof_hash:** `465649c01a2a55d6cd5bd299471deb0999ce582ce3ce86098c0aca65a4094f26`
- **Verification:** Challenger recomputes the same hash from their own copy of the shard and compares.
- **Spec reference:** §6 — Storage challenges

---

## 12. Reputation Bounds (§8)

### BOUNDS-01: New node starts at 0.2

- **Description:** All new nodes begin with reputation 0.2 (2000 fixed-point).
- **Expected:** Initial `overall` = 2000
- **Spec reference:** §8 — Initial Score

### BOUNDS-02: Hard cap at 1.0

- **Description:** Reputation cannot exceed 1.0 (10000 fixed-point).
- **Input:** Node at reputation 9900 earns +200
- **Expected:** Reputation = 10000 (capped, not 10100)
- **Spec reference:** §8 — Score

### BOUNDS-03: Floor at 0.1

- **Description:** Reputation cannot drop below 0.1 (1000 fixed-point).
- **Input:** Node at reputation 1100 receives -200 penalty
- **Expected:** Reputation = 1000 (floored, not 900)
- **Spec reference:** §8 — Floor

### BOUNDS-04: Daily velocity cap above 0.2

- **Description:** Daily gain capped at 0.02 (200 fixed-point) when above starting score.
- **Input:** Node at reputation 3000 (0.3) earns +500 in one day
- **Expected:** Gain capped to 200; reputation = 3200
- **Spec reference:** §8 — Velocity Limits

### BOUNDS-05: Recovery below 0.2 — uncapped

- **Description:** A penalized node below 0.2 can recover without velocity limits until reaching 0.2.
- **Input:** Node at reputation 1200 (0.12) earns +500 in one day
- **Expected:** Reputation = 1700 (no cap, still below 2000)
- **Spec reference:** §8 — Velocity Limits

### BOUNDS-06: Recovery crosses 0.2 boundary

- **Description:** When recovery crosses 0.2, the gain up to 0.2 is uncapped; velocity limits apply only to the portion above 0.2.
- **Input:** Node at reputation 1800 earns +500 in one day
- **Expected:** Uncapped up to 2000 (+200), then capped at +200 above 2000. Total = 2200.
- **Spec reference:** §8 — Velocity Limits

---

## 13. Proposal Rate Limiting (§6)

### RATE-01: Node at starting reputation can propose

- **Description:** Nodes with reputation ≥ 0.2 MAY submit proposals.
- **Input:** Node at reputation 2000 (0.2), 0 proposals in last 7 days
- **Expected:** Proposal accepted.
- **Spec reference:** §6 — Rate limiting

### RATE-02: Node below starting reputation — rejected

- **Description:** Nodes with reputation < 0.2 MUST NOT be allowed to propose.
- **Input:** Node at reputation 1900 (0.19)
- **Expected:** MUST reject proposal.
- **Spec reference:** §6 — Rate limiting

### RATE-03: 4th proposal in 7 days — rejected

- **Description:** Maximum 3 proposals per 7-day rolling window.
- **Input:** Node at reputation 5000, 3 proposals already submitted in last 7 days
- **Expected:** 4th proposal MUST be rejected.
- **Spec reference:** §6 — Rate limiting

### RATE-04: Rate limits transfer across KEY_ROTATE

- **Description:** Proposal rate limits transfer to the new key after key rotation.
- **Input:** Old key submitted 2 proposals, then KEY_ROTATE to new key
- **Expected:** New key can submit 1 more proposal in the rolling window. 2nd proposal from new key MUST be rejected.
- **Spec reference:** §6 — Rate limiting, §1 — Tenure continuity

---

## 14. Identity Linking (§1)

### LINK-01: DID_LINK happy path

- **Description:** Root (Keypair A) links child (Keypair B). Envelope signed by root, `child_signature` proves child consent.
- **Input:**
  - Root = Keypair A (`4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29`)
  - Child = Keypair B (`7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674`)
  - `child_signature` signing input: `root_key || child_key` as raw bytes (64 bytes total):
    ```
    4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba297422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674
    ```
  - `child_signature` (Keypair B signs the above):
    ```
    c3432f9706d926e7312c4b25555306e85db640dfa38ec8bbe453a5b99fd98d17c8df0d34622d8bed3bf03a1ca3b15dc135d45abf3d34db4153ca15fecc0a920b
    ```
  - Payload:
    ```json
    {
      "child_key": "7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674",
      "child_signature": "c3432f9706d926e7312c4b25555306e85db640dfa38ec8bbe453a5b99fd98d17c8df0d34622d8bed3bf03a1ca3b15dc135d45abf3d34db4153ca15fecc0a920b",
      "root_key": "4cb5abf6ad79fbf5abbccafcc269d85cd2651ed4b885b5869f241aedf0a5ba29"
    }
    ```
  - Envelope `from` = Keypair A public key, `type` = `"DID_LINK"`
- **Expected:**
  1. Verify envelope signature against Keypair A → `true`
  2. Verify `child_signature` against `root_key || child_key` using Keypair B → `true`
  3. Link accepted. Keypair B is now an authorized key for Keypair A's identity.
- **Spec reference:** §1 — DID_LINK

### LINK-02: DID_LINK conflict — same child, two roots

- **Description:** Second DID_LINK claiming the same child key for a different root MUST be rejected (first-seen wins).
- **Input:**
  - First DID_LINK: root=A, child=B (already processed, as in LINK-01)
  - Second DID_LINK: root=C (some other keypair), child=B, with valid child_signature from B
- **Expected:** Second DID_LINK MUST be rejected. Keypair B remains linked to A.
- **Spec reference:** §1 — "First valid DID_LINK for a given child key wins"

### LINK-03: DID_LINK — child is already a root

- **Description:** A key that is already a root of another identity MUST NOT be linked as a child.
- **Input:**
  - Keypair A is a root identity (has linked children or simply exists as a registered node)
  - DID_LINK: root=C, child=A, with valid child_signature from A
- **Expected:** MUST be rejected. A root key cannot become a child.
- **Spec reference:** §1 — "A child key MUST NOT also be a root key of another identity"

### LINK-04: DID_REVOKE happy path

- **Description:** Root revokes a child key. Messages from the revoked key are rejected going forward.
- **Input:**
  - Identity: root=A, child=B (linked via LINK-01)
  - DID_REVOKE signed by A: `revoked_key` = B
  - Subsequently, B sends a VOTE message
- **Expected:**
  1. DID_REVOKE accepted.
  2. VOTE from B MUST be rejected (B is no longer an authorized key).
  3. B is permanently deauthorized from A's identity.
- **Spec reference:** §1 — DID_REVOKE

### LINK-05: DID_REVOKE — non-child key

- **Description:** Root attempts to revoke a key that is not one of its authorized children.
- **Input:**
  - Identity: root=A, child=B
  - DID_REVOKE signed by A: `revoked_key` = C (not a child of A)
- **Expected:** MUST be rejected. Nodes MUST NOT process DID_REVOKE where `revoked_key` is not a child of the signing root. No effect on C.
- **Spec reference:** §1 — DID_REVOKE ("Nodes MUST reject DID_REVOKE messages where `revoked_key` is not a child of the signing root")

### LINK-06: DID_REVOKE — root revokes itself

- **Description:** Root attempts to revoke itself.
- **Input:**
  - DID_REVOKE signed by A: `revoked_key` = A (same as root_key)
- **Expected:** MUST be rejected. Root cannot revoke itself. Use KEY_ROTATE for root compromise.
- **Spec reference:** §1 — "The root key MUST NOT revoke itself"

### LINK-07: Vote attribution

- **Description:** Child votes on a proposal; vote is attributed to root identity. A subsequent vote from root supersedes.
- **Input:**
  1. Identity: root=A, child=B
  2. B sends VOTE(proposal_id=P, stance=endorse)
  3. A sends VOTE(proposal_id=P, stance=reject)
- **Expected:**
  1. B's vote is attributed to root identity A.
  2. A's subsequent vote supersedes B's vote.
  3. Identity A's effective vote on P = reject.
  4. Only 1 vote counted for identity A (not 2 distinct votes).
- **Spec reference:** §1 — "Single vote per proposal", §7 — Vote Rules

### LINK-08: Proposal rate limit per identity

- **Description:** The 3-proposals-per-7-days limit applies across all keys in an identity.
- **Input:**
  1. Identity: root=A, children=B, C
  2. B submits PROPOSE (1 of 3)
  3. A submits PROPOSE (2 of 3)
  4. C submits PROPOSE (3 of 3)
  5. B submits PROPOSE (4th attempt)
- **Expected:** 4th proposal MUST be rejected. The identity has exhausted its 3-proposal allowance.
- **Spec reference:** §1 — "Single proposal rate limit", §6 — Rate limiting

### LINK-09: Collusion exemption

- **Description:** Keys within the same identity with 100% vote correlation MUST NOT trigger collusion detection.
- **Input:**
  - Identity: root=A, child=B
  - A and B vote identically on 25 consecutive proposals (well above the 95%/20+ threshold)
- **Expected:** No collusion flag. Keys in the same identity are exempt from vote correlation and VDF timing correlation analysis.
- **Spec reference:** §1 — "Collusion detection exemption", §10 — Collusion Detection

### LINK-10: Headcount counting

- **Description:** An identity with multiple authorized keys counts as 1 for minimum voter and cold start headcount.
- **Input:**
  - Identity: root=A, children=B, C, D (4 keys total, 1 identity)
  - 2 other independent identities (E and F)
  - All vote on proposal P during cold start
  - active_nodes = 6 (A, B, C, D, E, F as connected peers)
- **Expected:**
  - Distinct voter count = 3 (identity A, identity E, identity F) — meets minimum voter threshold of 3
  - For cold start headcount: 3 identities voted, not 6 nodes
  - Identity A counts as 1 regardless of how many of its keys (A, B, C, D) cast votes
- **Spec reference:** §1 — "Quorum and headcount", §7 — Minimum voters, Cold Start

### LINK-11: KEY_ROTATE by root — new root inherits authorized keys

- **Description:** When a root key rotates, the new root inherits all authorized children.
- **Input:**
  1. Identity: root=A, child=B
  2. A performs KEY_ROTATE to A' (new keypair)
  3. B sends a VOTE message
- **Expected:**
  1. KEY_ROTATE accepted (standard dual-signature verification).
  2. B remains an authorized key under root A'.
  3. B's vote is attributed to identity A' (the new root).
  4. No DID_LINK re-signing required.
- **Spec reference:** §1 — "KEY_ROTATE Interaction"

### LINK-12: Re-registration after revocation

- **Description:** A revoked key re-registers independently at starting reputation; it cannot rejoin any identity.
- **Input:**
  1. Identity: root=A, child=B
  2. A revokes B via DID_REVOKE
  3. B computes new VDF proof and re-registers as independent node
  4. C attempts DID_LINK: root=C, child=B
- **Expected:**
  1. B re-registers successfully with reputation 0.2 (2000 fixed-point).
  2. B has no connection to identity A — A's reputation is unaffected.
  3. DID_LINK(root=C, child=B) MUST be rejected — revoked keys cannot be re-linked to any identity.
  4. B operates as an independent root identity only.
- **Spec reference:** §1 — "After revocation"

### LINK-13: Child KEY_ROTATE blocks hostile DID_LINK

- **Description:** When a child key rotates, the new key is implicitly linked — an attacker cannot claim it via DID_LINK.
- **Input:**
  1. Identity: root=A, child=B
  2. B performs KEY_ROTATE to B' (new keypair, dual-signed by B and B')
  3. Attacker E submits DID_LINK: root=E, child=B'
- **Expected:**
  1. KEY_ROTATE accepted — B' inherits B's authorized status under root A.
  2. B' is treated as linked to A (no explicit DID_LINK required).
  3. DID_LINK(root=E, child=B') MUST be rejected — B' is already an authorized key via KEY_ROTATE chain.
  4. Messages signed by B' are attributed to identity A.
- **Spec reference:** §1 — "KEY_ROTATE Interaction": "A key that became an authorized key via KEY_ROTATE of an existing child MUST be treated as linked"

### LINK-14: WITHDRAW by sibling key in same identity

- **Description:** Any key in an identity can withdraw a proposal authored by another key in the same identity.
- **Input:**
  1. Identity: root=A, child=B, child=C
  2. B submits PROPOSE (from=B, proposal_id=P1)
  3. C submits WITHDRAW (from=C, proposal_id=P1)
  4. Unrelated node D submits WITHDRAW (from=D, proposal_id=P1)
- **Expected:**
  1. PROPOSE accepted — B is authorized to propose as identity A.
  2. WITHDRAW from C accepted — C is in the same identity as B (both authorized under root A).
  3. Votes for P1 stop being counted after C's WITHDRAW.
  4. WITHDRAW from D rejected — D is not in identity A.
- **Spec reference:** §6 — WITHDRAW: "For proposals authored by an authorized key (§1 Identity Linking), any key in the same identity (including root) MAY withdraw."
