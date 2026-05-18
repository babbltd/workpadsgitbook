# §T — Template System

**Status:** v1 (settled 2026-05-12)
**Phase:** 1 (manifest + render engine live; page ingestion Phase 2)

---

## Overview

The Workpads Template System defines how visual presentation templates are authored, identified, distributed, cached, and rendered across the full platform range — KaiOS feature phones through desktop browsers.

A template is a self-contained bundle (HTML + CSS + optional JS) that takes a Workpads data object and produces a rendered view. Templates are separate from records. A record is permanent data; a template is a replaceable lens through which data is shown.

---

## Template Identity

Every template has two identifiers:

| Identifier | Form | Purpose |
|---|---|---|
| **Canonical URI** | `urn:` or `https://` | Globally unique, stable across all installs |
| **Local manifest key** | same as canonical URI | Key in the device's manifest store |

### URI formats

```
urn:workpads:tpl:{type}:{name}:{version}
urn:workpads:tpl:note:default:v1          ← built-in
urn:workpads:tpl:note:invoice-clean:v2    ← community

https://example.com/tpls/site-report.json ← third-party fetch URI
```

URN-form identifiers are reserved for first-party and built-in templates. Third-party authors should use an HTTPS URL they control as the canonical URI.

### Version field

`version` is a positive integer. The registry accepts an incoming template only when `incoming.version > stored.version`. This allows in-place upgrades without reinstall.

---

## Manifest Entry Fields

Every installed template has a manifest entry (~200 bytes). The manifest is always resident in localStorage.

| Field | Type | Required | Description |
|---|---|---|---|
| `uri` | string | **Yes** | Canonical URI (unique key) |
| `name` | string | **Yes** | Human-readable display name |
| `schema` | string | **Yes** | `'A'` `'B'` `'P'` `'C'` — see Schema Types |
| `type` | string | **Yes** | Record type this template targets (`'note'`, `'job'`, …) |
| `scope` | string[] | Yes | Sub-types within the record type (defaults to `[type]`) |
| `domain` | string | Yes | Industry/context domain; `'general'` matches all |
| `source` | string\|null | No | Display string for provenance (site name, author) |
| `version` | integer | Yes | Monotonically increasing; start at `1` |
| `trust` | string | Yes | `'built-in'` `'official'` `'community'` `'local'` |
| `origin` | string\|null | No | URL of origin page (filled by ingestion path) |
| `addedAt` | timestamp | auto | Unix ms when first installed |
| `usedAt` | timestamp | auto | Unix ms of most recent render |
| `platforms` | string[] | **Yes** | Target platforms — see Platform Targeting |
| `lineage` | object | Yes | `{ derivedFrom, supersedes, components }` |
| `cached` | boolean | auto | `true` once payload is locally stored |

---

## Platform Targeting

Templates declare which platforms they are designed for. The runtime uses this to filter the template picker and fallback chain.

### Platform values

| Value | Meaning |
|---|---|
| `'all'` | Works acceptably on every platform |
| `'kaios'` | Optimised for 240×320px, D-pad, KaiOS 2/3 |
| `'mobile'` | Touchscreen phones (Android/iOS browsers) |
| `'desktop'` | Wide viewport, mouse, full CSS/JS support |
| `'print'` | Print stylesheets, PDF export targets |

A template may list multiple values: `["mobile", "desktop"]`.

### Platform manifest in authored pages

Third-party pages that carry embedded templates should include a platform manifest block alongside the template payload. This tells the app which variant to surface:

```html
<!-- Preferred form: one <script> block per platform variant -->
<script type="application/workpads-template" data-platform="kaios">
  <!-- base64-compressed template payload for KaiOS -->
  eJyNj0...
</script>
<script type="application/workpads-template" data-platform="desktop">
  <!-- base64-compressed template payload for desktop -->
  eJyNk1...
</script>
```

The registry's `detectOnPage()` scanner reads the `data-platform` attribute and matches against the current runtime's platform token. If no `data-platform` is set the template is treated as `'all'`.

For single-file pages that serve all platforms, use `platforms: ['all']` in the template object and write CSS with responsive breakpoints:

```css
/* KaiOS baseline — 240px wide, 320px tall */
.wpt-card { font-size: 12px; padding: 8px; }

@media (min-width: 480px) {
  .wpt-card { font-size: 14px; padding: 14px; }
}

@media (min-width: 960px) {
  .wpt-card { font-size: 16px; padding: 20px 24px; }
}
```

---

## Schema Types

| Schema | Description | Payload fields |
|---|---|---|
| **A** | Single HTML string with `{{placeholders}}` | `html`, `css` |
| **B** | Named slot map — each slot rendered independently | `html` (wrapper), `css`, `slots: { name: htmlStr }` |
| **P** | Ordered sections (mini-presentation) | `css` (global), `sections: [{ html, css }]` |
| **C** | Component / partial — not rendered directly | `html`, `css`, `interface` (for future composition) |

### Template engine syntax (Schemas A, B, P)

```
{{fieldName}}           variable substitution (HTML-escaped)
{{a.b.c}}               dot-path into nested objects
{{#key}}…{{/key}}       conditional block — renders if value is truthy
{{^key}}…{{/key}}       inverted block — renders if value is falsy
```

### Schema A example

Simplest form. One HTML string, one CSS string, full template engine.

```json
{
  "uri": "urn:workpads:tpl:note:clean:v1",
  "schema": "A",
  "type": "note",
  "version": 1,
  "platforms": ["all"],
  "name": "Clean Note",
  "html": "<div class='wn'><div class='wn-ts'>{{ts}}</div><div class='wn-text'>{{text}}</div></div>",
  "css": ".wn{padding:12px;font-family:sans-serif}.wn-ts{font-size:10px;color:#888}.wn-text{font-size:13px;line-height:1.5}"
}
```

### Schema B example

Named slots — useful when the host app needs to pull specific regions separately (e.g. header vs. body vs. footer in a card layout).

```json
{
  "uri": "https://example.com/tpls/job-card-b.json",
  "schema": "B",
  "type": "job",
  "version": 1,
  "platforms": ["mobile", "desktop"],
  "slots": {
    "header": "<div class='jc-hdr'>{{job}} · {{customer}}</div>",
    "body":   "<div class='jc-body'>{{details}}</div>",
    "footer": "<div class='jc-foot'>{{date}}</div>"
  },
  "css": ".jc-hdr{font-weight:bold}.jc-body{margin:8px 0}.jc-foot{font-size:11px;color:#888}"
}
```

### Schema P example

Ordered sections — intended for note presentations, shareable slides, and rich single-page exports.

```json
{
  "uri": "https://example.com/tpls/note-pres.json",
  "schema": "P",
  "type": "note",
  "version": 1,
  "platforms": ["desktop"],
  "css": "body{margin:0;background:#111;color:#eee;font-family:Georgia,serif}",
  "sections": [
    {
      "html": "<section class='cover'><h1>{{job}}</h1><p>{{date}}</p></section>",
      "css":  ".cover{height:100vh;display:flex;flex-direction:column;justify-content:center;padding:60px}"
    },
    {
      "html": "<section class='body-sec'><p>{{text}}</p></section>",
      "css":  ".body-sec{padding:40px 60px;font-size:18px;line-height:1.7}"
    }
  ]
}
```

---

## Payload Storage

The manifest entry and payload are stored separately.

| Store | Key pattern | Size guidance |
|---|---|---|
| Manifest | `wp_tpl_manifest` (single JSON object) | ~200 B per entry |
| Payload | `wp_tpl_payload_{uri}` | No hard limit; evict by LRU `usedAt` if storage pressure |

Manifest entries survive payload eviction — the app knows the template exists and can re-fetch the payload on demand. The manifest `cached` field reflects whether the payload is currently stored.

---

## Fallback Chain

When rendering, the system walks this chain and stops at the first available template:

```
1. Requested URI (from manifest + payload cache)
2. Fetch canonical URI (if HTTPS and online)
3. Fetch fallback URI (from template's lineage.supersedes, if set)
4. Built-in default: urn:workpads:tpl:note:default:v1
```

The built-in default is hardcoded in the app binary and always available. It is the guaranteed terminus — rendering never fails.

---

## Carry Mechanism (Page-Embed)

A static HTML page can carry a template payload in a `<script>` tag. When a user visits the page in the in-app browser, the app detects and optionally ingests it — no server roundtrip required.

### Embedding a template

```html
<script type="application/workpads-template" data-platform="all">
eJyNj0EKwjAQRfc...   ← fflate-deflated, base64url-encoded template JSON
</script>
```

Encode with `TemplateRegistry.encode(tplObj)` (dev utility).

### What the app does on detection

1. Scans for `<script type="application/workpads-template">` tags
2. Decodes and reads the header (uri, name, version) — no full parse yet
3. Skips if: already at current version, or user has refused this URI before
4. Presents consent prompt: template name, origin domain, trust level
5. On accept: calls `ingest()`, stores manifest + payload
6. On refuse: records URI in refusals store — not surfaced again from same origin

---

## Template Object (Wire Format)

The full object passed to `ingest()` or embedded in a page:

```json
{
  "uri":      "urn:workpads:tpl:note:example:v1",
  "name":     "Example Note",
  "schema":   "A",
  "type":     "note",
  "scope":    ["note"],
  "domain":   "general",
  "version":  1,
  "trust":    "community",
  "platforms": ["kaios", "mobile"],
  "lineage":  { "derivedFrom": null, "supersedes": null, "components": [] },
  "html":     "…",
  "css":      "…"
}
```

`lineage.supersedes` may point to an older URI this template replaces. `lineage.derivedFrom` credits the parent if this is a fork.

---

## Built-in Default

`urn:workpads:tpl:note:default:v1` is shipped with every app build. It:

- Cannot be overwritten or removed
- Appears first in all template listings
- Is the fallback terminus — rendering never falls past it
- Has `trust: 'built-in'`, `platforms: ['all']`

---

## Registry API Reference

```js
// Query
TemplateRegistry.query(opts)          // { type, schema, domain, cached } → entries[]
TemplateRegistry.allEntries()         // all entries, built-in first
TemplateRegistry.getEntry(uri)        // manifest entry or null
TemplateRegistry.getPayload(uri)      // payload or built-in fallback

// Write
TemplateRegistry.ingest(tplObj, origin)         // → { ok, entry, isNew, isUpgrade } | { ok:false, error }
TemplateRegistry.ingestEncoded(b64str, origin)  // decode + ingest
TemplateRegistry.remove(uri)                    // removes from manifest + payload store
TemplateRegistry.markUsed(uri)                  // updates usedAt timestamp

// Discovery
TemplateRegistry.detectOnPage(doc)    // → [{ b64, header, origin }] not yet ingested/refused
TemplateRegistry.hasRefused(uri)      // boolean
TemplateRegistry.recordRefusal(uri)   // persists refusal

// Render
TemplateRegistry.render(uri, data)    // → HTML string, always succeeds (built-in fallback)

// Utilities
TemplateRegistry.encode(tplObj)       // → base64url string for page-embed
TemplateRegistry.compress(jsonStr)    // fflate deflate → base64url
TemplateRegistry.decompress(b64str)   // base64url → fflate inflate → string
TemplateRegistry.BUILTIN_URI          // 'urn:workpads:tpl:note:default:v1'
```

---

## Authoring Checklist

For template authors shipping a page that carries a Workpads template:

- [ ] Set `uri` to a stable URL or URN you control
- [ ] Set `version` and increment it on every breaking change
- [ ] Declare `platforms` accurately — do not use `'all'` for desktop-only templates
- [ ] Use `{{#block}}…{{/block}}` guards around optional data fields
- [ ] Test at 240×320px (KaiOS baseline) if `platforms` includes `'kaios'` or `'all'`
- [ ] Use responsive CSS (`@media min-width`) for multi-platform templates
- [ ] Embed with `<script type="application/workpads-template" data-platform="…">` and the output of `TemplateRegistry.encode(tplObj)`
- [ ] Set `lineage.supersedes` when replacing an older template URI

---

## Phases

| Phase | Scope | Status |
|---|---|---|
| 1 | Registry, manifest store, payload cache, render engine (A/B/P), built-in default | **Live** |
| 2 | Page ingestion — scanner active in in-app browser, consent banner UI | Planned |
| 3 | Gallery index format, `<link rel="workpads-gallery">`, Discover tab in Management | Planned |
| 4 | Schema B wired to record share, Schema P wired to note presentations, component resolution (C) | Planned |

---

## Wire Encoding Extensions (SUI-006)

**Status:** v1.1 — 2026-05-18  
**Kaios source:** `draft_specs/TEMPLATE-SYSTEM-DESIGN.md`

### Extended Template Object Fields

Templates may carry formula and FLAGS4 extension fields alongside the standard manifest fields:

| Field | Type | Description |
|---|---|---|
| `label_map` | object | `{ field_key: "Display Label" }` — overrides default labels for field_flags slots |
| `formula` | string | Infix formula string for computed fields (e.g. `"qty * rate"`) |
| `formula_bytecode` | string | RPN bytecode for the formula (base64url-encoded), carried in TRIG block |
| `block_order` | string[] | Override field display order (list of field_keys in desired render order) |
| `custom_fields` | object[] | FLAGS4 slot allocations for template-specific fields (see below) |
| `template_hash` | string | Canonical SHA-256 of the template content (hex), used for template-keyed encryption |

### custom_fields array

Each entry in `custom_fields` allocates a FLAGS4 bit:

```json
{
  "bit": 3,
  "key": "warranty_expiry",
  "label": "Warranty expires",
  "type": "date",
  "encoding": "uint16_days"
}
```

`bit`: FLAGS4 bit (0–6) allocated to this field.  
`type`: `"date"`, `"text"`, `"uint8"`, `"uint16"`, `"uint24"`.  
`encoding`: overrides default for the type (e.g. `"uint16_days"` for compact date fields).

### Custom Template ID (EXT_TEMPLATE Variant Type)

For community templates that need a wire-level ID without central registration, use the EXT_TEMPLATE variant type (EXT_SIGNAL=100 in meta1 bits 5-3):

```
3 bytes:
  byte 1: uint8  = CRC-8 of creator identity hash (256 namespace slots)
  bytes 2-3: uint16 = CRC-16 of (identity + template name + creation date)
```

Collision probability: ~1 in 16M across all namespaces. For built-in templates, use the BASE_TEMPLATE 3-bit codes (no extension bytes needed).

### Formula Encoding

When a template includes computed fields, the formula is encoded in the TRIG block using RPN bytecode. The `formula` string in the template object is for human readability and authoring tools only — the wire format uses `formula_bytecode`. Both must be present for the encoder to validate consistency.
