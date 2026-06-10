
## 1. Protocol Fundamentals and Data Model

### 1.1 The PADS Model

The "Process, Actions, Details, Story" (PADS) framework defines the structural and temporal lifecycle of a Workpads record. Information is categorized into four distinct phases to support progressive disclosure and field-service workflows.

|   |   |   |
|---|---|---|
|Section|Role|Temporal Phase|
|**P**rocess|Defines core identity: job title, customer, and site.|Pre-job (Planning/Dispatch)|
|**A**ctions|Procedural log or checklist items.|During job (Execution)|
|**D**etails|Logistics: timing, personnel attribution, and contacts.|During or After job|
|**S**tory|Narrative findings and technical observations.|Post-job (Documentation)|

### 1.2 Naming Conventions

Workpads identifiers map to BASICS standard concepts but utilize concise, role-neutral synonyms to optimize for constrained-device input (e.g., D-pads) and human memorability (Reference: **DEV-WP-001**).

|   |   |   |
|---|---|---|
|Workpads Field|BASICS Concept|Compact Key|
|`job`|Command / Work-order descriptor|`j`|
|`customer`|Operator-designated recipient|`c`|
|`date`|Record timestamp / event date|`d`|
|`location`|Deployment site / operator context|`l`|
|`meeting_time`|Scheduled activation time|`mt`|
|`start_time`|Work commencement timestamp|`st`|
|`end_time`|Work completion timestamp|`et`|
|`customer_phone`|Operator contact reference|`cp`|
|`worker`|Assigned operator / executing agent|`w`|
|`actions[].title`|Command sequence step|`at`|
|`actions[].notes`|Command sequence step notes|`an`|
|`details`|Field observation record|`de`|
|`story`|Narrative evidence record|`sy`|

### 1.3 Field Invariants

- **Mandatory Field:** The `job` field (bit 0) is the only required field; all other fields are optional.
- **Scalar Types:** All scalar data MUST be treated as UTF-8 strings.
- **Omission Rule:** Absent or null fields MUST be omitted from the binary frame to conserve bandwidth. Empty strings ("") are treated as present and encoded with a length of 0.
- **Flat Structure:** While conceptually grouped by PADS, the runtime record is a flat object to ensure encoding efficiency and stable bit-slot mapping.

--------------------------------------------------------------------------------

## 2. The pads-v1 Codec Specification

### 2.1 The Encoding Pipeline

The codec transforms a record object into a URL-safe hash fragment via a four-step deterministic sequence:

1. **Assemble:** Construct the binary frame header and body.
2. **Compress:** Apply DEFLATE compression using `fflate.deflateSync` at Level 9 complexity.
3. **base64url:** Convert the compressed bytes to a base64url string (replacing `+` with `-`, `/` with `_`, and stripping all `=` padding).
4. **Hash Fragment:** Append the payload to the URL origin and scheme tag.

### 2.2 URL Structure

The Workpads URL is a "URL-as-Document," carrying the complete state of the record within the hash fragment to ensure serverless portability.

|   |   |   |
|---|---|---|
|Component|Example/Value|Description|
|**Origin**|`https://workpads.me`|Canonical base for sharing.|
|**Path**|`/p`|Record receiver endpoint.|
|**Scheme Tag**|`1bg`|3-character encoding descriptor.|
|**Separator**|`/`|Literal path separator.|
|**Payload**|`<base64url>`|Compressed binary frame.|
|**Optional Chain**|`&c=XXXX`|4-character base64url chain reference.|

### 2.3 The Scheme Tag

The scheme tag identifies the codec generation. Decoders MUST check the second character (Codebook) to determine the parsing logic for bit slots 12–15.

|   |   |   |
|---|---|---|
|Position|Character|Representation|
|0|`1`|Format Version (pads-v1).|
|1|`a` / `b`|Codebook (`a` = initial/legacy; `b` = financial block enabled).|
|2|`g`|Compression (DEFLATE via fflate).|

--------------------------------------------------------------------------------

## 3. Binary Frame Architecture

### 3.1 Frame Header Construction

The header is a variable-length bitstream defining the record type and field availability.

**Meta Byte 1 (Byte 0):**

- Bit 7: `META2_PRESENT` (1 if Meta Byte 2 follows).
- Bits 6–3: `TEMPLATE_ID` (4-bit value; 0x1 = `svc-basic`).
- Bit 2: `ACK_REQUEST` (Sender requests a State Commit ACK).
- Bit 1: `CHAIN` (1 if `&c=` reference is present in URL).
- Bit 0: `RECIPIENT_TYPE` (0 = Customer, 1 = Colleague). Controls progressive disclosure.

**Meta Byte 2 (Optional):** Required if any extended bit is set. Includes `DOMAIN` (bits 3-2), where `01` or `11` enables the Financial Block.

**Presence Flags (2 Bytes):** A 16-bit field signaling which data blocks follow. If Bit 15 is set, a third flags byte MUST follow.

All multi-byte integers, including the Presence Flags, uint16 lengths, and uint24 monetary values, MUST use **Big-Endian** byte order.

- Correct Flag Read: `flags = (frame[offset] << 8) | frame[offset + 1]`
- Incorrect Little-Endian reads will result in catastrophic field misalignment.

### 3.2 Field Slot Assignments (Codebook `b`)

Blocks follow the header in strict ascending bit-position order.

|   |   |   |
|---|---|---|
|Bit|Field Key|Type|
|0|`job`|Scalar (Required)|
|1|`customer`|Scalar|
|2|`date`|Scalar / Binary (if COMPACT_TIME)|
|3|`location`|Scalar|
|7|`customer_phone`|Scalar|
|8|`worker`|Scalar (Legacy)|
|9|`actions`|Actions Blob|
|10|`details`|Scalar|
|11|`story`|Scalar|
|12|`financial block`|Structured Financial Data (uint24)|
|15|`FLAGS3_PRESENT`|Meta-flag for Byte 3 presence|

--------------------------------------------------------------------------------

## 4. Data Block Structures

### 4.1 Scalar and Actions Blobs

- **Scalar Format:** Encoded as `[uint16 length][UTF-8 data]`.
- **Actions Array:** Encoded as `[uint8 count]`, followed by sequential items. Each item contains a `title blob` and a `notes blob` (both `uint16` length-prefixed).
    - _Constraint:_ Records MUST NOT exceed 20 actions.

### 4.2 The Financial Block (Bit 12)

Amounts are stored as **uint24 Big-Endian** integers to ensure exact precision without floating-point errors.

- **Formula:** `monetary_amount = uint24_value / 10^DECIMAL_POS`.

**Setup Byte (Binary Environment):**

- Bits 7–5: `DECIMAL_POS` (0–5; default 2).
- Bits 4–3: `CURRENCY` (**00=GBP**, 01=EUR, 10=USD, 11=Local).
- Bits 2–1: `TAX_CODE` (**00=None**, 10=Standard, 11=Explicit).
- Bit 0: `COMPOUND_VALUE` (1 = Multi-line).

**Transaction Byte (Classification):** Includes `DIRECTION` (Bit 7), `TIME` (Bit 6), and `EFFECT` (Bit 5). These 3 bits form the I>O state.

### 4.3 The Participants Block (Meta 2: Bit 4)

This block handles identity and roles. It MUST be filtered during encoding if `RECIPIENT_TYPE` is set to 0 (Customer).

**Role Type Codebook:** | Code | Role | | :--- | :--- | | 000 | Worker (General) | | 001 | Job owner / Supervisor | | 100 | Site contact | | 110 | Witness / Verifier |

**Progressive Disclosure:** If `RECIPIENT_TYPE=0`, the encoder MUST omit internal colleague identities and internal business costs (e.g., `worker_amount`).

--------------------------------------------------------------------------------

## 5. Chain Protocol and State Management

### 5.1 24-Bit Composite Chain ID

Chains allow records to reference ancestors without a central database.

|   |   |   |   |
|---|---|---|---|
|Bits|Name|Width|Description|
|23–7|**Anchor**|17 bits|Device-specific origin identifier.|
|6–3|**Slot**|4 bits|Participant slot (Slot 0 = Authoritative).|
|2–0|**Sequence**|3 bits|Position (0–6 = Sequence; 7 = Continuation).|

**Anchor Generation:** Anchors MUST be generated using `crypto.getRandomValues()` and persisted locally as `wp_device_anchor`.

### 5.2 Protocol Mechanisms

- **ACK Mechanism:** If `ACK_REQUEST` is set, the recipient MUST be prompted to generate a **State Commit (Template 0xD)** record (subtype `00`), which is returned to the sender.
- **Amendment Records (Template 0xE):** These records carry only changed fields. Decoders MUST merge these onto the sequence-0 record.
    - _Immutable Fields:_ Anchor, Timestamp, Sender, and Template Type MUST NOT be amended.
- **Sequence 7:** If a chain exceeds 7 records, sequence 7 serves as a **Chain Continuation** marker to anchor a new 24-bit ID.

### 5.3 Storage Requirements

1. **Binary Frames:** Records MUST be stored locally as raw binary frames (saving ~35% space vs. URL strings).
2. **On-Demand Reconstruction:** URLs MUST be reconstructed from stored frames only at the moment of sharing.

--------------------------------------------------------------------------------

## 6. Transaction Classification (I>O Notation)

The 8-state system (D:T:E) ensures BitLedger-aligned rounding and correct UI labeling.

|   |   |   |   |
|---|---|---|---|
|Notation|Binary Value|Worker UI Label|Subtype (01)|
|`I < I`|0:0:0|Payment received|Deposit|
|`I > I`|0:1:0|Invoice sent|Quote / Estimate|
|`O < O`|1:0:1|Expense paid|Travel / Materials|
|`O > O`|1:1:1|Bill received|Supplier Order|

**Arrow Logic:**

- `<` (Past/Settled): The exchange has occurred; money has landed.
- `>` (Future/Pending): The exchange is open; money is expected.

--------------------------------------------------------------------------------

## 7. Implementation and Interoperability

### 7.1 Codec Sync Protocol

To prevent data drift, all implementations MUST maintain identical configurations for:

- **Scheme Tag Regex:** `/[0-9][a-z][a-z]\//`
- **Scalar Fields Array:** Strict order of bits 0–11.
- **Byte Order:** Big-Endian for all multi-byte values.

### 7.2 Implementation Deviations

|   |   |   |
|---|---|---|
|ID|Description|Resolution|
|`DEV-WP-URL-001`|KaiOS v0.1 uses query-strings (`?v=1&alg=...`).|MUST migrate to hash-fragment (`#1bg/`) in v0.2.|

### 7.3 Forward Compatibility

- **Fallback Chain:** If a template URI is unavailable, the decoder MUST fall back through superseded versions to the **Built-in Default** (`urn:workpads:tpl:note:default:v1`).
- **Template Diffusion:** Templates are distributed via `<script type="application/workpads-template">`. Once ingested, payloads are cached locally, ensuring the record remains renderable even if the original source site is offline.