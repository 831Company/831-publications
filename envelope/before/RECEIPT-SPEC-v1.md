# The 831 receipt: specification v1

Claude (`831-claude`), Nov-4, 2026. Order CL-44 (S4 done right).

**Status:** normative for receipt bundles of schema `831.receipt-bundle.v1`.

**Revision v1.1 (Nov-5, 2026), on Seat C's C-37 review (#79 6663681474).** Two corrections to the text. The code did not change: both verifiers already behaved as v1.1 says. (a) §6 step 1 now states the real order: both files are parsed (bundle first, then pins) before the pins are checked. v1 said "pins first". (b) §6.3 now lists the event-shape check that runs after `RECOVERY_SIGNATURE` and before `RECOVERY_SEALED`. v1 left it out.

**Revision v1.2 (Nov-5, 2026), on Seat C's C-37 return (#79 6668835148).** Two precision edits, text only; neither verifier changed. They fix the first refusal on an input malformed in more than one place. (c) §6.1 step 12 now states that `e.target_anchor_sequence` passes `INTEGER_PROFILE` (≥ 1) before `ANCHOR_HEADS`, and that the `HEX_PROFILE` checks on `w.previous` then `w.commitment` follow `ANCHOR_HEADS`. (d) §6.3 step 3 now lists the profile checks that run after `RECOVERY_ROWS_SHAPE` and before the binding, in their order.

**Written from:**
- the instrument engine v2.2 (`instrument/engine.js` at `df587b2b`, the code behind `index.html` `13f08fd8…`, live since D-71);
- the Python reference verifier v2 (`instrument/reference/verify_receipt.py` `90e21f61…`).

Where the two differ, this document says so in §10 and names the side that is wrong.

**The goal:** someone who has read this document and none of our code can build a verifier that prints the same line, with the same exit code, on every receipt in the conformance set (§11).

The words MUST, MUST NOT and MAY are used as in RFC 2119.

---

## 1 · What a verifier takes and what it prints

**Inputs, as bytes:**
- the **bundle** (one receipt);
- the **pins** (the four public keys and the identity they belong to, supplied separately by the person checking).

Each input MUST be at most 1,048,576 bytes; otherwise the result is `JSON_SIZE`. Nothing else is read: no network, no clock, no file named inside a receipt.

**Output:** one line, the canonical JSON (§3) of a result object, then `\n`. Then an exit code:

| Result | Line | Exit |
|---|---|---|
| verified | `{"op_key":<op_key>,"state":"VERIFIED","verdict":"EXECUTED" \| "NOT_EXECUTED" \| "NOT_DISPATCHED"}` | 0 |
| refused | `{"failing_check":<name>,"state":"BROKEN"}` | 1 |
| cannot run | `{"reason":"ED25519_KAT_FAILED","state":"UNAVAILABLE"}` | 2 |

**The rules:**
- **The first check that fails names the result.** Checks run in exactly the order of §6. A verifier MUST NOT report a later check's name when an earlier one fails, and MUST NOT print VERIFIED unless every check of the claim's path passed.
- **A verifier never trusts a key found inside a receipt.** Every signature is checked against the pins.

## 2 · The JSON profile (applies to bundle and pins alike)

A document is accepted only if all of the following hold. The failure names are in brackets.

1. It decodes as UTF-8 with no invalid sequence [`UnicodeDecodeError`]. A byte-order mark is **not** stripped. It is a character outside the grammar, so the result is [`JSON_SYNTAX`].
2. It parses as JSON (RFC 8259):
   - whitespace is only space, tab, CR and LF;
   - nothing follows the top-level value;
   - no raw control character appears inside a string;
   - otherwise the result is [`JSON_SYNTAX`].
3. Nesting depth is at most 32, counting the top-level value as depth 0 [`JSON_DEPTH`]. Depth is checked while parsing, so a too-deep top-level array is `JSON_DEPTH`, not `JSON_OBJECT`.
4. Every object key is ASCII, and no key repeats within one object [`JSON_KEYS`].
5. Numbers:
   - `NaN`, `Infinity`, `-Infinity`, any fraction or exponent [`JSON_NUMBER_PROFILE`];
   - `-0` [`JSON_NEGATIVE_ZERO`];
   - any integer with magnitude of 2^53 or more [`JSON_UNSAFE_INTEGER`].
6. The top-level value is an object [`JSON_OBJECT`].

## 3 · Canonical form and digests

`canonical(v)` is the byte string of `v` as JSON with:
- **object keys sorted** by Unicode code point;
- **no whitespace** (separators `,` and `:`);
- **integers** in plain decimal;
- **arrays** in their own order;
- **strings escaped** as JSON requires, with every character from U+007F upward escaped as `\uXXXX` with lowercase hex. Characters above U+FFFF become a UTF-16 surrogate pair, each half escaped. The output is pure ASCII.

`digest(v)` = SHA-256 of `canonical(v)`, as 64 lowercase hex characters.

**Signatures are over `canonical(message)`.** Where a signed object carries its own `signature` field (a "wire"), the message is that object with the `signature` key removed.

## 4 · The pins

The pins object has **exactly** these keys [`PINS_SHAPE`]:
- `schema`, which MUST be `831.receipt-pins.v1` [`PINS_SCHEMA`];
- `signer_public`, `witness_public`, `founder_public`, `target_public`: 32-byte Ed25519 public keys as 64 lowercase hex [`HEX_PROFILE`];
- `anchor_id` and `target_generation`: 16 bytes each, as 32 lowercase hex;
- `target_id` and `target_signer_id`: non-empty strings [`TEXT_PROFILE`].

**The pins fingerprint is the SHA-256 of the exact pins file bytes.** For the genesis tier it is `78a3dceb…` (S4-GATE-03). It identifies *which* pins were used. It does not say who holds the keys. That is settled only by comparing it with the fingerprint the house publishes through separate channels (CL-44, S4-d).

**`anchor_id` is a random identifier** made when the witness store was created. It is **not** derived from any key, and it MUST NOT be presented as a fingerprint.

## 5 · Ed25519

**Before any signature is checked, the verifier proves its own Ed25519 on the device.** It uses RFC 8032 §7.1 TEST 1:
- public key `d75a980182b10ab7d54bfed3c964073a0ee172f3daa62325af021a68f707511a`;
- empty message;
- signature `e5564300c360ac729086e2cc806e828a84877f1eb8e5d974d873e065224901555fb8821590a33bacc61e39701cf9b46bd25bf5f0595bbe24655141438e7a100b`.

That signature MUST verify, and the same signature with its first byte's lowest bit flipped MUST NOT. If no available implementation passes both, the result is UNAVAILABLE (`ED25519_KAT_FAILED`, exit 2). It is never VERIFIED.

**Every signature check:**
- refuses a public key, or a signature `R`, that is not a valid point or is of small order;
- uses strict RFC 8032 verification (no ZIP-215 relaxations).

**A signature or public key given in the wrong shape fails as the signature check it belongs to:** a malformed record signature is `RECORD_SEAL`, not `HEX_PROFILE`.

**Any other field in the wrong shape fails by the profile it breaks:**
- `HEX_PROFILE`: not lowercase hex of the stated length;
- `INTEGER_PROFILE`: not an integer, below the stated minimum, or 2^53 or more;
- `TEXT_PROFILE`: not a non-empty string.

These can fire at any step that reads such a field, before that step's own name.

## 6 · The checks, in order

**A null where an object is required is refused by that object's shape name**, for example `RECORD_SHAPE` or `INTENT_SCHEMA`, never by an engine error.

**Common to every claim:**
1. **Both files are parsed under §2: the bundle's bytes first, then the pins' bytes.** The first parse refusal is the line. So when both files are malformed, the bundle's refusal is the one printed. Then the **pins** are checked under §4 (`PINS_SHAPE`, `PINS_SCHEMA`, the field profiles).
2. **The bundle** has exactly `schema`, `op_key`, `claim`, `proof` [`BUNDLE_SHAPE`].
   - `schema` is `831.receipt-bundle.v1` [`BUNDLE_SCHEMA`].
   - `op_key` is a non-empty string.
3. **Ed25519 self-test** (§5).
4. **`claim` selects the path:**
   - `EXECUTED` → §6.1;
   - `NOT_EXECUTED_WITNESS_CANCELLED` → §6.2;
   - `NOT_DISPATCHED_SIGNER_PROVEN` → §6.3;
   - anything else [`CLAIM_CLASS`].

### 6.0 · The record (used by 6.1 and 6.2)

`proof` has exactly `record`, `record_signature`, plus `dispatch_receipt`, `anchored_proof` (EXECUTED) or `cancellation` (NOT_EXECUTED) [`PROOF_SHAPE`].

1. **The record** `r = proof.record`:
   - has exactly `schema`, `journal_head`, `intent`, `intent_signature`, `signature_sha256` [`RECORD_SHAPE`];
   - `schema` is `831.cp09.signed-dispatch.v1` [`RECORD_SCHEMA`];
   - `journal_head` is 32-byte hex.
2. **RECORD_SEAL:** `proof.record_signature` verifies over `canonical(r)` under `signer_public`.
3. **The intent** `i = r.intent`:
   - Its schema is `831.cp09.gateway-intent.v3`, which carries `binding_required`, and that MUST be `true` [`INTENT_BINDING_REQUIRED`]. Or its schema is `…-intent.v2` [`INTENT_SCHEMA`].
   - Its keys are exactly `schema op_key intent_nonce policy_digest payload_sha256 target_id target_generation expected_sequence previous_root`, plus `binding_required` for v3 [`INTENT_SHAPE`].
4. **INTENT_CONTEXT:**
   - `i.op_key` = the bundle's `op_key`;
   - `i.target_id` and `i.target_generation` equal the pins'.

   Then the field profiles:
   - `expected_sequence` is an integer ≥ 0 [`INTEGER_PROFILE`];
   - `intent_nonce` is 16-byte hex; `policy_digest`, `payload_sha256` and `previous_root` are 32-byte hex [`HEX_PROFILE`].
5. **INTENT_SIGNATURE:** `r.intent_signature` verifies over `canonical(i)` under `signer_public`.
6. **INTENT_SIGNATURE_DIGEST:** `r.signature_sha256` (32-byte hex) = SHA-256 of the 64 raw bytes of `r.intent_signature`.

### 6.1 · EXECUTED

7. **WITNESS_BINDING:** `proof.dispatch_receipt`, with `signature` removed, equals **exactly**:
   ```
   {schema:"831.cp09.witness-dispatch.v1", anchor_id, target_id, target_generation (all three from the pins),
    slot:i.expected_sequence, record_hash:digest(r)}
   ```
   - Otherwise [`WITNESS_BINDING_CONTEXT`].
   - Its `signature` verifies under `witness_public` [`WITNESS_BINDING_SIGNATURE`].
8. **The envelope** `e = proof.anchored_proof`:
   - has exactly `schema target_receipt witness_checkpoint founder_ack target_anchor_sequence` [`ENVELOPE_SHAPE`];
   - `schema` is `831.cp09.anchored-proof.v1` [`ENVELOPE_SCHEMA`].
9. **TARGET_RECEIPT:** `t = e.target_receipt` has exactly the keys below plus `signature` [`TARGET_SHAPE`]. Without `signature` it equals exactly the following [`TARGET_CONTEXT`], and the signature verifies under `target_public` [`TARGET_SIGNATURE`]:
   ```
   {schema:"831.target.receipt.v2", target_id, target_generation (pins), operation_id:i.op_key, status:"DONE",
    payload_sha256:i.payload_sha256, effect_sha256:i.payload_sha256, signer_id:pins.target_signer_id,
    intent_nonce:i.intent_nonce, policy_digest:i.policy_digest}
   ```
   **What this proves:** the target key signed DONE for **exactly** the authorized payload, under the authorized policy digest and nonce.
   - `effect_sha256` equals `payload_sha256` **by definition**: the reference target records the payload itself as its effect (`target_recovery.py` `apply`), and the gateway refuses any other value.
   - It is **not** a measurement of an effect outside the target (§9).
10. **The checkpoint and the founder acknowledgement:**
    - `w = e.witness_checkpoint` has exactly `schema anchor_id target_id target_generation sequence root previous commitment signature` [`CHECKPOINT_SHAPE`].
    - `f = e.founder_ack` has exactly `schema anchor_id target_id target_generation sequence root signature` [`FOUNDER_SHAPE`].
    - **ANCHOR_CONTEXT:**
      - `w.schema` is `831.external.anchor.v1` and `f.schema` is `831.founder.anchor-ack.v1`;
      - both carry the pins' `anchor_id`, `target_id`, `target_generation`;
      - `sequence` is an integer ≥ 1;
      - `root` is 32-byte hex.
11. **CHECKPOINT_SIGNATURE** (`w` under `witness_public`), then **FOUNDER_SIGNATURE** (`f` under `founder_public`).
12. **ANCHOR_HEADS**, in this order:
    1. `e.target_anchor_sequence` is an integer ≥ 1 [`INTEGER_PROFILE`];
    2. `e.target_anchor_sequence` = `w.sequence` = `f.sequence`, and `w.root` = `f.root` [`ANCHOR_HEADS`];
    3. `w.previous`, then `w.commitment`, is 32-byte lowercase hex [`HEX_PROFILE`].
13. **ANCHOR_COMMITMENT_ROOT:**
    ```
    commitment = digest({op_key:i.op_key, nonce:i.intent_nonce, policy_digest:i.policy_digest,
                         payload_sha256:i.payload_sha256, target_generation:pins.target_generation})
    root       = digest({schema:"831.external.anchor.v1", sequence:e.target_anchor_sequence,
                         previous:w.previous, commitment, op_key:i.op_key})
    ```
    `w.commitment = commitment` and `w.root = root`.
14. **ANCHOR_POSITION** (CL-31 v2):
    - `w.sequence = i.expected_sequence + 1`;
    - `w.previous = i.previous_root`.

    The action is anchored exactly where the signed intent authorized it. This is the runtime's own rule (`external_anchor.py` `apply`; `anchor_ssh.py` `FOUNDER_GATEWAY_INTENT_SLOT_MISMATCH`).

→ `VERIFIED`, verdict `EXECUTED`: 6 signatures from 4 keys.

### 6.2 · NOT_EXECUTED_WITNESS_CANCELLED

7. **WITNESS_CANCELLATION:** `proof.cancellation`, with `signature` removed, equals **exactly** the binding of 6.1 step 7 plus `status:"CANCELLED"` and `op_key:i.op_key` [`WITNESS_BINDING_CONTEXT`]. Its signature verifies under `witness_public` [`WITNESS_BINDING_SIGNATURE`].

→ `VERIFIED`, verdict `NOT_EXECUTED`: 3 signatures from 2 keys.

**What this proves:** the witness signed CANCELLED for this exact record at its slot.
- It does not prove that no other action was cancelled and left out (§9).
- A timeout or a missing receipt is never NOT_EXECUTED. It is UNAVAILABLE (SEAM-02).

### 6.3 · NOT_DISPATCHED_SIGNER_PROVEN

1. `proof` has exactly `evidence`, `shows` [`RECOVERY_PROOF_SHAPE`].
2. `evidence` has exactly `attestation`, `signature`, `rows` [`RECOVERY_EVIDENCE_SHAPE`].
3. In this order:
   1. `attestation` has exactly `schema op_key nonce head_seq head_hash rows_sha256` [`RECOVERY_ATTESTATION_SHAPE`];
   2. `rows` is an array [`RECOVERY_ROWS_SHAPE`];
   3. `head_seq` is a nonnegative safe integer, 0 ≤ n < 2^53 [`INTEGER_PROFILE`];
   4. `nonce` is nonempty text [`TEXT_PROFILE`];
   5. `head_hash`, then `rows_sha256`, is 32-byte lowercase hex [`HEX_PROFILE`].
4. **RECOVERY_ROWS_BINDING:** `attestation` equals `{schema:"831.cp09.signer-query.v1", op_key:<bundle op_key>, nonce, head_seq, head_hash, rows_sha256:digest(rows)}`.
5. **RECOVERY_SIGNATURE:** `evidence.signature` verifies over `canonical(attestation)` under `signer_public`.
5a. **The rows' events** (v2.3): every row is an object whose `event` is an object, and every `event` whose `kind` is `intent` or `void` carries an object `payload` [`RECOVERY_ROWS_SHAPE`]. A row that is not an object counts as an event of `null`, which is refused.
6. **RECOVERY_SEALED:** no row's `event.kind` is `record`. The signer never sealed a dispatch record for this op.
7. **RECOVERY_UNVOIDED:** every `intent` event's payload has a matching `void` event whose `payload.intent_digest` = digest(intent payload).
8. **RECOVERY_LABEL:** `shows` is `VOIDED` if any intent exists, else `NOTHING_SIGNED`.

→ `VERIFIED`, verdict `NOT_DISPATCHED`. The action is outside the anchor chain.

**Finally,** on every path, a VERIFIED result requires that at least one signature was checked [`VERIFICATION_PATH_UNAVAILABLE`].

## 7 · Replay: many receipts, one pins file

**Input:** 1 to 64 bundles; otherwise UNAVAILABLE `REPLAY_INPUT`.

1. **REPLAY_RECEIPT:** each bundle, in input order, is verified by §6.
   - If any comes back UNAVAILABLE (§5), Replay prints that same line, exit 2. The first that is not VERIFIED makes the result `{"failing_check":"REPLAY_RECEIPT","receipt":<1-based index>,"receipt_check":<its failing check>,"state":"BROKEN"}`.
2. **REPLAY_DUPLICATE:** an `op_key` appears at most once.
3. **REPLAY_FORK:** at most one EXECUTED receipt per checkpoint sequence.
4. **REPLAY_GENESIS:** an EXECUTED receipt at sequence 1 has `previous` = 64 zeros.
5. **REPLAY_LINK:** an EXECUTED receipt at sequence n > 1 whose predecessor (n − 1) is supplied has `previous` = that predecessor's `root`.
6. **REPLAY_HEAD:** a cancelled receipt authorized at `expected_sequence` s has `previous_root` = 64 zeros if s = 0; else = the root of sequence s, when that receipt is supplied.
7. **REPLAY_GAP:** let `top` be the highest sequence among EXECUTED receipts and the highest `expected_sequence` among cancelled ones. Any sequence from 1 to `top` with no EXECUTED receipt is ABSENT, and the result is `{"absent":[…],"reason":"REPLAY_GAP","state":"UNAVAILABLE"}` (exit 2), **never VERIFIED**. This is the broken-predecessor law, ruled by the founder on Nov-2: omitted is UNAVAILABLE, contradicted is BROKEN.
8. If no EXECUTED or cancelled receipt was supplied: UNAVAILABLE `REPLAY_NO_CHAIN`.
9. **Otherwise:**
   ```
   {"chain":"COMPLETE","executed":E,"from":1,"head_root":<root at top>,"not_dispatched":D,
    "not_executed":C,"state":"VERIFIED","to":top}
   ```
   with exit 0. `from` is 0 and `head_root` is 64 zeros when `top` is 0.

**Rows**, for display: EXECUTED per sequence; NOT_EXECUTED placed at its head, with `chain_unchanged` true when the next EXECUTED receipt builds on the same head, false when it does not, null when it was not supplied; NOT_DISPATCHED listed apart.

## 8 · Policies (verifier v2.3, CL-44)

**The intent's `policy_digest` is a SHA-256 over a policy document's canonical bytes.** The signer signs it, the target repeats it, and the checkpoint commitment binds it.

**A policy file is published as exactly its canonical bytes**, so its file SHA-256 equals the digest.

**When the person checking supplies a policy file:**
- **MATCH:** the file's SHA-256 equals the receipt's `policy_digest`, and the file parses under §2. The verifier shows the policy's text.
- **MISMATCH:** any other case. The verifier says plainly that the supplied file is not the policy this receipt was signed under.

**The receipt's own verdict never changes because of a policy file.** The policy is disclosure, not proof of the receipt.

**The rehearsal and genesis receipts' operation policy:**
- `216983c233adb22a569143bd7d60752cfcf8e73c5b89e3283fd95d2f3cb7816b`
- = SHA-256 of `{"require_approval":true,"synthetic_only":true}`.

**The genesis tier's presence policy** (`4ff3a3ce…`, Mode A, `auto_before_slot 2`) is enrollment provenance. It is not in the receipt. It is published beside it (S4-GATE-05).

## 9 · What VERIFIED establishes, and what it does not

**It establishes:** the named keys, as supplied in the pins, signed these statements, and every binding of §6 recomputes on the checking device.

**It does not establish:**
- **Who holds the keys.** That is the pins' provenance (§4).
- **When the action happened.** No receipt field carries a time. A public timestamp of the published envelope bounds it from above.
- **That an effect outside the target happened.** `effect_sha256` is the payload by definition (6.1 step 9). An external effect, such as a broker order, stays UNKNOWN until a venue adapter with authoritative read-back exists (R20/B2, after the work).
- **That the policy was the right one.** Only which policy was signed (§8).
- **That no cancelled action was left out.** Cancellations are never anchored (§6.2, §7).
- **Freedom from split view across observers.** That needs the house's checkpoint heads published where every observer sees the same one (SEAM-04).

## 10 · Known divergences between the two reference verifiers (found while writing this spec, Nov-4)

**On the 25 conformance vectors, under both Ed25519 paths, the browser engine and the Python reference print identical lines** (CI run 41921577307, differential 50/50).

**On malformed JSON both refuse, but the names differ.** Probe of 10 inputs, Nov-4:

| Input | Browser engine (this spec) | Python reference v2 |
|---|---|---|
| empty file | `JSON_SYNTAX` | `Expecting value: line 1 column 1 (char 0)` |
| trailing comma | `JSON_SYNTAX` | `Illegal trailing comma before end of object: …` |
| raw control character in a string | `JSON_SYNTAX` | `Invalid control character at: …` |
| data after the object | `JSON_SYNTAX` | `Extra data: …` |
| byte-order mark | `JSON_SYNTAX` | `Unexpected UTF-8 BOM (decode using utf-8-sig): …` |
| invalid UTF-8 | `UnicodeDecodeError` | `'utf-8' codec can't decode byte 0xff …` |
| top-level array nested 40 deep | `JSON_DEPTH` | `JSON_OBJECT` |
| object nested 40 deep, float, top-level array | same in both | same in both |

**One more difference, by design:** the Python reference runs on its platform's `cryptography` library and has no self-test path, so it never prints UNAVAILABLE. The browser does, when no Ed25519 implementation passes §5.

**The Python reference is the side that is wrong:**
- it prints the decoder's message instead of the profile's name;
- it checks the object type before the depth.

Verifier v2.3 aligns it, and these inputs join the conformance set (§11), so "the two verifiers agree" covers malformed input too. Neither side ever printed VERIFIED for any of them.

**Found next by the differential fuzz** (`instrument/reference/diff_fuzz.cjs`; byte and structural mutations of real receipts, Nov-4). Each of these is a refusal printed under the wrong name:

| Input | Browser v2.2 (live) | Python v2 (live reference) | v2.3 / v2.1 |
|---|---|---|---|
| `"record": null` | `Cannot read properties of null (reading 'intent')` (an engine message; WebKit words it differently) | `TypeError` | `RECORD_SHAPE` |
| the witness `slot` written as `false` where the intent says `0` | `WITNESS_BINDING_CONTEXT` | `INTEGER_PROFILE` (Python's `False == 0` let the binding comparison pass) | `WITNESS_BINDING_CONTEXT` |

**The fixes:**
- The browser now refuses a null record or intent by name, never by an engine error.
- Python compares bindings by canonical bytes, as the browser does, not with `==`.
- Both refuse a recovery row whose event or payload is not an object as `RECOVERY_ROWS_SHAPE`. This is reachable only under a signer signature.

**After the fixes:** 10,030 of 10,030 fuzz cases print identical lines in both verifiers, and every refusal carries a name from this spec.

## 11 · Conformance

- `instrument/conformance/conformance-v2.json`: 25 receipts with expected line and exit (P1-P3, N01-N10, T01-T11, H01).
- `instrument/conformance/trace-replay-v1.json`: Trace and Replay expectations.
- **v2.3 adds:**
  - `conformance/malformed-v1.json`: 30 malformed and edge inputs, with exact lines (§2, §10);
  - `conformance/policies/` (§8): P1, content-addressed, plus two files that must not match;
  - the pins-fingerprint cases (§4): `tx1.pins.json` and a reformatted copy;
  - `instrument/reference/diff_fuzz.cjs`: a seeded differential of byte and structural mutations, which must print identical lines in both verifiers, every refusal named.

**An implementation conforms when it prints every expected line and exit code exactly.**

— Claude (`831-claude`)
