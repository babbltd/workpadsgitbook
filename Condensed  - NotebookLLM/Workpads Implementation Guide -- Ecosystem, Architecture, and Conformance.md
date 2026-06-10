# 1. Ecosystem Overview and Repository Roles

The Workpads ecosystem is a decentralized suite of specifications and tools designed to facilitate portable, sovereign commercial records. The architecture strictly separates normative standards from reference implementations to ensure cross-platform interoperability.

### Repository Matrix

|   |   |   |
|---|---|---|
|Repository Name|Primary Role|Implementation Status|
|`workpads-standard`|**Normative Specification.** The definitive authority on record encoding and service contracts.|v0.1 Live|
|`workpads-codec`|**Canonical Codec.** npm package (@workpads/codec) implementing the `pads-v1` algorithm.|v0.1|
|`workpads-cli`|**Integration Test Harness.** CLI tool for testing RecordService surface and round-trip encoding.|v0.x|
|`workpadsdev-cli`|**Developer Toolchain.** Orchestrates builds and runs conformance checks (WP-CONF-001–004).|v0.1|
|`workpadskaios`|**KaiOS Reference App.** Proof-of-concept for D-pad navigation on constrained hardware.|v0.1 (See Note)|
|`workpadsdotme`|**Web Reference Implementation.** Canonical browser entry point (workpads.me/p).|Active|
|`workpads-basicsconform`|**Research Control Plane.** Internal workspace for design decisions and BASICS logs.|Internal|

**Technical Note (DEV-WP-URL-001):** The `workpadskaios` v0.1 implementation uses a legacy query-string scheme tag (`?v=1&alg=...`). This is a known breaking deviation. Developers must use the canonical hash-fragment scheme tag (`#1ag/`) for interoperability with `workpadsdotme` and future spec-compliant clients.

--------------------------------------------------------------------------------

## 2. Core Data Model: The PADS Framework

The PADS model defines the temporal structure of a record, mapping data capture to the natural lifecycle of a job.

### The PADS Sections

|   |   |   |   |
|---|---|---|---|
|Section|Name|Temporal Phase|Responsibility|
|**P**|Process|Before the job|Defines the "What, Who, When, and Where."|
|**A**|Actions|During the job|Logs specific steps or checklist items.|
|**D**|Details|During or After|Captures logistics, timing, and worker identity.|
|**S**|Story|After the job|Narrative summary and technical observations.|

### Fundamental Invariants

- **Minimal Requirement:** The `job` field (Process section) is the only mandatory field.
- **Scalar Reality:** All scalar fields are treated strictly as strings. Numeric or boolean logic is restricted to the financial and action blocks.
- **The Omission Rule:** Absent fields (null, undefined, or empty strings) **must not be encoded**. The binary presence flags must skip these fields to preserve the byte budget.
- **Flat Structure:** While conceptually grouped, the underlying record object is flat to ensure encoding efficiency.

--------------------------------------------------------------------------------

## 3. Service Layer and Storage Architecture

Workpads utilizes a prefix-scoped abstraction layer to decouple business logic from the underlying storage technology.

### The Storage Adapter Contract

The `StorageAdapter` provides a consistent asynchronous interface, allowing future migration from `localStorage` to `IndexedDB` or remote gateways without modifying the service layer.

- **Mandatory Signatures:** `get(key)`, `set(key, value)`, `remove(key)`, `list()`, and `count()`.
- **Async Requirement:** All methods must return Promises to accommodate I/O latency.
- **Prefixing Convention:** Services use unique namespaces: `wp_rec_` (records), `wp_arc_` (archives), and `wp_per_` (personal).

### RecordService Interface

As the core of the **Exchange Engine**, `RecordService` manages the lifecycle of commercial atoms:

1. **create(fields):** Initializes records with IDs and `createdAt` timestamps.
2. **get(id):** Retrieves a record by its local ID.
3. **update(id, fields):** Performs partial merges for auto-saving during wizard transitions.
4. **save(id, record):** Executes a full replacement save (sets `updatedAt`).
5. **archive(id):** Transitions a record to the archive namespace.
6. **encodeUrl(record):** Generates a URL hash fragment via the `pads-v1` codec.
7. **storeReceived(record):** Persists incoming shared records with a `receivedAt` flag.

### Two-Engine Logic

The architecture maintains a strict boundary between the **Exchange Engine** (Transaction layer: `wp_rec_`) and the **Learning Engine** (Personal Platform layer: `wp_per_`). Personal captures and notes are sovereign to the worker; they may reference exchange records but are never transmitted in share links without explicit worker action.

--------------------------------------------------------------------------------

## 4. The pads-v1 Codec and Compact Encoding

The `pads-v1` codec facilitates the "URL-as-Document" philosophy, ensuring records are standalone, compact, honest, sovereign, and chain-capable.

### Binary Frame Structure

Unlike legacy versions, `pads-v1` utilizes a flexible **Meta-Byte architecture** to minimize overhead.

- **Meta Byte 1:** The frame's first byte.
    - Bit 7: `META2_PRESENT`
    - Bits 6–3: **Template ID** (4-bit value; e.g., `0001` for `svc-basic`).
    - Bit 2: `ACK_REQUEST`
    - Bit 1: `CHAIN` (Signal for `&c=` parameter in URL)
    - Bit 0: `RECIPIENT_TYPE` (0=Customer, 1=Colleague)
- **Presence Flags:** Two bytes (16 bits) always present after headers.

Presence flags and all multi-byte integers must be handled as **Big-Endian**. Flag Retrieval: `flags = (frame[flags_offset] << 8) | frame[flags_offset + 1]`. Little-endian interpretation will result in catastrophic decoding failure.

### Field Slot Mapping (svc-basic v2)

The presence flags (Bits 0–12) map to these fields in the `1ag` and `1bg` codebooks: | Bit | Field Key | Type | Bit | Field Key | Type | | :--- | :--- | :--- | :--- | :--- | :--- | | 0 | `job` | Scalar | 7 | `customer_phone` | Scalar | | 1 | `customer` | Scalar | 8 | `worker` | Scalar | | 2 | `date` | Scalar/Binary | 9 | `actions` | Array Block | | 3 | :--- | :--- | 10 | `details` | Scalar | | 4 | `meeting_time`| Scalar/Binary | 11 | `story` | Scalar | | 5 | `start_time` | Scalar/Binary | 12 | `value_block` | Financial Block | | 6 | `end_time` | Scalar/Binary | 13-15| — | Reserved |

### Extension Blocks

- **Participants Block:** (Meta2 Bit 4). Replaces bit 8 for multi-party records. Includes `ROLE_TYPE` codes (e.g., 000=Worker, 001=Supervisor) and `IS_SENDER` flags.
- **Financial Block (I>O Notation):** Triggered by Bit 12.
    - **I>O State Table:** | Notation | Direction | Time | Effect | Meaning | | :--- | :--- | :--- | :--- | :--- | | **I < I** | Inflowing | Past | Asset | Payment Received | | **I > I** | Inflowing | Future | Asset | Invoice Sent | | **O < O** | Outgoing | Past | Liability| Expense Paid | | **O > O** | Outgoing | Future | Liability| Bill Received |
    - **Value Encoding:** Amounts use `uint24` (3-byte big-endian).
    - **Calculation:** `monetary_amount = uint24_value ÷ 10^DECIMAL_POS`.

--------------------------------------------------------------------------------

## 5. Template Diffusion and Rendering System

Workpads avoids central template repositories via a diffusion model where templates are "carried" by any static HTML page.

- **Diffusion Mechanism:** Templates are embedded via `<script type="application/workpads-template">`. Apps scan, prompt for consent, and cache locally.
- **Schema Types:** **A** (Placeholders), **B** (Slots), and **P** (Sections for rich/print layouts).
- **The Fallback Chain:**
    1. Check Local Manifest Cache for the specified URI.
    2. Attempt to fetch from Canonical URI.
    3. Follow `lineage.supersedes` breadcrumbs.
    4. Render using the **Built-in Default** (**Guaranteed Terminus**).

--------------------------------------------------------------------------------

## 6. UI Architecture and Platform Strategy

Workpads employs a dual-build strategy to support the evolution from KaiOS 2.x (Gecko 48) to 3.x (Gecko 84).

|   |   |   |
|---|---|---|
|Property|KaiOS 2.x|KaiOS 3.x|
|**Manifest**|`manifest.webapp`|`manifest.webmanifest`|
|**Clipboard**|`execCommand('copy')`|`navigator.clipboard`|
|**ES Support**|ES5 (Transpiled)|ES6+ (Native Modules)|

### Panel Access Model

Two universal 80% width panels provide context via **ArrowLeft** (Workpads Panel) and **ArrowRight** (Personal Panel). A 20% **underlay** maintains visual continuity.

- **Conflict Avoidance:** Panel access is strictly **suppressed** on the **Actions sub-screen** and **Management screens** to prevent D-pad navigation collisions.

--------------------------------------------------------------------------------

## 7. Chain Protocol and ACKs

The chain protocol links related records (Quote → Job → Invoice) via a shared reference.

### The 24-Bit Chain ID

The `chain_ref` is a 4-character base64url string deconstructed into:

- **Anchor (17 bits):** In v0.1, this is a **device-specific identity** used to anchor all chains from a single origin.
- **Participant Slot (4 bits):** Identifies the holder (0=Authoritative/Sender).
- **Sequence (3 bits):** Positions 0 (Anchor), 1–6 (Amendments), and 7 (Wrap).

### The ACK Mechanism

Setting the `ACK_REQUEST` bit in Meta Byte 1 prompts the recipient to acknowledge receipt. This generates a **State Commit (0xD)** record (subtype `00`) which is returned to the sender as proof-of-receipt within the chain.

--------------------------------------------------------------------------------

## 8. BASICS Conformance Mapping

Workpads v0.1 conforms to **BASICS v0.1.1** at the **Core Tier**.

### Rule Satisfaction

- **SC-001/002:** Command surface defined and versioned in `command-surface.md`.
- **SC-041:** Offline operation verified via dirty-tests of the `pads-v1` in-memory codec.
- **SW-032:** Safe by default; offline-only with no authentication surface.

### Registered Deviations

- **DEV-WP-001 (Naming Surface):** Concise synonyms (e.g., "worker" instead of "technician") are used to accommodate D-pad typing and maintain role-neutrality across diverse trades (HVAC, Nursing, etc.).
- **DEV-WP-URL-001:** Documented URL scheme tag mismatch in `workpadskaios` v0.1.

**Verification:** Conformance is validated through "Dirty-Test" passes on both the `workpads/` (architecture) and `workpadskaios/` (implementation) repositories.