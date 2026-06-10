# Implementation Notes — Pre-Publication Scan

**Date:** 2026-04-28 (summary updated 2026-05-21)  
**Scope:** workpadskaios, workpadsdotme, workpads-cli, workpads-codec  
**Purpose:** Record deviations, innovations, and gaps found in implementations before or during standard publication.

**Live deviation register:** [`workpadskaios/system/dev_daily/DEVIATIONS.md`](../workpadskaios/system/dev_daily/DEVIATIONS.md)  
**Cross-repo process:** [`workpadskaios/system/project-process.md`](../workpadskaios/system/project-process.md)

---

## Summary

| ID | Repo | Type | Severity | Status |
|----|------|------|----------|--------|
| DEV-WP-URL-001 | workpadskaios | Deviation | **Breaking** | **Fixed** — pads-v1 `#1pa/` (2026-05-17) |
| INN-WP-FIN-001 | workpadsdotme | Innovation | Additive | Forward design; pending §5 extension |
| INN-WP-FIN-002 | workpadsdotme | Innovation | Additive | Helpers not yet in standard |
| INN-WP-CHAIN-001 | workpadsdotme | Innovation | Additive | Consistent with chain-protocol.md |
| GAP-WP-BLOCK-001 | workpadsdotme | Gap | Minor | Schema alignment needed in v0.2 |
| GAP-WP-SYNC-001 | both | Gap | Process | Codec sync protocol adopted; see codec-sync.md |

---

## Deviations

### DEV-WP-URL-001 — URL Scheme Tag Mismatch

**Repo:** workpadskaios  
**Status:** **Resolved** (2026-05-17)

**Resolution:** KaiOS v0.2 emits pads-v1 `#1pa/` per `codec.md` v2.0. Legacy query and hash schemes remain decode-only in `js/lib/codec.js`. See kaios `DEVIATIONS.md` for full history.

---

## Innovations

### INN-WP-FIN-001 — Financial Fields Forward Design in workpadsdotme Codec

**Repo:** workpadsdotme  
**Found in:** `js/lib/codec.js`  
**Severity:** Additive (compatible with any decoder that ignores unknown bits)

**Description:**  
The workpadsdotme codec includes field slots for bits 12–15:

| Bit | Field | Section |
|-----|-------|---------|
| 12 | `amount` | Financial (v0.2) |
| 13 | `currency` | Financial (v0.2) |
| 14 | `vat` | Financial (v0.2) |
| 15 | `record_type` | Financial (v0.2) |

These are implemented as plain string scalars in the current codec (not the full financial block encoding specified in `financial-block.md`). They are not yet exposed in the wizard UI.

**Status:** Consistent with the standard's v0.2 roadmap. A decoder that does not know about bits 12–15 will not set the presence flag for those fields and will skip their blobs — the core record remains intact. No action required before publication; this is forward-compatible.

**v0.2 action:** When the financial block is formalised, the string scalar encoding for bits 12–15 must be replaced by the full financial block binary encoding specified in `financial-block.md`. The field slots are already allocated correctly.

---

### INN-WP-FIN-002 — Financial Encoding Helpers (encodeFin / decodeFin / appendFin)

**Repo:** workpadsdotme  
**Found in:** `js/lib/codec.js`  

**Description:**  
workpadsdotme's codec exposes three methods not present in the standard:

- `WPCodec.encodeFin(record)` — encodes a separate financial JSON object, compresses and base64url-encodes it independently of the main record frame
- `WPCodec.decodeFin(finString)` — decodes a financial JSON object from an `encodeFin` output
- `WPCodec.appendFin(url, finData)` — merges viewer-added expense data into an existing fin object and appends it to the URL

These implement a **side-channel financial object** — a separately-encoded blob appended to the main workpad URL, outside the pads-v1 frame. This allows viewers to annotate a received record with their own expense data without modifying the original frame.

**Assessment:** This is a useful pattern that is not yet in the standard. The approach of a side-channel annotation blob (rather than requiring a full Amendment record in the chain protocol) is simpler for viewer-side expense capture.

**Status:** Not yet in any spec section. Before v0.2 publication, decide whether:
1. The `encodeFin` side-channel approach becomes a standard pattern (add to `financial-block.md`)
2. It is deprecated in favour of the full Amendment record pattern from `chain-protocol.md`
3. It is left as a workpadsdotme-specific extension with a registered deviation

**Immediate action:** Do not rely on `encodeFin` output in other implementations until a decision is made. workpadsdotme should document the format of the fin blob in a comment in `codec.js`.

---

### INN-WP-CHAIN-001 — Chain Support in workpadsdotme RecordService

**Repo:** workpadsdotme  
**Found in:** `js/services/RecordService.js`  
**Severity:** Additive (KaiOS v0.1 simply lacks these methods)

**Description:**  
workpadsdotme RecordService includes chain protocol support not present in workpadskaios v0.1:

- `genChainRef()` — generates a 3-byte random chain reference, base64url-encoded (4 chars)
- `findByChainRef(chainRef)` — looks up records by stored chain reference
- Records store a `chainRef` field locally (not encoded in the URL frame)

This is consistent with `chain-protocol.md` (the chain reference is appended as `&c=<chainRef>` to the URL). The implementation is a subset of the full chain protocol (no ANCHOR/PARTICIPANT_SLOT/SEQUENCE composite ID structure from the spec); it uses simpler random refs.

**Status:** Acceptable for v0.1 web. The full chain protocol (with composite chain IDs) is v0.2. The simplified random-ref approach in workpadsdotme is a stepping stone.

**v0.2 action:** Replace the random-ref approach in workpadsdotme with the full composite chain ID from `chain-protocol.md` when chain support is implemented in KaiOS.

---

## Gaps

### GAP-WP-BLOCK-001 — BlockRegistry Schema Gap

**Repos:** workpadsdotme, workpadskaios  
**Found in:** `js/services/BlockRegistry.js`

**Description:**  
`activity-profile.md` specifies two typed block registries:

```
Customers: {id, fn, tel?, flag}
Locations: {id, label, address?, postcode?, intersection?, flag}
```

The current workpadsdotme BlockRegistry stores a simpler schema:
```
{key, name, phone, createdAt, updatedAt}
```

It does not separate customers from locations; it does not implement the `flag` field (`''` or `'⚠'` for incomplete entries); it does not support the location-specific fields (address, postcode, intersection).

workpadskaios has a similar simplified implementation.

**Severity:** Minor for v0.1 (block registry is a UX convenience, not a codec concern). The schema gap becomes material in v0.2 when the `Expand field` pattern (`Location[+]` → sub-fields) is implemented.

**v0.2 action:** Align both implementations with the `activity-profile.md` schema. Migrate existing block entries at first launch after update.

---

### GAP-WP-SYNC-001 — Codec Sync Protocol Not Documented in Standard

**Repos:** workpadsdotme (SYNC.md), workpads-standard (missing)

**Description:**  
workpadsdotme maintains `system/SYNC.md`, a protocol document for keeping three inline codec copies in sync across the web app:
- `js/lib/codec.js` — main app codec
- `p/index.html` — decode-only standalone receiver
- `p/customer.html` — full encode+decode for customer ACK flow

The standard has no equivalent document. As the ecosystem grows (KaiOS, web, CLI, independent products), codec drift becomes a real risk — especially since the CLI currently uses a different encoding (pre-pads-v1 legacy) and KaiOS uses a different scheme tag.

**Resolution:** The sync protocol has been formalised and moved to `codec-sync.md` in this repo. All implementations should reference it.

---

## Items Not Requiring Standard Changes

The following workpadsdotme features were reviewed and confirmed as implementation-level choices that do not affect the standard:

- **Theme system** (Telegram / Classic) — CSS/UI, not a record or codec concern
- **Background picker** — UI, not a standard concern
- **Three-column layout** — web-specific layout; panel model is already in `panel-access-model.md`
- **Back/Cancel navigation buttons** — web UX pattern; not a D-pad navigation concern
- **Left panel footer** (archive count, New Workpad) — UI pattern, implementation choice
- **Main topbar breadcrumb** — UI pattern
