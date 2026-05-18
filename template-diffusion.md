# The Template Diffusion Model
## How Workpads templates spread, survive, and compose into a worldwide system

**Status:** v1 (2026-05-12) — working theory, Phase 1 live
**Author note:** This document speculates beyond current implementation.
The core mechanism is proven. The scale argument is a design intention.

---

## The Problem It Solves

A user on a KaiOS feature phone in Lagos shares a job note via WhatsApp. The recipient opens the link in Firefox on a laptop in London. The sender chose a presentation template they installed last week from a construction trade site. The recipient has never heard of Workpads.

How does the link render with the right template — or gracefully fall back — without any central server being involved in the render decision?

That is the problem this architecture solves.

---

## The Core Insight

**Presentation is data, not infrastructure.**

A template is a small JSON object: some HTML, some CSS, an identity string. It compresses to a few hundred bytes. It can be embedded in a `<script>` tag in any static HTML page — a blog post, a trade association site, a contractor's invoice page, a personal portfolio. It costs the host nothing to include.

When a Workpads app — on any device — visits that page, it can silently detect the embedded template, ask the user once for consent, and cache it locally. Forever. No CDN. No subscription. No dependency on any Workpads domain being alive.

The template has escaped the app.

---

## How It Works: The Carry Mechanism

Any static HTML page can carry a Workpads template by including:

```html
<script type="application/workpads-template" data-platform="all">
eJyNj0EKwjAQRf...   ← fflate-compressed, base64url-encoded template JSON
</script>
```

The payload is invisible to browsers (unrecognised `type` attribute). Search engines index the page normally. The template sits dormant until a Workpads app visits.

On visit, the app's page scanner:
1. Finds the `<script>` tag
2. Decodes the header (URI, name, version) — a few microseconds
3. Checks: is this URI in our refusals list? Already at this version?
4. If new: presents a consent prompt once — template name, origin domain, trust level
5. On accept: stores payload in localStorage — sub-millisecond write
6. Never asks again for this URI + version combination

The template is now a permanent part of that user's app. It survives clearing the browser cache. It survives reinstalling the app (if the user backs up localStorage). It works offline forever.

---

## The Fallback Chain

Every share link carries a template URI. When the recipient's app renders the link, it walks:

```
Requested URI (local cache)
    ↓ miss
Fetch canonical URI over HTTPS (if online and URI is https://)
    ↓ miss / offline
Fetch fallback URI from lineage.supersedes (if set)
    ↓ miss
Built-in default (urn:workpads:tpl:note:default:v1)
    ↓ always present
Rendered. Always.
```

The built-in default is hardcoded in the app binary. It cannot be evicted. It is the guaranteed terminus.

This means: a share link generated today will render on any future Workpads install, on any device, on any platform, even if workpads.me no longer exists, even if the template author's site is gone, even if the user is offline.

The system degrades gracefully, never catastrophically.

---

## Template Identity: URNs and URLs

First-party and built-in templates use URN form:

```
urn:workpads:tpl:{type}:{name}:{version}
```

This is a permanent identity. It resolves only within the Workpads registry — no fetch is ever needed. The app ships knowing what `urn:workpads:tpl:note:default:v1` looks like.

Third-party templates use HTTPS URLs as their canonical URI:

```
https://constructiontrades.io/tpls/site-report-v2.json
```

This URL is both the identifier and the fetch target. If the site moves, the template can declare `lineage.supersedes` pointing to the old URI — the fallback chain follows the breadcrumb.

---

## Scale Without Central Coordination

Here is where the model becomes interesting.

Workpads does not need to be the only site publishing templates. Any person, trade association, software developer, or company can embed a template in their static site. They do not need to register with Workpads. They do not need an API key. They just need to:

1. Write a template object (HTML + CSS + identity fields)
2. Compress it with fflate and base64url-encode it
3. Paste it into a `<script>` tag on any page their audience visits

That is the entire onboarding cost for a template publisher.

### What this enables at scale

**Trade verticals**: A construction equipment supplier embeds a job-site inspection template on their product manual page. Every Workpads user who visits for the manual silently discovers the template. No app store submission. No Workpads approval. No CDN contract.

**Regional professionals**: A Lagos-based contractor association publishes a Nigerian compliance report template on their member resources page. Templates reach their natural audience through existing browsing behaviour — the same pages people visit anyway.

**Software integrations**: An accounting SaaS that accepts Workpads record shares can publish its preferred invoice display template on its own domain. Users who connect the integration automatically pick it up.

**Peer to peer**: A single individual can publish a template on their personal site. If one person links to it in a WhatsApp group, every Workpads user who clicks that page can pick up the template. Virality through ordinary web traffic, not app stores.

**Forks and derivations**: The `lineage.derivedFrom` field lets authors credit and build on each other's work. A template for UK electrical compliance can be forked for Irish regulations, crediting the original. The fork carries its own URI and can be independently distributed.

---

## What Workpads.org / Workpads.me Provides

Within this decentralised model, first-party domains serve specific roles:

| Domain | Role |
|---|---|
| `workpads.me` | Receiving endpoint for shared record and note links (`/p#`, `/n#`) |
| `workpads.org` | Standard documentation, built-in template hosting, gallery index seed |
| `workpads.app` | Primary app distribution |

None of these are required for the template system to function. They are conveniences, not dependencies.

The first-party gallery (planned Phase 3) is a curated index — an editorially maintained `<link rel="workpads-gallery">` pointing to a JSON feed of reviewed templates. But the existence of this gallery does not prevent templates from existing and spreading entirely outside it.

This is the key architectural distinction from conventional template marketplaces: **the gallery discovers templates; it does not gate them**.

---

## Trust and the Consent Model

Because any page can publish any template, trust must be explicit.

The manifest `trust` field carries four levels:

| Level | Meaning |
|---|---|
| `built-in` | Shipped with the app binary. Unconditional. |
| `official` | Verified by Workpads. Signed manifest. |
| `community` | Self-published, unverified. User consented. |
| `local` | Created on-device by the user. |

When a page-embedded template is detected, the consent prompt always shows:
- Template name
- Origin domain
- Trust level badge

The user can refuse permanently. Refusals are stored by URI — the same template from the same origin will never prompt again.

Templates cannot execute JavaScript in Phase 1 or 2. The render engine is a pure string substitution engine (Schemas A and B) or a section assembler (Schema P). This is by design: a template that injects `<script>` is not a Workpads template — it is something else, and it will not be rendered as one.

Schema C (components/partials), planned for Phase 4, will introduce sandboxed evaluation with a defined capability API. The trust model for Schema C is stricter: only `official` and `local` trust levels will be permitted to use it.

---

## The Standard as a Multiplier

The Workpads Standard is a public specification. Any application that implements the manifest format, the codec, the fallback chain, and the consent model becomes a compatible consumer of the same template ecosystem.

A competing note-taking app that implements the Standard can render Workpads-published templates. A desktop app can detect page-embedded templates and offer to install them. A print service can consume Schema P templates and render PDFs.

The template ecosystem does not belong to Workpads. The Standard is the asset. Workpads is the first implementation.

If the Standard is adopted, the worldwide template graph grows independently of any single company's decisions, funding, or continued existence.

---

## Schema Types and Their Diffusion Paths

| Schema | Complexity | Diffusion model |
|---|---|---|
| A | Simple | Embeddable anywhere; renders on all platforms including KaiOS |
| B | Medium | Named slots for structured layouts; mobile/desktop primary |
| P | Rich | Ordered sections; desktop and print primary; survives as fallback to A on KaiOS |
| C | Composable | Cross-template component references; requires online or pre-cached deps |

Schema A is the lingua franca. Any template author targeting maximum reach should start here.

---

## What a "Worldwide Template System" Actually Means

Not a marketplace. Not a CDN. Not a registry with an API.

A **graph of static pages**, each potentially carrying a template payload, connected by ordinary web links and browsing behaviour. The graph grows whenever anyone adds a `<script type="application/workpads-template">` tag to any page. It shrinks when pages go offline — but because templates are cached locally, installed templates survive the source going dark.

The system has no centre. It has no chokepoint. It has no single point of failure.

Its growth mechanism is identical to the web's own growth mechanism: people publishing things, other people reading them.

The Workpads app is the reader. The Standard is the syntax. The pages are already there.

---

## Implementation Phases

| Phase | What ships |
|---|---|
| 1 ✓ | Registry, manifest store, render engine, built-in default, NoteCodec, note share screen |
| 2 | Page scanner active in in-app browser, consent banner UI |
| 3 | Gallery index format, Discover tab, `official` trust verification |
| 4 | Schema C (components), cross-template composition, Schema P note presentations |
| ∞ | Standard adoption by third-party apps and sites |

---

## Distribution Tiers (SUI-017)

**Status:** v1.1 — 2026-05-18  
**Kaios source:** `draft_specs/TEMPLATE-SYSTEM-DESIGN.md` §distribution

Three tiers govern how templates reach devices. Each tier works independently; lower tiers are preferred where available.

| Tier | Mechanism | Connectivity required | Use |
|------|-----------|----------------------|-----|
| **1 — Built-in** | Bundled in the app binary at install time | None — always available | System templates (Groups A–H in TEMPLATE-CATALOGUE.md): core record types, standard financial templates |
| **2 — CDN** | `workpads.me/t/<sha256-hex>` — served from Cloudflare CDN; cached to IndexedDB after first fetch; `Cache-Control: immutable` | On first use only; zero cost after that | Custom and sector templates (Groups I–J): electrician, construction, medical, custom community templates |
| **3 — Peer-to-peer** | Data Sync Bundle embeds the template payload inline in the share URL | None — template travels with the record | Offline / P2P contexts; first-time sector template distribution to receivers who have never been online with this template |

**CDN provider:** Cloudflare (primary) — 24 African PoPs including Lagos, Nairobi, Accra, Johannesburg. Content-addressed URLs never change; infinite cache TTL.

**Template not found fallback (all tiers exhausted):**
1. Check IndexedDB by content hash → found: render immediately
2. Not found: attempt CDN fetch (5s timeout on KaiOS)
3. CDN unavailable: render in raw mode — canonical field names, no labels, no formula
4. Display notice: "Template unavailable. Showing raw record."

**No startup CDN calls.** The app never fetches templates at launch. Templates are fetched lazily, on first encounter with a record that needs them. Built-in templates cover all core use cases without any network dependency.

**Template versioning:** content addressing means old records always reference the old hash, which the CDN retains indefinitely. No migration is ever required. The template registry maintains an index `{template_id → [hash_v1, hash_v2, ...]}` — all versions kept.
