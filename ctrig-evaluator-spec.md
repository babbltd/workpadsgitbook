# §CT — C-TRIG Evaluator Specification

**Status:** v0.1 (2026-05-18)  
**SUI:** SUI-011  
**Kaios source:** `draft_specs/CTRIG-EVALUATOR-DESIGN.md` draft-spec 2026-05-17  
**Depends on:** `trig-spec.md`, `agreements-spec.md`

---

## 1. Purpose

The C-TRIG evaluator executes obligation bytecode against a resolved chain state to determine what actions should be triggered in a commitment record. It is the runtime for the C-TRIG instruction set defined in `agreements-spec.md` §3.3.

**Key constraints:**
- KaiOS compatible — pure JS, no dependencies beyond the existing codec
- Max program: 32 bytes — bounded execution, no infinite loops possible
- Max stack depth: 8 values — sufficient for any valid 32-byte program
- Stateless — no persistence between runs; chain state is always re-read fresh

The C-TRIG evaluator and the display TRIG evaluator (see `trig-spec.md`) share the TRIG block header byte position. The MODE bit (header byte bit 5) distinguishes them:
```
MODE = 0 → display TRIG (rendering rules)
MODE = 1 → C-TRIG (obligation rules)
```

---

## 2. Evaluator Inputs

```
Evaluator.run(program, context) → Result

program:   Uint8Array        — C-TRIG bytecode (max 32 bytes)

context:
  record         PadsRecord   — the decoded pads-v1 frame being evaluated
  chain          ChainState   — pre-resolved chain context (see §2.1)
  timestamp      uint16       — current COMPACT_TIME value
  parties        Party[]      — resolved participant identities
  serverResults  Map          — cached server evaluation results (for async conditions)
```

### 2.1 ChainState Resolution Contract

The app resolves chain state once before each evaluation run. The evaluator never fetches records itself — it only reads the `context` object. This is the boundary between app logic and evaluator logic.

```
ChainState {
  records: [{
    uid:          string         — record identifier
    template:     uint4          — BASE_TEMPLATE value
    commit_type:  uint2|null     — COMMIT_TYPE if State Commit
    sender_uid:   string         — IS_SENDER participant identity
    ack_present:  boolean        — ACK_REQUEST was set AND a reply exists in chain
    financial: {
      total:      Amount|null    — customer_amount (BitLedger encoded)
      paid:       Amount|null    — confirmed paid amount from I<I records in chain
      deposit:    Amount|null    — deposit_paid if present
    }|null
    timestamp:    uint16         — COMPACT_TIME of record creation
    depth:        uint8          — chain depth (0 = root)
  }]

  party_count:      uint8        — distinct IS_SENDER parties in chain
  ack_count:        uint8        — parties who have sent an ACK
  milestone_states: boolean[]    — which milestone indexes are marked complete
  ratified:         boolean      — bilateral ratification detected
  disputed:         boolean      — DISPUTE_LINK amendment present in chain
}
```

---

## 3. Stack Machine

```
Stack:
  values: (boolean | uint24)[]   — max 8 entries

  push(v)   — add to top
  pop()     — remove from top; returns UNKNOWN if empty
  peek()    — read top without removing

Result:
  { status: 'resolved', value: boolean }
  { status: 'halted',   reason: string, condition_id: string }
  { status: 'error',    reason: string }
```

**Stack value types:**
- `boolean`: condition results (`true` / `false`)
- `uint24`: numeric values for arithmetic conditions
- `UNKNOWN`: unresolvable — propagates pessimistically (UNKNOWN AND true = UNKNOWN)

---

## 4. Instruction Set

High nibble = opcode, low nibble = inline immediate (same convention as display TRIG):

```
0x0_ COMPARE_AMT  — compare two numeric stack values (see §4.1)
0x1_ PUSH_COND    — resolve condition and push result; imm=0xF → escape byte follows (see §5)
0x2_ RELEASE_AMT  — prefill payment_request record for milestone index in imm; push true
0x3_ TRIGGER_OBL  — prefill obligation record for clause index in imm; push true
0x4_ ASSERT_STATE — prefill state_commit record for state code in imm; push true
0x5_ REQUIRE_ACK  — push (chain.ack_count > 0)
0x6_ TIME_LOCK    — if milestone date (imm) not reached: halt('timelock'); else push true
0x7_ MARKER_WRITE — prefill marker_write record for party index in imm (0=A, 1=B); push true
0x8_ AND          — pop b, pop a, push (a AND b)
0x9_ OR           — pop b, pop a, push (a OR b)
0xA_ NOT          — pop a, push (NOT a)
0xB_ IF_THEN      — pop condition; if false: skip next instruction (pc++)
0xC_ BRANCH       — 3-byte: pop condition; jump by true_offset or false_offset (see §4.3)
0xD_ COMPLETE     — prefill completion record; return resolved(true)
0xE_ DISPUTE      — prefill dispute record for clause ref in imm (0xF=global); return resolved(false)
0xF_ VERSION      — 3-byte version/extension gate (see §4.4)
```

**Pre-fill behaviour:** RELEASE_AMT, TRIGGER_OBL, ASSERT_STATE, MARKER_WRITE, COMPLETE, and DISPUTE do not execute actions directly — they call `app.prefillRecord()` which builds a pre-populated record for the user to review before sending. The `push true` signals the condition was triggered, not that the action was completed.

Max program length: 32 bytes. Programs exceeding 32 bytes are spec violations; evaluator returns `error('program_too_long')`.

### 4.1 COMPARE_AMT — Opcode 0x0

Pops two uint24 values from the stack, compares them, pushes boolean result.

| imm | Operator |
|-----|----------|
| 0x0 | a >= b |
| 0x1 | a > b |
| 0x2 | a == b |
| 0x3 | a < b |
| 0x4 | a <= b |
| 0x5 | a != b |
| 0x6–0xE | reserved — push false |
| 0xF | percentage threshold — next byte = uint8 threshold (units 0.5%; 200=100%); push (a >= b × threshold / 200) |

Stack underflow: if fewer than two numeric values available, push UNKNOWN.

Common amount comparisons (`FULL_PAYMENT_CONFIRMED`, `PARTIAL_PAYMENT`, `DEPOSIT_CONFIRMED`) use the named conditions in CAT 0x1 — no COMPARE_AMT needed.

### 4.2 TIME_LOCK Date Resolution

`milestoneDate(imm, context)`:
- imm 0–14: `context.chain.milestone_states[imm].date` (uint16 COMPACT_TIME days)
- imm 0xF: `context.record.fields.due_date` — uses record's due_date field

If the referenced date is null: evaluator halts with `'timelock_date_missing'`.

### 4.3 BRANCH Offset Semantics

3-byte instruction: `[0xC_][true_offset][false_offset]`

Offsets are relative forward byte counts from the first byte after the BRANCH instruction. Offset 0 = execute immediately following instruction. Negative offsets not supported — all offsets are uint8.

```
pc_after_branch = pc + 3
target_pc = pc_after_branch + (condition ? true_offset : false_offset)
if target_pc >= program.length OR target_pc >= 32:
  return error('branch_out_of_bounds')
```

### 4.4 VERSION Opcode Semantics

3-byte instruction: `[0xF0][min_version][feature_flags]`

- byte 2: `min_version` — minimum evaluator version required to run this program
- byte 3: `feature_flags` — capability bitmask this program requires

```
EVALUATOR_VERSION = 1

if min_version > EVALUATOR_VERSION:
  return halt('version_unsupported')

unsupported = feature_flags & ~SUPPORTED_FEATURES
if unsupported !== 0:
  return halt('unsupported_features')
```

**SUPPORTED_FEATURES bitmask (v1):**

| Bit | Feature |
|-----|---------|
| 0 | NUMERIC_STACK — uint24 values on stack (always 1 in v1) |
| 1 | COMPARE_AMT — 0x0 opcode (always 1 in v1) |
| 2 | EXTENDED_COND — CAT escape byte (always 1 in v1) |
| 3–7 | Reserved |

---

## 5. Condition Resolution

```
PUSH_COND (opcode 0x1) with imm 0x0–0xE: direct condition ID
PUSH_COND with imm 0xF: escape — consume next byte as [CAT:4][ID:4]
```

### 5.1 Direct Conditions (0x00–0x0E)

| ID | Condition | Evaluation |
|----|-----------|-----------|
| 0x00 | HAS_PHONE | participants block contains a phone number |
| 0x01 | IS_ORG | any participant is_org |
| 0x02 | ROLE_TYPE_CUSTOMER | any participant role_type == 0 |
| 0x03 | ROLE_TYPE_WORKER | any participant role_type == 1 |
| 0x04 | HAS_FINANCIAL_BLOCK | record has financial block present |
| 0x05 | DATE_REACHED | context.timestamp >= record.date |
| 0x06 | HAS_LOCATION | location field present |
| 0x07 | IS_CHAIN | meta1 CHAIN == 1 |
| 0x08 | ACK_RECEIVED | chain.ack_count > 0 |
| 0x09 | PAYMENT_CONFIRMED | I<I State Commit (COMMIT_TYPE=01) in chain |
| 0x0A | MILESTONE_MET | chain.milestone_states[imm] — uses imm as milestone index |
| 0x0B | RATIFICATION_COMPLETE | chain.ratified |
| 0x0C | HAS_COMPOUND | setup_byte compound_value == 1 |
| 0x0D | DISPUTED | chain.disputed |
| 0x0E | HAS_TRIG | meta2 has_trig_block == 1 |

### 5.2 Extended Conditions by Category

**CAT 0x0 — Time/date**

| ID | Condition |
|----|-----------|
| 0x0 | DATE_REACHED (from milestone table) |
| 0x1 | TIMELOCK_EXPIRED (time_lock target date passed) |
| 0x2 | WITHIN_WINDOW (current time between date_start and date_end) |
| 0x3 | DUE_DATE_PASSED (record.due_date < context.timestamp) |
| 0x4–0xF | Reserved |

**CAT 0x1 — Payment**

| ID | Condition |
|----|-----------|
| 0x0 | DEPOSIT_CONFIRMED (deposit_paid field in chain record) |
| 0x1 | AMOUNT_THRESHOLD_MET (paid amount meets threshold via COMPARE_AMT) |
| 0x2 | FULL_PAYMENT_CONFIRMED (total paid >= invoice total in chain) |
| 0x3 | PARTIAL_PAYMENT (0 < paid < total) |
| 0x4–0xF | Reserved |

**CAT 0x2 — Identity/role**

| ID | Condition |
|----|-----------|
| 0x0 | ROLE_VERIFIED (role_code present AND cert signal set) |
| 0x1 | CERT_HELD (ROLE_SIGNALS CERT bit set for any participant) |
| 0x2 | QUORUM_MET (party writes >= threshold; direct threshold from low nibble or clause block via escape) |
| 0x3 | LEAD_PARTY_CONFIRMED (ROLE_SIGNALS LEAD participant has written) |
| 0x4–0xF | Reserved |

**CAT 0x3 — Chain/state**

| ID | Condition |
|----|-----------|
| 0x0 | ALL_MILESTONES_MET (all milestone_states true) |
| 0x1 | CHAIN_COMPLETE (CHAIN_COMPLETE=1 in any chain record) |
| 0x2 | AMENDMENT_PRESENT (BASE_TEMPLATE=110 record in chain) |
| 0x3 | CHAIN_DEPTH_REACHED (chain record count >= threshold) |
| 0x4–0xF | Reserved |

**CAT 0x4 — Document/ACK**

| ID | Condition |
|----|-----------|
| 0x0 | RECEIPT_CONFIRMED (State Commit COMMIT_TYPE=01 in chain) |
| 0x1 | SATISFACTION_CONFIRMED (State Commit with satisfaction signal in story) |
| 0x2 | SIGNATURE_PRESENT (cryptographic signature field in chain record) |
| 0x3 | TERMS_AGREED (State Commit COMMIT_TYPE=10 in chain) |
| 0x4–0xF | Reserved |

**CAT 0x5 — Location**

| ID | Condition |
|----|-----------|
| 0x0 | LOCATION_VERIFIED (location field present AND matches expected) |
| 0x1 | WITHIN_ZONE (location within declared service area) |
| 0x2–0xF | Reserved |

**CAT 0x6 — Server/async**

| ID | Condition |
|----|-----------|
| 0x0 | SERVER_EVALUATION_RECEIVED (serverResults has entry for this condition) |
| 0x1 | PAYMENT_GATEWAY_CONFIRMED (payment gateway callback received) |
| 0x2 | THIRD_PARTY_VERIFIED (external identity verification complete) |
| 0x3–0xF | Reserved |

**CAT 0x7 — Cryptographic**

| ID | Condition |
|----|-----------|
| 0x0 | PREIMAGE_REVEALED (hash preimage delivered — HTLC-style) |
| 0x1 | HASH_COMMITMENT_MET (SHA-256 of record bytes matches stored commitment) |
| 0x2 | WRITE_SIG_VALID (Stone write signature verifies against party public key) |
| 0x3–0xF | Reserved |

**CAT 0x8–0xE — Reserved.** Evaluator must treat as UNKNOWN and halt with `'unknown_condition'`.

**CAT 0xF — Further escape.** 3-byte form: byte 3 carries full 8-bit extended condition ID. Evaluators not implementing a given 8-bit ID must halt.

### 5.3 QUORUM_MET Threshold Encoding

When `CAT 0x2, ID 0x2 = QUORUM_MET`:
- Low nibble 0x1–0xE: direct threshold (1–14 parties required) — covers common quorums (2-of-3, 3-of-5) in zero extra bytes
- Low nibble 0xF: escape — threshold defined in clause block or milestone table

### 5.4 PUSH_COND Extended Registry — Design Status

The escape byte `0x1F` (PUSH_COND with imm=0xF) reserves a second byte as free design space for extended conditions. The CAT/ID encoding above covers the v1 registry (CAT 0x0–0x7). CAT 0x8–0xF remain reserved pending OQ-40 (smart contract conditions, Ricardian contract requirements, P2P enforcement models).

**Interim rule:** evaluators encountering an unrecognised CAT or ID must treat the condition result as UNKNOWN and halt — surfacing "agreement contains unrecognised conditions" to the user.

---

## 6. Arithmetic Scope

The evaluator implements a numeric stack — uint24 values alongside boolean values. This enables direct amount comparisons using `BitLedger.decode()` (the same function used by the codec):

```
chain.total_paid → BitLedger.decode() → uint24
record.customer_amount → BitLedger.decode() → uint24
COMPARE_AMT(>=) → boolean on stack
```

This makes payment domain conditions (the most important domain for trade agreements) first-class in the evaluator. The formula-forward record design aligns with numeric stack values as a natural extension.

Named payment conditions in CAT 0x1 (FULL_PAYMENT_CONFIRMED, DEPOSIT_CONFIRMED, PARTIAL_PAYMENT) pre-compute the common comparisons so most programs never need explicit COMPARE_AMT instructions.

---

## 7. Example Programs

**"50% deposit upfront, balance on completion" — 4 bytes**
```
0x1E  PUSH_COND(PAYMENT_CONFIRMED)  — check initial payment settled
0x21  RELEASE_AMT(milestone 1)       — release milestone 1 (deposit confirmed)
0x1D  PUSH_COND(ACK_RECEIVED)        — check job-complete acknowledgment
0x22  RELEASE_AMT(milestone 2)       — release milestone 2 (balance payment)
```

**"Conditional: if parties in same city, use local rate clause" — 3 bytes**
```
0x16  PUSH_COND(HAS_LOCATION)
0xB0  IF_THEN
0x31  TRIGGER_OBL(clause 1)
```

**"Write Marker when both parties accept" — 4 bytes**
```
0x1D  PUSH_COND(ACK_RECEIVED)
0x1E  PUSH_COND(PAYMENT_CONFIRMED)
0x80  AND
0x70  MARKER_WRITE(party 0)
```

**"Release payment when full amount confirmed, else dispute if overdue" — 6 bytes**
```
0x1F 0x12  PUSH_COND(CAT=1, ID=2, FULL_PAYMENT_CONFIRMED)
0xB0        IF_THEN
0xD0        COMPLETE
0x1F 0x03  PUSH_COND(CAT=0, ID=3, DUE_DATE_PASSED)
0xEF        DISPUTE(global)
```

---

## 8. Evaluation Flow

```
pc = 0
while pc < program.length AND pc < 32:
  byte   = program[pc]
  opcode = byte >> 4
  imm    = byte & 0xF

  execute opcode (see §4)
  pc++

// End of program: AND all remaining stack values
result = stack.reduce((acc, v) => acc AND v, true)
return resolved(result)
```

At end of program with empty stack: return `resolved(true)` (no conditions = unconditional pass).

---

## 9. Error Handling

| Condition | Evaluator response |
|-----------|-------------------|
| Program > 32 bytes | `error('program_too_long')` — agreement marked malformed |
| Stack underflow (pop from empty stack) | Push UNKNOWN; continue |
| Stack overflow (> 8 values) | `error('stack_overflow')` — agreement marked malformed |
| Unknown opcode | `error('unknown_opcode')` — agreement marked malformed |
| Async condition (CAT 0x6) not in serverResults | `halt('awaiting_server')` — re-evaluate on server response |
| CAT 0x8–0xE (reserved) | `halt('unknown_condition')` — shows "contains unrecognised conditions" |
| CAT 0xF without third byte | `error('escape_truncated')` |
| BRANCH target out of bounds | `error('branch_out_of_bounds')` |
| TIME_LOCK: date not found | `halt('timelock_date_missing')` |

**Halt vs error distinction:**
- `halt` — temporary: re-evaluation may succeed later (async condition, version mismatch with newer evaluator)
- `error` — permanent: program is malformed, agreement is flagged as malformed

---

## 10. Async Condition Resolution (Halt-and-Re-trigger)

When evaluator halts on an unresolvable async condition:

1. Evaluator returns `{ status: 'halted', reason: 'awaiting_server', condition_id }`
2. App registers a server listener for the expected confirmation event
3. When confirmation arrives, app stores result in `context.serverResults`
4. Full C-TRIG program re-evaluates from byte 0 (stateless — no partial state preserved)
5. Agreement UI shows "awaiting confirmation" during halt

The chain record state (not evaluator memory) is the source of truth for what has been confirmed. Re-evaluation is always complete and always fresh.

---

## 11. Reference Implementation (JS pseudocode)

```javascript
const EVALUATOR_VERSION = 1
const SUPPORTED_FEATURES = 0b00000111  // NUMERIC_STACK | COMPARE_AMT | EXTENDED_COND

function evaluate(program, context) {
  if (program.length > 32) return error('program_too_long')

  const stack = []
  let pc = 0

  while (pc < program.length) {
    const byte   = program[pc]
    const opcode = byte >> 4
    const imm    = byte & 0xF

    switch (opcode) {
      case 0x0: { // COMPARE_AMT
        const b = stack.pop(), a = stack.pop()
        if (imm === 0xF) {
          pc++
          const pct = program[pc]
          stack.push(a >= Math.round(b * pct / 200))
        } else {
          const ops = [(a,b)=>a>=b,(a,b)=>a>b,(a,b)=>a===b,(a,b)=>a<b,(a,b)=>a<=b,(a,b)=>a!==b]
          stack.push(imm < ops.length ? ops[imm](a, b) : false)
        }
        break
      }
      case 0x1: { // PUSH_COND
        if (imm === 0xF) {
          pc++
          if (pc >= program.length) return error('unexpected_end')
          const r = resolveExtended(program[pc], context)
          if (r && r.halt) return halt(r.reason, context)
          stack.push(r)
        } else {
          const r = resolveDirect(imm, context)
          if (r && r.halt) return halt(r.reason, context)
          stack.push(r)
        }
        break
      }
      case 0x2: context.app.prefillRecord('payment_request', { milestone: imm }); stack.push(true); break
      case 0x3: context.app.prefillRecord('obligation',      { clause:    imm }); stack.push(true); break
      case 0x4: context.app.prefillRecord('state_commit',    { state:     imm }); stack.push(true); break
      case 0x5: stack.push(context.chain.ack_count > 0); break
      case 0x6: {
        const date = milestoneDate(imm, context)
        if (!date) return halt('timelock_date_missing')
        if (context.timestamp < date) return halt('timelock')
        stack.push(true); break
      }
      case 0x7: context.app.prefillRecord('marker_write', { party: imm }); stack.push(true); break
      case 0x8: { const b = stack.pop(), a = stack.pop(); stack.push(a && b); break }
      case 0x9: { const b = stack.pop(), a = stack.pop(); stack.push(a || b); break }
      case 0xA: stack.push(!stack.pop()); break
      case 0xB: if (!stack.pop()) pc++; break
      case 0xC: {
        const cond = stack.pop()
        const trueRef = program[pc + 1], falseRef = program[pc + 2]
        const target = pc + 3 + (cond ? trueRef : falseRef)
        if (target >= 32 || target >= program.length) return error('branch_out_of_bounds')
        pc = target - 1; break
      }
      case 0xD: context.app.prefillRecord('completion'); return { status: 'resolved', value: true }
      case 0xE: context.app.prefillRecord('dispute', { clause: imm }); return { status: 'resolved', value: false }
      case 0xF: {
        if (pc + 2 >= program.length) return error('version_truncated')
        const minVer = program[pc + 1], features = program[pc + 2]
        if (minVer > EVALUATOR_VERSION) return halt('version_unsupported')
        if (features & ~SUPPORTED_FEATURES) return halt('unsupported_features')
        pc += 2; break
      }
    }
    pc++
  }

  const result = stack.length ? stack.reduce((a, b) => a && b, true) : true
  return { status: 'resolved', value: result }
}
```
