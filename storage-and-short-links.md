# Optional Storage and Short Links
## Extending the URL-as-document model with server-side optionality

**Status:** v1 (2026-05-12) — architectural analysis and service design
**Prerequisite:** url-as-document.md — this document extends that model

---

## The Fragment Model's Natural Limits

The URL-as-document model encodes content entirely in the URL fragment. This is its strength — no server dependency, no storage cost, works offline, permanent. It is also the source of three practical constraints:

**Length.** A compressed note payload is typically 150–300 characters of base64url. A full job record with financial data can reach 600–900 characters. The resulting URL is shareable on WhatsApp and works as a link, but it is impractical as a printed QR code (high-density QR codes are harder to scan), unusable in SMS on low-end phones, and unwieldy in email footers or printed documents.

**Immutability.** Content encoded in a fragment is frozen at the moment of encoding. If the underlying record changes — a quote is revised, an invoice is corrected, an inspection report gains a follow-up note — the original URL reflects the original state. There is no way to update what a URL shows without generating and re-sharing a new URL.

**Ephemerality.** Fragment URLs exist only where they have been shared. If a user clears their app, loses their phone, or wants to access a document from a second device, the URL must be retrieved from wherever it was sent — an SMS thread, a WhatsApp conversation, an email. It is not stored in any retrievable service.

None of these are flaws in the core model. They are consequences of a deliberate design choice that eliminates infrastructure dependencies. The question is what minimal infrastructure, added optionally, resolves them without compromising what the model gets right.

---

## The Optional Storage Layer

The answer is a simple key-value store where:

- The **key** is a short opaque ID (8–10 characters)
- The **value** is the compressed payload (the same bytes that would otherwise go in the URL fragment)
- The **resolution endpoint** is `workpads.me/s/{id}` — a static-feeling URL that performs a single lookup and redirects to a rendering page with the payload

The critical design constraint is that the storage service is **optional at every step**. A document can exist purely as a fragment URL and never touch the service. A user can choose to store it and get a short ID. The short ID resolves to the same content as the fragment URL. Both rendering paths lead to the same orchestrator page.

This means:

- The fragment URL is always the canonical form
- The short URL is a convenience alias that points to the canonical payload
- Neither is required for the system to function

---

## Content Addressing and Deduplication

Because the payload is deterministic — the same record, same template, same scope always produces the same compressed bytes — the storage service can use content addressing: **the key is derived from the payload's hash**, not from a random ID or a sequential counter.

A practical scheme: take the first 8 characters of the base64url-encoded SHA-256 of the payload. Two documents with identical content produce the same short ID. The service stores each payload once regardless of how many users submit it.

The practical consequences:

**For businesses generating many similar documents**, the deduplication is minor (each document differs). But for templates themselves (which are payloads too, when stored), deduplication is significant — every user who saves the same community template shares the same storage record.

**For auditability**, content addressing means a short ID is a verifiable claim: given the ID and the content, anyone can confirm the content hasn't been altered by the service. The hash is in the ID.

**For trust**, a content-addressed store cannot silently modify what a URL points to. The payload you stored is the payload retrieved. This is different from a conventional URL shortener, where the redirect target can be changed at will.

---

## Living Documents: The Updateable Path

Content addressing solves deduplication and integrity. It does not solve updateability — by definition, changing content produces a different hash and therefore a different ID.

For the use case where a URL must remain stable while the underlying content can change — a price list, a standing quote, a physical asset's maintenance record — a second path is needed: **named document storage**.

A named document has:
- A user-assigned or system-assigned stable ID (e.g., `workpads.me/s/acme-quote-0042`)
- A current payload that can be replaced
- A version history (each update stores the previous payload with its timestamp)

When the URL is accessed, it resolves to the current payload. Previous versions remain accessible by version number for audit purposes.

This is a fundamentally different model from the fragment URL:

| Property | Fragment URL | Content-addressed | Named document |
|---|---|---|---|
| Server required | Never | On store, not on render if cached | Always |
| Content mutable | No | No | Yes |
| URL stable across edits | N/A (frozen) | No (new hash) | Yes |
| Audit history | No | Implicit (hash) | Explicit |
| Offline render | Yes (if template cached) | Yes (if payload cached) | Requires fetch on update |

These are not competing approaches. They cover different use cases. An invoice should be immutable (fragment or content-addressed). A product catalogue should be updateable (named document). The infrastructure supports both.

---

## QR Codes and Physical Assets

A QR code encoding a 250-character URL produces a version 5 or 6 QR code (roughly 37×37 modules). A QR code encoding a 30-character URL produces a version 2 or 3 code (25×25 modules). The smaller code is faster to scan, prints reliably at smaller sizes, and survives more physical damage before becoming unreadable.

This matters for several practical scenarios:

**Field work**: A piece of equipment has a QR code label. A worker scans it to access the service history, the current inspection record, or the last known fault. The QR URL needs to be short (printed on a label), and it needs to be updateable (the record changes after each visit). This requires a named document with a short stable URL — a fragment URL will not serve this use case.

**Printed documents**: An invoice printed on paper carries a QR code so the recipient can open it on their phone without typing a URL. The QR encodes a short URL pointing to the stored payload. The document is immutable (content-addressed), the code is compact, and the printed invoice now has a live digital counterpart.

**Physical locations**: A job site, a property, a vehicle fleet can each have a QR code linked to their current Workpads record. The code is printed once. The record behind it updates as work is done. Workers scan the code, open the record, add notes, and the URL always shows the current state.

**NFC tags**: The same logic applies to NFC tags on equipment, vehicles, or doors. A tap launches the record URL. NFC payloads are limited to a few hundred bytes, making the short URL essential.

The QR/NFC use case also illustrates why the registry service's template hosting matters here: if a short URL is on a physical label and will be scanned years from now, the template it references must still resolve. A self-hosted template URI on a contractor's personal site carries more risk of going offline than a registry-hosted one.

---

## Multi-Device Access and the Sync Gap

localStorage is per-device and per-browser. A user who creates records on a KaiOS phone cannot access them on a desktop browser without re-sharing. This is currently a stated limitation of the Phase 1 architecture.

Optional server-side storage resolves this without requiring a full sync architecture.

The mechanism: when a user stores a document (fragment or named), they receive a short URL that resolves on any device. They can bookmark it, email it to themselves, or keep it in any list. Opening it on any device with any browser renders the document correctly using that device's locally cached templates (or fetching them if not cached).

This is not sync in the traditional sense — it is not bidirectional, it does not replicate a local database to the cloud. It is closer to a **personal document URL library**: a list of short URLs the user has generated, accessible from any device.

A logged-in storage service account would show the user their stored documents, listed by type, date, and title (extracted from the payload). Clicking one opens the document in the rendering page. This is functionally equivalent to a document library but stored as a list of short IDs pointing to compressed payloads — the entire library for a sole trader who generates 3,000 documents per year is a few megabytes at most.

---

## Document Expiry and Conditional Rendering

The fragment model cannot express expiry — the content is frozen in the URL. A server-side storage layer can.

Practical cases:

**Quotes**: A business sends a quote valid for 30 days. After 30 days, the URL should show a clear expiry message rather than the original quote, which may now have different pricing. The stored payload includes an `expiry` field. The resolution service checks it and returns either the payload or a standardised expiry response.

**Temporary access links**: A document shared for a specific purpose — a one-time price, a time-limited proposal — can have its storage entry deleted or marked inactive after a defined period.

**Compliance document windows**: Some regulatory contexts require that certain documents are only accessible within a specific time window (not before a certain date, not after a certain date). This is not achievable without server-side state.

Expiry is not a core feature of the URL-as-document model — it requires infrastructure. But it is a feature that makes the model practically complete for business document use cases, where time-validity is often a legal and commercial requirement.

---

## Read Receipts and View Analytics

When a document exists only as a fragment URL, there is no way to know whether it was opened, on what device, or when.

When a document is stored server-side, the resolution endpoint can record:

- That a resolution was requested (document opened)
- The timestamp
- The platform (user-agent string, platform token from the app)
- Whether the rendering completed (a second signal from the orchestrator page, sent after successful render)

This enables a practical read receipt: a business that sends an invoice can see whether it was opened, and when. A contractor who sends a quote can see whether the customer looked at it before the meeting.

The privacy constraint here is important: the storage service learns only that a resolution occurred. It does not see the document content (it stored only the compressed payload, not a parsed version), and it does not know anything about the recipient beyond their IP address and user agent — the same information any web server has for any page view.

If even this is undesirable, the content-addressed path provides an alternative: the payload is stored but the resolution request goes through a CDN or cache layer that does not log individual requests.

---

## The Service Architecture and Storage Costs

The economics of the storage service are worth stating explicitly, because they are quite different from file hosting.

Each stored document is a compressed JSON payload, typically 200–800 bytes. At 800 bytes per document:

- 1 million stored documents = 800 MB raw storage
- At current object storage pricing (~$0.023/GB/month), 1 million documents costs approximately $0.018 per month to store
- 10 million documents = $0.18/month

These are not meaningful storage costs. The cost driver for this service is not storage — it is request handling (each short URL resolution is an HTTP request that must be served quickly) and the operational overhead of the service itself.

This means the service can offer free tiers with generous document counts without storage economics being a concern. Tier differentiation is based on:

| Tier | Retention | Max documents | Named documents | Analytics | Expiry |
|---|---|---|---|---|---|
| Free | 90 days | 500 | No | No | No |
| Personal | 2 years | 10,000 | Yes | Basic | Yes |
| Business | Permanent | Unlimited | Yes | Full | Yes |
| Business+ | Permanent | Unlimited | Yes | Full + webhook | Yes |

The storage service and the template registry service are natural companions — a business account that pays for verified template hosting also benefits from permanent document storage. But they are separable services. A business can use the template registry without storing documents, and vice versa.

---

## The Delegation Effect

One underappreciated consequence of short URLs stored server-side: **the recipient can forward the document without being a Workpads user**.

A contractor sends a job completion report to a client. The client opens it, sees the rendered document, and wants to forward it to their property manager. They copy and paste the URL into an email. The property manager opens it in any browser and sees the same document, rendered through the same template.

No app required. No account required. No format negotiation. The URL is the document and the document travels freely.

This is how the system grows beyond its installed base. Every shared document is a potential first contact with Workpads for a recipient who has never used it. If the template is well-designed and the document is clearly labelled as coming from Workpads, the experience itself is the product demonstration.

Fragment URLs already have this property. Short URLs make it practical at the point of physical media (printed QR codes, NFC tags, SMS).

---

## Summary of Distinct Use Cases

The full matrix of URL types and storage modes covers a complete set of practical requirements:

| Use case | URL type | Storage | Mutable |
|---|---|---|---|
| Quick note share via chat | Fragment URL | None | No |
| Invoice to client | Short content-addressed URL | Content-addressed | No |
| Quote (time-limited) | Short content-addressed URL | Content-addressed + expiry | No |
| Equipment service record | Short named URL | Named document | Yes |
| Price list or standing offer | Short named URL | Named document | Yes |
| QR code on physical asset | Short named URL | Named document | Yes |
| Personal document library | Short URLs | Stored with account | Varies |
| Cross-device access | Short URL | Stored | Varies |
| Public business document page | Short named URL | Named document (public) | Yes |

The fragment URL remains the foundation — zero infrastructure, always available, works offline, cannot be taken down. Every other mode is additive. A user who never stores anything loses no functionality. A user who stores everything gains link brevity, updateability, multi-device access, and — where needed — expiry and analytics.

The architecture does not force a choice between the two modes. It provides both, and the choice is made per-document at the moment of sharing.
