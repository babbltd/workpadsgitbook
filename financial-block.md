# Financial Block Specification

**Status:** v0.2 — 2026-05-18 (SUI-005: fin_control BILLED/EXPENSE_CAT/PARITY added)
**Decision source:** decisions.md R-PADS-005, R-PADS-009, R-PADS-010, R-PADS-011, R-PADS-012
**Cross-references:** pads-v2-encoding-spec.md §6-8, transaction-classification.md
**BitLedger alignment:** BitLedger_Protocol_v3.md N = A × 2^S + r value formula; Layer 2 scaling; rounding rules

---

## 1. Overview

The financial block is the pads-v1 mechanism for encoding monetary amounts in workpad records. It is activated by:
- DOMAIN ≥ 01 (financial) in meta2 bits 3-2
- Field flags bit 12 set

The financial block is designed around three constraints:
1. **Compact** — most invoices are a single amount. The common case should be minimal bytes.
2. **Complete** — an invoice needs: amount, optional tax, optional worker cost, optional breakdown (qty × rate). All present without extra signaling once the setup context is established.
3. **Universal** — currency, decimal position, and tax rate are configuration, not structure. The same block format works for GBP at 20% VAT and USD at 0% or EUR at any rate.

---

## 2. Context: Setup Byte and Transaction Byte

Before the financial block is read, two context bytes establish the numeric environment:

**Setup byte** (see pads-v2-encoding-spec.md §6):
- DECIMAL_POS (bits 7-5): decimal places for all amounts in this record
- CURRENCY (bits 4-3): 2-bit currency code — default sourced from active Activity Profile `currency` field
- TAX_CODE (bits 2-1): tax treatment — default `10` (standard rate) if active Activity has `vatRegistered: true`, else `00` (none). See `workpads-standard/activity-profile.md`.
- COMPOUND_VALUE (bit 0): single vs multi-line

**Transaction byte** (see pads-v2-encoding-spec.md §7 and transaction-classification.md):
- DIRECTION + TIME + EFFECT: 8-state classification
- SUBTYPE: specific transaction type within state
- QTY_SPLIT: lump sum vs qty × rate
- ROUNDING: rounding direction for the amount

---

## 3. uint24 Value Encoding

All monetary amounts in pads-v1 are encoded as uint24 — a 3-byte big-endian unsigned integer.

**Interpretation:** monetary_amount = uint24_value ÷ 10^DECIMAL_POS

**Examples at DECIMAL_POS=2:**
| uint24 value | Monetary amount |
|-------------|----------------|
| 0x00_07_D0 (2000) | £20.00 |
| 0x00_22_B8 (8888) | £88.88 |
| 0x00_5B_8D (23437) | £234.37 |
| 0xFF_FF_FF (16777215) | £167,772.15 |

**Encoder DECIMAL_POS selection:**
- Default: DECIMAL_POS=2 (pennies/cents)
- If amount > £167,772.15 at D=2: set DECIMAL_POS=0 (whole pounds). Encode: `round(amount_in_pounds)` as uint24. Max at D=0: £16,777,215.
- If sub-penny precision required (fuel rates, per-minute billing): DECIMAL_POS=3. Max at D=3: £16,777.215.

**Why uint24 (not uint16 or uint32):**
- uint16 max at D=2: £655.35 — insufficient for most jobs
- uint32 = 4 bytes — 1 extra byte vs uint24 with no benefit for practical amounts
- uint24 at D=2 covers £167K — the 99th percentile of field service invoices

**BitLedger lineage:** BitLedger uses N = A × 2^S + r (25-bit value with 7-bit scaling factor). pads-v1 simplifies this to fixed DECIMAL_POS (no per-record scaling factor) for encoder simplicity. The concept — a compact integer with separately declared scale — is directly inherited from BitLedger Layer 3.

---

## 4. Tax Block

When TAX_CODE (setup byte bits 2-1) = 11 (explicit rate):

```
[1 byte ] tax_rate_permille  — tax rate as permille (‰): value/10 = rate%
                               e.g. 200 = 20.0% VAT; 125 = 12.5%; 75 = 7.5%
[2 bytes] tax_amount         — uint16: tax amount with same DECIMAL_POS as main amount
```

**When TAX_CODE = 10 (standard rate):** No tax block bytes. The rate is contextually implied from the currency and the parties' jurisdiction. The decoder labels the amount as "inc. tax" or shows the applicable standard rate from locale context. Encoders using code 10 should include the rate in the Details field if there is any ambiguity about jurisdiction.

**When TAX_CODE = 00 or 01:** No tax block bytes. Code 00 = no tax applies. Code 01 = zero-rated (tax is applicable but rate is 0%).

**Tax rounding:** Always UP (BitLedger rule). If the calculated tax rounds to a fractional unit, round the tax_amount up.

**Example:** Invoice for £125.00 at 20% VAT explicit (TAX_CODE=11):
```
customer_amount: uint24 = 15000  (DECIMAL_POS=2, £150.00 total inc tax)
tax_rate_permille: 200            (20.0%)
tax_amount: uint16 = 2500         (£25.00 tax)
```

---

## 5. Worker Amount

The worker amount is the sender's internal cost or margin figure — what the job costs the sender, separate from what the customer pays. It is encoded when field_flags3 bit 7 (WORKER_AMOUNT) is set.

```
[3 bytes] worker_amount  — uint24, same DECIMAL_POS as customer_amount
```

**Purpose:** A tradesperson charging a customer £150 may have £60 in materials and subcontractor costs. The worker amount records the sender's actual cost, enabling margin calculation in the management screen's Exchange tab.

**Progressive disclosure:** The worker_amount is never included in customer-facing URLs. Encoders must omit field_flags3 and the worker_amount block when RECIPIENT_TYPE=0 (customer). Colleague-facing URLs may include it.

---

## 6. Quantity × Rate Split

When QTY_SPLIT=1 (transaction byte bit 2):

```
[3 bytes] qty    — uint24: quantity (hours, units, items, etc.)
[3 bytes] rate   — uint24: rate per unit
```

Both qty and rate use the same DECIMAL_POS as the main amount. The decoder may display:
- qty formatted as a human count (e.g. "2.5 hours")
- rate formatted as a monetary rate (e.g. "£45.00/hr")
- customer_amount as the total

If `qty × rate ≠ customer_amount` due to rounding or discounts, the customer_amount is the authoritative total. The qty × rate breakdown is for transparency / audit, not recalculation.

**Unit label:** Optional. When qty and rate represent non-standard units (not hours), a unit label may be carried in a free-text Details field rather than in the financial block. There is no unit string field in the financial block itself — encoders use the Details field for context.

---

## 6b. Financial Block Control Byte (fin_control)

The first byte of every financial block is `fin_control`. It is always present when field_flags bit 12 is set.

### DOMAIN=01 (simple mode — I>O perspective)

```
[fin_control]    1 byte — always present when financial block active

  bit 7: BILLED         1=charge passed to customer (appears as invoice line item)
                        Note: EXPENSE_CAT=00 forces BILLED=1 automatically.
  bit 6: 0              must be 0 — mode indicator; integrity check 1
  bit 5: QTY_TYPE       0=units/items  1=time/hours (display context for qty)
  bit 4: PARITY         even parity of bits 7, 5, 3, 2, 1, 0 — integrity check 2
  bits 3-2: EXPENSE_CAT 00=job charge (billed to customer; BILLED forced to 1)
                        01=job cost (COGS, absorbed margin; not shown to customer)
                        10=running cost (business overhead; no parent job)
                        11=RESERVED (integrity check 3: must not appear)
  bit 1: CUSTOMER_AMT   1=customer_amount uint24 follows
  bit 0: WORKER_AMT     1=worker_amount uint24 follows
```

**Parity formula:**
```
PARITY = BILLED XOR QTY_TYPE XOR EXPENSE_CAT[1] XOR EXPENSE_CAT[0] XOR CUSTOMER_AMT XOR WORKER_AMT
```

**Four integrity checks (simple mode):**
1. bit 6 = 0 (mode indicator; mismatch signals DOMAIN mismatch or corruption)
2. bit 4 = PARITY (detects any single-bit flip across 6 content bits)
3. EXPENSE_CAT ≠ 11 (reserved code; signals future-version record or corruption)
4. QTY_TYPE=1 requires QTY_SPLIT=1 in transaction_byte (cross-field consistency)

**EXPENSE_CAT semantics (for field service workers):**

| Code | Category | BILLED | Visible to customer |
|------|----------|--------|---------------------|
| `00` | Job charge | Yes (forced) | Yes — appears on invoice |
| `01` | Job cost (COGS) | No | No — internal margin tracking |
| `10` | Running cost | No | No — business overhead, no specific job |

**Examples (DECIMAL_POS=2):**
```
Payment received, £125.50:
  fin_control = 0x12  (BILLED=0, bit6=0, QTY_TYPE=0, PARITY=1, EC=00, CUST=1, WORK=0)
  customer_amount uint24 = 12550

Expense paid, materials £45 (job cost):
  fin_control = 0x06  (BILLED=0, bit6=0, QTY_TYPE=0, PARITY=0, EC=01, CUST=1, WORK=0)
  Wait: PARITY = 0 XOR 0 XOR 0 XOR 1 XOR 1 XOR 0 = 0 → bit4=0
  fin_control = 0b00000110 = 0x06

Running cost, insurance £200:
  EXPENSE_CAT=10, BILLED=0, CUSTOMER_AMT=1, WORKER_AMT=0
  PARITY = 0 XOR 0 XOR 1 XOR 0 XOR 1 XOR 0 = 0 → bit4=0
  fin_control = 0b00001010 = 0x0A
```

### DOMAIN=10 (standard mode — BitLedger Account Pair)

```
  bit 7: BILLED         1=charge passed to customer
  bit 6: 1              must be 1 — mode indicator; integrity check
  bits 5-2: ACCOUNT_PAIR  4-bit BitLedger Account Pair (0000–1101 active; 1110/1111 reserved)
  bit 1: CUSTOMER_AMT   1=customer_amount uint24 follows
  bit 0: WORKER_AMT     1=worker_amount uint24 follows
```

See `transaction-classification.md §3` for the Account Pair code table.

---

## 7. Simple Financial Block

When COMPOUND_VALUE=0 (single amount):

```
[3 bytes] customer_amount         — always present when bit 12 set
[if TAX_CODE=11] tax block        — 3 bytes (rate + amount)
[if WORKER_AMOUNT flag] [3 bytes] worker_amount
[if QTY_SPLIT=1] [3+3 bytes] qty + rate
```

**Minimum simple block:** 3 bytes (customer_amount only, no tax, no worker amount, no qty/rate).
**Maximum simple block:** 3 + 3 + 3 + 6 = 15 bytes.

---

## 8. Compound Financial Block

When COMPOUND_VALUE=1 (multi-line):

```
[1 byte ] line_count              — number of line items (1–20, uint8)
for each line:
  [1 byte ] charge_type           — codebook code (0–15), see §9
  [3 bytes] amount                — uint24 for this line item
  [if TAX_CODE=11] tax block      — 3 bytes
  [if QTY_SPLIT=1] [3+3 bytes] qty + rate
  [if charge_type=0] [2+N bytes] label  — uint16 + UTF-8 custom label
```

**Line count 1 is valid** — a compound block with one line item is semantically the same as a simple block but carries a charge_type code. Use simple (COMPOUND_VALUE=0) for generic amounts; use compound even with one line when a specific charge type needs to be declared.

**Total:** The decoder sums all line amounts for display as a total. Decoders should show: individual line amounts + labels, subtotal, tax (if applicable), total.

**Example: T&M billing with urgency surcharge:**
```
line_count: 3
  line 1: charge_type=0 (custom), amount=9000 (£90.00), label="Labour 2hr @ £45"
  line 2: charge_type=3 (travel), amount=1500 (£15.00)
  line 3: charge_type=1 (urgency premium), amount=2000 (£20.00)
Total: £125.00
```

---

## 9. Charge Type Codebook (codebook package `a`)

| Code | Charge type | Notes |
|------|------------|-------|
| 0 | Custom | Free-text label block required |
| 1 | Urgency / emergency premium | After-hours callout surcharge, bank holiday, same-day |
| 2 | After-hours / out-of-hours | Standard after-hours rate (not emergency) |
| 3 | Travel / mileage | Vehicle or transit cost; may use QTY_SPLIT for distance × rate |
| 4 | Delivery / courier | Physical item delivery charge |
| 5 | Equipment hire | Rented equipment for the job |
| 6 | Consumables / materials markup | Worker's markup on parts/supplies purchased for the job |
| 7 | Subcontractor cost | Third-party work included in the quote |
| 8 | Cancellation / no-show fee | Charged when job was cancelled after dispatch |
| 9 | Deposit / retainer | Initial payment, deductible from final invoice |
| 10 | Credit / discount | Negative adjustment (encoder should handle as negative uint24 via complement or as signed convention documented in role_text) |
| 11 | Warranty / guarantee adjustment | Remedial work under warranty (zero or reduced charge) |
| 12 | Regulatory / compliance levy | Environmental levy, licensing surcharge, compliance fee |
| 13 | Currency / FX adjustment | Multi-currency adjustment for cross-border work |
| 14 | Payment handling / processing fee | Card payment surcharge, bank transfer fee |
| 15 | Multi-charge | Nested compound sub-block (for complex multi-tier billing — future extension) |

**Credit/discount (code 10):** When a line item is a reduction (discount, credit note), the amount is the reduction value as a positive uint24. The decoder's display logic should show it as a negative/subtracted line item based on the charge_type code. This avoids signed integer complexity in the uint24 scheme.

**Custom (code 0):** Required to carry a label block (2 bytes uint16 + UTF-8). No default label. Use this for any charge type not in the codebook.

---

## 10. Example Byte Counts

**Simple invoice, £150 inc 20% VAT (TAX_CODE=11), DECIMAL_POS=2:**
```
setup byte: 1 (DECIMAL_POS=2, GBP, TAX_CODE=11, COMPOUND_VALUE=0)
transaction byte: 1 (I > I, invoice, exact)
field_flags: 2 (bit 12 set)
customer_amount: 3 (uint24: 15000 = £150.00)
tax block: 3 (rate=200‰, tax=uint16 2500 = £25.00)
Total financial area: 10 bytes
```

**T&M billing, 2.5hr @ £45/hr, travel £15, urgency £20, no tax:**
```
setup byte: 1 (DECIMAL_POS=2, GBP, TAX_CODE=00, COMPOUND_VALUE=1)
transaction byte: 1 (I > I, invoice, QTY_SPLIT=1 for line 1)
field_flags: 2 (bit 12 set)
line_count: 1 (3 lines)
line 1: charge_type(1) + amount(3) + qty(3) + rate(3) + label(2+17) = 29 bytes
line 2: charge_type(1) + amount(3) = 4 bytes
line 3: charge_type(1) + amount(3) = 4 bytes
Total financial area: 42 bytes raw
```

After deflate+b64url on a full T3 record with this financial block: approximately 97–120 chars URL.

---

## 11. Notes on Design Decisions

### Why not JSON for amounts?

JSON would encode `{"amount": 125.50, "tax": 25.10, "currency": "GBP"}` as ~50 bytes before the field names. The financial block encodes the same as ~10 bytes. For a record that may be shared via SMS (160 chars), the difference is the record being shareable or not.

### Why not a decimal float?

Floating-point amounts introduce rounding errors at the representation level. uint24 with declared DECIMAL_POS is an exact integer encoding — £125.50 is exactly 12550 at D=2, with no floating-point representation error. This matches BitLedger's design philosophy.

### Why implicit standard tax rate (TAX_CODE=10)?

Adding 3 bytes per record to carry a rate that is the same for every record in a jurisdiction wastes the budget of every record. Workers in a single jurisdiction always operate at the same rate — encoding it each time is redundant. The explicit rate (TAX_CODE=11) exists for cross-jurisdiction records or non-standard rates.

### Credit as positive uint24

Encoding a credit as a negative number requires signed integer handling (two's complement, sign bit). This complicates the decoder. Encoding credits as positive amounts with a charge_type=10 code keeps all uint24 values non-negative and places the sign semantics in the codebook — a cleaner architecture for a constrained-device decoder.
