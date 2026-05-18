# Chain Protocol Specification

**Status:** v0.1 — 2026-04-27
**Decision source:** decisions.md R-PADS-012, R-PADS-013, R-PADS-014, R-PADS-015, R-PADS-016, R-PADS-019
**Cross-references:** pads-v2-encoding-spec.md §8, §11, §12, participants-block.md, transaction-classification.md

---

## 1. What a Chain Is

A chain is a sequence of related workpad records grouped by a shared anchor. Records in a chain speak to one another — an initial quote leads to a job record, which leads to an invoice, which leads to a completion certificate. Each record is self-contained and URL-transmissible. The chain is the relationship between them.

A chain is not a database table, a conversation thread, or a folder. It is a lightweight reference system: a 24-bit anchor value that each record in the chain shares, plus a sequence number indicating order.

Records can exist without a chain. A standalone quote sent once, with no follow-up, is a complete record with no chain reference. Adding a chain is the worker's deliberate act of linking records in a sequence.

---

## 2. Chain ID: 24-bit Composite

The chain reference is a 24-bit composite value encoded as 4 base64url characters.

### 2.1 Bit layout

| Bits | Name | Width | Meaning |
|------|------|-------|---------|
| 23–7 | ANCHOR | 17 bits | Device-specific chain origin |
| 6–3 | PARTICIPANT_SLOT | 4 bits | Which participant's copy of the chain |
| 2–0 | SEQUENCE | 3 bits | Position in chain (0–7) |

**ANCHOR:** 17 bits = 131,072 unique anchor values per device. Derived from device-specific entropy (see §2.3). The anchor identifies a specific chain — all records in the same chain share the same anchor.

**PARTICIPANT_SLOT:** 4 bits = 16 participant copies. Slot 0 = the sender's authoritative copy. Slots 1–15 = copies distributed to other participants (workers, customers, colleagues). Pre-seeding N worker URLs at share time means each worker receives a URL where only the PARTICIPANT_SLOT differs — the binary frame data is identical. Workers' apps store their copy indexed by their slot.

**SEQUENCE:** 3 bits = 8 positions (0–7) within a chain. Sequence 0 = the chain anchor record (the first record that established the chain). Sequence 1–6 = subsequent records. Sequence 7 = chain wrap or State Commit closing record.

### 2.2 Encoding

The 24-bit value is encoded as 4 base64url characters:
```
bytes[0] = (chain_id >> 16) & 0xFF
bytes[1] = (chain_id >> 8) & 0xFF
bytes[2] = chain_id & 0xFF
chain_ref = base64url(bytes[0..2])  → always exactly 4 chars
```

In the URL: appended as `&c=XXXX` after the data.

**Full URL example:**
```
workpads.me/p#1ag/eJyLjgUA...&c=AbCd
```

### 2.3 Anchor generation

Anchors are generated at first chain creation using `crypto.getRandomValues()`:
```javascript
var buf = new Uint8Array(4);
crypto.getRandomValues(buf);
var device_bits = (buf[0] << 4) | (buf[1] >> 4);       // 12 bits random
var install_bits = (Date.now() / 86400000) & 0x1F;     // 5 bits: day modulo 32
var anchor = ((device_bits << 5) | install_bits) & 0x1FFFF;  // 17 bits
```

`crypto.getRandomValues()` is available in KaiOS 3.x (Gecko 63+) and KaiOS 2.5 (Gecko 48 — available from Gecko 21). The anchor is stored as `wp_device_anchor` in localStorage and reused for all chains from this device. Each new chain uses the same device anchor but a new SEQUENCE starting at 0. Multiple concurrent chains are distinguished by their SEQUENCE bit allocation — the anchor alone does not differentiate chains; the SEQUENCE does.

**Wait:** actually, multiple concurrent chains from the same device share the same 17-bit anchor. The anchor is a device identity, not a per-chain identifier. Chain identity = anchor + the first record's timestamp, not anchor + sequence alone.

**Correction (design clarification):** For v0.1, the chain system assumes a single active chain per device at a time (one job, sequential records). Multiple concurrent chains are v0.2 scope. The chain index in localStorage maps anchors to record arrays — if two concurrent chains share the same anchor, they are distinguished by their first-record timestamp. This limitation is acceptable for v0.1 and will be resolved with a per-chain nonce in v0.2.

---

## 3. Chain Reference in the URL

When CHAIN bit (meta1 bit 1) = 1, the URL carries `&c=XXXX` after the data.

**Detector:** presence of `&c=` after the base64url data portion. If absent, the record is standalone (no chain reference).

**Decoder:**
```javascript
var hash = url.slice(url.indexOf('#') + 1);
var dataEnd = hash.indexOf('&c=');
var data = dataEnd !== -1 ? hash.slice(4, dataEnd) : hash.slice(4);  // skip "1ag/"
var chainRef = dataEnd !== -1 ? hash.slice(dataEnd + 3) : null;
```

If chainRef is present, decode 4 base64url chars → 3 bytes → extract ANCHOR (17 bits), PARTICIPANT_SLOT (4 bits), SEQUENCE (3 bits).

---

## 4. Chain Storage

Records in a chain are stored locally as binary frames (raw pads-v1 frame bytes, before compression). URLs are reconstructed on demand.

### 4.1 localStorage structure

```javascript
// Chain index: one key per anchor
"wp_chain_<anchor_hex>": JSON.stringify({
  anchor: 131234,              // the 17-bit anchor value
  created: 1714204800000,      // first record timestamp (epoch ms)
  records: [
    {
      seq: 0,
      record_id: "wp_1714204800000abc4",
      frame: "0102...",        // hex-encoded frame bytes
      timestamp: 1714204800000
    },
    {
      seq: 1,
      record_id: "wp_1714291200000def8",
      frame: "0102...",
      timestamp: 1714291200000
    }
  ],
  participants: [
    { slot: 0, name: "Alice Plumb", role: "worker" }
  ]
})
```

### 4.2 Storage estimates

- Average frame (T2-level record, 5 fields): ~156 bytes raw → hex: ~312 chars
- Chain index entry (2 records, 1 participant): ~700 chars JSON
- 1000 chains × 700 chars: ~700KB total
- With localStorage overhead: ~800KB — well within KaiOS 3.x localStorage quota (typically 5–10MB)

For v0.1 with one active chain at a time: storage impact is negligible.

### 4.3 URL reconstruction

When the worker shares a record that is part of a chain:
```javascript
function buildChainUrl(frame, anchor, participant_slot, seq, codebook, compression) {
  var compressed = fflate.deflateSync(frame, { level: 9 });
  var data = base64url(compressed);
  var chain_id = (anchor << 7) | (participant_slot << 3) | seq;
  var chain_ref = base64url([(chain_id >> 16) & 0xFF, (chain_id >> 8) & 0xFF, chain_id & 0xFF]);
  return 'workpads.me/p#' + codebook[0] + codebook[1] + compression + '/' + data + '&c=' + chain_ref;
}
```

Different participant slots (for multi-worker pre-seeding) change only the `chain_ref` — the compressed data is identical. This makes multi-worker URL generation O(N) in the number of participants with O(1) compression cost.

---

## 5. Multi-Worker Pre-Seeding

When a supervisor dispatches a job to multiple field workers, the supervisor pre-seeds participant slot URLs at share time:

1. Compress the frame once
2. For each worker (participant slots 1, 2, ...):
   - Compute chain_id with their slot
   - Append `&c=<slot_chain_ref>`
   - Send that URL to that worker

Each worker's app receives a URL with their slot identifier. The app stores the record indexed by their slot. When the worker later shares their version (e.g. after completing the job), they use their slot's chain_ref.

The supervisor (slot 0) holds the authoritative chain view. The workers (slots 1+) hold their copies. All share the same anchor — the chain index on any device with all the records can reconstruct the full job history.

---

## 6. ACK Mechanism

### 6.1 ACK request

When ACK_REQUEST bit (meta1 bit 2) = 1, the recipient's app presents an ACK prompt after decoding the record:

```
"[Job: Fix boiler — Alice Plumb]
 Confirm receipt?
 Your name (optional): [___________]

 [CSK: Send ACK] [RSK: Skip]"
```

Tapping "Send ACK" generates an ACK URL. The worker copies it and sends it back via the same channel (WhatsApp, SMS) the original workpad arrived on.

### 6.2 ACK URL structure

An ACK URL is a pads-v1 record with:
- Template: State Commit (0xD), subtype=00 (completion acknowledgement)
- CHAIN=1 (references the original record's chain anchor)
- Field flags: job (bit 0, carries "ACK: " + original job title), date (bit 2)
- Participants block (if recipient entered their name): one participant, IS_SENDER=1, role=site contact (100)
- No financial block

ACK URLs are very short — typically 60–80 chars. Example:
```
workpads.me/p#1ag/eJy...&c=AbCd
```
where `&c=AbCd` is the original chain reference with PARTICIPANT_SLOT = the recipient's slot.

### 6.3 ACK receipt

The sender's app, upon receiving an ACK URL (detecting template=0xD + chain match):
1. Matches the chain anchor to a local chain record
2. Stores the ACK as the next sequence in the chain
3. Updates the chain index with ACK status
4. Displays: "ACK received from [name if present] — [timestamp]"

### 6.4 ACK as proof

An ACK record is lightweight proof that the recipient received and opened the workpad. It is not a digital signature. It is:
- Stronger than a read receipt (the recipient actively generated a URL)
- Weaker than a cryptographic acknowledgement (could be spoofed)
- Appropriate for: job completion confirmations, invoice delivery acknowledgement, handoff acceptance

For v0.1, ACK is a social protocol, not a cryptographic one. Future versions may add a challenge-response mechanism.

---

## 7. State Commit Records in Chains

State Commit records (template 0xD) are the natural closing records in a chain. They represent a bilateral agreement: "at this moment, the relationship is in this state."

### 7.1 Job completion certificate

**Scenario:** Alice completes a plumbing job. She creates a State Commit record carrying: job title, date, details ("All work completed — no leaks, tested"), customer name. She sends it to the customer with ACK_REQUEST=1.

The customer receives a workpad URL. They view the completion certificate. They optionally tap "Send ACK" — this is their digital acknowledgement of the completion.

Alice's chain now contains: [quote → job record → invoice → completion cert + customer ACK]. The full commercial history is in the chain.

### 7.2 Pay summary

A pay summary State Commit carries a financial block summarizing work done and amounts. Used by workers to create a formal record for their own files (proof of income) or to share with an employer/accountant.

### 7.3 Dispute snapshot

If a customer disputes a charge, the worker creates a Dispute Snapshot State Commit referencing the original chain. The snapshot freezes the state of all records in the chain at the moment of dispute. Both parties can view the same snapshot — the chain anchor makes it clear they are looking at the same records.

---

## 8. Amendment Records in Chains

Amendment records (template 0xE) carry only the changed fields.

### 8.1 Amendment sequence

Original record: chain anchor, sequence=0.
Amendment: same chain anchor, same participant slot, sequence=1.

The decoder, on receiving a record with template=0xE, looks up the chain index for the matching anchor, retrieves sequence=0 (the original), and merges: all fields from original, overwritten by fields present in the amendment.

### 8.2 What can be amended

Any field in the original record except: chain anchor, original timestamp, original IS_SENDER identity, template type. These four are frozen at the moment of chain anchoring.

**Practical amendments:**
- Corrected invoice amount (customer_amount)
- Updated completion date
- Added actions that were omitted
- Corrected job title (spelling error)
- Updated customer phone

### 8.3 Amendment display

When a viewer decodes a record that has amendments in the chain:
- Show the current (merged) state as the primary display
- Show "Amended — [date]" notice
- Offer "View original" as an expandable section

The amendment history is the chain sequence. Any viewer with all the chain records can reconstruct the full amendment history.

---

## 9. Chain Index Management

### 9.1 Chain index size limit

For v0.1, no hard limit is enforced. The management screen (Records tab) shows: chain count, estimated chain storage. Workers can clear old chains from the management screen.

### 9.2 Chain expiry

No automatic chain expiry in v0.1. Workers decide when to archive or clear chains. EXPIRY bit (meta2 bit 1) is reserved for future use — it will signal that a chain anchor should be treated as expired after a certain date (encoded in a future meta extension).

### 9.3 Chain index cleanup

When a worker archives a record (soft delete), its chain index entry is not removed — the chain index references all versions. Archived records in a chain appear as "[archived]" in the chain history view. Permanent deletion (management screen → clear archive) removes the frame bytes but retains the chain index entry as a stub (seq, record_id, timestamp, no frame).

---

## 10. Notes on Design

### Why 4 base64url chars (not 3 or 6)?

4 chars = 24 bits. 3 chars = 18 bits = 262,144 combinations — enough for device anchor + sequence but no participant slot. 6 chars = 36 bits — more than needed, adds URL cost. 24 bits at 4 chars is the optimal balance: 131K device anchors, 16 participant slots, 8 sequences, zero URL overhead beyond `&c=` prefix.

### Why not encode the chain reference in the frame?

Encoding the chain reference in the frame would mean the frame changes for each participant (different slot = different frame = different compression output). Appending it as a URL parameter keeps the frame identical for all participants — one compress operation, N participant URLs.

### Why binary frames in storage (not URLs)?

URLs require decompression to access the frame. Storing frames means: read frame → compute chain_ref for desired slot → compress → base64url → prepend tag. Three steps vs. five. Also: frame storage is smaller than URL storage by ~35% (no base64 overhead, no scheme tag).

### Why 8 sequence positions (3 bits)?

Most job chains have 3–5 records: quote → booking → job → invoice → completion. 8 is sufficient for all practical chains. For chains that extend beyond 8 records, the sequence wraps — sequence=7 becomes a "chain continuation" State Commit that anchors a new chain with a new anchor. This keeps the chain reference compact while allowing unlimited chain length.

---

## 11. Agreement Chain Extension (SUI-018)

**Status:** v0.2 addition — 2026-05-18  
**Kaios source:** `dev_refs/FRAME-SPEC.md` §9–10; `draft_specs/AGREEMENTS-DESIGN.md`

The chain protocol is the foundation for the agreement layer. This section documents how State Commit and Amendment records encode agreement state within a chain.

### 11.1 State Commit Wire Format

When `BASE_TEMPLATE=101` (State Commit), a `state_commit` byte follows the field_flags block:

```
[state_commit]    1 byte — present when BASE_TEMPLATE=101

  bits 7-6: COMMIT_TYPE
    00 = job close        (job finalised; balance outstanding)
    01 = payment confirmed (settlement of a specific amount received/made)
    10 = terms agreed     (bilateral acceptance of a quoted/proposed record)
    11 = reserved         (disputes handled via Amendment + DISPUTE_FLAG, not here)

  bits 5-4: PERIOD_TYPE   (meaningful for COMMIT_TYPE=10)
    00 = calendar month   01 = tax week
    10 = tax month        11 = custom (date_end field required)

  bit 3: CHAIN_COMPLETE   1=all chained child records settled; no outstanding balance
  bit 2: DISPUTE_FLAG     1=at least one chained record is in dispute / uncorrected
  bits 1-0: reserved
```

**COMMIT_TYPE semantics:**

| Code | Name | Meaning | Typical trigger |
|------|------|---------|----------------|
| 00 | Job close | Job is done; invoice issued or balance noted | Worker taps "Close job" |
| 01 | Payment confirmed | Specific payment received or made | Worker records cash/transfer received |
| 10 | Terms agreed | Both parties have accepted the offer/quote | Party B's acceptance reply |
| 11 | Reserved | — | Not used; disputes use Amendment |

### 11.2 Amendment Wire Format

When `BASE_TEMPLATE=110` (Amendment), the frame omits `field_flags` and uses `changed_mask` bytes instead. An optional `amendment_flags` extension byte signals dispute and parent linkage:

```
[amendment_flags]   1 byte — present when flagged in changed_mask byte 2 bit 14

  bit 7: HAS_PARENT_UID   1=8-byte parent_uid follows after changed masks
  bit 6: DISPUTE_LINK     1=this amendment is a dispute record
  bits 5-0: reserved
```

When `HAS_PARENT_UID=1`, an 8-byte parent identifier follows:
```
[parent_uid]        8 bytes — SHA-256(parent_frame_bytes)[0:8]
```

`parent_uid` is mandatory when writing to a Marker (`#1pm/`) or for financial amendments (BASE_TEMPLATE=110 + DOMAIN≥01). Optional for web-shared amendments where the `&c=` URL suffix is sufficient.

### 11.3 Dispute Records

A dispute is an Amendment record with `DISPUTE_LINK=1` + `HAS_PARENT_UID=1`. It links into the agreement trail via `parent_uid` without polluting the State Commit sequence.

```
Agreement chain with dispute:

  [offer record, seq=0]
  [acceptance State Commit COMMIT_TYPE=10, seq=1]  → Agreement RATIFIED
  [dispute Amendment DISPUTE_LINK=1, seq=2]         → Agreement DISPUTED
  [resolution State Commit COMMIT_TYPE=10, seq=3]   → Agreement RESOLVED
```

The shell displays a dispute badge on the parent record when a `DISPUTE_LINK` amendment is detected anywhere in the chain.

### 11.4 C-TRIG in Chain Records

Chain records may carry a C-TRIG block in the TRIG block position (meta2 bit 5, `HAS_TRIG_BLOCK=1`). The header byte's MODE bit (bit 5) distinguishes display TRIG (MODE=0) from commitment C-TRIG (MODE=1).

A State Commit carrying C-TRIG encodes conditional release logic evaluated by the C-TRIG evaluator on receipt. See `ctrig-evaluator-spec.md` for the full instruction set.

### 11.5 Agreement Chain Lifecycle

```
Offer record (ACK_REQUEST=1, CHAIN=0)
  ↓ Party B replies
State Commit (COMMIT_TYPE=10, CHAIN=1, &c=<offer_uid>)
  ↓ Bilateral ratification detected
RATIFIED — optional Marker write
  ↓ Either party disputes
Amendment (DISPUTE_LINK=1, HAS_PARENT_UID=1)
  ↓ Server arbitration or off-protocol resolution
State Commit (COMMIT_TYPE=10 or 00, CHAIN_COMPLETE=1)
FULFILLED
```

See `agreements-spec.md` for the full agreement protocol and `markers-spec.md` for Marker integration.
