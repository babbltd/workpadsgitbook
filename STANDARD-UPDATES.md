# Standard Update Register

**Purpose:** Tracks every design advance in the kaios implementation docs that must be reflected in this standard before the standard can be considered normative for the current design generation.  
**Maintained by:** Babb (kaios implementation team)  
**Process:** See §3 below — the update cycle integrates with OPEN-QUESTIONS.md and CODEC-EVOLUTION.md in the kaios repo.

---

## 1. How to Read This File

Each row in the tables below is a **Standard Update Item (SUI)**. Items are grouped by tier:

- **Tier 1 — Must-before-public:** standard is normatively incomplete without this; a third-party implementor reading only the standard would build the wrong thing.
- **Tier 2 — Must-before-v1.0:** architectural extension that requires standard coverage before the standard can claim v1.0.
- **Tier 3 — Post-v1.0 / advisory:** useful to standardise but not blocking current implementations.

**Status values:**
- `pending` — no work started in standard
- `in-progress` — kaios source doc is being drafted/finalised
- `ready-to-write` — kaios source doc is draft-spec or above; standard update can be written now
- `done` — standard doc updated and consistent with kaios source

**SUI ID format:** `SUI-NNN` — stable identifiers; never reused.

---

## 2. Update Item Tables

### Tier 1 — Must-before-public

| SUI | Standard doc to update | What to add / replace | Kaios source | Status |
|-----|----------------------|----------------------|--------------|--------|
| SUI-001 | `codec.md` | Replace codebook-b frame layout (16-bit flags) with full pads-v1 frame layout: meta1 + meta2 + setup_byte + transaction_byte + field_flags (16-bit) + field_flags3 + field_flags4 + all conditional blocks. Codebook tag: `1pa`. | `dev_refs/FRAME-SPEC.md` v1.0 | done |
| SUI-002 | `codec.md` → new `security-wrapper.md` | Add complete security wrapper spec: 5-layer stack (deflate seed, field scramble, AES-CTR 128-bit, HMAC, preamble byte), key derivation hierarchy (passphrase→salt→master→cipher_key + scramble_seed), per-contact derivation, template-keyed derivation, IV reuse justification, wrapper URL format. | `draft_specs/SECURITY-DESIGN.md` | done |
| SUI-003 | `participants-block.md` | Extend ROLE_TYPE from 3-bit (8 slots) to 2-bit quick-select (Customer/Worker/Supplier/Extended) + 1-byte full codebook (240 roles in 16 groups) + 2-byte extended (224 specialist roles) + ROLE_SIGNALS byte (CERT/AUTH/LEAD). Add HAS_ALT_ID path (OQ-31 Option A). | `dev_refs/ROLE-CODEBOOK.md` v1.1 | done |
| SUI-004 | `codec.md` → update DOMAIN table | Document DOMAIN=11 hybrid mode: account_pair_byte after transaction_byte, AP_DIRECTION + AP_STATUS + AP_COMPLETENESS + AP_EXTENSION bits. Update meta2 DOMAIN bits table (00/01/10/11). | `dev_refs/FRAME-SPEC.md` §17 | done |
| SUI-005 | `financial-block.md` | Update fin_control for DOMAIN=01: add BILLED flag (bit 7), mode-indicator (bit 6=0), QTY_TYPE, PARITY, EXPENSE_CAT (job charge / COGS / running cost). Document DOMAIN=10 fin_control layout. Add DOMAIN=11 account_pair_byte. | `dev_refs/FRAME-SPEC.md` §2 financial block | done |
| SUI-006 | `template-system.md` | Add template definition schema (JSON): label_map, formula (infix string), formula_bytecode (RPN), block_order, custom_fields (FLAGS4 slot allocations), template_hash (canonical SHA-256). Document custom template ID namespace via EXT_TEMPLATE variant type (CRC-8 + CRC-16). | `draft_specs/TEMPLATE-SYSTEM-DESIGN.md` | done |
| SUI-007 | new `trig-spec.md` | Create TRIG specification from scratch: stack machine architecture, instruction set (opcodes + immediates), condition registry (CAT 0x0–0x1, escape byte), display mode tokens, CSS/theme codebooks, bot-resistance model. | `dev_daily/TRIG-DESIGN.md` + `dev_refs/TECH-REFERENCE.md` TRIG tables | done |
| SUI-008 | `transaction-classification.md` | Add Entry Type Matching Table: deterministic I>O state → Account Pair mapping for all 24 wizard entry types (income-side, expense-side, balance sheet). Add type-change reconciliation rules. Add accounting detail display (plain-English account names). | `dev_refs/FRAME-SPEC.md` §17.5–17.7 | done |

---

### Tier 2 — Must-before-v1.0

| SUI | Standard doc to update | What to add / replace | Kaios source | Status |
|-----|----------------------|----------------------|--------------|--------|
| SUI-009 | new `markers-spec.md` | Create Marker (Stone) specification: RATIFIED_FRAME wire encoding, bilateral ratification detection algorithm, write modes (WORM vs appendable-tail), hardware options (NTAG213/215 NFC, QR+server), P2P software-only Option E, write token format, #1pm/ URL scheme, offline connectivity matrix. | `draft_specs/MARKERS-DESIGN.md` draft-spec | done |
| SUI-010 | new `agreements-spec.md` | Create Agreements/Obligations specification: offer record (EXT_TEMPLATE path, ACK_REQUEST), acceptance State Commit (COMMIT_TYPE=10, ratification bitmap), bilateral ratification detection, dispute Amendment (DISPUTE_LINK, HAS_PARENT_UID), C-TRIG evaluator integration. | `draft_specs/AGREEMENTS-DESIGN.md` draft-spec | done |
| SUI-011 | new `ctrig-evaluator-spec.md` | Create C-TRIG evaluator specification: stack machine inputs/outputs, ChainState resolution contract, full opcode table (0x0 COMPARE_AMT through 0xF VERSION), §4.1–4.4 special semantics, SUPPORTED_FEATURES bitmask v1, pseudocode reference implementation. | `draft_specs/CTRIG-EVALUATOR-DESIGN.md` draft-spec | done |
| SUI-012 | `codec.md` → amount encoding | Add CODEC-1/2/3 decisions: BitLedger N=A×2^S+r scaled value encoding, DECIMAL_POS=111 flat uint24 backup mode, dual-layer formula encoding (infix JSON + RPN bytecode in TRIG block). | `CODEC-EVOLUTION.md` CODEC-1/2/3 | done |
| SUI-013 | new `project-association.md` | Document project association convention: `proj:<uuid>` prefix in tag field, comma-separated multi-project, single-project constraint for financial records, future first-class wire structure path. | `draft_specs/PROJECT-ASSOCIATION-DESIGN.md` + `dev_refs/STANDARD-FIELDS.md` §3.1 | done |
| SUI-014 | new `anonymous-mode.md` | Document DATA_SOURCE=11 anonymous/stealth mode: SUBMIT_ACTION=11, blind pickup via HMAC-derived pickup_slot_key, shell placeholder text, share-sheet warnings. | `draft_specs/ANON-MODE-DESIGN.md` | done |
| SUI-015 | new `attachment-spec.md` | Document attachment strategy: FLAGS3 bit 4 wire encoding, URL-reference vs inline-hash decision, storage strategy, hash validation. | `draft_specs/ATTACHMENT-DESIGN.md` | done |

---

### Tier 3 — Post-v1.0 / advisory

| SUI | Standard doc to update | What to add / replace | Kaios source | Status |
|-----|----------------------|----------------------|--------------|--------|
| SUI-016 | new `data-sync-bundle.md` | Document Data Sync Bundle: deflate-compressed multi-item bundle URL, sequenced dependency hydration, partial failure graceful degradation. | `dev_refs/TEMPLATE-SYSTEM-RESEARCH.md` §5 | done |
| SUI-017 | `template-diffusion.md` | Update CDN tier strategy to match kaios three-tier model: bundled (core) / peer-to-peer Data Sync Bundle (sector) / CDN on-demand (fallback only, no startup calls). | `draft_specs/TEMPLATE-SYSTEM-DESIGN.md` §distribution | done |
| SUI-018 | `chain-protocol.md` | Document agreement chain extension: C-TRIG block in chain records, COMMIT_TYPE values (00=job close, 01=payment confirmed, 10=terms agreed), DISPUTE_FLAG, amendment chain convention. | `dev_refs/FRAME-SPEC.md` §9–10 | done |
| SUI-019 | `codec-sync.md` | Update three-repo sync checklist for pads-v1 (`1pa` codebook). Replace `1eg/` / `1dg/` entries. | `dev_daily/CODEC-SYNC.md` | done |
| SUI-020 | `record-schema.md` | Add FLAGS4 standard cross-template assignments: `gps_binary` (bit 2, financial template), preamble byte HKDF_KEY (bit 3). | `dev_refs/FRAME-SPEC.md` §14 | done |
| SUI-021 | `codec.md` + tag dispatch | New scheme `#1pv/`: Path C record header (C1–C6), parser paths standard/shortcut/solo. Dual-decode with `#1pa/`. | `CODEC-V2-SCOPE-LOCKED.md`, Path C Full Adoption spec | done |
| SUI-022 | `codec.md` | Record type byte 0 table incl. `need`/`offer`/`connection`; mandatory-group elision (C5) per type. | `CODEC-V2-SCOPE-LOCKED.md`, `PRODUCT-SURFACE-LOCKED.md` | done |
| SUI-023 | `codec.md` | Flag byte + CRC-16-CCITT on `1pv/` frames (Doc 3 §8). | Doc 3, `CODEC-V2-SCOPE-LOCKED.md` | done |
| SUI-024 | `codec-sync.md`, TAG-REFERENCE | Register `1pv/` in cross-repo sync and kaios tag dispatch. | `CODEC-V2-SCOPE-LOCKED.md` | done |
| SUI-025 | `chain-protocol.md`, `codec.md` | `relationship` 4+4 on `chainRef` in `1pv/`; unknown → responds. | `CHAIN-EXECUTION-LOCKED.md`, Doc 8 §3 | done |
| SUI-026 | `codec.md`, share spec | `changedMask` at share time; `_ratifiedFrame` outbound. | `CHAIN-EXECUTION-LOCKED.md`, Doc 8 §4.3 | done |
| SUI-027 | `codec.md` / action-list annex | 16-bit `confirmed_mask` + `declined_mask` on ack records. | Doc 6 §8.2, `CHAIN-EXECUTION-LOCKED.md` | done |

---

## 3. Update Cycle Process

### How kaios design decisions flow into this standard

The kaios implementation operates an **advance-then-standardise** model. The lifecycle of a decision is:

```
OPEN-QUESTIONS.md          CODEC-EVOLUTION.md          dev_refs/ or draft_specs/
     OQ raised          →   Decision logged as D-N    →   File promoted to draft-spec
          ↓                                                         ↓
     SUI item added here ←─────────────────────────────── SUI status → ready-to-write
          ↓
     Standard doc updated
          ↓
     SUI status → done
```

### When to add an SUI

Add an SUI entry to this file when **any** of these happen:

1. A kaios `draft_specs/` file reaches `draft-spec` status (its OPEN-QUESTIONS entry gets DRAFT-SPEC annotation)
2. A kaios `dev_refs/` doc version bumps (e.g. FRAME-SPEC v0.2 → v1.0)
3. CODEC-EVOLUTION.md adds a new decision row (CODEC-N or D-N)
4. A kaios DEVIATIONS.md entry is created (which means the standard is now wrong)

### Where to look to check for missed SUI items

| Check | Frequency | What to look for |
|-------|-----------|-----------------|
| `dev_daily/OPEN-QUESTIONS.md` summary tables | Per session | Any row with `DRAFT-SPEC` or `RESOLVED` that has no SUI entry |
| `dev_refs/FRAME-SPEC.md` §16 Design Status | Per session | Any newly-resolved row |
| `CODEC-EVOLUTION.md` Decision Log | Per session | Any new CODEC-N row |
| `dev_daily/DEVIATIONS.md` | Per session | Any new DEV-WP-* entry (means standard is diverged) |

### Naming convention for new standard docs

New standard docs created under this process follow the same naming as existing workpads-standard files:
- `kebab-case.md`
- Lead with a version and date in the frontmatter block (match existing style)
- Cross-reference the originating SUI item and the kaios source doc

### When an SUI is "done"

An SUI is done when:
1. The standard doc is updated or created
2. The content matches the kaios source at the time of writing (minor variations for standard-appropriate abstraction are fine)
3. Any kaios DEVIATIONS.md entry that the SUI was addressing is closed

---

## 4. Version Alignment Reference

| Domain | Kaios (current) | Standard (current) | Delta |
|--------|-----------------|-------------------|-------|
| Codec codebook tag | `1pa` (pads-v1, package a) | `1pa` ✓ | SUI-001 done |
| Frame flags width | 16-bit + FLAGS3 + FLAGS4 | 16-bit + FLAGS3 + FLAGS4 ✓ | SUI-001 done |
| Security wrapper | 5-layer (FRAME-SPEC §12) | `security-wrapper.md` ✓ | SUI-002 done |
| DOMAIN modes | 00/01/10/11 | 00/01/10/11 ✓ | SUI-004 done |
| fin_control | BILLED + EXPENSE_CAT + PARITY | BILLED + EXPENSE_CAT + PARITY ✓ | SUI-005 done |
| ROLE_TYPE | 2-bit + 240 extended codes | 2-bit + extended ✓ | SUI-003 done |
| Template schema | Full label_map + formulas | label_map + formulas ✓ | SUI-006 done |
| Entry type matching | 24 deterministic pairs | §8 in transaction-classification ✓ | SUI-008 done |
| TRIG bytecode | Full spec (TRIG-DESIGN.md) | `trig-spec.md` ✓ | SUI-007 done |
| Markers | Full spec (MARKERS-DESIGN.md) | `markers-spec.md` ✓ | SUI-009 done |
| Agreements | draft-spec (AGREEMENTS-DESIGN.md) | `agreements-spec.md` ✓ | SUI-010 done |
| C-TRIG evaluator | draft-spec (CTRIG-EVALUATOR-DESIGN.md) | `ctrig-evaluator-spec.md` ✓ | SUI-011 done |
| Amount encoding | BitLedger N=A×2^S+r | `codec.md` §Amount Encoding ✓ | SUI-012 done |
| Project association | `proj:` tag convention | `project-association.md` ✓ | SUI-013 done |
| Anonymous mode | DATA_SOURCE=11, blind pickup | `anonymous-mode.md` ✓ | SUI-014 done |
| Attachment spec | FLAGS3 bit 4, tier CDN | `attachment-spec.md` ✓ (OQ-AT1/AT3 open) | SUI-015 done |

**Current standard version:** v1.0 (2026-05-18, all SUI-001–020 done)  
**Kaios design generation:** pads-v1 / FRAME-SPEC v1.0 (2026-05-17)  
**Status:** Standard is normatively complete for pads-v1 design generation.
