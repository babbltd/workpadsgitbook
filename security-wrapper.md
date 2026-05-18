# Security Wrapper Specification

**Status:** v0.1 — 2026-05-18  
**SUI:** SUI-002  
**Kaios source:** `system/dev_refs/FRAME-SPEC.md` §12 + `draft_specs/SECURITY-DESIGN.md`  
**Cross-references:** codec.md (URL tag table), participants-block.md (RECIPIENT_TYPE)

---

## 1. Overview

The security wrapper is an optional encryption and obfuscation layer applied to a pads-v1 frame before base64url encoding. It is signalled by the URL scheme tag — the tag character at position 1 distinguishes plain records (`1pa/`) from wrapped variants (`1ps/`, `1ph/`, `1pt/`).

The wrapper is applied *after* the pads-v1 frame is assembled and *before* the final base64url encoding step. Decoders apply the inverse in the opposite order.

**Security goal:** Records containing financial data, identity information, or internal cost figures must not be readable by anyone who intercepts the URL unless they possess the shared key.

---

## 2. URL Tag Dispatch Table

| Tag | Name | Purpose |
|-----|------|---------|
| `#1pa/` | Plain record | Standard records: financial, service, contact. No encryption. |
| `#1pb/` | Public billboard | Non-financial presentation: business cards, service menus, contact forms. Public circulation. Never carries financial data. |
| `#1pf/` | Financial presentation | Invoice display, statements, pay summaries. Not for public circulation. Use with `#1ps/` or `#1pt/` for sensitive records. |
| `#1ps/` | Full scramble | AES-CTR encryption + field scramble. Private and colleague records. |
| `#1ph/` | Partial scramble | Header bytes unencrypted (meta1, meta2, setup_byte, transaction_byte); field data encrypted. Receiver sees record type before entering passphrase. |
| `#1pt/` | Template-keyed | Template content is the encryption key. Template ID advertised plainly; payload meaningless without the template file. |
| `#l/` | List share | Standalone option list for form builder multi-select fields. Not a record. |

---

## 3. Five-Layer Security Stack

Layers are applied in order when encoding. The order is reversed for decoding.

### Layer 1 — Deflate Seed Poisoning

DEFLATE is applied with a deterministic non-standard seed derived from a shared key. The receiver must know the seed to decompress; an incorrect seed produces garbage output.

- Provides lightweight obfuscation without full cryptographic strength
- No computational cost beyond standard DEFLATE
- Signalled by `SEED_POISON=1` in the preamble byte

### Layer 2 — Field Scramble

The field_flags byte order is permuted using a key-derived shuffle. The actual field data bytes are unchanged; only the flag-to-offset mapping is scrambled. A receiver who does not know the shuffle key cannot identify field boundaries.

- Signalled by `SCRAMBLE=1` in the preamble byte
- Uses `scramble_seed[4:8]` (4 bytes from the derived master key)

### Layer 3 — AES-CTR Payload Encryption

The full frame (after deflate + field scramble) is encrypted using AES-CTR with a 128-bit key.

```
Key:  cipher_key = master[0:16]
IV:   iv = master[0:16]  (same bytes as cipher_key)
```

IV reuse is safe because the salt (random 4 bytes per record) makes every `master` unique. Two records with the same passphrase produce different `master` values and therefore different IVs.

- Signalled by `AES=1` in the preamble byte

### Layer 4 — Receiver Commitment HMAC

An 8-byte HMAC-SHA256 truncated tag is embedded inside the encrypted envelope (the last 8 bytes before AES-CTR is applied). The HMAC commits to the intended receiver's phone hash — only the named receiver can verify.

```
HMAC input: cipher_key || receiver_phone_hash
Tag:        HMAC-SHA256(...)[0:8]
```

- Signalled by `HMAC=1` in the preamble byte
- Requires `RECIPIENT_TYPE=1` (named recipient) in meta1
- Verification failure on decryption → reject record

### Layer 5 — Preamble Byte

One byte prepended to the encrypted output, before base64url encoding. It signals which security layers are active:

```
bit 7: SCRAMBLE      1=field scramble applied
bit 6: AES           1=AES-CTR encryption applied
bit 5: HMAC          1=receiver commitment HMAC present (last 8B inside envelope)
bit 4: SEED_POISON   1=deflate seed poisoning applied
bit 3: HKDF_KEY      1=HKDF-derived key (domain-separated)
                     0=direct SHA-256 of template bytes (#1pt/ only)
bits 2-0: KEY_HINT   lower 3 bits of cipher_key[0]; helps receiver select passphrase from key ring
```

---

## 4. Key Derivation

```
master       = SHA-256(passphrase || salt)   [32 bytes]
cipher_key   = master[0:16]
scramble_seed = master[16:32]
iv           = master[0:16]   (same as cipher_key; safe because salt is fresh per record)
```

**Salt:** 4 random bytes generated at encode time. Included in the URL as the first segment before the `.` separator. Not a secret.

**KEY_HINT:** The lower 3 bits of `cipher_key[0]`. Receivers with multiple passphrases try the one matching KEY_HINT first — eliminates most wrong-passphrase attempts without revealing key material.

---

## 5. URL Structure for Secured Records

```
workpads.me/p#1ps/<b64url(salt_4B)>.<b64url(preamble_byte + encrypted_inner)>
```

The salt (4 random bytes → 6 base64url chars) precedes the `.` separator. It is a derivation nonce — include it in key derivation but do not treat it as secret.

**Partial scramble (`#1ph/`) structure:**

```
workpads.me/p#1ph/<b64url(salt)>.<b64url(preamble + clear_header + encrypted_inner)>

clear_header = meta1[1B] + meta2[0-1B] + setup_byte[0-1B] + transaction_byte[0-1B]
```

The receiver sees the record type (BASE_TEMPLATE) and DOMAIN before entering a passphrase.

---

## 6. Encode Path

```
frame
  → deflate(seed = scramble_seed[0:4])
  → field_scramble(seed = scramble_seed[4:8])
  → [append hmac_tag_8B if HMAC=1]
  → AES-CTR(cipher_key, iv)
  → prepend preamble_byte
  → base64url
  → URL fragment after salt.
```

Full URL: `#1ps/` + base64url(salt) + `.` + base64url(preamble + encrypted).

---

## 7. Decode Path

```
1. Split URL fragment on `.` → extract salt (first segment, base64url)
2. Derive master, cipher_key, scramble_seed from passphrase + salt
3. Check KEY_HINT (preamble bits 2-0) against cipher_key[0] & 0x07 — abort if mismatch
4. AES-CTR decrypt
5. If HMAC=1: extract last 8 bytes; verify HMAC-SHA256(cipher_key || receiver_phone_hash, inner)[0:8] → abort on mismatch
6. Un-field-scramble using scramble_seed[4:8]
7. Inflate with seed = scramble_seed[0:4]
8. Parse pads-v1 frame
```

---

## 8. Per-Contact Key Derivation

Workers may derive per-contact keys to avoid sharing the same passphrase across all recipients.

```
contact_key = SHA-256(master_key || contact_phone_hash)
```

This allows key revocation per contact without changing the master passphrase.

---

## 9. Template-Keyed Derivation (`#1pt/`)

For template-keyed records, the template content (HTML + CSS bytes) is the passphrase:

```
cipher_key = SHA-256(template_bytes)[0:16]     (HKDF_KEY=0)
```

or with HKDF domain separation:

```
cipher_key = HKDF(template_bytes, salt, "workpads-tpl-key")[0:16]  (HKDF_KEY=1)
```

The template ID is advertised plainly in the URL. The record is only readable by parties who possess the template file.

---

## 10. Guidance by Record Type

| Content | Minimum security | Rationale |
|---------|-----------------|-----------|
| Invoice (customer-facing) | `#1pa/` or `#1pf/` | Customer must be able to open without passphrase |
| Worker amount / internal cost | `#1ps/` | Internal — never in plain URLs |
| EXPENSE_CAT=01 (COGS) or EXPENSE_CAT=10 (running cost) | `#1ps/` | Internal cost data |
| Pay summary (State Commit) | `#1ps/` | Worker financial data |
| Colleague record | `#1ps/` + RECIPIENT_TYPE=1 | Named delivery + HMAC |
| Business card (public) | `#1pb/` | No financial data; no encryption needed |

The codec itself does not refuse to encode plain records with sensitive data — enforcement is at the application share sheet layer.
