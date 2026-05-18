# §DS — Data Sync Bundle

**Status:** v0.1 (2026-05-18)  
**SUI:** SUI-016  
**Kaios source:** `dev_refs/TEMPLATE-SYSTEM-RESEARCH.md` §5

---

## 1. Purpose

A Data Sync Bundle packages multiple dependency items (template JSON, contact data, list data, and a record) into a single shareable URL. The receiver's app installs the dependencies before displaying the record — enabling reliable offline-first record sharing when the CDN is unavailable.

The bundle is the peer-to-peer layer of the three-tier template distribution model (see `template-diffusion.md` §Distribution Tiers). It removes the CDN dependency for offline field contexts.

---

## 2. When to Use

| Context | Mechanism | Bundle needed? |
|---------|-----------|---------------|
| Online receiver, template in CDN | CDN fetch on first open | No |
| Online receiver, template not in CDN | CDN fetch, fallback to raw mode | No |
| Offline receiver, template not yet installed | Template travels in bundle | Yes |
| P2P sharing (both devices offline) | Template embedded inline | Yes |
| Sector template first-time distribution | Bundle seeds the registry | Yes |

The common case (online receiver, template in CDN) does not need a bundle. Bundles are for offline/P2P contexts and first-time sector template distribution.

---

## 3. Bundle URL Format

```
workpads.me/sync#1pa/<bundle-payload>
```

`bundle-payload` is a deflate-compressed byte sequence, base64url-encoded (no padding).

---

## 4. Bundle Wire Format

```
[bundle_header]    1 byte
  bits 7-4: BUNDLE_TYPE   0x0=record bundle (standard)  0x1–0xF=reserved
  bits 3-0: ITEM_COUNT    number of items following (1–15)

Per item (ITEM_COUNT times):
[item_type]        1 byte
  0x01 = template payload (raw template JSON bytes)
  0x02 = list payload (contact list or price list)
  0x03 = contact record (participant data)
  0x04 = record (pads-v1 frame — MUST be the final item)

[item_length]      2 bytes — big-endian uint16
[item_payload]     item_length bytes
```

The record item (type 0x04) MUST be the last item in the bundle. All preceding items are dependencies. The app installs items in order; the record is opened after all dependencies are processed.

---

## 5. Delivery Model

**Resolved design (record-first with background dependency fetch):**

In the common case, the record is shared directly (no bundle URL). Dependencies (templates, contacts) are fetched in the background from CDN after the record is opened. The bundle mechanism is reserved for contexts where background CDN fetch is not viable.

**Bundle delivery fallback path:**
1. Sender detects offline receiver context (or no CDN available)
2. Sender app assembles bundle: template JSON → contact bytes → record frame
3. Bundle is deflate-compressed and base64url-encoded
4. Shared as a single URL or QR code

---

## 6. Size Budget

| Item | Typical deflated size |
|------|----------------------|
| Sector template JSON | 400–800 bytes |
| Contact record | 50–100 bytes |
| Service record (pads-v1) | 30–60 bytes |
| **Total bundle** | **500–1000 bytes deflated** |

Base64url encoding adds ~33%: typical bundle = **700–1400 chars** as a URL.

KaiOS 3.0 Gecko and modern browsers support URLs up to 8 KB. The bundle fits comfortably within limits even with multiple items.

---

## 7. Partial Failure Handling

If a dependency item fails to install, the receiver continues rather than blocking on the error:

| Item fails | Behaviour |
|------------|-----------|
| Template install fails | Skip; open record in raw mode (canonical field names, no labels) |
| List install fails | Record opens without autofill; list fetchable later via `#t/` URL |
| Contact install fails | Record opens; contact auto-created from participants block in the record |

Template failure degrades gracefully. List/contact failure is invisible to the user beyond missing autofill.

---

## 8. Relationship to Template Distribution Tiers

```
Tier 1 (built-in):         System templates bundled in app binary — zero connectivity
Tier 2 (CDN):              workpads.me/t/<sha256-hex> — cached to IndexedDB after first fetch
Tier 3 (peer-to-peer):     Data Sync Bundle embeds template inline — zero connectivity
```

The Data Sync Bundle is the Tier 3 delivery mechanism. A sender who knows the receiver is offline or has not yet installed the relevant sector template embeds the template in the bundle. No CDN required.
