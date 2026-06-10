# 1. The Foundational Architecture: URL-as-Document

The Workpads infrastructure initiates a structural shift in commercial record management by adopting the "URL-as-Document" model. In this paradigm, we abandon the traditional dependency on centralized file hosting and SaaS databases. Instead, we treat content and presentation as decoupled entities that converge only at the moment of rendering on a stateless **Orchestrator Page** (`workpads.me/p`).

Content is compressed and encoded directly into the URL hash fragment. Because hash fragments are not transmitted to servers by browsers, the record data remains strictly client-side. This ensures that the "document" is not a stored object on a server, but a portable string of intelligence.

### Comparison of Architectural Models

|   |   |   |
|---|---|---|
|Feature|Traditional Model (File Hosting/SaaS)|Workpads Model (Stateless Orchestration)|
|**Storage Growth**|Linear: Every document adds server-side overhead and cost.|Zero: Content lives in the URL string; storage is local to the device.|
|**Dependency**|High: Requires active subscriptions and 100% server uptime for access.|Low: Operates offline; independent of vendor persistence.|
|**Accessibility**|Restricted: Often gated by accounts, portals, or proprietary apps.|Universal: Any browser renders the URL via the static Orchestrator shell.|

### The Five Properties of a Workpad Record

For a record to function as an "atomic unit of commerce," it must adhere to five structurally enforced properties:

- **Complete:** The record must contain sufficient standalone context (Job, Customer, Date, Actions, Details, Story) to be understood by a recipient without external metadata.
- **Compact:** Every byte is optimized for transport via SMS, WhatsApp, or low-density QR codes, ensuring accessibility for workers on constrained hardware.
- **Honest:** Records carry explicit "truth claims" through meta-byte signals. Bitwise flags for **DRAFT**, **ACK_REQUEST**, and **CHAIN** status structurally prevent state ambiguity.
- **Sovereign:** The creator retains total authority over the data. Through progressive disclosure, the user decides exactly what intelligence is encoded for the recipient.
- **Chained:** Records can reference one another via a sequence protocol, allowing individual snapshots to compose a cohesive commercial relationship.

--------------------------------------------------------------------------------

## 2. Advanced Scaling: Optional Server-Side Storage

While fragment-only URLs provide maximum independence, physical constraints (URL length) and operational requirements (immutability and updateability) necessitate an optional server-side storage layer.

### Content Addressing and Deduplication

The storage layer utilizes a **Content-Addressed** mechanism. The 8-character document ID is derived from the **first 8 characters of the base64url-encoded SHA-256 hash** of the payload.

This architecture provides massive efficiency through **Deduplication**. Because templates are themselves payloads, common community templates are stored exactly once globally. For business records, this hash-based ID serves as a verifiable integrity claim; any alteration to the content would result in a mismatch with the ID, fostering absolute audit trust.

### Infrastructure Scaling Matrix

|   |   |   |   |
|---|---|---|---|
|Property|Fragment URLs|Content-Addressed Storage|Named Documents|
|**Mutability**|Immutable (Frozen)|Immutable (New hash for edits)|Mutable (Stable alias)|
|**URL Stability**|N/A|No (ID changes with content)|Yes (Static URL)|
|**Offline Rendering**|Yes (if template cached)|Yes (if payload cached)|Requires fetch on update|
|**Primary Use**|Ephemeral chat shares|Invoices / Audit trails|Asset tags / Price lists|

--------------------------------------------------------------------------------

## 3. Physical Integration and Asset Management

The transition from digital fragments to physical media (QR/NFC) is a critical scaling frontier. By utilizing stored short links (8–10 characters) rather than raw fragment URLs (600–900 characters), we achieve significant technical advantages.

A short URL allows for a **Version 2 QR code** (25x25 modules) instead of a high-density Version 6 code. This lower module density is vital for **reliability in harsh field conditions**, where damaged labels or low-light scanning would otherwise cause capture failure.

### Use Case Scenarios

**Field Work (Named Documents)** Physical equipment is fitted with a permanent QR label linked to a stable, mutable _Named Document_. Technicians scan the code to access the current service history. Updates to the record do not change the URL, ensuring that a single printed label remains the "live" entry point for the asset's entire lifecycle.

**Printed Invoices (Content-Addressed)** Paper invoices include an immutable _Content-Addressed_ QR code. The hash-based ID ensures the digital counterpart precisely matches the printed terms. Scans redirect the recipient to a rendered version on their device, eliminating re-keying errors.

**Physical Locations (Dynamic Handoff)** Job sites carry a single QR code. As work transitions through phases (Quote → Active → Completion), the record associated with that location's ID is updated. The URL remains the static site identifier, while the content evolves with the project.

--------------------------------------------------------------------------------

## 4. Template Registry Economics and Ecosystem Diffusion

Workpads treats presentation as data, allowing for a "Diffusion Model" where templates spread organically across the web. Templates "escape the app" by being embedded in `<script>` tags on static pages; once visited, they are **cached locally forever**, surviving cache clears and app uninstalls if localStorage is backed up.

### The Economic Case for Stateless Storage

The storage service is highly sustainable because we store only compressed payloads (~800 bytes). From an infrastructure perspective, **1 million documents occupy roughly 800 MB**. At standard object storage rates, this results in a cost of approximately **$0.018 per month**. This enables a high-volume free tier where the primary cost is request handling rather than data volume.

### Service Tier Structure

|   |   |   |   |
|---|---|---|---|
|Tier|Retention|Max Documents|Features|
|**Free**|90 Days|500|Fragment & Content-Addressed only|
|**Personal**|2 Years|10,000|Named Documents, Basic Analytics|
|**Business**|Permanent|Unlimited|Full Analytics, Document Expiry|
|**Business+**|Permanent|Unlimited|Webhooks, Verified JS Templates|

--------------------------------------------------------------------------------

## 5. v0.2 Enterprise Feature Roadmap

The v0.2 release formalizes the protocol for high-stakes professional environments, prioritizing financial logic and multi-worker orchestration.

### Financial Logic: I>O Notation and 8-State System

The **Financial Block (bit 12)** utilizes the **I>O Notation** to classify transactions. This system describes the Direction (Income/Outgoing), Time (Past/Future), and Effect (Asset/Liability).

- **I < I**: Settled Income (Payment received).
- **O > O**: Future Expense (Bill received).

Monetary amounts are encoded as **uint24 values**, interpreted via the formula: `Monetary Amount = uint24 ÷ 10^DECIMAL_POS`. By default, `DECIMAL_POS` is 2 (pennies), allowing the uint24 to cover invoices up to £167,772.15—capturing the 99th percentile of field service jobs with zero floating-point error.

### The Chain Protocol

To handle complex workflows, we utilize a **24-bit composite Chain ID**:

1. **Anchor (17 bits):** Device-specific origin ID.
2. **Participant Slot (4 bits):** Identifies the worker (up to 16 participants).
3. **Sequence (3 bits):** Tracks record position (0–7).

This allows for **multi-worker pre-seeding**: a supervisor can generate N unique URLs for different workers with a single compression operation, as only the 4-bit Participant Slot varies in the URL.

### State Commit Records (Template 0xD)

V0.2 introduces **State Commit Records** to close chains. These snapshots serve as Job Completion Certificates and Pay Summaries, providing a bilateral agreement on the state of a commercial relationship at a specific point in time.

--------------------------------------------------------------------------------

## 6. Cross-Platform Build Strategy and Interoperability

Infrastructure must bridge the gap between legacy hardware (KaiOS 2.5) and modern browsers (KaiOS 3.x/Web). We utilize a **Two-Build Strategy** to avoid "runtime feature detection guards," which degrade performance and testability.

### Technical Requirement Comparison

|   |   |   |
|---|---|---|
|Property|KaiOS 2.5 (Gecko 48)|KaiOS 3.x (Gecko 84)|
|**Manifest**|`manifest.webapp`|`manifest.webmanifest`|
|**Service Worker**|No|Yes|
|**Modules**|Not supported (Requires UMD)|Native ES Modules|
|**Clipboard**|`execCommand('copy')`|`navigator.clipboard`|

Gecko 48 requires **UMD bundles** for the codec because it lacks ES module support. This ensures the same cryptographic and encoding logic runs on a $20 feature phone as on a high-end desktop.

### Codec Sync and Scheme Tags

The **Scheme Tag** is our primary interoperability signal. All implementations must adhere to the tag versioning:

- `1ag/`: v0.1 canonical (Codebook A).
- `1bg/`: v0.2 upgrade (Codebook B), which adds the **Financial Block** and structured line items.

--------------------------------------------------------------------------------

## 7. Conclusion: The Global Standard Vision

The Workpads infrastructure does not aim to be a closed platform, but a "worldwide template system" devoid of central chokepoints. By maintaining the **Standard as the primary asset**, we enable third-party adoption while the reference app serves merely as the initial proof of concept.

### Trust Levels and Execution

Security is maintained through four **Trust Levels**, which dictate JavaScript execution (Schema C):

- **Built-in:** Unconditional trust; shipped with app.
- **Official:** Verified/Signed by Workpads; **JS Enabled**.
- **Community:** Self-published; user consent required; **No JS**.
- **Local:** Created on-device; **JS Enabled**.

By enforcing these boundaries, we protect the user while allowing the decentralised template graph to grow. Our goal remains the maintenance of a public, stateless protocol that empowers workers to own their digital presence through a simple, universal URL string.