# Codec Sync Protocol

**Status:** v1.1 (2026-05-18) — updated for pads-v1 (`1pa` codebook); SUI-019  
**Source:** Adapted from workpadsdotme/system/SYNC.md  
**Applies to:** All Workpads implementations that inline or bundle the pads-v1 codec

---

## The Problem

The pads-v1 codec (`codec.md` §5) is implemented in multiple places across the Workpads ecosystem:

| Location | Codec copy | Purpose |
|----------|------------|---------|
| `workpads-codec` npm package | Canonical source | Used by CLI and as reference |
| `workpadskaios/js/lib/codec.js` | Inline UMD bundle | KaiOS runtime (no npm) — primary active implementation |
| `workpadskaios/js/lib/anon.js` | Inline UMD bundle | Anonymous mode helpers (DATA_SOURCE=11) |
| `workpadskaios/js/lib/security.js` | Inline UMD bundle | Security wrapper (5-layer stack) |
| `workpadsdotme/js/lib/codec.js` | Inline browser bundle | Web app main codec |
| `workpadsdotme/p/index.html` | Inline decode-only | Standalone receiver (no app required) |
| `workpadsdotme/p/customer.html` | Inline encode+decode | Customer ACK flow |

When the spec changes — a field is added, a slot is reassigned, the actions blob format is updated — every copy must be updated together. A missed copy produces a decoder that silently mis-parses records from newer encoders, or an encoder generating URLs no older receiver can decode.

The canonical specification is always `codec.md`. Code is truth; spec is spec. When they conflict, investigate before deciding which to change.

---

## What Must Stay in Sync

These are the codec elements most likely to drift:

### 1. Scheme tag regex

The scheme tag identifies the codec generation. All copies must parse the same tag.

```js
// Canonical (v1.1 — pads-v1 / 1pa codebook)
var SCHEME = /^(?:https?:\/\/workpads\.me\/p[/?]?)?#?1pa\//;

// Legacy support (decode-only — route 1ag/ and 1bg/ to their legacy decoders)
var SCHEME_LEGACY = /^(?:https?:\/\/workpads\.me\/p[/?]?)?#?(1ag|1bg)\//;
```

Active codebook: `1pa` — pads-v1, package a. Supersedes `1ag/` (codebook a, pre-financial) and `1bg/` (codebook b, financial block with fin_flags).

Routing by scheme tag char[1]:
- `'a'` → legacy decoder (codebook a, `1ag/`)
- `'b'` → legacy decoder (codebook b, `1bg/`)  
- `'p'` → pads-v1 decoder (current, `1pa/`)

If the scheme tag changes (new codebook char), update this regex in every copy simultaneously. A mismatch here causes complete decode failure — no partial degradation.

### 2. Field flags layout

In pads-v1, field presence is encoded across up to four bytes: `field_flags` (16 bits), and optional `field_flags3` (FLAGS3, 8 bits) and `field_flags4` (FLAGS4, 8 bits) when FLAGS3_PRESENT and FLAGS4_PRESENT are set.

#### field_flags (bits 0–15) — all template types

```js
// Canonical pads-v1 — 16 bits, big-endian uint16
// bit 12 = financial block (FIN_BLOCK) — when set, financial block follows data fields
// bit 9  = actions blob (not a scalar field)
var FIELD_FLAGS_NAMES = [
  'job',            // bit 0
  'customer',       // bit 1
  'date',           // bit 2
  'location',       // bit 3
  'start_time',     // bit 4
  'end_time',       // bit 5
  'meeting_time',   // bit 6
  'customer_phone', // bit 7
  'worker',         // bit 8
  // bit 9 = actions blob
  'details',        // bit 10
  'story',          // bit 11
  // bit 12 = financial block present (FIN_BLOCK)
  'ref_number',     // bit 13
  'due_date',       // bit 14
  'context_label',  // bit 15
];
```

#### FLAGS3 (bits 0–7) — when FLAGS3_PRESENT=1 in field_flags byte 2

```
bit 0: context_label   short display label
bit 1: tag             comma-separated tags (includes proj: prefixes)
bit 2: date_end        end date (uint16 COMPACT_TIME)
bit 3: expiry_date     offer/record expiry date (uint16 COMPACT_TIME)
bit 4: attachment      attachment reference URL (see attachment-spec.md)
bit 5: uid             record/contact UID
bit 6: url             associated URL
bit 7: FLAGS4_PRESENT  1=flags4 byte follows
```

#### FLAGS4 (bits 0–7) — template-defined; standard cross-template assignments

```
bit 2: gps_binary      compact GPS — [int16 lat×100][int16 lon×100] = 4 bytes (financial template)
bit 3: (preamble hint) preamble byte HKDF_KEY context — see security-wrapper.md
bits 0-1, 4-7: template-defined (see template-system.md §custom_fields)
```

**Field bit stability:** bit positions in field_flags are frozen for the lifetime of the `1pa` codebook. Never reuse a bit position once assigned. To add a new standard field: assign the next available bit in FLAGS3 or FLAGS4, document it in record-schema.md, and bump the codebook char only if the new bit conflicts with an existing implementation assumption.

### 3. Actions blob format

The actions blob structure (at bit 9) must be identical in all copies:

```
[count: uint8]
  for each action:
    [title_len: uint16 big-endian]
    [title: UTF-8 bytes]
    [notes_len: uint16 big-endian]
    [notes: UTF-8 bytes]
```

Max actions: 20. A count byte > 20 is a decode error.

### 4. Frame header — meta1 byte

The first byte of every pads-v1 frame is `meta1`, not a raw template byte:

```
meta1 bit layout:
  bit 7: META2_PRESENT    1=meta2 byte follows
  bit 6: EXT_TEMPLATE     1=ext_template signal in bits 5-3
  bits 5-3: BASE_TEMPLATE (EXT=0) or EXT_SIGNAL (EXT=1)
  bit 2: ACK_REQUEST
  bit 1: CHAIN
  bit 0: RECIPIENT_TYPE
```

BASE_TEMPLATE codes:
- `000` Service record
- `001` Financial record
- `010` Compound financial
- `011` Contact/entity
- `100` Document/media
- `101` State Commit
- `110` Amendment
- `111` Generic / extension

All copies must emit and parse the `meta1` byte as described in `codec.md`. Records with unrecognised BASE_TEMPLATE values must be flagged as unrecognised (not silently discarded).

### 5. Byte order

All multi-byte integers are **big-endian**. `uint16` lengths, presence flags — big-endian throughout. Do not change this; it is fixed for the lifetime of the `1ag` codebook.

---

## Change Procedure

When `codec.md` is updated:

1. **Identify which sync elements changed** (scheme tag, field list, actions format, template byte, byte order).
2. **Update `workpads-codec` first** — this is the canonical implementation. Tests must pass.
3. **Update inline copies in order:**
   - `workpadsdotme/js/lib/codec.js`
   - `workpadsdotme/p/index.html` (decode-only — only update decode path)
   - `workpadsdotme/p/customer.html` (encode + decode — update both paths)
   - `workpadskaios/js/lib/codec.js`
4. **If SCALAR_FIELDS changed:** bump the codebook char in the scheme tag. Update the SCHEME regex in all copies at the same time. Old decoders will reject new URLs (by design — the scheme tag is the compatibility signal).
5. **If only the actions format or byte limit changed without a field set change:** the codebook char does not need to bump, but add a version comment noting the change and the date.
6. **Run tests:** `workpads-codec` round-trip tests, KaiOS `test/flow.test.js`, workpadsdotme manual encode/decode smoke test.
7. **Update `codec.md`** section version (e.g., v1.1 → v1.2) if the change is breaking; add a note if additive.

---

## Checklist for Codec PRs

Before merging any change that touches codec logic:

- [ ] `codec.md` section version updated if breaking
- [ ] `workpads-codec` tests pass
- [ ] `workpadsdotme/js/lib/codec.js` updated
- [ ] `workpadsdotme/p/index.html` decode path updated (if decode changed)
- [ ] `workpadsdotme/p/customer.html` updated (if encode or decode changed)
- [ ] `workpadskaios/js/lib/codec.js` updated
- [ ] `workpadskaios/js/lib/anon.js` updated if anonymous mode changed
- [ ] `workpadskaios/js/lib/security.js` updated if security wrapper changed
- [ ] Scheme tag bumped if field_flags layout changed
- [ ] SCHEME regex updated in all copies if scheme tag changed
- [ ] `implementation-notes.md` updated if a deviation is introduced or resolved

---

## Decode-Only Copies

`workpadsdotme/p/index.html` is a standalone receiver. It only needs the decode path. When updating it:

- Do **not** import encode logic — keep the file self-contained and minimal
- Update SCHEME regex, SCALAR_FIELDS, and the actions blob parser
- Do **not** add fields it doesn't yet render to the UI — add a TODO comment if a new field lands in the codec before the receiver UI handles it

---

## Detecting Drift

If you suspect copies have drifted:

```bash
# Compare scheme tag regex across copies
grep -h '1pa\|1ag\|1bg\|SCHEME' \
  workpads-codec/src/codec.js \
  workpadskaios/js/lib/codec.js \
  workpadsdotme/js/lib/codec.js \
  workpadsdotme/p/index.html \
  | sort | uniq -c | sort -rn

# Compare field_flags bit assignments across copies
grep -h 'bit 0\|bit 1\|bit 12\|FIN_BLOCK\|FLAGS3\|FLAGS4' \
  workpads-codec/src/codec.js \
  workpadskaios/js/lib/codec.js \
  | sort | uniq -c | sort -rn
```

If the same pattern appears more than once with different values, there is drift. `workpads-codec` is authoritative; `workpadskaios/js/lib/codec.js` is the primary active runtime.

---

## See Also

- `codec.md` — §5 normative specification
- `implementation-notes.md` — DEV-WP-URL-001 (scheme tag mismatch, KaiOS v0.1 vs web)
- `build-strategy.md` — §7 two-build strategy and codec bundling approach
