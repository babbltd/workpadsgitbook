# §AT — Attachment Specification

**Status:** v0.1 (2026-05-18)  
**SUI:** SUI-015  
**Kaios source:** `draft_specs/ATTACHMENT-DESIGN.md` design 2026-05-17

---

## 1. Philosophy

Attachments follow an **early internet / MMS-era progressive delivery** model: fast, low-overhead, 2G-viable. Four principles:

1. Something displays immediately — Tier 0 inline thumbnail requires no network call
2. Useful quality appears fast on any connection — Tier 1 in under 3 seconds on EDGE
3. Full quality only when explicitly requested — Tier 2–3 on user tap
4. The wire frame is not bloated by images — it carries a reference, not the image bytes

---

## 2. Wire Encoding

The attachment field is located at **FLAGS3 bit 4**.

```
Field type: [u16 len][UTF-8], max 500 bytes
Content:    structured string — Tier 0 thumbnail + content URL
```

**Format:**

```
"t0:<thumbnail>:<content_url>?t=<available_tiers>"
```

| Segment | Description |
|---------|-------------|
| `t0:` | Tier 0 prefix — inline thumbnail follows |
| `<thumbnail>` | Compact image preview (see §3 — encoding decision pending) |
| `:` | Separator |
| `<content_url>` | Content-addressed URL for higher quality tiers |
| `?t=123` | Optional: bitmask of available tiers (1=T1, 2=T2, 3=T3; `t=13`=T1+T3 only) |

When no Tier 0 is available (legacy attachments or pre-upload state): field value is just the content URL with no `t0:` prefix.

---

## 3. Tier 0 Inline Thumbnail Encoding

**Encoding: quantized 8×8 micro-thumbnail** (OQ-AT1 resolved)

An 8×8 pixel image quantized to 256 colours, base64-encoded:

```
8×8 at 256 colours = 64 pixels × 8 bits = 64 bytes = 86 base64 chars

attachment field example:
  "t0:AQID...base64...xyz:https://workpads.me/a/sha256:abc123"
       ^^^^^^^^^^^^^^^^^^^
       86-char base64 of 8×8×1B pixel data
```

- No library required — encode/decode in ~20 lines of JS
- Decoder scales to display size using nearest-neighbour interpolation (blocky pixel output)
- The blocky aesthetic intentionally communicates "field photo, not polished asset" — fits workpads visual identity for on-site documentation
- Zero bundle overhead on RAM-constrained KaiOS devices

**Encoding algorithm:**
1. Resize source image to 8×8 pixels (area averaging)
2. Quantize each pixel to nearest colour in a 256-entry palette (6-bit RGB: 2 bits per channel, 64 colours; or 8-bit indexed)
3. Store as 64 raw bytes — one byte per pixel, row-major
4. Base64-encode the 64 bytes → 86 ASCII characters

**Decoding algorithm:**
1. Base64-decode → 64 bytes
2. Reconstruct 8×8 pixel grid
3. Scale to display size using nearest-neighbour → blocky preview at any resolution

---

## 4. Quality Tier Architecture

| Tier | Resolution | Size target | When fetched |
|------|-----------|-------------|-------------|
| 0 | Inline thumbnail (no fetch) | 20–86 bytes in frame | Always — immediate display |
| 1 | 320×240 px, JPEG Q40–50 | 8–20 KB | On record open (default) |
| 2 | 800×600 px, JPEG Q65 | 60–120 KB | On explicit expand tap |
| 3 | Original dimensions | Camera original | On explicit "full quality" request only |

Tier 1 at 320×240 is full-screen quality on KaiOS (240×320 screen). No need to auto-load Tier 2 on KaiOS card view.

Tier 3 is never auto-loaded — explicit user request only. KaiOS RAM limit (256MB typical) cannot hold multiple full-resolution images.

---

## 5. Content URL and CDN Tier Routing

**Routing: single hash, quality via `?q=` parameter** (OQ-AT3 resolved)

```
https://workpads.me/a/<sha256_of_original>         → Tier 1 (default)
https://workpads.me/a/<sha256_of_original>?q=2     → Tier 2
https://workpads.me/a/<sha256_of_original>?q=3     → Tier 3 (original)
```

One hash represents all quality tiers of the same image. The server generates and stores each tier on upload; the CDN routes the appropriate tier based on `?q=`. The attachment field carries one URL — the `?t=` bitmask declares which tiers are available.

The `?q=` parameter is not part of the content hash — the hash identifies the original. A future upgrade path to per-tier content-addressing (for IPFS compatibility) would require a new URL scheme; the `?q=` model does not foreclose this.

---

## 6. Content URL Formats

Three valid content URL types in the attachment field:

```
Workpads CDN (primary):   https://workpads.me/a/<sha256hash>
IPFS (optional):          ipfs://bafybei<CID>
Bare hash (offline ref):  sha256:<hex_hash>
```

**Bare hash** is used for offline records where the image is in local storage keyed by hash. The receiver fetches from whatever source they have. No URL = the record is a valid reference even with no connectivity.

---

## 7. Image Capture and Processing Pipeline

When a user attaches a photo in the workpads app:

```
1. CAPTURE
   Camera API → raw JPEG buffer

2. PROCESS
   a. Generate Tier 0 thumbnail from buffer
   b. Resize to 320×240, JPEG Q40 → Tier 1 buffer
   c. Resize to 800×600, JPEG Q65 → Tier 2 buffer
   d. Compute SHA-256 of original → content hash

3. ENCODE IN FRAME (immediate — before upload)
   attachment field = "t0:<thumbnail>:<CDN_URL>?t=0"
   Tier 0 is embedded NOW — frame is shareable before upload completes
   ?t=0 signals: Tier 0 only currently available

4. UPLOAD (when connected)
   POST /upload with [hash, tier1_bytes, tier2_bytes, tier3_bytes]
   Server stores all tiers, CDN distributes

5. FRAME UPDATE (on upload complete)
   attachment field updated: ?t=0123
   (Note: this requires a mutable field update post-encode — see §8)
```

**Offline-first implication:** a record with a photo attachment can be shared and displayed with Tier 0 preview BEFORE the upload completes. The receiver sees a thumbnail immediately. Tier 1 becomes available after the sender's next sync window.

---

## 8. Pre-Upload Mutability

The `?t=` parameter signals which tiers are currently available. This creates a tension with immutable frame design:

- Before upload: `?t=0` (only Tier 0 available)
- After upload: `?t=0123` (all tiers available)

**Resolution:** the `?t=` parameter is advisory only. Decoders treat a missing Tier 1 as a transient state — they attempt the fetch; if they receive a 404 or timeout, they show the Tier 0 thumbnail and schedule a retry. No frame mutation required. The field value with `?t=0` is valid and complete; it simply tells the receiver not to bother fetching Tier 1 yet.

Frames are still immutable — the `?t=` field in the original shared URL will always say `?t=0` if shared before upload. Receivers that open the URL after upload succeeds will attempt Tier 1 regardless of the `?t=` value (the advisory is optimistic, not authoritative).

---

## 9. Multi-Image Support

FLAGS3 bit 4 supports a single attachment field with 500B budget. For records needing multiple photos:

**MVP (comma-separated):**
```
"t0:<thumb1>:<url1>,t0:<thumb2>:<url2>"
```
Approximately 2–3 images with ThumbHashes within the 500B limit.

**Post-MVP:** `BASE_TEMPLATE=100` (Document/media) with compound lines — each line is a media attachment with optional per-image caption.

---

## 10. Privacy

- **EXIF stripping:** app strips all EXIF metadata before upload (GPS, device model, timestamp removed). Only pixel data retained.
- **Attachment URL in URL:** the attachment content hash is visible in plain `#1pa` records — anyone with the URL can fetch the image from CDN.
- **For sensitive attachments:** use `#1ps/` encrypted records — attachment URL is inside encrypted payload. Or use tokenised CDN URL (time-limited signed URL generated by server).
- **Face blurring:** post-MVP — automatic face detection + Gaussian blur on upload, toggleable for portrait records.

---

## 11. KaiOS-Specific Constraints

| Constraint | Impact | Design response |
|------------|--------|-----------------|
| Screen: 240×320 px | Tier 1 (320×240) is full-screen | No need for Tier 2 on KaiOS card view |
| RAM: 256MB typical | Cannot hold multiple full-res images | Never auto-load Tier 3 |
| 2G/EDGE common | 56 kbps in rural areas | Tier 1 at 10–20KB loads in <3s on EDGE |
| Camera: 2–5MP | Original 500KB–2MB | Tier 1 = ~90% size reduction |
| No WebP (older KaiOS) | JPEG only | All tiers encoded as JPEG |
| Local storage limited | Cannot cache all images | LRU cache 10MB; Tier 0 always cached (in frame) |

---

## 12. Open Items

- **OQ-AT5** — NTAG213 Marker with image: 86 bytes of quantized Tier 0 thumbnail leaves ~58 bytes for agreement frame on NTAG213 — too tight for a full agreement. Requires NTAG215 (504B) when Marker + image combination is needed. Design note only; no wire change required.
