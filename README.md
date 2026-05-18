# Workpads Standard

**Version:** v0.1 (2026-04-27)  
**Maintained by:** [Babb](https://babb.tel)  
**BASICS conformance:** Core tier, BASICS v0.1.1  
**License:** Open standard — see [Conformance and Use](#conformance-and-use)

---

## What is Workpads?

Workpads is a record format and protocol for portable, sovereign, URL-transmissible commercial records. A workpad record documents a piece of work — a job, a quote, an invoice, an expense, a completion — in a compact binary encoding that travels as a URL hash fragment over SMS, WhatsApp, QR code, or any link transport, without a server or account.

The atomic unit is the **workpad record**: complete enough to stand alone, compact enough to travel as a URL, honest about its state, sovereign to its creator, and able to reference other records through a chain protocol.

This repository is the **normative specification** for all Workpads implementations. When implementation and standard conflict, the standard governs. When the standard is silent, implementations document their decisions and feed them back here.

---

## An Open Door, Not a Protected Territory

Babb created the Workpads Standard to grow a format that serves workers, not to own a market.

Babb builds reference implementations — KaiOS, web, CLI — to prove the standard works in practice and to establish a baseline for conformance testing. These are not the only valid implementations. They are proofs of concept with a conformance record.

**The standard is designed to be implemented by anyone, for any purpose.** Experimental apps, open-source tools, proprietary products, and platform-specific derivatives are all welcome. The standard defines what a workpad record is and how it is encoded; what you build with that is yours.

What Babb asks in return is simple: if your product generates or decodes workpad URLs (`workpads.me/p#1ag/...`), it should decode correctly the records it receives, and encode records that any conformant decoder can read. Interoperability is the standard's core value — not exclusivity.

**BASICS Standard is the benchmark.** The Workpads Standard is a BASICS-conformant product standard. BASICS provides the meta-framework: naming conventions, conformance claims, deviation registration, compatibility policy. If you are building on Workpads, conforming to BASICS gives your product a recognised conformance tier and a path for formally registering deviations rather than silently breaking compatibility.

Babb actively supports independent products that:
- Implement the codec and record schema faithfully
- Extend the format through registered deviations or v0.2+ extension points
- Use the standard as a foundation for experimental, open-source, or proprietary applications
- Conform to BASICS to establish a credible interoperability claim

The standard grows through real implementation experience. Deviations found in practice feed the next version. Every conformant product makes the ecosystem stronger.

---

## The Standard

Eight numbered sections form the normative core:

| Section | File | Version |
|---------|------|---------|
| §1 — PADS Data Model | [`pads-model.md`](pads-model.md) | v1 (2026-04-27) |
| §2 — Record Schema (svc-basic) | [`record-schema.md`](record-schema.md) | v1 (2026-04-27) |
| §3 — Naming Convention | [`naming-convention.md`](naming-convention.md) | v1 (2026-04-26) |
| §4 — RecordService Interface | [`record-service.md`](record-service.md) | v1 (2026-04-27) |
| §5 — Codec & Compact Encoding | [`codec.md`](codec.md) | v1.1 (2026-04-28) |
| §6 — Storage Adapter Contract | [`storage-adapter.md`](storage-adapter.md) | v1 (2026-04-27) |
| §7 — Two-Build Strategy | [`build-strategy.md`](build-strategy.md) | v1 (2026-04-27) |
| §8 — BASICS Conformance Mapping | [`basics-conformance.md`](basics-conformance.md) | v1 (2026-04-27) |

### Extended Specifications

These documents are part of the standard and cover features in active implementation or scheduled for v0.2:

| File | Scope |
|------|-------|
| [`record-philosophy.md`](record-philosophy.md) | The five properties of a workpad record — foundational rationale for all design decisions |
| [`activity-profile.md`](activity-profile.md) | Activity Profile and Block Registry — sender identity, contact store, location store |
| [`chain-protocol.md`](chain-protocol.md) | Record chain protocol — linked records, ACK mechanism, multi-worker pre-seeding (v0.2) |
| [`participants-block.md`](participants-block.md) | Participants block — replaces simple `worker` text field with typed party list (v0.2) |
| [`financial-block.md`](financial-block.md) | Financial block — amounts, currency, tax, qty×rate split, compound invoicing (v0.2) |
| [`transaction-classification.md`](transaction-classification.md) | I>O notation for income/expense direction, time, and settlement state |
| [`personal-platform.md`](personal-platform.md) | Two-engine architecture: Exchange Engine + Learning Engine |
| [`panel-access-model.md`](panel-access-model.md) | Panel navigation model (ArrowLeft/ArrowRight) — platform input adaptation |
| [`pads-v2-encoding-spec.md`](pads-v2-encoding-spec.md) | Extended bitpad-v2 encoding forward design |
| [`bitpad-record-encoding.md`](bitpad-record-encoding.md) | Bitpad record encoding reference |

### Maintenance Documents

| File | Purpose |
|------|---------|
| [`ECOSYSTEM.md`](ECOSYSTEM.md) | Map of all repos, tools, and how they connect |
| [`implementation-notes.md`](implementation-notes.md) | Pre-publication scan: deviations and innovations found in v0.1 implementations |
| [`codec-sync.md`](codec-sync.md) | Protocol for keeping codec implementations in sync across products |
| [`STANDARD-UPDATES.md`](STANDARD-UPDATES.md) | **Standard Update Register** — tracked list of all kaios design advances not yet reflected in this standard, with SUI IDs, source docs, and status. The authoritative to-do list for keeping this standard current. |

---
 cc
## The Ecosystem

The Workpads Standard is supported by a set of reference products maintained by Babb. These are the canonical implementations of the standard — not the only ones.

| Repo | Role | Status |
|------|------|--------|
| `workpads-standard` (this repo) | Normative specification | v0.1 live |
| `workpads-codec` | `@workpads/codec` npm package — implements §5 | v0.1 |
| `workpads-cli` | CLI reference implementation — implements §4 surface, used for integration testing | v0.x |
| `workpadsdev-cli` | Developer toolchain — build, packaging, softkey validation, conformance checks | v0.1 |
| `workpadskaios` | KaiOS 3.x reference app — full implementation, D-pad navigation | v0.1 |
| `workpadsdotme` | Web reference app (`workpads.me`) — three-column layout, mouse + keyboard | active |
| `workpads-basicsconform` | Research control plane — quiz rounds, decision log, deviation registry | internal |
| `workpads` | Original architecture documents (pre-2026). Historical reference. | archived |

See [`ECOSYSTEM.md`](ECOSYSTEM.md) for a detailed description of each repo's role and how they interconnect.

---

## Conformance and Use

### Using the Standard

The Workpads Standard is open. You may implement it, extend it, and build proprietary or open-source products on it without permission from Babb.

If your product:
- **Generates** workpad URLs (`workpads.me/p#1ag/...`): encode according to §5 so any conformant decoder can read the record.
- **Decodes** workpad URLs: implement §5 faithfully so records from any conformant encoder are readable.
- **Extends** the format: use the reserved field slots and template bytes in §5, or register a deviation via BASICS.

### BASICS Conformance

The Workpads Standard conforms to BASICS v0.1.1 at the Core tier. BASICS provides a meta-framework for interoperability claims, deviation registration, and compatibility policy.

If you are building a product on Workpads and want a formal conformance claim:
1. Read [`basics-conformance.md`](basics-conformance.md) for the mapping.
2. Use `workpads-basicsconform/` (Babb's internal conformance control plane) as a reference for the BASICS evidence artifact format.
3. Register deviations in your own `deviation-registry.md` following the BASICS pattern.

Formal BASICS conformance is not required to implement Workpads — it is a signal to other implementors that your product has a documented interoperability posture.

---

## Versioning

Section versions use `v1`, `v2`, etc. A section version increments on breaking changes (field removed, encoding slot reassigned). Additive changes (new optional fields, new sections) do not increment existing section versions.

The overall standard version (`v0.1`) tracks the implementation generation. `workpadskaios v0.1` and `workpadsdotme` at launch both implement Workpads Standard v0.1.

### v0.2 Roadmap

The following are settled in design and documented in extended specs, pending implementation:

- Financial block in codec (§`financial-block.md`) — amounts, currency, tax, compound invoicing
- Chain protocol (§`chain-protocol.md`) — linked records, ACK, multi-worker chains
- Participants block (§`participants-block.md`) — replaces `worker` text field
- KaiOS 2.5 build target (§`build-strategy.md`)
- URL scheme tag alignment across all implementations (see [`implementation-notes.md`](implementation-notes.md))
- Block Registry v0.2 schema — typed customers/locations with flag field

---

## History

The Workpads Standard was developed through a series of research-quiz rounds in `workpads-basicsconform/`, with decisions logged in `workpads-basicsconform/system/logs/decisions.md`. The `original/` subfolder in this repo contains a separate git repository with the pre-2026 Workpads Standard drafts. Those documents informed but do not supersede the sections in this repo.
