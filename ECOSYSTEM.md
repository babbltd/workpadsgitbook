# Workpads Ecosystem

**Status:** v0.2 alignment (2026-05-21)

This document maps the full Workpads ecosystem: every repository, its role, and how the pieces connect. It is descriptive, not normative — the spec sections govern.

**Process:** [`workpadskaios/system/project-process.md`](../workpadskaios/system/project-process.md)

---

## Overview

```
┌─────────────────────────────────────────────────────┐
│              Workpads Standard (this repo)          │
│  Normative spec: data model, codec, services,       │
│  BASICS conformance, chain protocol, financial      │
└────────────────────┬────────────────────────────────┘
                     │ specifies
        ┌────────────┼────────────────┐
        ▼            ▼                ▼
  workpads-codec  workpads-cli   workpadsdev-cli
  npm package     CLI reference  Build toolchain
  (@workpads/     + test harness + conformance
   codec)                          checks
        │
        │ implements §5
        ▼
  ┌─────────────────────────────────┐
  │        Reference Apps           │
  │  workpadskaios  workpadsdotme   │
  │  KaiOS 3.x app  Web app         │
  └─────────────────────────────────┘
                     │
                     │ research feeds
                     ▼
           workpads-basicsconform
           (internal research control plane)
```

---

## Repositories

### workpads-standard (this repo)
**Role:** Live normative specification.

All settled design decisions become sections here. The standard is the definitive answer to "what is a workpad record, how is it encoded, how does the service layer work." No implementation supersedes it; conflicts are resolved by updating either the implementation or the standard.

**What lives here:**
- 8 numbered core sections (§1–§8)
- Extended specs: chain protocol, financial block, participants block, activity profile, transaction classification, panel access model
- Maintenance documents: ecosystem map, implementation notes, codec sync protocol

---

### workpads-codec
**Role:** `@workpads/codec` npm package — the canonical JavaScript implementation of §5.

The codec package is the portable form of the pads-v1 encoder/decoder. It is the reference for anyone building a Workpads-compatible product in JavaScript or Node.js. KaiOS and web implementations bundle it directly or inline it; the CLI uses it as a package dependency.

**Key facts:**
- Algorithm: pads-v1
- Compression: fflate DEFLATE (level 9, `deflateSync`)
- URL encoding: base64url, no padding
- Scheme tag: `1pa/` (pads-v1 codebook `a`; supersedes `1ag/` / `1bg/` legacy)
- Frame: meta1 + meta2 + field_flags (+ optional FLAGS3/FLAGS4)
- Scalar fields: per `codec.md` v2.0; actions at bit 9 (newline-joined titles on wire)

**Dependency graph:**
```
workpads-cli        → @workpads/codec
workpadsdev-cli     → @workpads/codec (conformance check WP-CONF-004)
workpadskaios       → inlines codec.js (UMD bundle)
workpadsdotme       → inlines codec.js (browser bundle)
```

---

### workpads-cli
**Role:** CLI reference implementation and integration test harness.

The CLI is the earliest-stage Workpads implementation. It exposes the full §4 RecordService surface as shell commands: create, edit, share, import, render, list, export, delete, dashboard. It also implements template validation, storage policy management, and a browser service (local HTTP server with embedded UI).

**Encoding (v0.2+):** Share URLs use pads-v1 `#1pa/` via `@workpads/codec` (see `workpads-file-format.md` for `.wpf` binary). Legacy JSON share docs in older README sections are obsolete.

**Use as a test harness:** Because the CLI exposes every RecordService method as a command, it is the simplest way to test round-trip encode/decode, template validation, and chain protocol interactions. Integration tests for the codec should be run against the CLI before a codec change is shipped.

**Storage:**
- `.workpads/records.json` — active records
- `.workpads/storage-policies.json` — storage policy configuration

---

### workpadsdev-cli
**Role:** Developer toolchain for building and validating Workpads implementations.

`workpads-dev` orchestrates the build pipeline for KaiOS apps and runs conformance checks. It is the build-time counterpart to the runtime standard.

**Commands:**
- `build:3.0` — package KaiOS 3.x app (manifest + dated .zip)
- `build:2.5` — stub for KaiOS 2.5 (v0.2)
- `set-keys` — validate softkeys.yaml screen key assignments
- `conform` — run conformance checks (WP-CONF-001 through WP-CONF-004)

**Conformance rules enforced:**
| Rule | Check |
|------|-------|
| WP-CONF-001 | Manifest present |
| WP-CONF-002 | `@workpads/codec` in package.json |
| WP-CONF-003 | `softkeys.yaml` present |
| WP-CONF-004 | Codec round-trip test passes |

**Exit codes:** 0=OK, 10=config error, 20=discovery error, 30=eval error, 40=blocking, 50=runtime.

Any product using `workpadsdev-cli conform` and passing WP-CONF-001–004 has a baseline conformance claim. Additional BASICS evidence artifacts (see `basics-conformance.md`) complete the formal claim.

---

### workpadskaios
**Role:** KaiOS 3.x reference implementation — the first complete Workpads application.

workpadskaios is the canonical mobile implementation of the Workpads Standard. It targets KaiOS 3.x (240×320px, Gecko 63+, D-pad only, no touch) and is the proof that the standard works on resource-constrained hardware with no account, no server, and no framework dependency.

**Architecture:**
- **Two-engine design:** Exchange Engine (records, sharing, receiving) + Learning Engine (notes, activity)
- **Service layer:** StorageAdapter, RecordService, ActivityService, PersonalService, BlockRegistry — all vanilla ES5, all localStorage-backed, all Promise-returning
- **Screen layer:** list, wizard (4-step PADS), view, share, management
- **Panel layer:** WorkpadsPanel (ArrowLeft), PersonalPanel (ArrowRight)
- **Codec layer:** inlined `codec.js` (UMD bundle of `@workpads/codec`)

**Navigation:** Entirely D-pad driven. ArrowLeft/Right for panels, Enter/CSK for confirm, SoftLeft/Backspace for back, SoftRight for save/options, `*` or long-press CSK for quick note.

**Known deviation:** v0.1 uses scheme tag `v=1&alg=bitpad-v1&d=...` (pre-standard URL format). The canonical scheme tag is `1ag/` as specified in §5. URLs generated by workpadskaios v0.1 are not interoperable with workpadsdotme or any implementation using the `1ag/` format. This is tracked in `implementation-notes.md` as DEV-WP-URL-001 and is the first breaking fix in v0.2.

**Chain support:** Not implemented in v0.1. Chain protocol is v0.2.

---

### workpadsdotme
**Role:** Web reference implementation at `workpads.me` — the canonical browser entry point.

workpadsdotme is a full Workpads application for desktop and mobile browsers. It uses the same service layer as workpadskaios (identical JS files) and the same pads-v1 codec, adapted for mouse, keyboard, and responsive layout.

**Architecture:**
- **Three-column layout:** Left panel (records list) + centre screen (wizard/view/management) + right panel (personal notes, theme, settings)
- **Screens:** list, wizard, view, share, management, archive — all hash-routed (`#/`, `#/new`, `#/edit/:id`, etc.)
- **Services:** Identical to workpadskaios (no changes)
- **Codec:** Inlined `codec.js` — implements canonical `1ag/` scheme tag

**Extensions over KaiOS (v0.1 only):**
- Chain support: `genChainRef()`, `findByChainRef()`, `chainRef` field on records
- Financial field forward design: bits 12–15 (amount, currency, vat, record_type) present in codec.js
- `encodeFin()`, `decodeFin()`, `appendFin()` — financial encoding helpers (see `implementation-notes.md`)

**Receiver:** `/p` route is a standalone HTML page that decodes any `workpads.me/p#1ag/...` URL and renders the record. It contains an inline decode-only codec copy. This is the primary entry point for record recipients who do not have the app installed.

---

### workpads-basicsconform
**Role:** Internal research control plane — feeds the standard but does not host it.

This is Babb's internal workspace for design decisions: quiz rounds, task cards, the decision log (`decisions.md`), BASICS naming commentary, deviation registry drafts, and BASICS-claim.yaml. When a decision settles, its outcome is written into the appropriate section in `workpads-standard/`.

**This repo is not the standard.** It is the process that produces the standard. External implementors do not need to read it; the standard sections are self-contained.

---

## How They Connect at Runtime

### Encode path (sender)

```
User fills in wizard (workpadskaios or workpadsdotme)
  → RecordService.create() / save()
  → RecordService.encodeUrl()
      → WPCodec.encode(record)
          → padsEncode() → deflate → base64url
          → prepend scheme tag "1ag/"
          → prepend "https://workpads.me/p#"
  → URL displayed / shared via SMS / QR
```

### Decode path (recipient)

```
Recipient opens workpads.me/p#1ag/<payload>
  → /p/index.html (standalone receiver)
      → inline decode-only codec
          → strip scheme tag → base64url decode → inflate → padsDecode()
      → record rendered in browser (no app required)

OR recipient opens URL in workpadskaios or workpadsdotme app
  → App.handleIncoming(url)
      → RecordService.decodeUrl(url)
          → WPCodec.decode(url)
      → RecordService.storeReceived(decoded)
      → record appears in received list
```

### Build path (developer)

```
workpadsdev-cli build:3.0
  → reads manifest.webmanifest
  → packages js/, css/, index.html
  → produces workpadskaios-YYYYMMDD.zip

workpadsdev-cli conform
  → WP-CONF-001: manifest present
  → WP-CONF-002: @workpads/codec in package.json
  → WP-CONF-003: softkeys.yaml present
  → WP-CONF-004: codec.encode(record) → codec.decode(url) → fields match
```

---

## For Independent Implementors

If you are building a product that generates or decodes workpad URLs:

1. **Read `codec.md` (§5).** The pads-v1 frame format, scheme tag, field slot table, and byte order rules are fully specified. Implement them exactly.
2. **Use `workpads-codec`.** The npm package is the reference; inlining the source is acceptable for browser targets.
3. **Run `workpadsdev-cli conform`** to get a baseline WP-CONF-001–004 pass.
4. **Register deviations** in your own `deviation-registry.md` using the BASICS format. If you add fields, use the reserved bits (12–15 for v0.2 financial fields are already allocated; contact Babb before claiming them in ways that conflict with the financial block spec).
5. **Do not change the scheme tag** (`1ag/`) without registering a new codebook character. The scheme tag is the primary interoperability signal. A different scheme tag means a different codec generation — decoders will reject the URL.

Babb is reachable via the repository and welcomes questions about conformance, extension points, and ecosystem integration.
