# §AN — Anonymous Mode (DATA_SOURCE=11)

**Status:** v0.1 (2026-05-18)  
**SUI:** SUI-014  
**Kaios source:** `draft_specs/ANON-MODE-DESIGN.md` design 2026-05-17

---

## 1. Purpose

Anonymous mode allows a workpads record to be shared publicly with zero sender identity in the payload. No name, phone, routing address, or reply path is visible to the receiver. The shell displays placeholder text only.

**Use cases:**

| Scenario | Why anonymity matters |
|----------|----------------------|
| Community service menu in a public WhatsApp group | Sender doesn't want strangers having their phone number |
| Anonymous feedback form | Receiver cannot pre-filter based on sender identity |
| Blind tender / quote submission | Parties submit without knowing each other's identity until close |
| Whistleblower reporting form | Identity protection is mandatory |
| Market price board | Vendor wants price visibility but not personal contact on a public channel |
| Sensitive professional services (counselling, legal aid, health) | Client confidentiality — the request itself must not expose the requestor |

---

## 2. Wire Mechanism

Anonymous mode is signalled by `DATA_SOURCE=11` in the display_schema DISPLAY_CONTROL byte (FRAME-SPEC §11.1):

```
DISPLAY_CONTROL bits 5-4: DATA_SOURCE = 11
```

**Required constraints when DATA_SOURCE=11:**
- `RECIPIENT_TYPE=0` in meta1 — cannot name a specific recipient
- `IS_SENDER=1` MUST NOT appear in any participant in the participants block
- `SUBMIT_ACTION=11` REQUIRED when a form schema is present — activates blind pickup
- The participants block, if present, contains only non-sender participants

**Fields OMITTED in anonymous mode:**
- Sender name (`worker` field, bit 8)
- Sender phone (contact phone field, bit 7)
- Any `IS_SENDER=1` participant
- The `url` field (FLAGS3 bit 6) if it would reveal sender identity
- Any UID that could be correlated to sender identity

**Fields that MAY still be present:**
- Service description (`job`, bit 0)
- Location (`location`, bit 3) — service area, non-identifying
- Financial block (prices, without payer identity)
- Display schema (TRIG programs, display type, form schema)
- An anonymous-stable content UID (see §7)

---

## 3. Shell Display Behaviour

When the shell decodes a frame with `DATA_SOURCE=11`:

```
Sender name:    "Anonymous"  (or template-defined alias — see §4)
Sender avatar:  Generic icon (no initials, no photo)
Contact button: Hidden
Reply button:   Shown only if form schema present AND SUBMIT_ACTION=11
Chain button:   Hidden (no chain reply path to anonymous sender)
```

**Warning banner (always shown, cannot be suppressed by template):**
```
"This record was shared anonymously. The sender cannot be identified
 or contacted through this link."
```

This protects receivers from confusing anonymous records with named ones.

---

## 4. Sender Alias

Anonymous records may carry a non-identifying descriptor chosen by the sender — a `sender_alias`. This is NOT a name — it does not identify the person.

Examples: "Local electrician", "Community health service", "Market Trader"

**Encoding:** `sender_alias` is a display-layer field in the display_schema, maximum 40 bytes. It is NOT encoded in the wire field_flags. Templates declare `anon_alias_field: true` to expose the alias in the rendered view.

The alias allows meaningful context ("Local electrician, available weekdays") without revealing identity. It appears where the sender name would otherwise appear.

---

## 5. Blind Pickup (SUBMIT_ACTION=11)

When a form is present with `SUBMIT_ACTION=11`, submissions go to a server-side blind pickup slot. The sender retrieves them without the server being able to link the retrieval to the sender's identity.

### Key Derivation

All key material is derived on-device. Nothing identifying leaves the sender's device:

```
master_secret    = device-held secret (never leaves device, never sent to server)
form_uid         = random 16-byte identifier embedded in the form record
pickup_code      = HMAC-SHA256(master_secret, form_uid)   [computed locally only]
pickup_slot_key  = SHA256(form_uid || pickup_code)         [sent to server as slot address]
```

The server stores submissions keyed by `pickup_slot_key`. It never sees `master_secret` or `pickup_code`. The sender retrieves by presenting `pickup_slot_key` — which the server cannot link to any identity.

### Exchange Protocol

```
Sender                                Server
──────                                ──────
1. Create form record
2. Generate form_uid (random 16B)
3. Derive pickup_code locally
4. Compute pickup_slot_key
5. Embed form_uid in frame ─────────→ (server stores nothing yet)
6. Share #1pb or #1pa URL

Receiver fills form ────────────────→ Server stores at hash(pickup_slot_key + seq)
                                       Payload: AES-256-GCM(submission, submission_key)
                                       submission_key = HMAC(pickup_code, "submission-key")
                                       Server holds ciphertext it cannot decrypt

7. Sender fetches:
   GET /pickup
   Auth: HMAC(pickup_slot_key, timestamp) → Server returns encrypted submissions
   Sender decrypts locally
```

### Submission Storage

The server stores:
```
(pickup_slot_key, AES-256-GCM(submission_content, submission_key), timestamp)
```

The server cannot decrypt the payload — it holds only ciphertext. `submission_key` is derived from `pickup_code` which is derived from `master_secret` — none of which the server ever sees.

**Retention policy:** submissions auto-deleted after 90 days (configurable in form_schema). Server stores a deletion timestamp alongside each submission.

### Replay and Enumeration Protection

- Pickup requests require a TOTP-style timestamp HMAC — replays older than 5 minutes rejected
- Server rate-limits pickup attempts per slot
- `form_uid` is 16 random bytes — 128-bit entropy prevents guessing

---

## 6. Anonymous Record UID

Anonymous records should not use device UIDs (which would correlate the record to the sender across multiple shares). Two options exist:

- **Content-derived UID**: hash of the service description — stable across re-shares of the same content; same service menu always produces the same UID
- **Random UID**: fresh random per record — unlinkable but unstable; re-sharing creates a new UID

**Convention:** use content-derived UIDs for public service menus (stability across re-shares), random UIDs for one-shot forms (no cross-record linkage). The app chooses based on whether `SUBMIT_ACTION=11` is present (form → random; menu → content-derived).

---

## 7. Primary URL Combinations

**`#1pb/` + `DATA_SOURCE=11`** — the "anonymous service menu" pattern:
- `#1pb/` — public billboard URL (safe for WhatsApp broadcast, QR code)
- `DATA_SOURCE=11` — no sender identity in payload
- Receiver views prices and submits a form; sender retrieves via blind pickup

**`#1pa/` + `DATA_SOURCE=11`** — anonymous data record (data-only, no display schema). Used for raw anonymous data submissions.

**`#1ps/` + `DATA_SOURCE=11`** — anonymous encrypted record. Content is private to key-holders AND sender is anonymous. Use case: anonymous quote submission in a sealed tender.

---

## 8. Chain Protocol Constraints

Anonymous records cannot participate in named chains. A chain link in the URL (`&c=<parent_uid>`) is visible metadata — if the anonymous record's UID appears in a named chain, the topology links the record to the named sender.

**Rules:**
- A record with `DATA_SOURCE=11` MUST NOT set `CHAIN=1` in meta1
- The `&c=` URL parameter MUST NOT appear in anonymous record URLs
- Amendment records (BASE_TEMPLATE=110) with a parent chain pointing to a `DATA_SOURCE=11` record inherit the anonymity constraint

**Anonymous-to-anonymous chains ARE allowed:** both records have `DATA_SOURCE=11`, no names in either. Used for multi-step anonymous forms (wizard-style submission across multiple records).

---

## 9. TRIG Interaction

TRIG programs in anonymous records MUST NOT use conditions that reveal sender identity.

| Condition | Safe for anon? |
|-----------|---------------|
| `HAS_PHONE` | No — tests receiver phone; reveals receiver context |
| `IS_ORG` | Yes — tests record structure |
| `ROLE_TYPE` | Yes — tests viewer role (customer/worker/supplier) |
| `HAS_FINANCIAL_BLOCK` | Yes — tests record structure |
| `DATE_REACHED` | Yes — time-based |

Intended TRIG use in anonymous records: display mode selection by viewer role — e.g. "Show price to role=Customer; hide internal cost to role=Worker" — without revealing sender identity.

---

## 10. Threat Model

### Protected

| Threat | Protection |
|--------|-----------|
| Receiver identifies sender from URL | Fragment never sent to server |
| Server logs reveal sender identity | No sender identity in payload; server sees only timing + IP |
| CDN / proxy logs | Fragment stripped by browser before any network request |
| Submission-to-sender correlation | Blind pickup — server cannot link retrieval to identity |
| UID correlation across records | Anonymous records use service-hash or random UIDs, not device UIDs |

### Not Protected

| Threat | Note |
|--------|------|
| IP address correlation | Sender's IP visible to server on pickup fetch. Use VPN/Tor for strong anonymity. |
| Timing correlation | Pickup fetch timing relative to submission may be correlated by a powerful adversary |
| Content fingerprinting | Unique service descriptions may identify the sender even without explicit identity fields |
| Shared device context | If the same device is used for anonymous and named records, a compromised device breaks anonymity |

**Guidance for high-risk use cases** (whistleblower, political context): use cellular data (not WiFi), fetch pickups from a different network than submission was created on, use the anon alias instead of recognisable descriptions, consider a rotating form_uid (see §11 open items).

---

## 11. Open Items

- **OQ-24c** — Pickup code rotation: should `form_uid` be rotatable (regenerated after each pickup window) for forward secrecy? This would require re-encoding and re-sharing the URL after each rotation window — higher security, higher UX friction. Decision needed: opt-in rotation (app offers "reset pickup slot" action) vs always-rotating (form auto-expires after N days and must be re-shared).
- **OQ-24d** — Anonymous Markers: can an anonymous record be ratified in a Marker? Markers require two-party writes, implying identity. Resolution path: anonymous Markers where both parties are anonymous (community cooperative pricing pact) — Marker holds terms but neither slot carries a named identity. Needs dedicated design.
