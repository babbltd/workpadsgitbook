# §A — Agreements and Commitment Records

**Status:** v0.1 (2026-05-18)  
**SUI:** SUI-010  
**Kaios source:** `draft_specs/AGREEMENTS-DESIGN.md` draft-spec 2026-05-17

---

## 1. Purpose

A commitment is any record where two or more parties have agreed to terms and that agreement itself is the artifact. Agreements in workpads range from a worker's accepted invoice (single-sentence, lightweight) to a multi-clause service contract with milestone-gated payments and a physical Marker holding the ratified state.

**Design principle:** commitments use the same wire format, share mechanism, and chain protocol as all other workpads records. An agreement IS a financial or service record with a ratification layer attached — not a separate "contract platform."

---

## 2. Existing Architecture Mapping

| Existing element | Commitment role |
|-----------------|-----------------|
| `ACK_REQUEST` in meta1 | "I am asking you to confirm this" |
| `BASE_TEMPLATE=101` State Commit | A point-in-time signed-off snapshot |
| `BASE_TEMPLATE=110` Amendment | Revision trail — mechanism for updating agreed terms |
| `CHAIN=1` + `&c=` suffix | Sequential record chain — a commitment thread |
| `story` field (bit 11) | Free-text terms — single paragraph prose |
| `due_date` (bit 14) | Time-binding on payment |
| I>O subtype `I>I` sub 01 = Retainer | Recurring/agreed obligation |
| compound_block | Line-item structure — maps to clause-level structure |

---

## 3. Three Tiers of Agreement Complexity

### Tier 1 — Light Agreement (v1.0 scope)

**Use cases:** "I accept this quote." / "Payment terms agreed." / "Job scope confirmed."

**Wire mechanism:**

1. Party A creates a record (Financial `I>I` Invoice/Quote, or Service) with `story` field containing agreed terms text and `ACK_REQUEST=1` in meta1.
2. Party B replies with a chained State Commit (`BASE_TEMPLATE=101`, `COMMIT_TYPE=10` terms agreed), also with `ACK_REQUEST=1`. Chain link `&c=<party_A_uid>` ties them.
3. Both records existing in the chain = bilateral ratification. Neither party can alter either record without issuing an Amendment (which itself requires a new chain reply).

**Ratification detection:** a record is bilaterally ratified when the chain contains at least two records from different `IS_SENDER` parties, both with `COMMIT_TYPE` present.

**Bilateral ratification detection algorithm:**

```javascript
function isRatified(chain) {
  const offer   = chain.find(r => r.ack_request && !r.chain)
  const replies = chain.filter(r => r.chain && r.commit_type !== undefined)
  if (!offer || replies.length === 0) return false

  const threshold    = offer.threshold_n ?? 2
  const uniqueAcceptors = new Set(
    replies.filter(r => r.sender_uid !== offer.sender_uid).map(r => r.sender_uid)
  )
  return uniqueAcceptors.size >= threshold - 1
}
```

**App-layer status badges:**
- Unratified: "Awaiting acceptance"
- Ratified: "Agreement confirmed ✓" + timestamp
- Disputed: "Dispute raised" (Amendment with DISPUTE_FLAG=1)

---

### Tier 2 — Structured Agreement (post-MVP)

**Use cases:** multi-clause service contract, milestone-gated payment, numbered terms.

**Record type:** `BASE_TEMPLATE=111` Generic with `EXT_TEMPLATE=1`, domain extension byte for agreement subtype:

| Code | Subtype |
|------|---------|
| 0x01 | Service contract |
| 0x02 | Employment |
| 0x03 | Supply |
| 0x04 | Joint venture |
| 0x05 | NDA |
| 0x06–0xFF | Available (registry in TEMPLATE-CATALOGUE.md) |

**Split-record acceptance model:** records are immutable once created. Party A creates the agreement record with clauses. Party B creates a chain-reply State Commit carrying an acceptance_mask. Acceptance state is read from the chain — never written back into Party A's clauses.

#### Clause Block Wire Format

```
[clause_header]    2 bytes
  bits 7-3: CLAUSE_COUNT      number of clauses (0–31)
  bit 2: HAS_MILESTONES       1=milestone table follows clause list
  bit 1: HAS_CONDITIONS       1=each clause carries a condition code byte
  bit 0: CLAUSE_FLAGS_PRESENT 1=each clause has a clause_flags byte

Per clause (repeated CLAUSE_COUNT times):

[clause_flags]     1 byte — if CLAUSE_FLAGS_PRESENT=1
  bits 7-6: CLAUSE_TYPE
    00 = standard term (both parties bound)
    01 = conditional (if <condition> then <obligation>)
    10 = milestone (triggers a subsequent payment or obligation)
    11 = recital / preamble (informational — not binding)
  bits 5-4: OBLIGATION_SIDE
    00 = both parties
    01 = Party A only (the agreement proposer)
    10 = Party B only (the agreement acceptor)
    11 = third party / authority
  bit 3: PARTY_A_DISPUTES  1=Party A has raised a dispute on this clause (in reply record)
  bit 2: PARTY_B_DISPUTES  1=Party B has raised a dispute on this clause (in reply record)
  bits 1-0: reserved

[condition_code]   1 byte — if HAS_CONDITIONS=1 AND CLAUSE_TYPE=01
  bits 7-4: CONDITION_ID  4-bit code; uses shared condition registry
  bit 3: NEGATE           1=condition is inverted (if NOT <condition>)
  bits 2-0: reserved

[clause_text]      [u8 len][UTF-8]  max 255B
```

#### Milestone Table (when HAS_MILESTONES=1)

```
[milestone_count]  1 byte

Per milestone:
[milestone_flags]  1 byte
  bits 7-6: TRIGGER_TYPE
    00 = date reached (date_milestone u16 follows)
    01 = explicit acknowledgment (ACK_REQUEST on linked chain record)
    10 = condition satisfied (condition_code byte follows)
    11 = manual release (both parties must signal)
  bits 5-4: RELEASE_TYPE
    00 = payment release (amount_milestone uint24 follows)
    01 = obligation trigger (clause_ref in bits 2-0)
    10 = state change (record status update)
    11 = Marker write event (Marker UID in milestone)
  bit 3: SEQUENTIAL        1=requires all prior milestones complete first
  bits 2-0: CLAUSE_REF     which clause index this milestone belongs to (7=global)

[date_milestone]   u16 days — if TRIGGER_TYPE=00
[amount_milestone] uint24   — if RELEASE_TYPE=00
[condition_code]   1 byte   — if TRIGGER_TYPE=10
```

#### Acceptance Reply

Party B's State Commit reply carries acceptance state:

**MVP:** `tag = "accept,mask:0b11111"` — bit-per-clause accept mask as text string.

**Post-MVP acceptance_block:** 1–4 bytes of acceptance bitmap in FLAGS4.
```
bits 0–6: per-clause acceptance (1=accepted, 0=disputed/pending)
bit 7: FULLY_RATIFIED (1=threshold met)
```

#### Shared Condition Registry Extensions

New conditions extending the TRIG 12-condition registry for clause conditionals:

| Code | Condition | Meaning |
|------|-----------|---------|
| 0x0C | `DATE_REACHED` | Current date ≥ the date in the linked milestone |
| 0x0D | `ACK_RECEIVED` | A chain reply with the expected ACK has been received |
| 0x0E | `PAYMENT_CONFIRMED` | A settled payment record (I<I) exists in the chain |
| 0x0F | `MILESTONE_MET` | A specific milestone index is marked complete |

---

### Tier 3 — Commitment Protocol (long-term)

**Use cases:** complex agreements with branching logic, automated release conditions, Marker integration.

**Architecture:** TRIG programs govern DISPLAY (who sees what). C-TRIG programs govern OBLIGATION (when a condition is met, what is triggered). Separate instruction sets, shared condition registry. A display-only shell ignores C-TRIG blocks. A commitment-aware shell runs both.

C-TRIG shares the TRIG header byte position. The MODE bit (header byte bit 5) distinguishes:
```
TRIG header byte bit 5: MODE
  0 = display TRIG (rendering rules)
  1 = commitment C-TRIG (obligation rules)
```

See `ctrig-evaluator-spec.md` for the C-TRIG instruction set and evaluator.

**Evaluation model:**
- Normal case: both parties evaluate C-TRIG locally against their chain state. No server required.
- When milestone/condition met: app pre-fills a new sub-record (State Commit, payment request, progress update) for user review before sending. User always has a review step — no auto-send.
- When parties disagree: dispute escalated to server arbitration (see §4 below).

---

## 4. Server Arbitration Protocol

When parties disagree on C-TRIG evaluation, either party may escalate to server arbitration. Server arbitration is the fallback — normal operation is always client-local.

`ARBITRATION_LEVEL` field (2 bits) in the dispute Amendment record:

| Code | Level | What server receives | Privacy |
|------|-------|---------------------|---------|
| 00 | Lightweight | C-TRIG bytecode + chain state summary (hashes + timestamps) | High |
| 01 | Standard | Full record frames from all chain participants | Medium |
| 10 | Privacy-preserving | Both parties sign chain summary independently; server evaluates against signed hashes only | High |
| 11 | Reserved | — | — |

Server returns a signed evaluation result. Both parties must accept or escalate to off-protocol dispute resolution.

**Async condition resolution (halt-and-re-trigger):**

When C-TRIG encounters an unresolvable condition (e.g. `SERVER_EVALUATION_RECEIVED`, payment gateway confirmation):
1. Evaluator halts entire program
2. App registers server listener for expected confirmation event
3. On confirmation arrival: full C-TRIG program re-evaluates from byte 0
4. Agreement shows "awaiting confirmation" during halt
5. Evaluator is stateless — chain record state is source of truth, not evaluator memory

---

## 5. Markers Integration

Markers are electronic commitment artifacts that accept a one-time write from each party at ratification, then become read-only. See `markers-spec.md` for the full Marker specification.

**Marker ↔ Agreement protocol:**

```
1. Party A creates agreement record, generates #1pa or #1pt URL
2. Party B reviews, creates chain-reply acceptance record
3. App detects bilateral ratification in chain
4. App assembles RATIFIED_FRAME = compact encoding of both records + chain link
5. App initiates Marker write:
     SLOT 0: Party A writes RATIFIED_FRAME
     SLOT 1: Party B writes commitment confirmation bytes
     Marker sets WRITE_LOCK after threshold slots filled
6. Marker URL: #1pm/<marker_uid>
```

**Write-once enforcement:**
- Hardware: WORM memory on NFC tag (NTAG OTP bytes)
- Software: server-side WORM — once threshold writes recorded, all further writes rejected and canonical content sealed with a timestamp signature

---

## 6. Offer Record Wire Encoding

```
meta1:
  BASE_TEMPLATE = 001 (Financial) or 000 (Service)
  ACK_REQUEST = 1
  CHAIN = 0 (first in chain)
  RECIPIENT_TYPE = 1 (named recipient)

field_flags:
  bit 0: job       — agreement title
  bit 11: story    — terms text
  bit 14: due_date — if time-bound

Optional EXT_TEMPLATE path (standalone Agreement records):
  EXT_SIGNAL = 100 (Variant type, 3 bytes)
  Domain extension byte: 0x01=service contract, 0x02=employment,
    0x03=supply, 0x04=joint venture, 0x05=NDA
```

## 7. Acceptance Reply Wire Encoding

```
meta1:
  BASE_TEMPLATE = 101 (State Commit)
  ACK_REQUEST = 1
  CHAIN = 1
  RECIPIENT_TYPE = 1

URL suffix: &c=<party_A_record_uid>

State Commit fields:
  COMMIT_TYPE = 10 (terms agreed)
  story: optional acceptance note

Acceptance mask (Tier 2, when clause block present in offer):
  MVP:      tag field "accept,mask:0b11111111"
  Post-MVP: FLAGS4 acceptance_block (bits 0–6 per-clause; bit 7 FULLY_RATIFIED)
```

## 8. Dispute Wire Encoding

```
When C-TRIG present:
  C-TRIG evaluator emits DISPUTE(clause_ref)
  → app prompts for grounds text
  → Amendment sent:
      BASE_TEMPLATE = 110 (Amendment)
      amendment_flags bit 6: DISPUTE_LINK = 1
      amendment_flags bit 7: HAS_PARENT_UID = 1
      parent_uid: 8-byte SHA-256 truncated of original offer record
      story: dispute grounds text

When no C-TRIG:
  Amendment with DISPUTE_FLAG=1 alone is sufficient
  → agreement state = DISPUTED on Amendment receipt
```

---

## 9. Agreement State Machine

```
PROPOSED → REVIEWED → ACCEPTED → ACTIVE → COMPLETED
                                ↘ DISPUTED → (resolved) → COMPLETED
                                                        ↘ CANCELLED
```

| Code | State | Trigger |
|------|-------|---------|
| 0 | PROPOSED | Original record created |
| 1 | REVIEWED | Party B has opened the record (app-layer signal) |
| 2 | ACCEPTED | Party B's chain reply with acceptance_mask received |
| 3 | ACTIVE | First milestone condition met |
| 4 | COMPLETED | COMPLETE instruction evaluated, or all milestones met |
| 5 | DISPUTED | DISPUTE instruction evaluated, or Amendment with DISPUTE_FLAG=1 |
| 6 | CANCELLED | Both parties signal cancellation (mutual Amendment) |

---

## 10. Template Controls

Agreement records use dedicated template variants:
- **Agreement — Simple**: story field, no clauses, Tier 1 only
- **Agreement — Service Contract**: clause block, milestone dates
- **Agreement — Employment**: compound block with pay terms

Templates declare:
```json
{
  "agreement_tier": 1,
  "requires_ack": true,
  "marker_eligible": true,
  "clause_display": "numbered",
  "acceptance_mode": "full_record"
}
```

---

## 11. Relationship to Other Record Types

| Scenario | Record type | Agreement layer |
|----------|------------|-----------------|
| Job quote | Financial `I>I` sub 01 | Light agreement on the same record |
| Service contract | Service record with story | Tier 2 agreement block attached |
| Employment terms | Compound financial (payroll) | Tier 2 with pay terms as clauses |
| One-off commitment | State Commit | Tier 1 ratification on the commit |

A record that IS the agreement uses `BASE_TEMPLATE=111` Generic with a commitment domain signal. A record that CARRIES an agreement (financial transaction + terms) uses the financial template with agreement block added.
