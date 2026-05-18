# Participants Block Specification

**Status:** v0.2 — 2026-05-18 (SUI-003: 2-bit ROLE_TYPE, HAS_ALT_ID, extended codebook path)
**Decision source:** decisions.md R-PADS-008
**Cross-references:** pads-v2-encoding-spec.md §9, chain-protocol.md

---

## 1. Purpose

The participants block is pads-v1's mechanism for encoding the human parties to a workpad record. It replaces the simple `worker` text field (pads-v1 bit 8) with a typed, extensible structure that handles:

- Sole trader attribution (just me)
- Employed worker + supervisor (two parties)
- Multi-worker jobs (supervisor dispatching to field worker, both identities in the record)
- Colleague-to-colleague records (lateral sharing)
- Witness or verifier (third party confirming work)
- Subcontractor attribution

The participants block is encoded in the binary frame, not in the URL query parameters. It is subject to progressive disclosure filtering — a customer-facing URL may omit internal worker identity details that would appear in a colleague-facing URL.

---

## 2. Wire Format

```
[meta2 bit 4 = PARTICIPANTS must be 1]

[1 byte ] block_header
    bits 7-5: participant_count (1-7)
    bits 4-0: reserved (must be 0)

for each participant (participant_count times):
  [1 byte ] part_flags
  [2+N bytes] name           — uint16 byte_len + UTF-8 name (required, minimum present)
  [if HAS_PHONE] [2+M bytes] phone
  [if HAS_EMAIL] [2+P bytes] email
  [if HAS_ROLE_TEXT] [2+Q bytes] role_text
```

### 2.1 part_flags layout

| Bit | Name | Meaning |
|-----|------|---------|
| 7 | IS_SENDER | This participant created the record |
| 6–5 | ROLE_TYPE | 2-bit quick-select (see §2.2) |
| 4 | HAS_ALT_ID | alt_id block follows (for participants without a phone number) |
| 3 | HAS_PHONE | Phone block follows name |
| 2 | HAS_EMAIL | Email block follows |
| 1 | HAS_ROLE_TEXT / HAS_TRADING_NAME | When ROLE_TYPE=11: free-text role label. When ROLE_TYPE≠11: trading name (sole trader or org name) |
| 0 | IS_ORG | 1=company/organisation; 0=individual |

ROLE_TYPE = bits 6–5 as 2-bit value.

### 2.2 Role type codebook — 2-bit quick-select

| Code | Role | Description |
|------|------|-------------|
| `00` | Customer | Customer / client / recipient |
| `01` | Worker | General field worker, sender's colleague, or self |
| `10` | Supplier / Vendor | Third-party supplier, subcontractor, or service provider |
| `11` | Extended | Role described via role_code byte or free-text role_text |

**Extended role path (ROLE_TYPE=11):**
- If `HAS_ROLE_TEXT=0`: a `role_code` byte (or 2 bytes for specialist roles) follows after phone/email fields. See §2.3.
- If `HAS_ROLE_TEXT=1`: a `[uint16 len][UTF-8]` free-text role label follows. Use when no codebook code fits.

### 2.3 Role code extended path

When `ROLE_TYPE=11` and `HAS_ROLE_TEXT=0`:

```
[role_code]    1 byte (0x00–0xFE) — 240 named roles in 16 groups (see ROLE-CODEBOOK.md §3)
               0xFF = escape → 1 additional byte (224 specialist roles; see ROLE-CODEBOOK.md §4)
```

Named roles include: Job owner/supervisor, Subcontractor, Colleague, Site contact, Referred by, Witness/verifier, and 230+ sector-specific roles. See `ROLE-CODEBOOK.md` for the complete codebook.

### 2.4 alt_id block (HAS_ALT_ID=1)

For participants without a phone number (common in low-connectivity markets):

```
[alt_id_type]   1 byte
  0x01 = app_uid          device-generated UID, SIM-stable
  0x02 = trade_name       business trading name
  0x03 = national_id      government-issued ID number (opaque)
  0x04 = location_label   contextual place reference ("Kigali market stall 7B")
  0x05–0xFE = reserved

[alt_id_value]  [uint8 len][UTF-8] — compact text, max 255 bytes
```

The `alt_id` block follows the phone/email/role_code/role_text fields for the participant.

---

## 3. IS_SENDER: Record Creator Identity

Exactly one participant per record should have IS_SENDER=1. This participant is the person who generated the URL — the record creator.

**Identity source:** IS_SENDER name and phone are drawn from the worker's active **Activity Profile** (`wp_activity_active`) at encode time. Changing the active Activity changes which identity appears on new records. See `workpads-standard/activity-profile.md`.

**Why IS_SENDER matters:**
- A customer receiving a workpad URL can identify who created it without a separate "From:" field
- A chain of records uses IS_SENDER identity for continuity — the same person is expected to appear as IS_SENDER across related records in a chain
- IS_SENDER identity is immutable after a record is chain-anchored (see Amendment rules in §10 of pads-v2-encoding-spec.md)

**Graceful degradation:** If no participant has IS_SENDER=1, decoders treat the first participant as the sender. Encoders must not produce records with zero or multiple IS_SENDER participants.

---

## 4. Usage Patterns

### 4.1 Sole trader sending to customer

One participant. IS_SENDER=1, ROLE_TYPE=000 (worker). Name is the worker's trading name or personal name. Optional: phone for callback.

```
participants:
  - name: "Alice Plumb"
    is_sender: true
    role: worker
    phone: "+44 7700 900123"  (optional)
```

URL generated with RECIPIENT_TYPE=0 (customer). Customer sees Alice's name and optionally phone.

### 4.2 Employed worker + supervisor

Two participants. Worker has IS_SENDER=1, ROLE_TYPE=000. Supervisor has IS_SENDER=0, ROLE_TYPE=001.

```
participants:
  - name: "Bob Field"
    is_sender: true
    role: worker
  - name: "Carol Manager"
    is_sender: false
    role: job owner
    phone: "+44 7700 900456"
```

Customer-facing URL: may include both names (Carol as point of contact). Colleague-facing URL: same.

### 4.3 Supervisor dispatching to worker (three-party)

Supervisor is IS_SENDER=1 (they created the URL), role=job owner. Worker is IS_SENDER=0, role=worker. The customer receives a record "from" the supervisor, with the worker attribution.

```
participants:
  - name: "Carol Manager"
    is_sender: true
    role: job owner
  - name: "Bob Field"
    is_sender: false
    role: worker
```

### 4.4 Witness / completion verifier

Job completion record (State Commit, template 0xD) with a third-party witness.

```
participants:
  - name: "Alice Plumb"
    is_sender: true
    role: worker
  - name: "Customer Name"
    is_sender: false
    role: site contact
  - name: "Dave Verify"
    is_sender: false
    role: witness
```

### 4.5 Colleague-to-colleague handoff

RECIPIENT_TYPE=1 (meta1 bit 0). Two participants, both workers. The sender includes their own identity (IS_SENDER=1) and the intended recipient (IS_SENDER=0, role=colleague) — so the URL "addresses" the colleague without requiring a backend routing layer.

```
participants:
  - name: "Alice Plumb"
    is_sender: true
    role: worker
  - name: "Frank Colleague"
    is_sender: false
    role: colleague
```

---

## 5. Relationship to Field Flags bit 8 (worker text)

The legacy format uses field flags bit 8 for a `worker` scalar text field. pads-v1 preserves this for backward compatibility. The interaction rules:

| Scenario | Use bit 8 | Use participants block |
|----------|-----------|----------------------|
| Simple attribution, one name, no identity detail | Yes | No |
| Two or more parties | No | Yes |
| Phone or email needed | No | Yes |
| IS_SENDER signal needed | No | Yes |
| Role type beyond "worker" needed | No | Yes |

Encoders choosing bit 8 (worker text) must not also set PARTICIPANTS=1 for the same worker — that would duplicate the attribution. If PARTICIPANTS=1 is set, bit 8 should not be set (the participants block is the authoritative source).

Decoders receiving a record with both bit 8 and PARTICIPANTS=1 should prefer the participants block.

---

## 6. Progressive Disclosure and Participants

When a worker generates a customer-facing URL (RECIPIENT_TYPE=0), the participants block may be filtered:

**Include in customer URL:**
- IS_SENDER participant (name + optional phone for customer callback)
- Job owner if relevant to customer (e.g. customer may need to call the supervisor)
- Site contact if it is the customer's own representative

**Omit from customer URL:**
- Internal colleague references
- Worker margin/rate details (in financial block — see financial-block.md)
- "Referred by" attribution (internal business intelligence)

The filtering is applied at URL generation time by the encoder. The RECIPIENT_TYPE bit signals to the decoder what filtering was applied, enabling the web viewer to display context-appropriate labels.

---

## 7. Privacy Notes

Names and contact details in the participants block travel in the URL. Once a URL is shared, the sender cannot recall the participants data. Workers should include only the participant information that is appropriate for the intended recipient.

For customer-facing URLs: include the sender's name and business phone. Omit internal email addresses, internal referral sources, and financial identities.

For colleague-facing URLs: include full participant detail as needed for the job handoff.

The RESTRICT_FORWARD bit (meta2 bit 0) is a signal to the recipient's app that the record should not be re-shared. It does not cryptographically prevent sharing — it is a protocol-level courtesy signal and an explicit consent indicator.

---

## 8. Byte Budget Examples

**Sole trader, name + phone:**
```
block_header: 1 byte
part_flags: 1 byte
name: 2 + 12 = 14 bytes ("Alice Plumb")
phone: 2 + 14 = 16 bytes ("+44 7700 900123")
Total: 32 bytes raw
```

**Employer + worker, names only:**
```
block_header: 1 byte
worker part_flags + name: 1 + 2 + 10 = 13 bytes ("Bob Field")
supervisor part_flags + name: 1 + 2 + 14 = 17 bytes ("Carol Manager")
Total: 31 bytes raw
```

After deflate, small participant blocks compress well (names have high entropy but the structural overhead is minimal and repeating patterns compress). For a T3-level record with a 32-byte participants block: URL length impact ≈ +25–35 chars after compress+b64url.
