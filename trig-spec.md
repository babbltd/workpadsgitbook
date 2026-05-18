# TRIG — Display Trigger Bytecode Specification

**Status:** v0.1 — 2026-05-18  
**SUI:** SUI-007  
**Kaios source:** `dev_daily/OPEN-QUESTIONS.md` OQ-32 (spec complete); `dev_daily/TRIG-DESIGN.md`; `js/lib/trig.js` (reference implementation, Round 9)  
**Cross-references:** codec.md (TRIG block wire format), security-wrapper.md (TRIG evaluation order relative to decryption)

---

## 1. Purpose

TRIG is a 1–20 byte bytecode program embedded in a pads-v1 frame. It is evaluated client-side by the workpads shell to determine **what to show, to whom, in which visual mode** — before any UI is rendered.

**The problem it solves:** A URL is public once shared. Link-preview bots (WhatsApp, Telegram, iMessage) fetch URLs before the human recipient reads them. Search engine crawlers index page content. TRIG ensures these automated systems see a blank page while qualifying human viewers see the full card, form, or service menu.

**How it works:** The URL fragment (`#1pb/...`) is never sent to the server in an HTTP request. Bots that only fetch HTML never see the fragment. TRIG bytes sit inside the fragment payload. Evaluating TRIG requires JavaScript execution — bots that do not run JS never reach it. TRIG conditions (KNOWN_CONTACT, HAS_APP, IS_HUMAN) are answered from local device state, not the network — a bot has none of these.

**TRIG is a presentation gate, not a security boundary.** A determined attacker with a real browser can circumvent most conditions. For sensitive content, use the encryption layer (`#1ps/`, `#1pt/`). TRIG handles the presentation surface; encryption handles the security surface. They are orthogonal and may be combined.

---

## 2. Wire Format

TRIG occupies the **TRIG block** in the pads-v1 frame. The block is present when `meta2 bit 5 (HAS_TRIG_BLOCK) = 1`. It appears after the participants block (or after the financial block if no participants block is present), before any security wrapper.

```
[trig_len]    uint8 — length of TRIG program in bytes (0–20)
[trig_bytes]  N bytes — TRIG v1 bytecode (N = trig_len)
```

| trig_len | Interpretation |
|----------|----------------|
| 0 | Block present but empty; shell defaults to SHOW_ALWAYS NATIVE |
| 1 | Pattern token (single byte, high nibble = `0x0`) |
| 2–20 | Bytecode program; byte 0 is the header byte |
| > 20 | Spec violation; shell renders BLANK and does not crash |

TRIG can be carried by any record type — not only presentation records. A financial record with HAS_TRIG_BLOCK=1 can gate which party sees the financial detail.

---

## 3. Two Modes

### Mode 1 — Pattern Token (1 byte)

A single byte `0x0P` (high nibble = 0, low nibble = pattern ID P) is the complete program. The stack machine is never entered. The shell expands the pattern to its equivalent display behaviour.

**Pattern token table:**

| Token | Name | Behaviour |
|-------|------|-----------|
| `0x00` | SHOW_ALWAYS | Show card to all visitors. Shell default when no TRIG block present. |
| `0x01` | KNOWN_CONTACT_SHOW | Show card only if sender is a known contact |
| `0x02` | HAS_APP_SHOW | Show card only if viewer has the app installed |
| `0x03` | CODE_VERIFIED_SHOW | Show card only if per-contact scramble code has been verified |
| `0x04` | HUMAN_SHOW | Show card if human visitor detected (anti-bot) |
| `0x05` | KNOWN_OR_APP_SHOW | Show card if known contact OR has app |
| `0x06` | KNOWN_AND_APP_SHOW | Show card if known contact AND has app |
| `0x07` | ALWAYS_BLANK | Show nothing (stealth / test mode) |
| `0x08` | FORM_ALWAYS | Show contact form to all visitors |
| `0x09` | FORM_IF_HUMAN | Show form only if human visitor |
| `0x0A` | SERVICE_MENU_ALWAYS | Show service menu to all visitors |
| `0x0B` | SERVICE_MENU_KNOWN | Show service menu to known contacts only |
| `0x0C`–`0x0F` | RESERVED | Future common patterns |

**Pattern + CSS theme (2-byte extension):** `0x0P 0x1T` — pattern P with theme T (0–15 from theme codebook). Example: KNOWN_CONTACT_SHOW with dark theme = `0x01 0x13`.

### Mode 2 — Bytecode Program (2–20 bytes)

Any TRIG block where the first byte has high nibble ≠ `0x0` is a bytecode program. Byte 0 is the header; subsequent bytes are the instruction stream.

---

## 4. Bytecode Header Byte

```
Byte 0: [VER:2][HAS_CSS:1][HAS_TERNARY:1][PROG_LEN:4]
```

| Field | Bits | Meaning |
|-------|------|---------|
| VER | 7–6 | `00` = TRIG v1. Unknown VER → shell renders BLANK (graceful degradation). |
| HAS_CSS | 5 | 1 if program contains LOAD_CSS — optimiser hint; shell may pre-fetch CSS module |
| HAS_TERNARY | 4 | 1 if program contains TERNARY — optimiser hint for fast-path evaluation |
| PROG_LEN | 3–0 | Instruction byte count following the header. 1–14 = direct. 15 = extended (next byte carries `length − 15`; range 15–269, though frame max is 20). |

HAS_CSS and HAS_TERNARY are **optimiser hints only** — a shell that ignores them evaluates identically. PROG_LEN lets a shell skip an unknown TRIG block (unknown VER) without parsing instruction bytes.

---

## 5. Instruction Set

Each instruction byte: `[OP:4][ARG:4]`

- **OP** = operation (high nibble)
- **ARG** = inline immediate 0–14. **ARG = 15** = read next byte as actual value (extended range, 0–255).

| OP | Mnemonic | ARG meaning | Stack effect |
|----|----------|-------------|--------------|
| `0x0_` | PATTERN | pattern_id (0–15) → splice in pattern expansion | inline |
| `0x1_` | LOAD_CSS | css_id (0–14; see §8) | side effect |
| `0x2_` | SET_LAYOUT | layout_id (0–14) | side effect |
| `0x3_` | SHOW | display_mode (0–7; see §7) | bool → render if true, BLANK if false |
| `0x4_` | SHOW_ALWAYS | display_mode (0–7) | — (unconditional render) |
| `0x5_` | AND | count (2–14): pop N bools | N bools → 1 bool |
| `0x6_` | OR | count (2–14): pop N bools | N bools → 1 bool |
| `0x7_` | NOT | 0 | bool → bool |
| `0x8_` | JZ | skip_bytes (0–14): skip N instruction bytes if top is false | bool → |
| `0x9_` | TERNARY | 0; next 3 bytes: cond_id, mode_true, mode_false | — (4-byte shortcut) |
| `0xA_` | LOAD_JS | js_id (0–14; see §10) | side effect |
| `0xB_` | SET_THEME | theme_id (0–14; see §9) | side effect |
| `0xC_` | BLOOM | 0; next 2 bytes: 16-bit filter operand | → bool |
| `0xD_` | PUSH_COND | condition_id (0–14; see §6) | → bool |
| `0xE_` | PUSH_LIT | ARG bit 0: 0=false, 1=true | → bool |
| `0xF_` | EXTENDED | next byte = full 8-bit secondary opcode | varies |

**Notes:**
- TRIG programs are always finite. There are no backward-jump instructions. EXTENDED (`0xF_`) is reserved for future opcode expansion; unknown secondary opcodes → shell renders BLANK.
- Maximum stack depth: 8. Stack underflow → BLANK.
- If the byte stream ends without a SHOW or SHOW_ALWAYS: render BLANK.
- Inside a bytecode program, PATTERN (`0x0P`) splices in the expansion of pattern P at the current position — useful when a common pattern forms part of a larger program.

---

## 6. Condition Registry

Used by PUSH_COND (ARG) and TERNARY (cond_id byte).

| ID | Mnemonic | True when |
|----|----------|-----------|
| 0 | HAS_APP | Viewer has the Workpads app installed |
| 1 | KNOWN_CONTACT | Sender appears in viewer's local contacts database |
| 2 | CODE_VERIFIED | Per-contact scramble code entered and verified for this sender |
| 3 | IS_HUMAN | JS execution context + human interaction signal detected (not a bot) |
| 4 | HAS_SAVED_RECORD | Viewer has previously saved a record from this sender |
| 5 | ORG_MATCH | Sender's org name matches a saved business contact on device |
| 6 | HAS_TEMPLATE | Viewer's app has the referenced template installed |
| 7 | DAYLIGHT_HOURS | Current local time is 06:00–20:00 |
| 8 | RECENT_CONTACT | Known contact with recorded interaction within 90 days |
| 9 | APP_VERSION_OK | Viewer app version ≥ version floor declared in record |
| 10 | REPLY_PENDING | Viewer has an unsent reply queued for this sender |
| 11 | LOCATION_NEAR | Device location within stated radius of geo block in record (geo block must be present) |
| 12–14 | RESERVED | Future conditions; push false if unknown |

**Condition evaluation properties:**
- All conditions are **synchronous local queries** — no network calls, no async operations.
- **IS_HUMAN (ID=3)**: shell defers evaluation until one of: (a) an interaction event (`touchstart`, `click`, `keydown`, `mousemove`) fires, or (b) a 500ms timeout expires. IS_HUMAN = true if interaction fires within 500ms; false on timeout. This adds a worst-case 500ms render delay for IS_HUMAN-gated content.
- **LOCATION_NEAR (ID=11)**: uses the last-known cached device position — no live GPS query is triggered during TRIG evaluation.
- Conditions with the same ID appearing multiple times in a program should be cached for the duration of one evaluation run.
- Unknown condition IDs → push false (graceful degradation).

---

## 7. Display Modes

Used by SHOW, SHOW_ALWAYS, and TERNARY (mode_true, mode_false bytes).

| ID | Name | Description |
|----|------|-------------|
| 0 | CARD | Full business card / record card view |
| 1 | LIST | Service list or compact line-item view |
| 2 | FORM | Interactive form (requires form schema block) |
| 3 | MINIMAL | Name + contact button only |
| 4 | TICKER | Single-line scrolling banner |
| 5 | BLANK | Empty page (record parsed, nothing shown) |
| 6 | NATIVE | Shell chooses display mode based on record type |
| 7 | RESERVED | — |

In TERNARY byte encoding: `mode_false = 0xFF` means BLANK; `mode_false = 0xFE` means NATIVE.

---

## 8. CSS Codebook (LOAD_CSS)

Shell-side CSS modules loaded by 1-byte ID. No CSS bytes travel in the frame or URL. Modules are bundled in the shell or installed as extensions.

| ID | Module | Description |
|----|--------|-------------|
| 0 | BASE | Reset, mobile-safe spacing, base font. Loaded implicitly by all display modes. |
| 1 | CARD_LIGHT | White card, shadow, clean typography |
| 2 | CARD_DARK | Dark card variant |
| 3 | FORM_STD | Input fields, labels, submit button, validation states |
| 4 | SERVICE_LIST | Compact service/price list layout |
| 5 | BILLBOARD | Large-format with accent colour support |
| 6 | MINIMAL | Name + contact icon only, zero chrome |
| 7–14 | RESERVED | Future CSS modules |

CSS modules map naturally to display modes (CARD → CARD_LIGHT or CARD_DARK; FORM → FORM_STD; LIST → SERVICE_LIST). LOAD_CSS gives the sender explicit control — a BILLBOARD CSS layout with CARD display mode is valid.

---

## 9. Theme Codebook (SET_THEME)

Applied as CSS custom-property overrides on top of the loaded CSS module (`--accent-color`, `--bg-color`, `--text-color`).

| ID | Theme |
|----|-------|
| 0 | NEUTRAL (grey/white, default) |
| 1 | WARM (amber/cream) |
| 2 | COOL (blue/white) |
| 3 | DARK (dark background) |
| 4–14 | RESERVED |

---

## 10. JS Codebook (LOAD_JS)

Pre-registered, shell-bundled JS modules. Not sender-provided code. Runs in a sandboxed iframe; communicates with the shell via postMessage.

| ID | Module | Function |
|----|--------|----------|
| 0 | CONTACT_FORM | Name + phone contact request form; on submit, creates a contact record on the viewer's device |
| 1 | BOOKING_FORM | Date picker + service selector; on submit, sends a booking request reply record |
| 2 | REPLY_ROUTER | Generic form-to-record router for any form schema block |
| 3–14 | RESERVED | Domain-specific JS modules |

LOAD_JS codebook modules are distinct from sender-provided inline/fetch-target JS (`#1ps/`/`#1pt/` only). Codebook modules are shell-vetted and version-pinned.

---

## 11. TERNARY Instruction

The most common complex TRIG program: "show X to audience A, Y to audience B." TERNARY compresses this to 4 bytes.

```
0x90  cond_id  mode_true  mode_false
```

- `0x90` — TERNARY opcode (OP=9, ARG=0)
- `cond_id` — full byte (0–255; standard conditions 0–11)
- `mode_true` — display mode if condition is true
- `mode_false` — display mode if false (`0xFF` = BLANK, `0xFE` = NATIVE)

A standalone TERNARY program requires a header byte (since byte 0 high nibble = `0x9` ≠ `0x0`):

```
header: VER=00, HAS_TERNARY=1, PROG_LEN=4  →  0x14
Full:   0x14  0x90  cond_id  mode_true  mode_false  (5 bytes total)
```

---

## 12. BLOOM Instruction

Anti-bot Bloom filter. Pushes `true` onto the stack if the viewer's JS environment passes a declared subset of browser capability checks that bots typically cannot satisfy.

```
0xC0  bloom_hi  bloom_lo
```

The 16-bit operand (`bloom_hi << 8 | bloom_lo`) is a bitmask selecting which checks to AND together:

| Bit | Test |
|-----|------|
| 15 | `requestAnimationFrame` timing consistency (≥ 60fps) |
| 14 | `PointerEvent` or `TouchEvent` support |
| 13 | Clipboard API accessible |
| 12 | `IntersectionObserver` present |
| 11 | `CSS.supports()` returns expected value for `accent-color` |
| 10 | Canvas fingerprint entropy above threshold |
| 9–0 | RESERVED (future capability checks) |

Example: test bits 15, 14, 13, 12 only → operand = `0xF0 0x00`.

BLOOM pushes a bool like PUSH_COND — AND, OR, NOT, SHOW all operate on it identically.

---

## 13. Evaluation Rules

1. **Single pass, left to right.** No backward jumps; programs always terminate.
2. **First byte determines mode:** high nibble `0x0` = pattern token; otherwise = bytecode program with header.
3. **Unknown VER in header → BLANK** (do not attempt to parse instructions).
4. **Unknown opcodes** (including unknown EXTENDED secondary codes) → BLANK.
5. **Stack underflow** → BLANK.
6. **No SHOW or SHOW_ALWAYS reached** after reading all bytes → BLANK.
7. **trig_len > 20** → BLANK (spec violation; do not parse).
8. **IS_HUMAN requires deferred evaluation**: render a loading state; commit to result after first interaction event or 500ms timeout, whichever comes first.
9. **Side effects (LOAD_CSS, SET_THEME, LOAD_JS) accumulate** throughout evaluation and apply after SHOW/SHOW_ALWAYS terminates. A SHOW that resolves to BLANK discards accumulated side effects.

---

## 14. Version Upgrade Path

VER bits (header byte bits 7–6) provide 3 future version slots (`01`, `10`, `11`). A shell receiving an unknown VER renders BLANK without crashing. The sender can trust that old shells degrade gracefully — they will show nothing rather than mis-render.

VER `01` and above may redefine the instruction set, condition registry, or codebook assignments. Any such change requires a new VER value and does not invalidate v1 programs.

The EXTENDED opcode (`0xF_`) reserves a full 8-bit secondary space (256 additional opcodes) for future expansion within VER `00`. Unrecognised secondary opcodes → BLANK.

---

## 15. Example Programs

| Use case | Bytes | Hex |
|----------|-------|-----|
| Show card to all (default) | 1 | `0x00` |
| Show card to known contacts only | 1 | `0x01` |
| Show form to all | 1 | `0x08` |
| Show service menu, known contacts only | 1 | `0x0B` |
| Always blank (stealth mode) | 1 | `0x07` |
| Show card, dark theme | 2 | `0x01 0x13` |
| Show card if human (anti-bot) | 1 | `0x04` |
| Show card if known contact → card, else form | 5 | `0x14 0x90 0x01 0x00 0x02` |
| Show card if HAS_APP AND KNOWN_CONTACT | 5 | `0x04 0xD0 0xD1 0x52 0x30` |
| BLOOM filter (real browser) + KNOWN_CONTACT → card | 7 | `0x05 0xC0 0xF0 0x00 0xD1 0x52 0x30` |
| Service menu + BILLBOARD CSS + WARM theme | 4 | `0x03 0x15 0x11 0x41` |
| Known → card, human (not known) → form, else blank (JZ ladder) | 9 | `0x08 0xD1 0x83 0x40 0x00 0xD3 0x83 0x32 0x05` |

**JZ ladder explanation (9-byte example):**
```
header:  0x08  (VER=0, LEN=8)
0xD1     PUSH_COND(KNOWN_CONTACT)
0x83     JZ skip=3 (skip next 3 bytes if false)
0x40     SHOW_ALWAYS(CARD)       — KNOWN_CONTACT was true
0x00     (skipped if KNOWN_CONTACT=false)
0xD3     PUSH_COND(IS_HUMAN)
0x83     JZ skip=3
0x32     SHOW(FORM)              — IS_HUMAN was true, not a known contact
0x05     SHOW_ALWAYS(BLANK)      — neither condition
```

---

## 16. Relationship to C-TRIG

TRIG programs control **display** — who sees what, which CSS/theme/JS loads. C-TRIG programs control **obligations** — when a condition is met, what is triggered in an agreement. They share the wire frame (both use the TRIG block), distinguished by the MODE bit in the program header:

```
Header byte bit 5 (when VER=00, interpreted as MODE):
  0 = display TRIG (rendering rules)
  1 = commitment C-TRIG (obligation rules)
```

A shell that only understands display TRIG ignores C-TRIG blocks (unknown MODE = BLANK, which is safe for display-only contexts). See `ctrig-evaluator-spec.md` for the full C-TRIG specification.

---

## 17. Implementation Reference

**Reference evaluator:** `workpadskaios/js/lib/trig.js` — `WPTrig.evaluate(trigBytes, ctx)` → `{ mode, show, css, theme, js }`.

The evaluator is approximately 80 lines of JavaScript. It is stateless between calls, requires no dependencies beyond the codec, and runs in under 0.5ms for any valid 20-byte program on KaiOS hardware.

**KaiOS performance notes:**
- Condition evaluation that requires IndexedDB (KNOWN_CONTACT, HAS_SAVED_RECORD, RECENT_CONTACT) should use an in-memory Set pre-populated at app startup — O(1) lookup, no IndexedDB call during TRIG evaluation.
- The HAS_CSS hint in the header byte enables CSS module pre-fetch before evaluation completes, eliminating style-recalculation jank on first paint.
