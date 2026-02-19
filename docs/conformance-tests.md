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

## 4. VDF (§10)

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
- **Spec reference:** §10 — VDF iteration semantics

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
- **Spec reference:** §10 — Intermediate checkpoints

### VDF-03: Segment verification

- **Description:** To verify segment between checkpoint[k] and checkpoint[k+1], recompute SHA-256 from checkpoint[k] and compare to checkpoint[k+1].
- **Input:**
  - checkpoint[3] = `258f310ec5d92fe424cf05afb208ffa9161575d51940af6dfa3fb876930418b5`
  - checkpoint[4] = `18a99c7ec36bd99d88cf34e6efa39ba2ce6304e8e94375796c43a1882a181536`
  - Segment length: 1 iteration (for difficulty=10)
- **Verification:** SHA-256(checkpoint[3]) = `18a99c7ec36bd99d88cf34e6efa39ba2ce6304e8e94375796c43a1882a181536` = checkpoint[4] ✓
- **Spec reference:** §10 — Verifiers MUST verify segments

---

## 5. Merkle Tree (§12)

### MERKLE-01: Empty proposal set

- **Description:** Empty active proposal set → Merkle root is SHA-256 of empty byte string.
- **Input:** No proposals
- **Expected root:** `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`
- **Spec reference:** §12 — Merkle Consistency

### MERKLE-02: Single proposal

- **Description:** Single proposal → leaf hash = SHA-256(id), root = leaf.
- **Input:** Proposal ID = `a1b2c3d4e5f6`
- **Leaf hash:** SHA-256("a1b2c3d4e5f6") = `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
- **Expected root:** `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
- **Spec reference:** §12

### MERKLE-03: Two proposals

- **Description:** Two proposals, sorted lexicographically by ID, root = SHA-256(left || right).
- **Input:** Proposal IDs = `["a1b2c3d4e5f6", "b2c3d4e5f6a1"]` (already sorted)
- **Leaves:**
  - L1 = SHA-256("a1b2c3d4e5f6") = `bde81e9384b7848e57951ec32c7344459233235bfa519d7396ae3406014a06f4`
  - L2 = SHA-256("b2c3d4e5f6a1") = `fabd7d26ffca35f755c4f17604bacd2ca74638b836e0ccc8f7c06cb5bc9a0e5d`
- **Expected root:** SHA-256(L1 || L2) = `cd4eacb7493443618c0a6325db660723d010873c4e188e62eda660021b0de9a0`
- **Spec reference:** §12

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
- **Spec reference:** §12

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
- **Spec reference:** §12

---

## 6. Manifest Hash (§6)

### MANIFEST-01: Shard manifest hash computation

- **Description:** Manifest hash = SHA-256 of: shard_count as 4-byte big-endian integer, then shard hashes sorted lexicographically as hex strings concatenated without delimiters, then content_hash hex string — all as UTF-8 bytes except shard_count which is raw bytes.
- **Input:**
  - `shard_hashes`: `["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc", "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"]`
  - `content_hash`: `"dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd"`
  - `shard_count`: 3
- **Sorted shard hashes:**
  1. `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
  2. `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`
  3. `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc`
- **Concatenated input (260 bytes total):**
  ```
  [0x00, 0x00, 0x00, 0x03] (shard_count as 4-byte BE u32)
  followed by UTF-8 bytes of:
  aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
  ```
- **Expected manifest_hash:** `36edd51a274af1b3411c0eaad5b414de7c492b665a2e194a13a0e6dea7370e86`
- **Spec reference:** §6 — Content Delivery, manifest_hash

---

## 7. Fixed-Point Reputation Formula (§9)

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
- **Spec reference:** §9 — Reputation Propagation, Evaluation order

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
- **Spec reference:** §9 — When α = 0

### REP-03: α=0 — uncapped with ≥3 ASN-distinct assessors

- **Description:** Same as REP-02 but with ≥3 assessors from distinct ASNs.
- **Expected result:** `6727` (uncapped)
- **Spec reference:** §9

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
- **Spec reference:** §9

### REP-05: Gain dampening for multi-key identity

- **Description:** Reputation gains are dampened by authorized key count using `effective_gain = raw_gain / authorized_key_count^0.75`.
- **Input (fixed-point ×10,000):**
  - raw_gain = 100 (represents 0.01)
  - authorized_key_count = 2
- **Computation (2 keys):**
  ```
  dampening_factor = 2^0.75 = 1.6818
  fixed_point_factor = 16818
  effective_gain = raw_gain × 10000 / 16818 = 100 × 10000 / 16818 = 59 (truncated)
  ```
- **Expected result (2 keys):** `59` (represents 0.0059)
- **Input (4 keys):**
  ```
  dampening_factor = 4^0.75 = 2.8284
  fixed_point_factor = 28284
  effective_gain = 100 × 10000 / 28284 = 35 (truncated)
  ```
- **Expected result (4 keys):** `35` (represents 0.0035)
- **Spec reference:** §1 — Gain dampening, §9

### REP-06: Velocity limits at 0.2 boundary

- **Description:** Recovery below 0.2 is uncapped. Above 0.2, daily velocity cap of 0.02 applies. When a gain crosses the boundary, the portion up to 0.2 is uncapped and velocity limits apply only above 0.2.
- **Input:**
  - Current reputation: 1800 (0.18)
  - Earned gain in one day: 500 (0.05)
- **Computation:**
  ```
  uncapped_portion = 2000 - 1800 = 200 (up to 0.2, uncapped)
  remaining_gain = 500 - 200 = 300
  capped_portion = min(300, 200) = 200 (daily cap of 0.02 = 200 above 0.2)
  total_gain = 200 + 200 = 400
  new_reputation = 1800 + 400 = 2200
  ```
- **Expected result:** `2200` (represents 0.22)
- **Spec reference:** §9 — Velocity Limits

---

## 8. Quorum Evaluation (§8)

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
- **Spec reference:** §8

### QUORUM-02: Cold start — headcount mode

- **Description:** During first 30 days after governance activation, voting uses headcount mode.
- **Input:**
  - active_nodes = 20
  - 15 nodes endorse, 3 reject, 2 abstain
  - Total voters: 20 (majority of 20 = 11 must vote ✓, all 20 voted)
  - Endorse ratio: 15 / (15 + 3) = 0.833... ≥ 0.67 ✓
- **Expected result:** **Ratified** (headcount majority voted, >67% endorse)
- **Spec reference:** §8 — Cold Start

### QUORUM-03: Constitutional quorum

- **Description:** Constitutional proposals require 30% of total known reputation.
- **Input:**
  - total_known_reputation = 52000 (5.2)
  - Constitutional quorum = 52000 × 3000 / 10000 = 15600 (represents 1.56)
  - Total voter reputation: 16000 (1.6) ≥ 15600 ✓
  - Threshold: 0.90 required
- **Expected:** Quorum met if voter rep ≥ 15600 AND endorsement ratio ≥ 9000 (0.90)
- **Spec reference:** §8 — Constitutional quorum

### QUORUM-04: Cold-start proposal under weighted voting

- **Description:** A proposal submitted during cold start whose deadline falls after cold start ends is evaluated under weighted voting, but MUST additionally require ≥50% of active nodes to have voted.
- **Input:**
  - active_nodes = 20
  - 8 nodes voted (< 50% of 20)
  - Weighted endorsement meets threshold
  - Quorum met by weight
- **Expected result:** **Not ratified** (50% headcount floor not met: 8 < 10)
- **Spec reference:** §8 — Cold Start

---

## 9. Activity Multiplier (§8)

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
- **Spec reference:** §8 — Network Phases

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
- **Spec reference:** §8

---

## 10. Message Validation (§2, §5, §14)

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
- **Spec reference:** §14 — Unknown Message Types

### VALID-06: Unknown type — rate limited

- **Description:** 11th unknown-type message from same sender within an hour MUST be dropped.
- **Input:** 11th message with unknown type from same `from` key within 1 hour
- **Expected:** MUST NOT propagate (exceeds 10/sender/hour limit).
- **Spec reference:** §14

### VALID-07: Missing signature — rejected

- **Description:** Messages without a signature MUST be rejected.
- **Input:** Envelope with `signature` field missing or empty
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §14 — Malformed Messages

### VALID-08: Invalid signature — rejected

- **Description:** Messages with a signature that doesn't verify MUST be rejected.
- **Input:** Envelope with valid structure but signature from a different key
- **Expected:** MUST reject. MUST NOT propagate.
- **Spec reference:** §14 — Malformed Messages

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

## 12. Reputation Bounds (§9)

### BOUNDS-01: New node starts at 0.2

- **Description:** All new nodes begin with reputation 0.2 (2000 fixed-point).
- **Expected:** Initial `overall` = 2000
- **Spec reference:** §9 — Initial Score

### BOUNDS-02: Hard cap at 1.0

- **Description:** Reputation cannot exceed 1.0 (10000 fixed-point).
- **Input:** Node at reputation 9900 earns +200
- **Expected:** Reputation = 10000 (capped, not 10100)
- **Spec reference:** §9 — Score

### BOUNDS-03: Floor at 0.1

- **Description:** Reputation cannot drop below 0.1 (1000 fixed-point).
- **Input:** Node at reputation 1100 receives -200 penalty
- **Expected:** Reputation = 1000 (floored, not 900)
- **Spec reference:** §9 — Floor

### BOUNDS-04: Daily velocity cap above 0.2

- **Description:** Daily gain capped at 0.02 (200 fixed-point) when above starting score.
- **Input:** Node at reputation 3000 (0.3) earns +500 in one day
- **Expected:** Gain capped to 200; reputation = 3200
- **Spec reference:** §9 — Velocity Limits

### BOUNDS-05: Recovery below 0.2 — uncapped

- **Description:** A penalized node below 0.2 can recover without velocity limits until reaching 0.2.
- **Input:** Node at reputation 1200 (0.12) earns +500 in one day
- **Expected:** Reputation = 1700 (no cap, still below 2000)
- **Spec reference:** §9 — Velocity Limits

### BOUNDS-06: Recovery crosses 0.2 boundary

- **Description:** When recovery crosses 0.2, the gain up to 0.2 is uncapped; velocity limits apply only to the portion above 0.2.
- **Input:** Node at reputation 1800 earns +500 in one day
- **Expected:** Uncapped up to 2000 (+200), then capped at +200 above 2000. Total = 2200.
- **Spec reference:** §9 — Velocity Limits

---

## 13. Proposal Rate Limiting (§7)

### RATE-01: Node at minimum reputation can propose

- **Description:** Nodes with reputation ≥ 0.3 MAY submit proposals.
- **Input:** Node at reputation 3000 (0.3), 0 proposals in last 7 days
- **Expected:** Proposal accepted.
- **Spec reference:** §7 — Rate limiting

### RATE-02: Node below minimum reputation — rejected

- **Description:** Nodes with reputation < 0.3 MUST NOT be allowed to propose.
- **Input:** Node at reputation 2900 (0.29)
- **Expected:** MUST reject proposal.
- **Spec reference:** §7 — Rate limiting

### RATE-03: 4th proposal in 7 days — rejected

- **Description:** Maximum 3 proposals per 7-day rolling window.
- **Input:** Node at reputation 5000, 3 proposals already submitted in last 7 days
- **Expected:** 4th proposal MUST be rejected.
- **Spec reference:** §7 — Rate limiting

### RATE-04: Rate limits transfer across KEY_ROTATE

- **Description:** Proposal rate limits transfer to the new key after key rotation.
- **Input:** Old key submitted 2 proposals, then KEY_ROTATE to new key
- **Expected:** New key can submit 1 more proposal in the rolling window. 2nd proposal from new key MUST be rejected.
- **Spec reference:** §7 — Rate limiting, §1 — Tenure continuity

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
- **Spec reference:** §1 — "Single vote per proposal", §8 — Vote Rules

### LINK-08: Proposal rate limit per identity

- **Description:** The 3-proposals-per-7-days limit applies across all keys in an identity.
- **Input:**
  1. Identity: root=A, children=B, C
  2. B submits PROPOSE (1 of 3)
  3. A submits PROPOSE (2 of 3)
  4. C submits PROPOSE (3 of 3)
  5. B submits PROPOSE (4th attempt)
- **Expected:** 4th proposal MUST be rejected. The identity has exhausted its 3-proposal allowance.
- **Spec reference:** §1 — "Single proposal rate limit", §7 — Rate limiting

### LINK-09: Collusion exemption

- **Description:** Keys within the same identity with 100% vote correlation MUST NOT trigger collusion detection.
- **Input:**
  - Identity: root=A, child=B
  - A and B vote identically on 25 consecutive proposals (well above the 95%/20+ threshold)
- **Expected:** No collusion flag. Keys in the same identity are exempt from vote correlation and VDF timing correlation analysis.
- **Spec reference:** §1 — "Collusion detection exemption", §11 — Collusion Detection

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
- **Spec reference:** §1 — "Quorum and headcount", §8 — Minimum voters, Cold Start

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
- **Spec reference:** §7 — WITHDRAW: "For proposals authored by an authorized key (§1 Identity Linking), any key in the same identity (including root) MAY withdraw."

---

## 15. Content Economics (§6)

### CONTENT-01: Scarcity multiplier computation

- **Description:** Compute the scarcity multiplier from network utilization using `scarcity_multiplier = 1 + 99 × network_utilization^4`.
- **Input → Expected output (fixed-point ×10,000):**
  - 0% utilization: `1 + 99 × 0^4 = 1.0` → `10000`
  - 20% utilization: `1 + 99 × 0.2^4 = 1 + 99 × 0.0016 = 1.1584` → `11584`
  - 50% utilization: `1 + 99 × 0.5^4 = 1 + 99 × 0.0625 = 7.1875` → `71875`
  - 80% utilization: `1 + 99 × 0.8^4 = 1 + 99 × 0.4096 = 41.5504` → `415504`
  - 100% utilization: `1 + 99 × 1^4 = 100.0` → `1000000`
  - `total_allocated = 0` (no storage nodes): multiplier = `1.0` → `10000`
- **Spec reference:** §6 — Network Capacity and Pricing

### CONTENT-02: Monthly rent calculation

- **Description:** Compute monthly rent for replicated content using `monthly_rent = content_size_mib × base_rate × scarcity_multiplier`.
- **Input:**
  - Content size: 10 MiB
  - Scarcity multiplier: 7.1875 (50% utilization)
  - Base rate: 0.001 per MiB per month
- **Computation:**
  ```
  rent = 10 × 0.001 × 7.1875 = 0.071875
  ```
- **Expected result (fixed-point ×10,000):** `718` (truncated from 718.75)
- **Spec reference:** §6 — Replication Rent

### CONTENT-03: Rent convergence — normal (≤10× divergence)

- **Description:** Rent price locking converges toward current market rate at 20% per cycle using `effective = locked + (current - locked) × min(1, cycles_elapsed / 5)`.
- **Input (fixed-point ×10,000):**
  - Locked multiplier: 10000 (1.0)
  - Current multiplier: 500000 (50.0)
  - Divergence: 50× (≤10×? No — but per the spec the 10× threshold triggers acceleration. Let's test normal convergence first with a ≤10× case.)
- **Revised input for normal convergence:**
  - Locked multiplier: 10000 (1.0)
  - Current multiplier: 50000 (5.0) — divergence = 5× (≤10×, normal rate)
- **Computation:**
  ```
  Cycle 1: factor = min(10000, 1 × 10000 / 5) = 2000
           effective = 10000 + (50000 - 10000) × 2000 / 10000
                     = 10000 + 40000 × 2000 / 10000
                     = 10000 + 8000 = 18000
  
  Cycle 3: factor = min(10000, 3 × 10000 / 5) = 6000
           effective = 10000 + 40000 × 6000 / 10000
                     = 10000 + 24000 = 34000
  
  Cycle 5: factor = min(10000, 5 × 10000 / 5) = 10000
           effective = 10000 + 40000 × 10000 / 10000
                     = 10000 + 40000 = 50000 (fully converged)
  ```
- **Expected results:**
  - Cycle 1: `18000` (1.8)
  - Cycle 3: `34000` (3.4)
  - Cycle 5: `50000` (5.0, fully converged)
- **Spec reference:** §6 — Rent price locking with convergence

### CONTENT-04: Rent convergence with acceleration (>10× divergence)

- **Description:** When current multiplier exceeds locked by more than 10×, convergence accelerates to 30% per cycle (`cycles_elapsed × 3 / 10`).
- **Input (fixed-point ×10,000):**
  - Locked multiplier: 10000 (1.0)
  - Current multiplier: 1000000 (100.0) — divergence = 100× (>10×, accelerated)
- **Computation:**
  ```
  Cycle 1: factor = min(10000, 1 × 3 × 10000 / 10) = min(10000, 3000) = 3000
           effective = 10000 + (1000000 - 10000) × 3000 / 10000
                     = 10000 + 990000 × 3000 / 10000
                     = 10000 + 297000 = 307000
  
  Cycle 4: factor = min(10000, 4 × 3 × 10000 / 10) = min(10000, 12000) = 10000 (capped)
           effective = 10000 + 990000 × 10000 / 10000
                     = 10000 + 990000 = 1000000 (fully converged)
  ```
- **Expected results:**
  - Cycle 1: `307000` (30.7)
  - Cycle 4: `1000000` (100.0, fully converged at cycle 4)
- **Spec reference:** §6 — Rent price locking with convergence

### CONTENT-05: Storage capacity limit

- **Description:** Maximum active replicated content scales with reputation using `max_active_replicated = base_allowance × (reputation / 0.2)^2`, where `base_allowance = 10 MiB`.
- **Input → Expected output:**
  - Rep 0.2: `10 × (0.2 / 0.2)^2 = 10 × 1 = 10 MiB`
  - Rep 0.3: `10 × (0.3 / 0.2)^2 = 10 × 1.5^2 = 10 × 2.25 = 22.5 MiB`
  - Rep 0.5: `10 × (0.5 / 0.2)^2 = 10 × 2.5^2 = 10 × 6.25 = 62.5 MiB`
  - Rep 0.8: `10 × (0.8 / 0.2)^2 = 10 × 4^2 = 10 × 16 = 160 MiB`
  - Rep 1.0: `10 × (1.0 / 0.2)^2 = 10 × 5^2 = 10 × 25 = 250 MiB`
- **Spec reference:** §6 — Reputation-based storage limits

### CONTENT-06: Grace period escalation

- **Description:** Repeated missed rent payments within 90 days trigger progressively shorter grace periods.
- **Input:** Same content, three missed payments within 90 days
- **Expected:**
  - 1st missed payment: 7-day grace period
  - 2nd missed payment within 90 days: 3-day grace period
  - 3rd missed payment within 90 days: 0 days (immediate GC eligibility)
- **Spec reference:** §6 — "If the uploader cannot pay rent"

### CONTENT-07: Provenance reward split

- **Description:** When `origin_share_id` is set, adoption rewards are split 70/30 between proposer and original host.
- **Input:**
  - Proposal receives 10 ADOPT messages → +0.05 total (capped)
  - `origin_share_id` is set
  - Original host reputation ≥ 0.3
- **Expected:**
  - Proposer (author): 70% × 0.05 = 0.035 (fixed-point: 350)
  - Original host: 30% × 0.05 = 0.015 (fixed-point: 150)
- **Input (host below 0.3):**
  - Original host reputation < 0.3
- **Expected:**
  - Original host: capped at 0.002 per proposal (fixed-point: 20)
  - Proposer: receives remainder = 0.05 - 0.002 = 0.048 (fixed-point: 480)
- **Spec reference:** §6 — Content Provenance

### CONTENT-08: Provenance + storage provider split (both apply)

- **Description:** When both provenance and storage provider splits apply, storage providers receive their 30% first, then provenance splits the author's 70%.
- **Input:**
  - Proposal receives 10 ADOPT messages → +0.05 total
  - `origin_share_id` is set, host reputation ≥ 0.3
- **Computation:**
  ```
  Storage providers: 30% × 0.05 = 0.015 (fixed-point: 150)
  Author side: 70% × 0.05 = 0.035 (fixed-point: 350)
  
  With provenance on author side:
    Proposer: 70% × 0.035 = 0.0245 (fixed-point: 245)
    Original host: 30% × 0.035 = 0.0105 (fixed-point: 105)
  
  Total: 0.0245 + 0.0105 + 0.015 = 0.05 ✓
  ```
- **Expected:**
  - Storage providers: 150 (0.015)
  - Proposer: 245 (0.0245)
  - Original host: 105 (0.0105)
- **Spec reference:** §6 — Content Provenance, Content Lifecycle

### CONTENT-09: FLAG stake costs

- **Description:** Flagging content costs reputation as a stake, with additional penalties for false severe flags.
- **Input → Expected costs:**
  - `severity: dispute` flag: -0.005 (fixed-point: -50)
  - `severity: illegal` flag: -0.02 (fixed-point: -200)
  - False `severity: illegal` flag (rejected by governance): stake forfeited (-0.02) + additional penalty (-0.05) = total -0.07 (fixed-point: -700)
- **Spec reference:** §6 — Content Flagging

### CONTENT-10: Rent payment timing

- **Description:** RENT_PAYMENT must be broadcast within the first 7 days of the billing cycle.
- **Input scenarios:**
  - RENT_PAYMENT broadcast on day 3 of billing cycle → **accepted**
  - RENT_PAYMENT broadcast on day 7 of billing cycle → **accepted**
  - RENT_PAYMENT broadcast on day 8 of billing cycle → **treated as missed** (grace period triggers)
  - Duplicate RENT_PAYMENT for same `(content_hash, billing_cycle)` → **first-seen wins**, subsequent rejected
- **Spec reference:** §6 — Rent Payments

### CONTENT-11: CONTENT_WITHDRAW rules

- **Description:** Content withdrawal requires 24-hour delay and cannot conflict with active proposals.
- **Input scenarios:**
  - `effective_after` = timestamp + 24h → **accepted**
  - `effective_after` = timestamp + 12h → **MUST reject** (< 24h from message timestamp)
  - Content with active proposal (voting_deadline not passed) → **MUST reject** withdrawal
  - Content with concluded proposal (voting_deadline passed) → **accepted**
- **Spec reference:** §6 — Content Withdrawal

### CONTENT-12: Illegal flag deletion threshold

- **Description:** Immediate shard deletion requires specific multi-source thresholds.
- **Input scenarios:**
  - 5 flags from 3 distinct ASNs → **immediate shard deletion** ✓
  - 2 hash-match flags from different databases → **immediate shard deletion** ✓
  - 4 flags from 3 ASNs → **NOT enough** (need ≥5 flags)
  - 5 flags from 2 ASNs → **NOT enough** (need ≥3 ASNs)
  - 1 hash-match flag from 1 database → **stop serving, but NOT shard deletion** (need ≥2 databases)
- **Spec reference:** §6 — Content Flagging, Illegal response

---

## 16. Capability Ramp (§9)

### CAP-01: Capability thresholds by reputation

- **Description:** Fresh identities have limited capabilities. Higher reputation unlocks higher-impact actions.
- **Input → Expected behavior:**

  **At rep 0.2 (starting):**
  - Store shards → ✓
  - Earn rent → ✓
  - Sync → ✓
  - Browse → ✓
  - Adopt → ✓
  - Propose → **REJECT**
  - Vote → **REJECT**
  - Flag (dispute) → **REJECT**

  **At rep 0.3:**
  - Propose (small) → ✓
  - Vote → ✓
  - Flag (dispute) → ✓
  - Replicate content → ✓
  - Flag (illegal) → **REJECT**

  **At rep 0.5:**
  - Flag (illegal) → ✓
  - Full proposal size → ✓

- **Spec reference:** §9 — Capability Ramp

---

## 17. Votes (§8)

### VOTE-01: Minimum reputation to vote

- **Description:** Nodes MUST have reputation ≥ 0.3 to submit votes.
- **Input scenarios:**
  - Node at rep 2900 (0.29) submits VOTE → **MUST reject**
  - Node at rep 3000 (0.30) submits VOTE → **accepted**
- **Spec reference:** §8 — Vote Rules

### VOTE-02: Close-margin confirmation period

- **Description:** When a proposal's endorsement ratio falls within ±0.02 of the threshold at voting_deadline, a 7-day confirmation period is required.
- **Input scenarios:**
  - Proposal with endorsement ratio 6600 (0.66), threshold 6700 (0.67):
    - Within ±0.02 range (0.65–0.69) → **MUST enter 7-day confirmation period**
  - Proposal with endorsement ratio 7000 (0.70), threshold 6700 (0.67):
    - Outside ±0.02 range → **no confirmation period needed**, ratification determined immediately
- **Spec reference:** §8 — Close-margin confirmation

---

## 18. Storage Validators (§6)

### VALIDATOR-01: Rent split between providers and validators

- **Description:** Storage rent is split 80/20 between providers and validators.
- **Input (fixed-point ×10,000):**
  - Monthly rent: 1000 (0.1)
- **Expected:**
  - Provider share (80%): 800 (0.08)
  - Validator share (20%): 200 (0.02)
- **Spec reference:** §6 — Storage Provider and Validator Revenue

### VALIDATOR-02: Pro-rated rent for insufficient challenges

- **Description:** Providers who pass fewer than the minimum 10 challenges receive pro-rated rent.
- **Input:**
  - Monthly rent: 1000 (0.1)
  - Provider full share: 800 (0.08)
  - Challenges passed: 5 of 10 minimum
- **Expected:**
  - Provider receives 50% of share: 400 (0.04)
- **Spec reference:** §6 — Minimum challenge requirements

### VALIDATOR-03: Self-validation rejected

- **Description:** CHALLENGE_RESULT messages where the challenger and challenged are the same identity MUST be rejected.
- **Input:**
  - CHALLENGE_RESULT with challenger identity = challenged identity
- **Expected:** MUST reject.
- **Spec reference:** §6 — "Validators MUST NOT validate their own shards"

### VALIDATOR-04: Validator earnings cap

- **Description:** Each validator's earnings are capped at 30 challenges per content per billing cycle.
- **Input:**
  - Validator issues 50 challenges for one content in one billing cycle
- **Expected:**
  - First 30 challenges earn validator share
  - Remaining 20 challenges verify storage integrity but do NOT earn additional validator share
- **Spec reference:** §6 — Validator earnings cap

---

## 19. Comment Rate Limits (§7)

### COMMENT-01: Rate scaling with reputation

- **Description:** Comment rate limits scale with reputation. Per-proposal limits apply uniformly.
- **Input → Expected limits:**
  - Rep 0.3: **1 comment per hour**
  - Rep 0.5: **5 comments per hour**
  - Rep 0.8+: **10 comments per hour**
  - Per-proposal: **3 per identity per rolling 24-hour window** (all reputation levels)
- **Spec reference:** §7 — COMMENT

---

## 20. State Reconciliation — Sync Protocol (§5)

### SYNC-01: Sync phase ordering

- **Description:** Phases 1, 2, and 3 MUST complete sequentially in that order. Phases 4 and 5 MAY proceed in parallel after phase 3 completes.
- **Input:** Node joins network and begins state reconciliation.
- **Expected behavior:**
  - Phase 1 (Identity: DID_LINK, DID_REVOKE, KEY_ROTATE) completes before phase 2 starts.
  - Phase 2 (Reputation: REPUTATION_GOSSIP) completes before phase 3 starts.
  - Phase 3 (Proposals & Votes: PROPOSE, VOTE, COMMENT) completes before phases 4/5 start.
  - Phase 4 (Content Metadata: SHARE, FLAG, CONTENT_WITHDRAW, RENT_PAYMENT) and Phase 5 (Storage State: REPLICATE_REQUEST, REPLICATE_ACCEPT, SHARD_ASSIGNMENT, SHARD_RECEIVED) MAY run in parallel after phase 3.
  - Starting phase 2 before phase 1 completes → **violation**.
  - Starting phase 4 before phase 3 completes → **violation**.
  - Running phases 4 and 5 simultaneously after phase 3 → **allowed**.
- **Spec reference:** §5 — Sync Phases

### SYNC-02: Gossip buffer phase classification

- **Description:** Live gossip messages received during sync are classified by phase. Messages for completed phases are applied immediately; messages for current or future phases are buffered.
- **Input:**
  - Node is currently syncing phase 2 (Reputation). Phase 1 (Identity) is already committed.
  - Receives live gossip: DID_REVOKE (phase 1), VOTE (phase 3), REPUTATION_GOSSIP (phase 2).
- **Expected:**
  - DID_REVOKE (phase 1 — already completed): **applied immediately** (validated against committed identity state).
  - REPUTATION_GOSSIP (phase 2 — current phase): **buffered**, applied in timestamp order when phase 2 completes.
  - VOTE (phase 3 — future phase): **buffered**, applied in timestamp order when phase 3 completes.
- **Spec reference:** §5 — Gossip buffering during sync

### SYNC-03: Gossip buffer priority eviction (phase 1)

- **Description:** When the gossip buffer for a phase fills, messages are evicted by priority. Phase 1 priority: DID_REVOKE > KEY_ROTATE > DID_LINK.
- **Input:**
  - Phase 1 gossip buffer is full (at capacity limit).
  - New DID_REVOKE message arrives for phase 1 buffer.
  - Buffer contains: 10 DID_LINK messages, 5 KEY_ROTATE messages, 2 DID_REVOKE messages.
- **Expected:**
  - The oldest DID_LINK message is evicted first (lowest priority, oldest first within priority).
  - The new DID_REVOKE is buffered.
  - If all DID_LINK messages were already evicted, the oldest KEY_ROTATE would be evicted next.
  - DID_REVOKE messages are evicted last (highest priority).
- **Spec reference:** §5 — Gossip buffer priority

### SYNC-04: Gossip buffer size cap

- **Description:** The gossip buffer per phase is bounded to 100,000 messages or 100 MiB, whichever is reached first.
- **Input scenarios:**
  - Buffer has 99,999 messages totaling 50 MiB → new message **accepted**.
  - Buffer has 100,000 messages totaling 50 MiB → new message triggers **eviction** (message count limit).
  - Buffer has 50,000 messages totaling 100 MiB → new message triggers **eviction** (byte limit).
  - Buffer has 99,999 messages totaling 99.9 MiB → new message **accepted** (neither limit reached).
- **Expected:** When either limit is reached, lowest-priority oldest messages are evicted. The node MUST track the timestamp of the oldest dropped message per phase for follow-up incremental sync.
- **Spec reference:** §5 — Gossip buffer cap

### SYNC-05: REPUTATION_CURRENT trimmed minimum aggregation (5 peers)

- **Description:** With 5 sync peers reporting REPUTATION_CURRENT for a subject, discard the highest and lowest, then use the minimum of the remaining 3.
- **Input (fixed-point ×10,000):**
  - Peer 1 reports subject X reputation: 5000 (0.50)
  - Peer 2 reports subject X reputation: 6000 (0.60)
  - Peer 3 reports subject X reputation: 7000 (0.70)
  - Peer 4 reports subject X reputation: 4000 (0.40)
  - Peer 5 reports subject X reputation: 8000 (0.80)
- **Computation:**
  ```
  Sorted: [4000, 5000, 6000, 7000, 8000]
  Discard lowest (4000) and highest (8000)
  Remaining: [5000, 6000, 7000]
  Result = min(5000, 6000, 7000) = 5000
  ```
- **Expected result:** `5000` (represents 0.50)
- **Spec reference:** §5 — REPUTATION_CURRENT integrity

### SYNC-06: REPUTATION_CURRENT sparse coverage

- **Description:** When fewer than 5 sync peers report a value, adjusted aggregation rules apply.
- **Input (fixed-point ×10,000) — 4 peers:**
  - Reports: [4000, 5000, 7000, 8000]
  - Computation: discard only the highest (8000), min of bottom 3 → min(4000, 5000, 7000) = 4000
  - **Expected:** `4000`
- **Input — 3 peers:**
  - Reports: [5000, 6000, 7000]
  - Computation: use minimum directly (no trim) → min(5000, 6000, 7000) = 5000
  - **Expected:** `5000`
- **Input — 2 peers:**
  - Reports: [5000, 7000]
  - **Expected:** Fall back to **full REPUTATION_GOSSIP replay** for this subject.
- **Input — 1 peer:**
  - Reports: [6000]
  - **Expected:** Fall back to **full REPUTATION_GOSSIP replay** for this subject.
- **Spec reference:** §5 — REPUTATION_CURRENT sparse coverage

### SYNC-07: REPUTATION_CURRENT divergence threshold

- **Description:** If the difference between the 2nd-highest and 2nd-lowest reported values exceeds 0.1, fall back to full replay.
- **Input (fixed-point ×10,000) — 5 peers:**
  - Reports: [3000, 4000, 7000, 8000, 9000]
  - Sorted: [3000, 4000, 7000, 8000, 9000]
  - 2nd-lowest = 4000, 2nd-highest = 8000
  - Difference = 8000 − 4000 = 4000 (represents 0.40) > 1000 (0.1)
  - **Expected:** Fall back to **full REPUTATION_GOSSIP replay** for this subject.
- **Input — 5 peers (within threshold):**
  - Reports: [5000, 5500, 6000, 6200, 6500]
  - 2nd-lowest = 5500, 2nd-highest = 6200
  - Difference = 6200 − 5500 = 700 (0.07) ≤ 1000 (0.1)
  - **Expected:** Proceed with trimmed minimum aggregation. Discard 5000 and 6500, min(5500, 6000, 6200) = `5500`.
- **Spec reference:** §5 — REPUTATION_CURRENT divergence threshold

### SYNC-08: Sync completeness criteria

- **Description:** A node is sync-complete when its identity and proposal Merkle roots each match at least 3 of 5 sync peers.
- **Input:**
  - Identity Merkle root matches peers 1, 2, 3 (3/5) ✓
  - Proposal Merkle root matches peers 1, 3, 4 (3/5) ✓
  - All phases completed, at least one REPUTATION_GOSSIP batch from each peer ✓
- **Expected:** Node transitions to `sync_status: "synced"`. MAY now vote, propose, flag, and participate in storage.
- **Input (insufficient agreement):**
  - Identity Merkle root matches peers 1, 2 (2/5) ✗
  - Proposal Merkle root matches peers 1, 2, 3, 4, 5 (5/5) ✓
- **Expected:** Node does NOT transition to "synced". Must resolve identity Merkle divergence first.
- **Spec reference:** §5 — Sync Completeness

### SYNC-09: Snapshot freshness

- **Description:** Snapshots with `snapshot_timestamp` older than 24 hours MUST be rejected.
- **Input scenarios:**
  - `snapshot_timestamp` = current_time − 86,400,000 ms (exactly 24h) → **accepted** (at boundary).
  - `snapshot_timestamp` = current_time − 86,400,001 ms (24h + 1ms) → **MUST reject**.
  - `snapshot_timestamp` = current_time − 43,200,000 ms (12h) → **accepted**.
  - `snapshot_timestamp` = current_time + 300,001 ms (5 min + 1ms in future) → **MUST reject** (standard timestamp tolerance).
- **Spec reference:** §5 — State Snapshots

### SYNC-10: Snapshot publisher reputation threshold

- **Description:** Only nodes with reputation ≥ 0.7 SHOULD publish snapshots. Syncing nodes MUST request snapshots from peers with reputation ≥ 0.7.
- **Input scenarios:**
  - Peer with reputation 7000 (0.70) publishes STATE_SNAPSHOT → **valid publisher**.
  - Peer with reputation 6999 (0.6999) publishes STATE_SNAPSHOT → **below threshold**, syncing nodes MUST NOT use this snapshot.
  - Peer with reputation 8000 (0.80) publishes STATE_SNAPSHOT → **valid publisher**.
- **Spec reference:** §5 — State Snapshots (publisher threshold)

### SYNC-11: Snapshot minimum publishers

- **Description:** A syncing node MAY bootstrap from a snapshot if it receives matching Merkle roots from ≥5 snapshot publishers across ≥3 ASNs, plus 2 non-publisher cross-check peers.
- **Input scenarios:**
  - 5 publishers from 3 ASNs, all with matching identity + proposal Merkle roots → **snapshot bootstrap allowed**.
  - 5 publishers from 2 ASNs (all matching) → **MUST NOT bootstrap** (insufficient ASN diversity).
  - 4 publishers from 3 ASNs (all matching) → **MUST NOT bootstrap** (insufficient publishers).
  - 5 publishers from 3 ASNs (matching), but cannot find 2 non-publisher cross-check peers after 10 additional connections → **MAY proceed with cross-checking against any 2 synced non-publisher peers** (small network fallback).
- **Spec reference:** §5 — State Snapshots (minimum publishers, post-snapshot verification)

### SYNC-12: Degraded mode 50% vote weight

- **Description:** Nodes in degraded mode with phases 1–2 complete MAY vote with 50% weight. The 50% reduction applies uniformly to all degraded states.
- **Input:**
  - Node has completed phases 1 (Identity) and 2 (Reputation) but phase 3 failed.
  - Node enters degraded mode (`sync_status: "degraded"`).
  - Node casts a VOTE with reputation weight 6000 (0.60).
- **Expected:**
  - Node MAY vote (phases 1–2 complete).
  - VOTE message MUST include `sync_status: "degraded"`.
  - Other nodes apply 50% weight: effective weight = 3000 (0.30).
  - Weight reduction applies via `min(self_declared_weight, weight_from_observed_sync_status)`.
- **Input (phases 1–2 NOT complete):**
  - Node in degraded mode, only phase 1 complete.
  - **Expected:** Node MUST NOT vote (phases 1–2 not both complete).
- **Spec reference:** §5 — Degraded mode

### SYNC-13: Degraded mode cold start interaction

- **Description:** During cold start, degraded nodes count as full participants for headcount but their votes count as 0.5 in the endorsement calculation.
- **Input:**
  - Cold start active. 10 nodes vote: 7 synced, 3 degraded (all with phases 1–2 complete).
  - All 10 endorse.
- **Computation:**
  ```
  Total headcount: 10 (all count as participants)
  Effective endorsement votes: 7 × 1.0 + 3 × 0.5 = 8.5
  Effective total votes: 8.5 (all endorsed)
  Endorsement ratio: 8.5 / 8.5 = 1.0 ≥ 0.67 ✓
  ```
- **Input (close margin):**
  - 10 nodes: 7 synced endorse, 3 degraded reject.
  - Effective endorsement: 7 × 1.0 = 7.0
  - Effective rejection: 3 × 0.5 = 1.5
  - Endorsement ratio: 7.0 / (7.0 + 1.5) = 7.0 / 8.5 = 0.8235 ≥ 0.67 ✓
  - **Expected:** Ratified.
- **Spec reference:** §5 — Degraded mode cold start interaction

### SYNC-14: DID_REVOKE retroactive invalidation

- **Description:** When a DID_REVOKE is applied, the node MUST scan committed state for messages from the revoked key with timestamps after the revocation's `effective_from` and invalidate them.
- **Input:**
  1. Identity: root=A, child=B. B has voted on proposals P1, P2, P3.
  2. DID_REVOKE arrives: `revoked_key` = B, `effective_from` = T.
  3. B's votes: P1 vote at T−1000 (before effective_from), P2 vote at T+1000 (after), P3 vote at T+5000 (after).
- **Expected:**
  - P1 vote (timestamp < effective_from): **retained** (vote was before suspected compromise).
  - P2 vote (timestamp > effective_from): **invalidated**.
  - P3 vote (timestamp > effective_from): **invalidated**.
  - Any buffered gossip messages from B MUST be discarded at buffer-drain time.
  - Revocation status MUST be checked both when buffering and when applying buffered messages.
- **Spec reference:** §5 — Retroactive invalidation

### SYNC-15: Incremental sync identity phase jitter

- **Description:** Identity messages (phase 1) are included in every 3rd to 5th incremental sync cycle (randomly selected), approximately once per 45–75 minutes. Event-triggered on DID_REVOKE.
- **Input scenarios:**
  - Incremental sync interval: 15 minutes. Identity phase included in cycles 3, 7, 12 (random selection within 3rd–5th each time).
  - **Expected:** Identity sync occurs approximately every 45–75 minutes (3–5 × 15 min).
  - Previous cycle's live gossip included a DID_REVOKE message.
  - **Expected:** Next incremental sync MUST include identity messages **regardless of cycle counter**.
- **Spec reference:** §5 — Incremental Sync (identity phase jitter)

### SYNC-16: Sync jitter scaling (partition-aware)

- **Description:** Sync jitter scales based on partition detection. Normal jitter is 0–30s. Partition-detected jitter is 0–300s.
- **Input scenarios:**
  - Node reconnects, Merkle divergence from ≤20% of peers → base jitter: **uniform 0–30 seconds**.
  - Node reconnects, Merkle divergence from >20% of peers → elevated jitter: **uniform 0–300 seconds**.
  - Node offline for 36,000 seconds (10 hours) → jitter: `min(300, 36000 / 100)` = **min(300, 360) = 300 seconds**.
  - Node offline for 5,000 seconds (~83 min) → jitter: `min(300, 5000 / 100)` = **min(300, 50) = 50 seconds**.
  - Node offline for 100,000 seconds (~27.8 hours) → jitter: `min(300, 100000 / 100)` = **min(300, 1000) = 300 seconds**.
- **Spec reference:** §5 — Backpressure (jitter)

### SYNC-17: Sync peer requirements

- **Description:** Nodes MUST sync from at least 5 peers from at least 3 distinct ASNs. First-join requires at least one bootstrap node.
- **Input scenarios:**
  - 5 peers from 3 ASNs (ASN1: 2 peers, ASN2: 2 peers, ASN3: 1 peer) → **valid sync peer set**.
  - 5 peers from 2 ASNs → **MUST NOT proceed** (insufficient ASN diversity).
  - 4 peers from 4 ASNs → **MUST NOT proceed** (insufficient peer count).
  - First join (`since_timestamp = 0`): 5 peers from 3 ASNs, none are bootstrap nodes → **MUST NOT proceed** (first-join requires bootstrap node).
  - First join: 5 peers from 3 ASNs, 1 is a bootstrap node → **valid**.
  - Incremental sync (not first join): 5 peers from 3 ASNs, no bootstrap → **valid** (bootstrap requirement only for first join).
- **Spec reference:** §5 — Sync Peer Selection

### SYNC-18: Mid-sync stale phase check

- **Description:** If more than 1 hour has elapsed since phase 1 completed, the node MUST re-run phase 1 before resuming later phases.
- **Input scenarios:**
  - Phase 1 completed at T. Currently at T + 3,599,999 ms (< 1 hour). Resuming phase 3.
  - **Expected:** Resume phase 3 normally (phase 1 still fresh).
  - Phase 1 completed at T. Currently at T + 3,600,001 ms (> 1 hour). Resuming phase 3.
  - **Expected:** MUST re-run phase 1 before resuming phase 3.
- **Spec reference:** §5 — Mid-Sync Failure Recovery

### SYNC-19: Sync serving uptime credit limits

- **Description:** Sync serving earns uptime credit with anti-farming limits: at most 1 counted interaction per peer per 15 minutes, only for non-empty responses, from ≥3 distinct peers in 24 hours.
- **Input scenarios:**
  - Node serves sync response with non-empty messages to peer X at T. Same peer X requests again at T + 14 min.
  - **Expected:** Only the first interaction counts for uptime credit (< 15 min gap).
  - Node serves sync to peer X at T, peer Y at T + 5 min, peer Z at T + 10 min (all non-empty, 3 distinct peers).
  - **Expected:** All 3 count. Meets ≥3 distinct peer requirement for uptime credit.
  - Node serves sync to peer X and peer Y only (2 distinct peers) in a 24-hour window.
  - **Expected:** Does NOT qualify for uptime credit (< 3 distinct peers).
  - Node serves sync response with empty `messages` array to peer X.
  - **Expected:** Does NOT count toward uptime credit (empty response).
- **Spec reference:** §5 — Sync Serving Incentive

### SYNC-20: Post-snapshot sync window

- **Description:** After successful snapshot verification, incremental sync starts from `snapshot_timestamp − 1 hour` (safety margin).
- **Input:**
  - Snapshot verified with `snapshot_timestamp` = 1700000000000.
  - **Expected:** Incremental sync uses `since_timestamp` = 1700000000000 − 3,600,000 = **1699996400000**.
  - This ensures messages published slightly before the snapshot (including backdated messages) are captured.
- **Spec reference:** §5 — State Snapshots (post-snapshot sync)

---

## 21. Identity Merkle Divergence Severity (§5, §12)

### PART-06: Identity Merkle divergence — additive only (info)

- **Description:** When identity Merkle divergence between two "synced" nodes involves only DID_LINK or KEY_ROTATE (additive messages), severity is info.
- **Input:**
  - Node A and Node B are both `sync_status: "synced"`.
  - Identity Merkle roots diverge.
  - Merkle narrowing reveals the difference is 2 DID_LINK messages present on A but not B.
- **Expected:** Severity = **info**. Will converge via gossip. No immediate action required.
- **Spec reference:** §5 — Identity divergence severity, §12 — Severity Classification

### PART-07: Identity Merkle divergence — DID_REVOKE (critical)

- **Description:** When identity Merkle divergence involves any DID_REVOKE, severity is critical regardless of other message types.
- **Input:**
  - Node A and Node B are both `sync_status: "synced"`.
  - Identity Merkle roots diverge.
  - Merkle narrowing reveals: 1 DID_REVOKE present on A but not B.
- **Expected:**
  - Severity = **critical** (security-relevant).
  - Node B MUST immediately fetch the missing DID_REVOKE.
  - Node B MUST revalidate any messages accepted from the revoked key.
- **Spec reference:** §5 — Identity divergence severity, §12 — Severity Classification

### PART-08: Identity Merkle divergence — mixed additive + subtractive (critical)

- **Description:** Mixed divergence (additive + subtractive) is classified as critical because subtractive takes precedence.
- **Input:**
  - Divergence includes: 3 DID_LINK (additive) + 1 DID_REVOKE (subtractive).
- **Expected:** Severity = **critical** (subtractive takes precedence over additive).
- **Spec reference:** §5 — Identity divergence severity, §12 — Severity Classification

### PART-09: Identity divergence does NOT trigger partition detection for syncing nodes

- **Description:** Peers MUST NOT trigger partition detection based on Merkle divergence with nodes whose `sync_status` is "syncing".
- **Input:**
  - Node A (`sync_status: "synced"`) compares identity Merkle root with Node B (`sync_status: "syncing"`).
  - Roots diverge significantly.
- **Expected:** No partition event triggered. Divergence is expected for syncing nodes. Partition detection applies only between two "synced" nodes.
- **Spec reference:** §5 — Sync Status

---

## 22. Key Rotation Edge Cases (§1)

### SIG-03: KEY_ROTATE first-seen-wins rejection

- **Description:** A second KEY_ROTATE for the same `old_key` with a different `new_key` MUST be rejected (first-seen wins).
- **Input:**
  1. KEY_ROTATE: `old_key` = Keypair A, `new_key` = Keypair B (valid dual signatures). Accepted.
  2. KEY_ROTATE: `old_key` = Keypair A, `new_key` = Keypair C (valid dual signatures from A and C).
- **Expected:** Second KEY_ROTATE MUST be rejected. Keypair A has already been rotated to Keypair B. The node MUST NOT update the rotation target.
- **Spec reference:** §1 — "Subsequent KEY_ROTATE messages with the same `old_key` but a different `new_key` MUST be rejected"

### SIG-04: KEY_CONFLICT generation on conflicting rotations

- **Description:** When a node observes two KEY_ROTATE messages for the same `old_key` with different `new_key` values, it MUST broadcast a KEY_CONFLICT and apply suspension.
- **Input:**
  - KEY_ROTATE message 1: `old_key` = A, `new_key` = B (valid)
  - KEY_ROTATE message 2: `old_key` = A, `new_key` = C (valid, arrives later — e.g., from partition heal)
- **Expected:**
  1. Node MUST broadcast a KEY_CONFLICT message:
     ```json
     {
       "type": "KEY_CONFLICT",
       "payload": {
         "old_key": "<A public key>",
         "rotate_1": "<message_id of first KEY_ROTATE>",
         "rotate_2": "<message_id of second KEY_ROTATE>"
       }
     }
     ```
  2. Both `new_key` B and `new_key` C MUST be suspended:
     - Reputation frozen at floor: 1000 (0.1)
     - Voting weight reduced to 10% of normal
  3. Suspension persists until resolved by governance proposal.
- **Spec reference:** §1 — KEY_CONFLICT

### SIG-05: KEY_ROTATE grace period — old key accepted within 1 hour

- **Description:** After KEY_ROTATE, messages from the old key are accepted during the 1-hour grace period (measured from the KEY_ROTATE message timestamp).
- **Input:**
  - KEY_ROTATE message: `old_key` = A, `new_key` = B, `timestamp` = T
  - Message from A with `timestamp` = T + 3,599,999 ms (just under 1 hour)
- **Expected:** Message from A MUST be accepted (within grace period).
- **Spec reference:** §1 — "messages signed by the old key MUST be rejected after `KEY_ROTATE.timestamp + 3,600,000 ms`"

### SIG-06: KEY_ROTATE grace period — old key rejected after 1 hour

- **Description:** After KEY_ROTATE, messages from the old key MUST be rejected after the 1-hour grace period.
- **Input:**
  - KEY_ROTATE message: `old_key` = A, `new_key` = B, `timestamp` = T
  - Message from A with `timestamp` = T + 3,600,001 ms (just over 1 hour)
- **Expected:** Message from A MUST be rejected (grace period expired).
- **Spec reference:** §1 — "messages signed by the old key MUST be rejected after `KEY_ROTATE.timestamp + 3,600,000 ms`"

### SIG-07: KEY_CONFLICT suspension effects

- **Description:** When KEY_CONFLICT is in effect, both conflicting keys have reputation frozen and voting weight reduced.
- **Input:**
  - KEY_CONFLICT active for `old_key` = A, conflicting `new_key` values B and C.
  - B has pre-conflict reputation 7000 (0.7). C has pre-conflict reputation 5000 (0.5).
  - B casts a VOTE with normal weight 7000.
- **Expected:**
  - B's reputation is frozen at 1000 (0.1 floor) — no gains or losses applied.
  - B's effective voting weight = 1000 × 10% = 100 (0.01 effective).
  - C's reputation similarly frozen at 1000.
  - Neither B nor C can earn reputation gains until conflict resolution.
- **Spec reference:** §1 — "reputation frozen at floor (0.1), voting weight reduced to 10% of normal"

---

## 23. Authentication (§3)

### AUTH-01: AUTH_RESPONSE replay detection — wrong initiator key

- **Description:** An AUTH_RESPONSE that binds the wrong initiator key MUST be rejected, preventing replay attacks.
- **Input:**
  - Node X sends AUTH_CHALLENGE to Node Y, including X's public key.
  - Y constructs AUTH_RESPONSE signing: `challenge_nonce || X_public_key`.
  - Attacker Z replays Y's AUTH_RESPONSE to a different node W.
- **Expected:** W MUST reject the AUTH_RESPONSE because the bound initiator key (X) does not match W's own key. The signature is valid but the binding is wrong.
- **Spec reference:** §3 — "Binding the initiator's key into the signed response prevents replay attacks"

---

## 24. Close-Margin Confirmation (§8)

### VOTE-03: Close-margin confirmation — full deterministic test vector

- **Description:** A proposal whose endorsement ratio falls within ±0.02 of the threshold at voting deadline MUST enter a 7-day confirmation period.
- **Input (fixed-point ×10,000):**
  - Threshold: 6700 (0.67)
  - Close-margin range: 6500–6900 (0.65–0.69)
  - Voters (all synced):
    - Voter 1: rep weight 5000, stance = endorse
    - Voter 2: rep weight 3000, stance = endorse
    - Voter 3: rep weight 2000, stance = endorse
    - Voter 4: rep weight 4000, stance = reject
    - Voter 5: rep weight 1000, stance = reject
  - Total voter rep: 15000 (quorum assumed met)
- **Computation:**
  ```
  weighted_endorse = 5000 + 3000 + 2000 = 10000
  weighted_reject = 4000 + 1000 = 5000
  endorsement_ratio = 10000 × 10000 / (10000 + 5000)
                    = 100000000 / 15000
                    = 6666  (truncated)
  ```
  - 6666 is within range [6500, 6900] → close margin detected.
- **Expected:**
  - Proposal MUST NOT be ratified or rejected immediately.
  - Proposal enters 7-day confirmation period.
  - During the confirmation period, new votes are accepted and the ratio is recalculated.
  - At the end of the 7-day period, the final ratio determines the outcome.
- **Spec reference:** §8 — Close-margin confirmation

### VOTE-04: Close-margin — outside range, no confirmation needed

- **Description:** A proposal whose endorsement ratio is outside the ±0.02 range is decided immediately.
- **Input (fixed-point ×10,000):**
  - Same voters as VOTE-03 except Voter 5 switches to endorse.
  - weighted_endorse = 5000 + 3000 + 2000 + 1000 = 11000
  - weighted_reject = 4000
  - endorsement_ratio = 11000 × 10000 / 15000 = 7333
  - 7333 > 6900 → outside close-margin range.
- **Expected:** Proposal is ratified immediately (ratio 0.7333 ≥ 0.67 threshold, outside ±0.02 band).
- **Spec reference:** §8 — Close-margin confirmation

---

## 25. Identity Linking Edge Cases (§1)

### LINK-15: DID_LINK first-seen-wins — same child to different root rejected

- **Description:** A second DID_LINK claiming the same child key for a different root MUST be rejected.
- **Input:**
  1. DID_LINK: root=A, child=C (valid, accepted — first seen).
  2. DID_LINK: root=B, child=C (valid signatures from B and C).
- **Expected:** Second DID_LINK MUST be rejected. Child C is already linked to root A. This is a MUST regardless of whether the child provided a valid signature for the second link.
- **Note:** This differs from LINK-02 only in that it uses a third keypair (C) as the child. The rule is: first valid DID_LINK for a given child key wins unconditionally.
- **Spec reference:** §1 — "First valid DID_LINK for a given child key wins"

### LINK-16: Identity gain dampening — 3 keys

- **Description:** Gain dampening with 3 authorized keys uses `raw_gain / 3^0.75`.
- **Input (fixed-point ×10,000):**
  - raw_gain = 100 (represents 0.01)
  - authorized_key_count = 3
- **Computation:**
  ```
  dampening_factor = 3^0.75 = 2.2795
  fixed_point_factor = 22795
  effective_gain = 100 × 10000 / 22795 = 43 (truncated)
  ```
- **Expected result (3 keys):** `43` (represents 0.0043)
- **Cross-check with REP-05 (2 keys):** `59` (represents 0.0059) — 3 keys gives less gain than 2 keys ✓
- **Spec reference:** §1 — Gain dampening: `effective_gain = raw_gain / authorized_key_count^0.75`

### LINK-17: Unlinked key 60-day voting cooldown

- **Description:** A key that was revoked from an identity and re-registers independently faces a 60-day voting cooldown from the DID_REVOKE timestamp.
- **Input:**
  1. Identity: root=A, child=B.
  2. DID_REVOKE at timestamp T: revokes B.
  3. B re-registers as independent node.
  4. B attempts to VOTE at timestamp T + 59 days (< 60 days).
  5. B attempts to VOTE at timestamp T + 61 days (> 60 days).
- **Expected:**
  - Vote at T + 59 days: MUST be rejected (cooldown active).
  - Vote at T + 61 days: accepted (cooldown expired, assuming rep ≥ 0.3).
- **Spec reference:** §1 — "Re-registered keys that were previously part of an identity face a 60-day voting cooldown"

---

## 26. Sync Edge Cases (§5)

### SYNC-21: Sync peer selection — reject syncing/degraded peers

- **Description:** Nodes with `sync_status` of `"syncing"` or `"degraded"` MUST NOT be selected as sync peers.
- **Input:**
  - Available peers: P1 (synced), P2 (syncing), P3 (synced), P4 (degraded), P5 (synced), P6 (synced), P7 (synced)
  - All from ≥3 distinct ASNs.
- **Expected:**
  - Eligible sync peers: P1, P3, P5, P6, P7 (5 peers, all synced).
  - P2 (syncing) and P4 (degraded) MUST NOT be selected.
  - If only 4 synced peers are available from ≥3 ASNs, the node MUST NOT proceed with sync (insufficient peer count).
- **Spec reference:** §5 — "Nodes with status `syncing` or `degraded` MUST NOT be selected as sync peers"

### SYNC-22: Mid-sync phase 1 stale check boundary

- **Description:** If more than 1 hour has elapsed since phase 1 completed, phase 1 MUST be re-run. Exactly 1 hour is the boundary.
- **Input scenarios:**
  - Phase 1 completed at T. Current time = T + 3,540,000 ms (59 minutes):
    - **Expected:** Phase 1 is still fresh. Resume later phases normally.
  - Phase 1 completed at T. Current time = T + 3,660,000 ms (61 minutes):
    - **Expected:** Phase 1 is stale. MUST re-run phase 1 before resuming later phases.
  - Phase 1 completed at T. Current time = T + 3,600,000 ms (exactly 60 minutes):
    - **Expected:** Boundary case — at exactly 1 hour, phase 1 is still fresh (the check is "> 1 hour", not "≥ 1 hour").
- **Spec reference:** §5 — "If more than 1 hour has elapsed since phase 1 completed, the node MUST re-run phase 1"

### SYNC-23: Gossip buffer phase classification — full mapping

- **Description:** Every message type is classified into exactly one sync phase for gossip buffer purposes.
- **Expected mapping:**

  | Message Type | Phase | Buffer Priority |
  |---|---|---|
  | `DID_LINK` | 1 (Identity) | Low (evicted first) |
  | `DID_REVOKE` | 1 (Identity) | High (evicted last) |
  | `KEY_ROTATE` | 1 (Identity) | Medium |
  | `REPUTATION_GOSSIP` | 2 (Reputation) | Equal (oldest first) |
  | `PROPOSE` | 3 (Proposals & Votes) | Medium |
  | `VOTE` | 3 (Proposals & Votes) | High (evicted last) |
  | `COMMENT` | 3 (Proposals & Votes) | Low (evicted first) |
  | `SHARE` | 4 (Content Metadata) | Equal (oldest first) |
  | `FLAG` | 4 (Content Metadata) | Equal (oldest first) |
  | `CONTENT_WITHDRAW` | 4 (Content Metadata) | Equal (oldest first) |
  | `RENT_PAYMENT` | 4 (Content Metadata) | Equal (oldest first) |
  | `REPLICATE_REQUEST` | 5 (Storage State) | Equal (oldest first) |
  | `REPLICATE_ACCEPT` | 5 (Storage State) | Equal (oldest first) |
  | `SHARD_ASSIGNMENT` | 5 (Storage State) | Equal (oldest first) |
  | `SHARD_RECEIVED` | 5 (Storage State) | Equal (oldest first) |

- **Verification:** For each message type above, if the node is currently syncing phase 2 and receives the message via gossip:
  - Phase 1 types (DID_LINK, DID_REVOKE, KEY_ROTATE): **applied immediately** (phase 1 already committed).
  - Phase 2 type (REPUTATION_GOSSIP): **buffered** in phase 2 buffer.
  - Phase 3 types (PROPOSE, VOTE, COMMENT): **buffered** in phase 3 buffer.
  - Phase 4 types (SHARE, FLAG, etc.): **buffered** in phase 4 buffer.
  - Phase 5 types (REPLICATE_REQUEST, etc.): **buffered** in phase 5 buffer (or discarded if node has no `store` capability).
- **Spec reference:** §5 — Gossip buffering during sync, message type → phase mapping table
