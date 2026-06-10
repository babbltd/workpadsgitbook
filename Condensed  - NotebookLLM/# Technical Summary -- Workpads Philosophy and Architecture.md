
This document serves as the normative architectural summary for the Workpads ecosystem. It details the transition from traditional centralized record-keeping to a sovereign, URL-transmissible framework designed for high-consequence fieldwork on resource-constrained hardware.

## 1. Core Philosophy: The Atomic Unit of Commerce

In the Workpads architecture, the record is the **Atomic Unit of Commerce**. We reject the model where records are transient views of a central database; instead, each workpad record is an "atom"—a standalone, portable, and sovereign entity.

### The Five Properties of a Workpad Record

Every conformant record must adhere to these five structural properties:

- **Completeness:** The record must be self-describing. A recipient must understand the full commercial context without requiring external cover messages or platform access.
- **Compactness:** Records are architected for the "Zero-Data" worker. They must be small enough to travel via SMS (160-character limits) or high-density QR codes.
- **Honesty:** Honesty is a protocol-level signal. The codec enforces "truth claims" through specific flags (e.g., `DRAFT`, `ACK_REQUEST`), making honest state-sharing the default and deceptive state-tampering structurally difficult.
- **Sovereignty:** The worker is the sole arbiter of the record. Data is user-sovereign, residing locally and only transmitted through explicit user action.
- **Chain-ability:** Records are not isolated; they "speak" to one another via a cryptographic chain protocol, allowing a sequence of snapshots (Quote → Job → Invoice) to form a verifiable commercial relationship.

### The URL-as-Document Model

Workpads shifts the paradigm from file hosting to data-encoding. By separating data from its visual representation, we eliminate the need for document storage.

"Content is data; Presentation is a template."

In this model, **Content** (the PADS data) is encoded into a URL hash fragment, while **Presentation** (the template) is a cached local resource. The "document" is rendered client-side by a stateless orchestrator, ensuring it exists only at the moment of viewing.

## 2. The Personal Platform: Dual-Engine Architecture

Workpads is a "Personal Platform" that bifurcates the worker's digital life into two distinct engines. This split is managed by the **Active Activity Profile**, which provides the identity context (`IS_SENDER`) and financial defaults (e.g., `TAX_CODE`) to the encoder.

|   |   |   |
|---|---|---|
|Dimension|Exchange Engine|Learning Engine|
|**Primary Service**|`RecordService`|`PersonalService`|
|**Storage Namespace**|`wp_rec_*` (Active), `wp_arc_*` (Archive)|`wp_per_*` (Captures)|
|**Function**|What the system **does** (Generation/Transmission)|What the worker **accumulates** (Insights/Knowledge)|
|**Sharing Mechanism**|Explicit (Share link / QR)|Opt-in (Export / Link)|

### The Meta-Activity Concept

The architecture recognizes that every work task generates an "ether" of ambient observations and personal learnings. The **Learning Engine** captures this meta-activity without interrupting the primary workflow, storing captures privately and separately from the commercial exchange records.

## 3. The PADS Data Model

The PADS model provides the semantic structure for every record. It is a **flat object structure** where all scalar fields are strings.

|   |   |   |   |
|---|---|---|---|
|Section|Compact Key|Purpose|Filled|
|**P — Process**|`j`, `c`, `d`, `l`|"What and for whom?" (Job, Customer, Date, Location)|Pre-Job|
|**A — Actions**|`at`, `an`|"Specific steps taken?" (Actions Array: Title/Notes)|During Job|
|**D — Details**|`w`, `st`, `et`, `mt`, `cp`|"Who and when?" (Worker, Times, Contact)|During/Post|
|**S — Story**|`sy`, `de`|"What happened?" (Narrative Story, Technical Details)|Post-Job|

### Technical Invariants and Constraints

- **Required Field:** `job` (`j`) is the only mandatory field for a valid record.
- **Byte-Length Limits:** All field limits are defined by **UTF-8 byte counts**, not character counts. For example, the `job` field is limited to 120 bytes to ensure URL stability across different languages (CJK, Arabic, etc.).
- **Identity Fields:** System-managed fields (`id`, `createdAt`, `updatedAt`, `receivedAt`) are distinct from human-entered PADS data and are never transmitted in the URL fragment.

## 4. Compact Encoding & Codec Specification (pads-v1)

The `pads-v1` codec is a binary-text hybrid designed for maximum density.

### The Encoding Process

1. **Binary Frame Assembly:** Constructing a byte array following the frame format.
2. **Deflate Compression:** Applying `fflate.deflateSync` at compression level 9.
3. **base64url Encoding:** Conversion to a URL-safe string with **all padding (**`**=**`**) stripped**.
4. **Hash Fragment Production:** Prepending the 3-character **Scheme Tag**.

### Scheme Tag Specification (e.g., `1bg`)

|   |   |   |
|---|---|---|
|Position|Character|Meaning|
|0|`1`|Format Version (pads-v1)|
|1|`b`|Codebook (e.g., `a` = initial, `b` = financial extension)|
|2|`g`|Compression (always `g` for DEFLATE via fflate)|

### Binary Frame Format

The frame begins with a **3-byte header**:

- **Byte 0:** Template ID (e.g., `0x01` for `svc-basic`).
- **Bytes 1-2:** 16-bit **Presence Flags** (Big-Endian `uint16`). Bit 9 is reserved for the **Actions Blob**, which contains a `count` byte followed by length-prefixed title/notes pairs.

All multi-byte integers in the frame (Presence Flags, lengths) MUST use **Big-Endian** bit-shifting. Decoders must implement as `(frame[1] << 8) | frame[2]`. Little-Endian reading will cause catastrophic byte-offset cascades and data corruption.

**Codec Sync Protocol:** To prevent silent data corruption, all implementations (NPM, KaiOS inline, Web inline) must remain byte-for-byte identical as per the `codec-sync.md` specification.

## 5. Financial Architecture and Transaction Classification

Workpads utilizes **I>O Notation** (Direction, Time, Effect) to classify transactions, mapping 8 primary states to specific worker-facing labels.

### The 8 Primary States

|   |   |   |
|---|---|---|
|Notation|Plain Meaning|Worker UI Label|
|**I < I**|Settled income|Payment received|
|**I > I**|Future income|Invoice|
|**I < O**|Income given back|Refund given|
|**I > O**|Credit note issued|Credit note|
|**O < O**|Settled expense|Expense paid|
|**O > O**|Future expense|Bill received|
|**O < I**|Expense recovered|Reimbursed|
|**O > I**|Future reimbursement|Reimbursement pending|

### Technical Encoding & BitLedger Lineage

Inherited from BitLedger, monetary values are encoded as a `**uint24**` big-endian integer.

- **Formula:** `monetary_amount = uint24_value / 10^DECIMAL_POS`
- **Rounding Rules:** To ensure financial conservatism, Income (I < I) rounds **DOWN**, while Liabilities and Tax (O > O) round **UP**.

The **Financial Block** structure includes explicit tax codes (`TAX_CODE`), worker-only internal margins (subject to **Progressive Disclosure**), and a `QTY_SPLIT` flag for time-and-materials billing.

## 6. Context Stores: Participants and Chain Protocol

### Participants Block & Progressive Disclosure

The participants block replaces simple text strings with a typed structure (Role Types, `IS_SENDER` flags).

- **Filtering:** The encoder applies **Progressive Disclosure Filtering** based on the `RECIPIENT_TYPE` (bit 0 of Meta Byte 1). Internal worker margins and colleague details are automatically stripped from customer-facing URLs.

### Chain Protocol

A chain is a 24-bit composite **Chain ID** consisting of:

- **Anchor (17 bits):** Device-specific origin.
- **Participant Slot (4 bits):** Index of the participant's copy.
- **Sequence (3 bits):** Position (0–7).

**Multi-Worker Pre-Seeding:** This architecture allows for O(N) URL generation with O(1) compression cost. By modifying only the 4-bit `PARTICIPANT_SLOT` in the URL tail, a supervisor can generate unique links for multiple workers from a single compressed data frame.

## 7. The Template Diffusion and Distribution Model

Workpads treats presentation as data embedded in static HTML via `<script type="application/workpads-template">`.

### The Fallback Chain

The system ensures that a record never fails to render by walking a fallback chain: **Registry → Local Cache → Built-in Default**. This ensures graceful degradation even if the central registry is offline.

### Schema Types

- **Schema A:** String-substitution HTML.
- **Schema B:** Named slot layouts.
- **Schema P:** Ordered rich-media sections.
- **Schema C:** Composable components. **Trust Model:** Schema C execution is restricted to "Verified" (Official) or "Local" templates to prevent unauthorized JS execution.

## 8. Implementation Architecture and Ecosystem

### Storage Adapter Contract

The system abstracts backends (localStorage vs. IndexedDB) via an async, Promise-based **Storage Adapter**. Namespaces are isolated using strict **Prefix-Scoping** (e.g., `wp_rec_` for records, `wp_per_` for personal data).

### Two-Build Strategy

The ecosystem targets different Gecko generations through **Platform Adapters** (e.g., `clipboard.js`, `storage.js`) swapped at build time:

- **KaiOS 2.5 (Gecko 48):** Babel transpilation to ES5.
- **KaiOS 3.x (Gecko 84):** Modern ES6+ execution.

### Optional Infrastructure

While offline-first, an optional layer provides **content-addressed short links** (`workpads.me/s/{id}`). This improves QR code density and provides URL stability for physical assets without compromising the URL-as-document model.

## 9. Technical Metadata and Conformance

- **BASICS Conformance:** Workpads claims **Core Tier** status.
- **Verification:** To maintain the claim, both the architecture and the application must pass the **mandatory "Dirty-Test"** as specified in `basics-conformance.md`.
- **Compatibility Signals:** Forward and backward compatibility are maintained via two in-band signals: the **Scheme Tag** (codec version) and the **Template Byte** (Byte 0 of the frame), ensuring decoders can reject unknown formats gracefully.