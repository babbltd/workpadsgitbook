# Transaction Classification: I>O Notation and 8-State System

**Status:** v0.2 — 2026-05-18 (SUI-008: Entry Type Matching Table, type-change rules, accounting display)
**Decision source:** decisions.md R-PADS-006, R-PADS-007
**Cross-references:** pads-v2-encoding-spec.md §7, financial-block.md
**BitLedger alignment:** BitLedger_Protocol_v3.md §Layer 2, BitLedger_Universal_Domain.md

---

## 1. Overview

All financial records in pads-v1 carry a transaction classification. This classification determines how the record is labelled in the UI, how rounding is applied to amounts, and how the record is interpreted in the context of a chain.

The classification system uses two design inputs:
1. **I>O notation** — a compact three-part description of every financial event
2. **The transaction byte** — the 8-bit binary encoding of the same information

These are the same system expressed differently: the notation for design and documentation, the byte for wire encoding.

---

## 2. I>O Notation

### 2.1 Structure

Every financial transaction is described as:
```
<Direction> <Time> <Effect>
```
Where:
- **Direction** = `I` (income — money flows toward the sender) or `O` (outgoing — money flows away from the sender)
- **Time** = `<` (Past/settled — the exchange has occurred) or `>` (Future/pending — the exchange is open)
- **Effect** = `I` (increases sender's asset position) or `O` (increases sender's liability / decreases assets)

### 2.2 Arrow direction convention

`<` points backward in time = Past = settled. The transaction has closed.
`>` points forward in time = Future = pending. The transaction is open.

This mirrors the natural reading: "money flows `In`, it is in the `Past`, it has landed `In` my account" = `I < I`.

### 2.3 The 8 primary states

| Notation | Dir | Time | Effect | Plain meaning | Worker label |
|----------|-----|------|--------|---------------|--------------|
| `I < I` | I | < | I | Settled income | Payment received |
| `I > I` | I | > | I | Future income | Invoice sent |
| `I < O` | I | < | O | Income given back | Refund given |
| `I > O` | I | > | O | Credit note issued | Credit note |
| `O < O` | O | < | O | Settled expense | Expense paid |
| `O > O` | O | > | O | Future expense | Bill received |
| `O < I` | O | < | I | Expense recovered | Reimbursed |
| `O > I` | O | > | I | Future reimbursement | Reimbursement expected |

### 2.4 Common mapping for field service workers

| Scenario | Notation | Template subtype |
|----------|----------|-----------------|
| Customer paid at completion | `I < I` | 00 |
| Invoice sent to customer (net 30) | `I > I` | 00 |
| Partial deposit received | `I < I` | 01 (deposit) |
| Customer cancelled, refund issued | `I < O` | 00 |
| Supplier invoice for parts | `O > O` | 00 |
| Parts paid for on job | `O < O` | 00 |
| Employer reimbursed travel | `O < I` | 01 (travel) |
| Expense claim submitted, pending | `O > I` | 01 (travel) |

---

## 3. Transaction Byte Encoding

### 3.1 Bit layout

| Bits | Name | Values |
|------|------|--------|
| 7 | DIRECTION | 0 = I, 1 = O |
| 6 | TIME | 0 = Past (`<`), 1 = Future (`>`) |
| 5 | EFFECT | 0 = I (asset), 1 = O (liability) |
| 4–3 | SUBTYPE | 00–11 (see §3.2) |
| 2 | QTY_SPLIT | 0 = lump sum, 1 = qty × rate |
| 1–0 | ROUNDING | 00=exact, 10=down, 11=up, 01=error |

### 3.2 Subtypes per primary state

Each of the 8 primary states has 4 subtypes (2 bits). Codebook package `a` assigns:

**`I < I` (settled income) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | Standard payment |
| 01 | Deposit / partial payment |
| 10 | Final payment (job closed) |
| 11 | Tip / gratuity |

**`I > I` (future income) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | Invoice |
| 01 | Quote / estimate |
| 10 | Retainer agreement |
| 11 | Recurring billing |

**`O < O` (settled expense) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | General expense |
| 01 | Travel / mileage |
| 10 | Materials / supplies |
| 11 | Subcontractor payment |

**`O > O` (future expense) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | Bill received |
| 01 | Scheduled payment |
| 10 | Supplier order pending |
| 11 | Future subcontractor cost |

**`I < O` (settled outgoing income — refund given) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | Full refund |
| 01 | Partial refund |
| 10 | Warranty / goodwill credit |
| 11 | Overpayment return |

**`I > O` (future outgoing income — credit note) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | Credit note |
| 01 | Return authorisation |
| 10 | Discount issued |
| 11 | Credit balance applied |

**`O < I` (settled expense recovery — reimbursed) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | General reimbursement |
| 01 | Travel reimbursed |
| 10 | Materials recovered |
| 11 | Advance returned |

**`O > I` (future expense recovery — reimbursement pending) subtypes:**
| Code | Meaning |
|------|---------|
| 00 | Claim submitted |
| 01 | Travel claim pending |
| 10 | Materials claim pending |
| 11 | Advance requested |

---

## 4. Rounding Rules (BitLedger Alignment)

pads-v1 adopts BitLedger's rounding rules directly:

| Transaction type | Rounding direction | Rationale |
|----------------|--------------------|-----------|
| Income (DIRECTION=I, EFFECT=I) | DOWN (bits 10) | Conservative — do not overstate income |
| Liability / expense (DIRECTION=O, EFFECT=O) | UP (bits 11) | Conservative — do not understate what is owed |
| Tax component | UP (bits 11) | Regulatory conservatism — round tax up |
| Recovered expense (O < I) | DOWN (bits 10) | Do not overstate recovery |
| Future reimbursement (O > I) | UP (bits 11) | Claim the full amount owed |

**ROUNDING bit codes (BitLedger standard):**
- `00` = exact (no rounding applied)
- `10` = down
- `11` = up
- `01` = error (invalid — not a valid rounding instruction)

Encoders should set ROUNDING based on the transaction type as above. For whole-unit amounts (no fractional pennies), ROUNDING=00 (exact) is appropriate.

---

## 5. QTY_SPLIT: Time-and-Materials Billing

When QTY_SPLIT=1, the financial block carries a quantity and rate rather than (or in addition to) a single sum amount.

```
customer_amount   — total (uint24): what the customer pays
qty               — uint24: quantity (hours, units, items)
rate              — uint24: rate per unit
```

Both qty and rate use DECIMAL_POS from the setup byte. For hours-based billing at DECIMAL_POS=2: qty=200 → 2.00 hours; rate=4500 → £45.00/hr.

**Display:** "2 hours @ £45/hr = £90.00" — the decoder reconstructs this from qty, rate, and customer_amount. If `qty × rate ≠ customer_amount` (rounding or adjustment), display customer_amount as the authoritative total.

Worker rate (what the worker costs, not what the customer pays): encoded in the worker_amount block (field_flags3 bit 7). This field is never included in customer-facing URLs (progressive disclosure).

---

## 6. Relationship to State Commit Records

State Commit records (template 0xD) do not carry a transaction byte. They are state snapshots, not transactions — there is no directional flow.

**Pay summary subtype (State Commit 01)** may carry a financial block summarizing accumulated amounts. In this case, the setup byte is present (for currency/decimal context) but the transaction byte is absent. The amounts in a pay summary State Commit are cumulative totals, not single transactions.

**Period summary (State Commit 11):** An annual or periodic aggregate. Carries total income (I), total expense (O), and net. Useful for a year-end summary record in a chain. The financial block uses COMPOUND_VALUE=1 with line items by category.

---

## 7. UI Labels

Workers do not see I>O notation. They see derived labels:

| State | Worker UI label | Customer UI label |
|-------|----------------|-------------------|
| `I < I` | Payment received | Paid |
| `I > I` | Invoice sent | Invoice |
| `I > I` sub=01 | Quote sent | Quote |
| `I < O` | Refund given | Refunded |
| `O < O` | Expense recorded | — |
| `O > O` | Bill pending | — |
| `O < I` | Reimbursed | — |
| `O > I` | Reimbursement pending | — |

Labels are derived at decode time from the transaction byte. Future codebook packages may define label strings directly in the codebook — for now, labels are hardcoded in the decoder per state + subtype combination.

---

## 8. Entry Type Matching Table (DOMAIN=11 Deterministic Mapping)

When the entry type is known at creation time (from the wizard screen the worker used), every combination resolves deterministically to a single Account Pair code with no inference required. This table is used for DOMAIN=11 hybrid records.

The `1110` Correction/Netting code is written only for API-generated records that bypass the wizard.

### Income-side entry types

| Wizard Entry Type | I>O State | Account Pair | AP_DIRECTION |
|-------------------|-----------|--------------|--------------|
| Invoice (send bill) | I>I | `0101` Op Income / Liability | 0 (debit receivable) |
| Cash sale (paid now) | I<I | `0100` Op Income / Asset | 0 (debit cash) |
| Payment received (settling invoice) | I<I | `0101` Op Income / Liability | 1 (credit receivable — clears it) |
| Refund given | I<O | `0100` Op Income / Asset | 1 (credit cash out) |
| Credit note issued | I>O | `0101` Op Income / Liability | 1 (credit receivable reversal) |
| Quote / Estimate | I>I | `0101` Op Income / Liability | 0 (same as invoice; DRAFT=1) |

### Expense-side entry types

| Wizard Entry Type | I>O State | EXPENSE_CAT | Account Pair | AP_DIRECTION |
|-------------------|-----------|-------------|--------------|--------------|
| Running cost — cash paid | O<O | `10` | `0000` Op Expense / Asset | 0 |
| Running cost — bill received | O>O | `10` | `0001` Op Expense / Liability | 0 |
| Running cost — bill paid | O<O | `10` | `0001` Op Expense / Liability | 1 (clears payable) |
| Job cost / COGS — cash paid | O<O | `01` | `0000` Op Expense / Asset | 0 |
| Job cost / COGS — bill received | O>O | `01` | `0001` Op Expense / Liability | 0 |
| Job charge — billed to customer | O<O | `00` | `0100` Op Income / Asset | 0 (COGS offset) |
| Reimbursement given (advance) | O>I | — | `1001` Asset / Equity | 0 |
| Reimbursement received | O<I | — | `1001` Asset / Equity | 1 |

### Balance sheet entry types (post-MVP wizard)

| Wizard Entry Type | I>O State | Account Pair | AP_DIRECTION |
|-------------------|-----------|--------------|--------------|
| Asset purchase — cash | O<O | `1011` Asset / Asset | 0 |
| Asset purchase — on finance | O>O | `1000` Asset / Liability | 0 |
| Loan repayment | O<O | `1000` Asset / Liability | 1 |
| Owner contribution | I<I | `1001` Asset / Equity | 0 |
| Owner draw / distribution | O<O | `1001` Asset / Equity | 1 |
| Internal transfer | O<O | `1011` Asset / Asset | 0 |

---

## 9. Type-Change Reconciliation

When a worker changes the entry type mid-creation, the app reconciles the frame encoding in cascade order:

**Step 1 — Direction change (I↔O flip):**
- Amounts preserved; DIRECTION bit in transaction_byte flips
- EXPENSE_CAT resets to `10` (running cost) as the safe default when direction flips to O
- Account Pair recalculates from new entry type

**Step 2 — Settlement state change (Past↔Future, TIME bit):**
- No field data changes — only TIME bit flips
- Account Pair recalculates (e.g. `O<O` → `O>O` shifts Asset pair to Liability pair)

**Step 3 — EXPENSE_CAT change (within O-direction):**
- No field data changes; fin_control EXPENSE_CAT bits update
- Account Pair recalculates (job charge ↔ COGS ↔ running cost have different pairs)
- BILLED flag updates (EXPENSE_CAT=00 forces BILLED=1)

**Invariant:** Worker-entered amounts are never silently discarded on direction changes. The amount is relabelled, not erased.

---

## 10. Accounting Detail Display

When "Show accounting detail" is toggled on, plain-English labels are derived from `ACCOUNT_PAIR` + `AP_DIRECTION`:

| Account Pair | AP_DIRECTION=0 | AP_DIRECTION=1 |
|---|---|---|
| `0000` Op Expense / Asset | Debit: Expenses / Credit: Cash | Debit: Cash / Credit: Expenses (reversal) |
| `0001` Op Expense / Liability | Debit: Expenses / Credit: Payable | Debit: Payable / Credit: Expenses (payment) |
| `0100` Op Income / Asset | Debit: Cash / Credit: Income | Debit: Income / Credit: Cash (refund) |
| `0101` Op Income / Liability | Debit: Receivable / Credit: Income | Debit: Income / Credit: Receivable (credit note) |
| `1000` Asset / Liability | Debit: Fixed Asset / Credit: Finance Liability | Debit: Finance Liability / Credit: Cash (repayment) |
| `1001` Asset / Equity | Debit: Cash / Credit: Capital | Debit: Capital / Credit: Cash (distribution) |
| `1011` Asset / Asset | Debit: Asset (dest) / Credit: Asset (src) | reverse |
| `1110` Correction / Netting | "Unclassified — review needed" | — |

**Display format in app:**
```
▼ Accounting detail
  Type:    Operating Expense
  Debit:   Expenses (Operating)
  Credit:  Assets (Cash)
```

No Account Pair code numbers are shown to workers. Labels are derived from the pair's primary account category.

---

## 12. Notes on Design

The I>O notation was developed during the pads-v1 design session from first principles, then cross-checked against BitLedger's 8-state accounting model and Universal Domain archetypes. The alignment is intentional but not mechanical — workpads uses the 8-state grid as a conceptual framework, not as a literal implementation of BitLedger Layer 2.

**Why `<` for Past:** The arrow points backward, toward the completed transaction. The past is behind the present moment. `I < I` reads "Income, closed, landed."

**Why not debit/credit terminology:** Debit/credit is accounting convention, not field-worker language. "Payment received" and "Invoice sent" are the worker's natural language. The I>O grid maps to these labels without requiring accounting literacy.

**BitLedger relationship:** BitLedger defines a full financial record with scaling factors, decimal positions, and value formulas. pads-v1 borrows the rounding rules, the 8-state grid concept, and the uint-based value encoding. The BitLedger N = A × 2^S + r value formula is the inspiration for pads-v1's uint24 + DECIMAL_POS approach, simplified for the workpads use case.
