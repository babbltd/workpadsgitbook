# §5 — Codec & Compact Encoding

**Status:** v2.2 (pads-v1 `1pa` + pads-v2 `1pv` native G0–G6 — 2026-05-24)  
**Replaces:** v2.1 (bridge-only `1pv`)  
**Algorithm:** pads-v1 codebook `a`; pads-v2 codebook `v` (Path C + native groups)  
**Compression:** fflate deflateSync (DEFLATE, RFC 1951)  
**URL encoding:** base64url (no padding)  
**Cross-references:** financial-block.md, participants-block.md, transaction-classification.md  
**Kaios source:** `system/dev_refs/FRAME-SPEC.md` v1.0 (SUI-001, SUI-004)

---

## Purpose

The codec converts a workpad record to a compact string suitable for use as a URL hash fragment. This allows a record to be shared as a single URL — via SMS, QR code, clipboard, or any link transport — and decoded by any Workpads client without server involvement.

The encoded string is placed after the `#` character in the URL, making it a hash fragment. Hash fragments are not sent to servers by browsers, ensuring the record data remains client-side during transmission.

---

## URL Structure

```
https://workpads.me/p#1pa/<base64url>
```

| Component | Value | Description |
|-----------|-------|-------------|
| Origin | `https://workpads.me` | Canonical base URL |
| Path | `/p` | Workpad receiver path |
| Scheme tag | `1pa` | 3-character encoding descriptor (see below) |
| `/` | literal | Separator between scheme tag and payload |
| `<base64url>` | variable | The compressed, base64url-encoded binary frame |

### Scheme Tag

The scheme tag is always 3 ASCII characters:

| Position | Character | Meaning |
|----------|-----------|---------|
| 0 | `1` | pads format version 1 |
| 1 | `p` | codebook package a (pads-v1 full frame) |
| 2 | `a` | compression: DEFLATE via fflate |

Detection: a hash fragment matching `/^[0-9][a-z][a-z]\//` is a pads-v1 URL. Extract the payload as `hash.slice(4)`. Read char[1] to determine codebook and route to the appropriate decoder.

### Chain Reference

When a record participates in a chain, a chain reference is appended to the URL:

```
https://workpads.me/p#1pa/<base64url>&c=<chainRef>
```

The chain reference is 4 base64url characters (3 random bytes). Strip `&c=<chainRef>` before passing the payload to the frame decoder. Store `_chainRef` on decoded records for chain lookups.

### Security-Wrapped URLs

Records with sensitive content use security-tagged variants. The scheme tag position 1 changes:

| Tag | Security level |
|-----|---------------|
| `#1pa/` | Plain record (no encryption) |
| `#1pv/` | pads-v2 Path C record (see §5.2) — **preferred encode** when supported |
| `#1ps/` | Full scramble (AES-CTR + field scramble) |
| `#1ph/` | Partial scramble (header clear, data encrypted) |
| `#1pt/` | Template-keyed encryption |
| `#1pb/` | Public billboard (presentation only, no financial data) |
| `#1pf/` | Financial presentation (invoice display, pay summaries) |

See `security-wrapper.md` for the full security specification.

---

## Encoding Steps

Given a valid record object:

### Step 1 — Assemble the binary frame

Construct a byte array following the pads-v1 frame format (see below).

### Step 2 — Compress

Apply DEFLATE compression using `fflate.deflateSync`:

```js
var compressed = fflate.deflateSync(frameBytes, { level: 9 });
```

### Step 3 — base64url encode

Convert the compressed bytes to a base64url string:

```
base64url = base64(bytes)
              .replace(/\+/g, '-')
              .replace(/\//g, '_')
              .replace(/=+$/, '')
```

### Step 4 — Produce hash fragment

```
1pa/<base64url>
```

Append `&c=<chainRef>` if a chain reference is present.

---

## Decoding Steps

Given a hash fragment string:

1. Check `/^[0-9][a-z][a-z]\//.test(hash)` — reject if not matched
2. Read char[1] to determine codebook. Route `1ag/` and `1bg/` to their legacy decoders.
3. Extract payload: `hash.slice(4)` (skip the 3-char tag and `/`)
4. Reverse base64url: add `=` padding, replace `-`→`+`, `_`→`/`, then base64-decode to bytes
5. Decompress: `fflate.inflateSync(bytes)`
6. Parse the binary frame (see below) to reconstruct the record object

---

## pads-v1 Binary Frame Format

### Byte Order

All multi-byte integers are **big-endian** (most significant byte first):

| Field | Size | Encoding |
|-------|------|----------|
| Field flags | 2 bytes | big-endian uint16 |
| Scalar field length | 2 bytes | big-endian uint16 |
| Date field (COMPACT_TIME=1) | 2 bytes | uint16 days since 2000-01-01 |
| Time field (COMPACT_TIME=1) | 2 bytes | uint16 minutes since midnight |
| Amount | 3 bytes | big-endian uint24 (see §Amount Encoding below) |

### Frame Header

```
[meta1]          1 byte — ALWAYS PRESENT

  bit 7: META2_PRESENT    1=meta2 byte follows; 0=meta2 absent
  bit 6: EXT_TEMPLATE     1=ext_template bytes follow; 0=BASE_TEMPLATE in bits 5-3
  bits 5-3: BASE_TEMPLATE (when EXT=0) or EXT_SIGNAL (when EXT=1)
  bit 2: ACK_REQUEST      1=sender requests delivery acknowledgement
  bit 1: CHAIN            1=record is chained (parent ref via &c= URL suffix)
  bit 0: RECIPIENT_TYPE   0=any receiver, 1=named recipient
```

**BASE_TEMPLATE codes (EXT_TEMPLATE=0):**

| Code | Template |
|------|----------|
| `000` | Service record (no financial block) |
| `001` | Financial record (single I>O transaction) |
| `010` | Compound financial (multiple line items) |
| `011` | Contact/entity (person or organisation) |
| `100` | Document/media (file reference) |
| `101` | State Commit (snapshot — period summary, pay summary, job close) |
| `110` | Amendment (edit/revision of a prior record) |
| `111` | Generic (DOMAIN bits provide further context) |

**EXT_SIGNAL codes (EXT_TEMPLATE=1):**

| Code | Extension |
|------|-----------|
| `001` | +1 byte (256 domain types per codebook package) |
| `010` | +2 bytes (uint16 BE, 65,536 domain types) |
| `011` | +3 bytes (24-bit, 16M domain types) |
| `100` | Variant type (3 bytes: CRC-8 namespace + CRC-16 local ID, decentralised) |

```
[ext_template]   1–3 bytes — if EXT_TEMPLATE=1
```

```
[meta2]          1 byte — if META2_PRESENT=1

  bit 7: SELF_DESCRIBING  1=self-describing blocks (forward-compat signal; decoder sets flag, continues)
  bit 6: COMPACT_TIME     0=date/time as UTF-8 text; 1=uint16 binary (days/minutes)
  bit 5: HAS_TRIG_BLOCK   1=TRIG bytecode block present after participants block
  bit 4: PARTICIPANTS     1=participants block present
  bits 3-2: DOMAIN        00=none (service record, no financial)
                          01=simple (I>O perspective mode)
                          10=standard (BitLedger Account Pair mode)
                          11=hybrid (I>O + Account Pair — see §DOMAIN=11 below)
  bit 1: DRAFT            1=working draft, not finalised
  bit 0: RESTRICT_FORWARD 1=record must not be forwarded by recipient
```

### Financial Context (DOMAIN ≥ 01)

Present when DOMAIN bits ≥ 01:

```
[setup_byte]     1 byte — if DOMAIN ≥ 01

  bits 7-5: DECIMAL_POS   000=0 (whole units)  001=1  010=2 (pence/cents)
                          011=3  100=4  101=5  110=6  111=flat uint24 mode
  bits 4-3: CURRENCY      00=sender home currency  01=first codebook common
                          10=second codebook common  11=extended (currency_ext follows)
  bits 2-1: TAX_CODE      00=no tax  01=tax inclusive  10=tax exclusive  11=compound (post-MVP)
                          Note: 01/10 both require a tax_block with explicit rate+amount.
  bit 0: SF_PRESENT       1=sf_byte follows

[currency_ext]   1 byte — if CURRENCY=11 (uint8 extended currency code, 256 slots)

[sf_byte]        1 byte — if SF_PRESENT=1

  bits 7-5: SCALING_FACTOR  000=×1  001=×10  010=×100  ...  111=×1,000,000,000
  bit 4: COMPOUND_VALUE     1=compound financial block (multiple line items)
  bit 3: QTY_COMPACT        1=qty+rate packed into customer_amount uint24
  bits 2-0: SPLIT_POINT     bit count for qty when QTY_COMPACT=1 (0=default 8 bits)

[transaction_byte]  1 byte — if setup_byte present

  DOMAIN=01 (I>O simple mode):
    bit 7: DIRECTION   0=I (income)   1=O (outgoing)
    bit 6: TIME        0=Past (settled)  1=Future (pending)
    bit 5: EFFECT      0=I (net positive)  1=O (net negative)
    bits 4-3: SUBTYPE  00–11 (see transaction-classification.md)
    bit 2: QTY_SPLIT   1=qty × rate encoding active
    bits 1-0: ROUNDING 00=exact  10=down  11=up  01=error (never set)

  DOMAIN=10 (BitLedger Account Pair mode):
    bits 7-4: ACCOUNT_PAIR  0000–1101 active (14 pairs)
    bit 3: DIRECTION        0=debit primary  1=credit primary
    bit 2: STATUS           0=past/settled  1=future/pending
    bit 1: QTY_SPLIT        1=qty × rate active
    bit 0: ROUNDING         0=exact or down  1=up

  DOMAIN=11 (hybrid — same as DOMAIN=01, plus account_pair_byte follows):

[account_pair_byte]  1 byte — if DOMAIN=11

  bits 7-4: ACCOUNT_PAIR   BitLedger 4-bit Account Pair code (0000–1101 active)
  bit 3: AP_DIRECTION      0=debit primary  1=credit primary
  bit 2: AP_STATUS         0=settled  1=pending
  bit 1: AP_COMPLETENESS   0=complete  1=partial
  bit 0: AP_EXTENSION      0=none  1=account_pair_ext byte follows (reserved)
```

### Field Flags

```
[field_flags]    2 bytes — ALWAYS PRESENT
```

**Field Slot Assignments:**

| Bit | Flag mask | Field key | Encoding |
|-----|-----------|-----------|----------|
| 0 | `0x0001` | `job` | `[uint16 len][UTF-8]` |
| 1 | `0x0002` | `customer` | `[uint16 len][UTF-8]` |
| 2 | `0x0004` | `date` | uint16 days* or `[uint16][UTF-8 ISO]` |
| 3 | `0x0008` | `location` | `[uint16 len][UTF-8]` |
| 4 | `0x0010` | `meeting_time` | uint16 minutes* or `[uint16][UTF-8]` |
| 5 | `0x0020` | `start_time` | uint16 minutes* or `[uint16][UTF-8]` |
| 6 | `0x0040` | `end_time` | uint16 minutes* or `[uint16][UTF-8]` |
| 7 | `0x0080` | `customer_phone` | `[uint16 len][UTF-8]` |
| 8 | `0x0100` | `worker` | `[uint16 len][UTF-8]` |
| 9 | `0x0200` | `actions` | `[uint16 len][UTF-8]` |
| 10 | `0x0400` | `details` | `[uint16 len][UTF-8]` |
| 11 | `0x0800` | `story` | `[uint16 len][UTF-8]` |
| 12 | `0x1000` | financial block | triggers fin_control block (see financial-block.md) |
| 13 | `0x2000` | `ref_number` | `[uint8 len][UTF-8, max 255 B]` |
| 14 | `0x4000` | `due_date` | uint16 days* or `[uint16][UTF-8]` |
| 15 | `0x8000` | FLAGS3_PRESENT | field_flags3 byte follows |

`*` when COMPACT_TIME=1; otherwise text encoding

```
[field_flags3]   1 byte — if FLAGS3_PRESENT=1

  bit 0: context_label   [uint8 len][UTF-8, max 255 B]
  bit 1: tag             [uint8 len][UTF-8]
  bit 2: qty_unit        [uint8 len][UTF-8]
  bit 3: date_end        uint16 days* or [uint16][UTF-8]
  bit 4: attachment      [uint16 len][UTF-8] (URL or hash)
  bit 5: uid             [uint16 len][UTF-8]
  bit 6: url             [uint16 len][UTF-8]
  bit 7: FLAGS4_PRESENT  flags4 follows (template-defined)

[field_flags4]   1 byte — if FLAGS4_PRESENT=1 (template-defined; see FRAME-SPEC §14)
```

### Date and Time Encoding

**COMPACT_TIME=1 (binary):**
- `date`: uint16 days since 2000-01-01. Range: 2000-01-01 to ~2179.
- `time` fields: uint16 minutes since midnight (0–1439).

**COMPACT_TIME=0 (text):**
- `date`: `[uint16 len][UTF-8 ISO date]` — e.g. `"2026-05-18"` = 12 bytes total
- `time` fields: `[uint16 len][UTF-8]` — e.g. `"09:30"` = 7 bytes total

COMPACT_TIME=0 does not require meta2. When meta2 is absent, the decoder defaults COMPACT_TIME=0. Adding meta2 to signal COMPACT_TIME=1 costs 1 byte but saves 10 bytes per date field — a net 9-byte saving per record with a date.

### Actions Blob (field_flags bit 9)

```
[action_count]     uint8 — number of actions (0–20)
per action:
  [title]          [uint16 len][UTF-8]
  [notes]          [uint16 len][UTF-8]  (length 0 is valid; prefix still present)
```

### Financial Block (field_flags bit 12)

See `financial-block.md` for the complete specification.

Summary: when bit 12 is set, a `fin_control` byte follows all data fields, then `customer_amount` (uint24), optional `worker_amount` (uint24), optional `tax_block` (3 bytes), and optional `qty_rate_block` (6 bytes) or packed `customer_amount` (when QTY_COMPACT=1 in sf_byte).

### Participants Block (meta2 bit 4)

See `participants-block.md` for the complete specification.

When meta2 PARTICIPANTS=1, the participants block follows all data blocks. It encodes structured identity for each party to the record (sender, customer, workers, etc.).

### TRIG Block (meta2 bit 5)

A 1–20 byte stack machine program controlling conditional rendering. Appears after the participants block (or after the financial block if no participants block present). See `FRAME-SPEC.md §13` for the bytecode specification.

```
[trig_len]   uint8 — length of TRIG program (0–20)
[trig_bytes] N bytes — TRIG v1 bytecode
```

---

## Frame Body Layout

Blocks appear in this order:

```
[meta1]
[ext_template]              — if EXT_TEMPLATE=1
[meta2]                     — if META2_PRESENT=1
[setup_byte]                — if DOMAIN ≥ 01
[currency_ext]              — if CURRENCY=11
[sf_byte]                   — if SF_PRESENT=1
[transaction_byte]          — if setup_byte present
[account_pair_byte]         — if DOMAIN=11
[field_flags]               — always present (2 bytes)
[field_flags3]              — if FLAGS3_PRESENT=1
[field_flags4]              — if FLAGS4_PRESENT=1
[data blocks for flag bits 0–14, in ascending bit order]
[financial block]           — if field_flags bit 12 set
[participants block]        — if PARTICIPANTS=1
[TRIG block]                — if HAS_TRIG_BLOCK=1
[display_schema]            — if presentation URL (#1pb/, #1pf/)
[form_schema]               — if DISPLAY_TYPE=02 or 03
[security wrapper]          — if encrypted (#1ps/, #1ph/, #1pt/)
```

---

## DOMAIN=11 Hybrid Mode

DOMAIN=11 carries both the I>O perspective classification (worker/customer UI) and the BitLedger Account Pair classification (accounting integrations). The `setup_byte` and `transaction_byte` are identical to DOMAIN=01. One additional byte — `account_pair_byte` — follows the transaction_byte.

This allows a single record to serve both the worker's natural-language view ("Invoice sent") and an accounting integration's double-entry view ("Debit: Receivable / Credit: Income") without re-entry.

See `transaction-classification.md §8` for the Entry Type Matching Table that deterministically maps wizard entry types to Account Pair codes.

---

## Worked Example

Record: `{ job: "Fix tap", customer: "Alice", date: "2026-05-18" }` with COMPACT_TIME=1.

**meta1:** `0b10000000 = 0x80` (META2_PRESENT=1, BASE=000 service)  
**meta2:** `0b01000000 = 0x40` (COMPACT_TIME=1)  
**field_flags:** bits 0+1+2 set = `0x0007` → byte1=`0x00`, byte2=`0x07`

```
0x80              meta1
0x40              meta2
0x00 0x07         field_flags
0x00 0x07         job length = 7
46 69 78 20 74 61 70   "Fix tap"
0x00 0x05         customer length = 5
41 6C 69 63 65    "Alice"
[2 bytes]         date uint16 (days since 2000-01-01)
```

Compressed with `deflateSync` → base64url → prepend `1pa/` → append to `https://workpads.me/p#`.

**Profile A (33B raw, COMPACT_TIME=1):**
```
meta1:       1B  0x80
meta2:       1B  0x40
field_flags: 2B  (bits 0+2: job + date)
job block:   2+25 = 27B  (25-char job description)
date block:  2B   uint16 days
Total: 33B
```

**Profile B (36B raw, simple payment):**
```
meta1:             1B  0x88 (Financial template, META2=1)
meta2:             1B  0x44 (COMPACT_TIME=1, DOMAIN=01)
setup_byte:        1B  0x40 (DECIMAL_POS=2, home currency, no tax, no SF)
transaction_byte:  1B  0x00 (I<I settled, sub=00 payment, exact)
field_flags:       2B  0x1007 (job+customer+date+financial)
job block:         2+13 = 15B
customer block:    2+7 = 9B
date block:        2B
fin_control:       1B  0x12 (BILLED=0, QTY_TYPE=0, PARITY=1, EC=00, CUST=1, WORK=0)
customer_amount:   3B  uint24 = 12550 (£125.50 at DECIMAL_POS=2)
Total: 36B  →  after deflate+base64url: ~48–52 chars
```

---

## Size Characteristics

| Record content | Approx. URL fragment length |
|----------------|--------------------------|
| Service note, job only | ~40–50 chars |
| Job + customer + date (COMPACT_TIME=1) | ~50–65 chars |
| Simple payment (Profile B) | ~48–55 chars |
| Invoice with qty/rate | ~55–70 chars |
| Full record with participants | ~90–120 chars |

pads-v1 frames compress 20–30% better than `1eg/` frames due to binary integer encoding (no repeated decimal digits) and more predictable field structure.

---

## Amount Encoding (SUI-012)

### uint24 + DECIMAL_POS

All financial amounts are encoded as big-endian 24-bit unsigned integers (0–16,777,215). The actual decimal value is recovered by applying `DECIMAL_POS` from the `setup_byte`:

```
decoded_value = uint24_raw / 10^DECIMAL_POS

Examples at DECIMAL_POS=2 (pence/cents precision):
  uint24 = 12550 → £125.50
  uint24 = 100   → £1.00
  uint24 = 5000  → £50.00

Maximum representable at DECIMAL_POS=2: 167,772.15 (any currency unit)
Maximum representable at DECIMAL_POS=0: 16,777,215 (whole units only)
```

`DECIMAL_POS=111` (binary) = flat uint24 mode: the raw integer IS the value with no decimal interpretation. Used for quantities and identifiers stored as uint24 when no decimal scaling applies.

### BitLedger Lineage

The uint24 amount encoding is derived from the **BitLedger Layer 3** scaled value format: `N = A × 2^S + r`, where `N` is the stored value, `A` is the scaled mantissa, `S` is the scaling exponent, and `r` is a rounding remainder. In pads-v1, the BitLedger encoding simplifies to a direct uint24 (no mantissa/exponent split at the record layer — the DECIMAL_POS in the setup_byte carries the scaling information once per record).

For multi-record batches where a common scaling factor applies to all records, BitLedger Layer 2 batch headers carry the currency, decimal position, and scaling factor once for the whole batch. Pads-v1 single records carry these inline in `setup_byte`.

### Legacy comparison

| Codec | Amount encoding | Bytes per amount |
|-------|----------------|-----------------|
| `1ag/` (codebook a) | UTF-8 string via uint16 length prefix | 2 + len (e.g. "125.50" = 8 bytes) |
| `1bg/` (codebook b) | UTF-8 string via uint16 length prefix | 2 + len |
| `1pa/` (pads-v1, current) | big-endian uint24 + DECIMAL_POS in setup_byte | 3 bytes always |

The move to uint24 saves ~5 bytes per amount field and removes variable-length string parsing from the hot decode path.

---

## pads-v2 (`#1pv/`) — Path C + native groups (v0.4)

**SUI:** 021–027 (2026-05-24). **Kaios reference:** `pathc-v2.js`, `pathc-native.js`, `native-v1-split.js`, `FRAME-SPEC-1pv-ADDENDUM.md`. **Default wire:** native G0–G6 group chunks (phase 2b). Legacy bridge v1 (flag bit 0x01) decode-only.

### URL

```
https://workpads.me/p#1pv/<base64url-deflated-inner-frame>[&c=<chainRef>][&r=<ratified>]
```

Same deflate + base64url pipeline as `1pa/`. Dual-decode: all clients **must** decode both `1pa/` and `1pv/`. Encoders **should** default to `1pv/` when Path C is supported.

### Scheme tag

| Position | Char | Meaning |
|----------|------|---------|
| 0 | `1` | pads format generation 1 |
| 1 | `p` | pads family |
| 2 | `v` | codebook v — Path C header + bridge |

### Inner frame (after deflate) — native default (v0.4 phase 2b)

```
[flag_byte]
[path_c_header]     ; 2 bytes (shortcut) or 3 bytes (standard)
[presence u8]        ; bits 0–6 = G0–G6 present
[u8 hdr_len][meta prefix]?   ; if flag bit 5 (0x20) HAS_HDR — legacy meta1/meta2 during migration
[u16_le chunk][group_body]  ; emit order: G1, G0, G2, G3, G4, G5, G6
[trail?]             ; relationship / ack_masks / v4 ext
[crc16_le?]          ; if flag bit 7 (0x80)
```

Each **group_body** carries that group’s **subset of field_flags + FLAGS3/4 + field payloads** (not a full monolithic v1 frame). Decoder merges groups to pads-v1 field stream for field parsing. See **§5.3**.

#### flag_byte (native)

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | BRIDGE_V1 | **Legacy:** uint16 v1 embed follows (v0.3 only) |
| 1 | _(reserved)_ | |
| 2 | _(trail)_ | See trail_flags below |
| 4 | GROUP_LOCAL | Each group body prefixed with `u8 group_local_flags` (phase 3) |
| 5 | HAS_HDR | Length-prefixed meta1/meta2 prefix before groups |
| 6 | HAS_TRAIL | Trail block before CRC |
| 7 | HAS_CRC | CRC-16-CCITT LE over all preceding bytes |

**group_local_flags:** bit0 = programmable rules (G6); bit1 = informational_ack (G0); bit2 = display schema (G6). **Light ack wire:** `relationship: acknowledges` + `confirmed_mask=0` + `declined_mask=0`.

**Trail** (when HAS_TRAIL): `trail_flags` u8 — bit1 relationship, bit2 ack_masks (4 bytes), bit3 v4 ext.

#### Legacy bridge v1 (flag bit 0 set)

```
[flag_byte]  ; bit 0 = 1
[path_c_header]
[u16_le v1_length]
[pads-v1 frame]
[relationship?] [ack_masks?] [crc16?]
```

Dual-decode required indefinitely.

CRC algorithm: CRC-16-CCITT (poly 0x1021), init 0xFFFF, reflected per byte — same as `pathc-v2.js`.

#### Path C header — standard path (3 bytes)

| Byte | Content |
|------|---------|
| 0 | `record_type` byte (see table below) |
| 1 | bit0 priority; bits 4–6 `chain_mode` on invoice/quote only |
| 2 | D-byte: draft (bit 2), ack_request (bit 4), restrict_forward (bit 5) |

`chain_mode` (3 bits): 0 INITIATING, 1 LIVE, 2 INFORMATIONAL, 3 CLOSING, 4 DISPUTING, 5 WITNESSING.

#### Path C header — shortcut path (2 bytes)

Byte 0 = `0x00`; byte 1 = `(type_nibble << 4) | d_bits` for compact invoice/work_record/note/log/payment when no `chain_mode` override.

#### record_type byte 0 (v0.3)

| Type | Byte |
|------|------|
| invoice | 0x01 |
| quote | 0x02 |
| work_record | 0x03 |
| payment | 0x07 |
| need | 0x16 |
| offer | 0x17 |
| connection | 0x18 |
| job (default) | 0x21 |

Full table in `pathc-v2.js` `TYPE_TO_BYTE`.

#### relationship byte (bridge extension)

4-bit core + 4-bit subtype (Doc 8). Unknown core values (>7) decode as `responds` with `_relationshipUnknown: true`.

| Core | Name |
|------|------|
| 0 | creates |
| 1 | amends |
| 2 | acknowledges |
| 3 | pays |
| 4 | disputes |
| 5 | reverses |
| 6 | responds |
| 7 | confirms |

Encoders may set `relationship` explicitly or rely on inference from `record_type` (e.g. `payment` → `pays`, `ack` → `acknowledges`).

#### Ack masks (bridge extension)

On `relationship: acknowledges`:

| Field | Width |
|-------|-------|
| confirmed_mask | uint16 LE |
| declined_mask | uint16 LE |

Up to **16** indexed actions (Doc 6 §8.2). Connection light ack: `acknowledges` with both masks zero.

#### Share-time behaviour (unchanged wire, v1 + URL suffix)

| Feature | Behaviour |
|---------|-----------|
| `changedMask` | Amendment sparse payload — `baseTemplate` 6, derived at share from field diff |
| `_ratifiedFrame` | State commit: `&r=<deflated-ratified-frame>` URL suffix |
| `chainRef` | `&c=<4-char ref>` when meta1 CHAIN=1 |

These apply to `1pa/` and `1pv/` equally; Path C adds header-level `chain_mode` and bridge-level `relationship`.

### 5.3 Native groups (G0–G6)

After Path C header, parser reads **`presence`** (1 byte; bits 0–6 = G0–G6). Each set bit is followed by **`u16_le` length** and group body bytes. Emit order on wire: **G1, G0, G2, G3, G4, G5, G6**. Optional **`HAS_HDR` (0x20)** length-prefixed meta1/meta2 before chunks during migration.

| Gn | Name | Role |
|----|------|------|
| G0 | Identity | job, customer, worker, ref, uid, context |
| G1 | Financial | setup_byte, financial block |
| G2 | Time & place | date, location, times, due_date |
| G3 | References | tags, attachment, url, service_ref |
| G4 | Work content | actions, details |
| G5 | Narrative & parties | story, participants block |
| G6 | Chain & extensions | TRIG, display_schema, programmable_rules (`0x50`) |

**Group body (phase 2b):** per-group subset of `field_flags` + optional FLAGS3/4 + field payloads for that group only. Encoders MUST NOT place non-G0 field payloads only inside G0.

**Mandatory groups** (implied by record type; presence bit required; zero-length chunk allowed):

| Type | Mandatory |
|------|-----------|
| invoice, quote, receipt, credit_note | G0, G1, G6 |
| work_record, task, need, offer, job | G0, G4 |
| note, log, broadcast, report, connection | G0, G5 |
| payment, expense, state_commit | G0, G1 |
| schedule | G0, G2 |
| template | G0, G6 |
| contact | G0 |
| contract | G0, G4, G6 |
| order | G0, G1, G4 |
| ack | G0, G6 |

Full matrix: [`native-groups-mandatory.json`](native-groups-mandatory.json). Field→group bit map: `workpadskaios/system/dev_refs/NATIVE-GROUPS-TABLE.md` §4.

**Trail** (after groups, flag `HAS_TRAIL`): `trail_flags` — bit1 relationship, bit2 ack_masks (4 bytes), bit3 v4_ext.

**Legacy bridge v1:** flag bit 0 — full pads-v1 embed; decode indefinitely; encode opt-in only.

### Conformance vectors

`test/fixtures/1pv-vectors.json` — version `1pv-native-2b`, eight scenarios (invoice, ack, payment, schedule, work_record, note, need, job). Mirrored in `workpadskaios/test/fixtures/`. Regenerate: `node scripts/regen-1pv-vectors.js` (kaios).

---

## Implementations

| Implementation | Location | Environment |
|----------------|----------|-------------|
| Browser codec | `workpadskaios/js/lib/codec.js` — `window.WPCodec` | Browser (fflate UMD) |
| Path C bridge | `workpadskaios/js/lib/pathc-v2.js` — `window.WPPathC` | Browser — load before codec.js |
| Native groups | `native-v1-split.js`, `pathc-native.js`, `native-groups-table.js` | Same load order as pathc |
| npm package | `workpads-codec` — `encode({ padsV2: true })` native default | Node / CLI |
| Anonymous mode helpers | `workpadskaios/js/lib/anon.js` — `window.WPAnon` | Browser |
| Security wrapper | `workpadskaios/js/lib/security.js` | Browser |

All implementations must produce frames that decode correctly in all other implementations.

---

## Backwards Compatibility

Legacy URLs remain decodable indefinitely:

| Tag | Codebook | Status |
|-----|----------|--------|
| `1ag/` | Codebook a (pre-financial) | Legacy — amounts as UTF-8 scalars |
| `1bg/` | Codebook b (financial block with fin_flags) | Legacy — 3-byte flat header |
| `1pa/` | pads-v1 package a (current) | Active |

Decoder routing by scheme tag char[1]:

```js
var c = hash[1];
if      (c === 'a') return decodeLegacyA(frame);  // 1ag/
else if (c === 'b') return decodeLegacyB(frame);  // 1bg/
else if (c === 'p') return decodePadsV1(frame);   // 1pa/ (and 1pb/, 1ps/, etc.)
```

New encoders emit `1pa/` (or the appropriate security variant). Existing legacy URLs decode indefinitely.

---

## Codec Identity

This codec is registered as:

```
pads-v1 / codebook package a / 1pa
pads-v2 / codebook package v / 1pv   (Path C header + native G0–G6 — v0.4)
```

**Kaios source authority:** `FRAME-SPEC.md` v1.0 for `1pa/` field blocks; `FRAME-SPEC-1pv-ADDENDUM.md` for `1pv/` header and bridge. On conflict, kaios `dev_refs` wins until merged into this standard.
