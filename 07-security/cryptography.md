# 🔑 Cryptography

> You will never implement a cipher. You will constantly make decisions about *which* primitive, *which* mode, and *how* to combine them — and that's where systems actually break.

**Prerequisite:** [AppSec](appsec.md)

---

## 📑 Contents

1. [The Golden Rule](#1-the-golden-rule)
2. [What Crypto Gives You](#2-what-crypto-gives-you)
3. [Randomness](#3-randomness)
4. [Hashing](#4-hashing)
5. [Symmetric Encryption](#5-symmetric-encryption)
6. [Asymmetric Encryption](#6-asymmetric-encryption)
7. [Key Exchange](#7-key-exchange)
8. [Digital Signatures](#8-digital-signatures)
9. [TLS](#9-tls)
10. [PKI and Certificates](#10-pki-and-certificates)
11. [Key Management](#11-key-management)
12. [Applied Patterns](#12-applied-patterns)
13. [Attacks Worth Knowing](#13-attacks-worth-knowing)
14. [Post-Quantum](#14-post-quantum)
15. [Interview Section](#15-interview-section)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. The Golden Rule

```
   ⭐⭐ DON'T ROLL YOUR OWN CRYPTO.

   And the more important corollary that people miss:

   ⭐ DON'T ROLL YOUR OWN CRYPTO **PROTOCOL** EITHER.

   Almost nobody writes their own AES. But people constantly
   invent their own way of COMBINING primitives — encrypt then
   sign, or sign then encrypt, or a custom token format, or a
   homemade key derivation scheme.

   That's where real systems break. The primitives are solid;
   the composition is where the bugs live.
```

```
   ⭐ USE, IN ORDER OF PREFERENCE

   1. A high-level library that makes the decisions for you
        libsodium / NaCl · Python `cryptography` (Fernet) ·
        Go `crypto` · Tink (Google)
      ⭐ These are "misuse-resistant" by design — the API
        doesn't offer you the footguns.

   2. A well-established protocol
        TLS · Signal Protocol · JWT with a vetted library ·
        age (file encryption)

   3. ⚠️ Low-level primitives (raw AES, raw RSA) — only if you
      genuinely know what you're doing, and probably not even then.

   ⚠️ WARNING SIGNS IN A CODE REVIEW
     • ECB mode anywhere
     • A hardcoded IV, or an IV of all zeros
     • ⭐ MD5 or SHA-1 for anything security-relevant
     • Encryption without authentication
     • `==` comparing secrets (timing attack)
     • A custom "encryption" scheme with a clever name
```

---

## 2. What Crypto Gives You

```
   ⭐ FOUR DISTINCT PROPERTIES — and they are NOT the same thing

   ┌──────────────────────────────────────────────────────────────┐
   │ CONFIDENTIALITY   Nobody else can READ it                    │
   │                   → encryption                               │
   ├──────────────────────────────────────────────────────────────┤
   │ INTEGRITY         Nobody can MODIFY it undetected            │
   │                   → MAC / AEAD / signature                   │
   ├──────────────────────────────────────────────────────────────┤
   │ AUTHENTICITY      It genuinely came from who it claims       │
   │                   → MAC (shared key) or signature (public)   │
   ├──────────────────────────────────────────────────────────────┤
   │ NON-REPUDIATION   The sender CANNOT LATER DENY sending it    │
   │                   → ⭐ signature ONLY, never a MAC            │
   └──────────────────────────────────────────────────────────────┘

   ⚠️⚠️ ENCRYPTION ALONE DOES NOT GIVE YOU INTEGRITY.

   With CBC or CTR mode and no MAC, an attacker can FLIP BITS in
   the ciphertext and predictably flip the corresponding bits in
   the plaintext — without knowing the key.

   ⭐ THIS IS WHY AEAD MODES EXIST. Always use authenticated
     encryption. "Encrypt then MAC" is the correct manual
     construction if you must build it yourself — but use AEAD.
```

```
   ⭐ MAC vs SIGNATURE — the distinction interviewers probe

   MAC (HMAC)        SHARED secret key. Both parties can create
                     AND verify. ⚠️ So a MAC proves the message
                     came from someone holding the key — which
                     could be either party. No non-repudiation.
                     ✅ Fast.

   SIGNATURE         PRIVATE key signs, PUBLIC key verifies.
                     ⭐ Only the holder of the private key could
                     have produced it → NON-REPUDIATION.
                     ⚠️ Much slower.
```

---

## 3. Randomness

```
   ⭐ THE MOST COMMON CRYPTO BUG IS BAD RANDOMNESS.

   Every key, IV, nonce, salt, session ID, and token depends on
   it. Predictable randomness silently destroys everything built
   on top, and it produces no error and no visible symptom.
```

```python
# ❌ NEVER for anything security-relevant
import random
random.randint(...)          # Mersenne Twister — ⭐ observing ~624
random.choice(...)           #   outputs lets you predict ALL future
                             #   values. Not a flaw; it's not a CSPRNG.

# ✅ ALWAYS
import secrets
secrets.token_bytes(32)      # raw bytes
secrets.token_urlsafe(32)    # ⭐ URL-safe token
secrets.token_hex(16)
secrets.compare_digest(a, b) # ⭐ constant-time comparison
```

```javascript
// ❌ Math.random()  — not cryptographically secure, period
// ✅
crypto.getRandomValues(new Uint8Array(32));   // browser
crypto.randomBytes(32);                       // Node
crypto.randomUUID();                          // both
```

```
   ⭐ ENTROPY SIZING
     128 bits (16 bytes)  minimum for anything meaningful
     256 bits (32 bytes)  ⭐ the sensible default for keys and tokens

   ⚠️ WHERE ENTROPY GOES WRONG IN PRACTICE
     • VMs cloned from a snapshot share entropy state
     • Embedded devices booting with no entropy source
     • ⭐ Containers starting before the kernel pool is seeded
     • Using a PRNG seeded from the current time
```

---

## 4. Hashing

```
   ⭐ THREE PROPERTIES OF A CRYPTOGRAPHIC HASH

   PREIMAGE RESISTANCE     given h(x), you can't find x
   SECOND PREIMAGE         given x, you can't find y ≠ x with
                           the same hash
   COLLISION RESISTANCE    ⭐ you can't find ANY x, y that collide
                           (this is the property that broke first
                            for MD5 and SHA-1)
```

```
   ┌──────────────┬────────┬──────────────────────────────────────┐
   │ Algorithm    │ Output │ Status                               │
   ├──────────────┼────────┼──────────────────────────────────────┤
   │ MD5          │ 128b   │ ⚠️⚠️ BROKEN — collisions in seconds   │
   │ SHA-1        │ 160b   │ ⚠️⚠️ BROKEN — SHAttered (2017)        │
   │ SHA-256      │ 256b   │ ✅ Solid, widely supported            │
   │ SHA-512      │ 512b   │ ✅ Often FASTER on 64-bit CPUs        │
   │ SHA-3        │ varies │ ✅ Different construction (sponge) —  │
   │              │        │ insurance against a SHA-2 break      │
   │ BLAKE3   ⭐   │ varies │ ✅ Very fast, parallelizable, modern  │
   └──────────────┴────────┴──────────────────────────────────────┘
```

```
   ⚠️⚠️ HASHES ARE FOR INTEGRITY, NOT FOR PASSWORDS.

   SHA-256 is designed to be FAST. A GPU computes billions per
   second. That's exactly wrong for passwords.

   ⭐ PASSWORDS NEED A SLOW, MEMORY-HARD KDF:
     Argon2id (best) · scrypt · bcrypt · PBKDF2 (FIPS only)

   ⭐ MEMORY-HARDNESS is the key property. GPUs and ASICs have
     enormous parallel compute but limited memory per core, so
     requiring 64 MB per hash is what actually defeats them —
     not just "more iterations."
```

```
   ⭐ HMAC — the right way to key a hash

   ❌ NEVER: hash(secret + message)
      → ⚠️ LENGTH EXTENSION ATTACK. With Merkle-Damgård hashes
        (MD5, SHA-1, SHA-2), an attacker who knows h(secret‖msg)
        and len(secret) can compute h(secret‖msg‖padding‖extra)
        WITHOUT knowing the secret.

   ✅ HMAC(key, message) — its nested construction is
      specifically designed to prevent this.
      hmac.new(key, msg, hashlib.sha256).hexdigest()

   ⭐ SHA-3 and BLAKE aren't vulnerable to length extension,
     but HMAC is still the standard, interoperable answer.
```

```
   OTHER HASH USES
   • Content addressing (git, Docker layers, IPFS)
   • Deduplication
   • ⭐ Merkle trees — efficiently prove which parts of two
     large datasets differ (see [Distributed Theory](../05-system-design/06-distributed-theory.md#6-quorums))
   • HMAC for webhook signatures and API request signing
   • Commitment schemes
```

---

## 5. Symmetric Encryption

```
   ⭐ ONE KEY, used for both encryption and decryption.
     Fast — hardware-accelerated AES runs at GB/s.
     ⚠️ The problem is DISTRIBUTING the key.
```

### Modes — this is where the decisions are

```
   ┌──────────────────────────────────────────────────────────────┐
   │ ⚠️⚠️ ECB — NEVER USE                                          │
   │   Each block encrypted independently → identical plaintext   │
   │   blocks produce identical ciphertext blocks.                │
   │   ⭐ THE "ECB PENGUIN": encrypt an image in ECB and you can   │
   │     still SEE the picture. That's the whole problem in one   │
   │     image.                                                   │
   ├──────────────────────────────────────────────────────────────┤
   │ CBC — chains blocks, needs a random IV                       │
   │   ⚠️ No integrity → padding oracle attacks                    │
   │   ⚠️ IV must be random and unpredictable, never reused        │
   ├──────────────────────────────────────────────────────────────┤
   │ CTR — turns a block cipher into a stream cipher              │
   │   ⚠️ No integrity. ⚠️⚠️ NONCE REUSE IS CATASTROPHIC — XOR the  │
   │     two ciphertexts and the keystream cancels out, leaving   │
   │     plaintext XOR plaintext.                                 │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ GCM — AEAD: encryption + authentication together           │
   │   ✅ The standard choice. Hardware accelerated.               │
   │   ⚠️⚠️ NONCE REUSE BREAKS IT COMPLETELY — it leaks the        │
   │     authentication key, not just plaintext.                  │
   │   ⚠️ 96-bit nonce → don't use random nonces beyond ~2³² msgs  │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ ChaCha20-Poly1305 — AEAD, no hardware needed               │
   │   ✅ Faster than AES on devices without AES-NI (mobile, IoT)  │
   │   ✅ ⭐ Constant-time by construction — no cache-timing risk   │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ XChaCha20-Poly1305 — 192-bit nonce                         │
   │   ✅ Nonces can safely be RANDOM. Removes the entire class    │
   │     of nonce-management bugs. Prefer this when available.    │
   └──────────────────────────────────────────────────────────────┘
```

```python
# ✅ THE HIGH-LEVEL WAY — let the library decide
from cryptography.fernet import Fernet

key = Fernet.generate_key()          # store this securely
f = Fernet(key)
token = f.encrypt(b"secret data")    # AES-CBC + HMAC, IV handled,
plaintext = f.decrypt(token)         # timestamped, versioned

# ✅ AEAD when you need associated data
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

key = AESGCM.generate_key(bit_length=256)
aesgcm = AESGCM(key)
nonce = os.urandom(12)               # ⭐ UNIQUE per message, always
ct = aesgcm.encrypt(nonce, plaintext, associated_data)
pt = aesgcm.decrypt(nonce, ct, associated_data)
# ⭐ associated_data is authenticated but NOT encrypted — perfect
#   for routing headers, IDs, or version numbers that must not
#   be tampered with but don't need hiding
```

```
   ⚠️⚠️ THE NONCE RULE — the single most important operational
     rule in symmetric crypto

   NEVER REUSE A (key, nonce) PAIR.

   With GCM, reuse doesn't just leak two plaintexts — it leaks
   the authentication subkey, letting an attacker FORGE arbitrary
   messages. It is a total break.

   ⭐ SAFE STRATEGIES
     • A counter, if you can guarantee it never resets or
       duplicates (⚠️ hard across restarts and replicas)
     • Random 96-bit nonces, with a message limit around 2³²
     • ⭐ XChaCha20 with a 192-bit nonce — random is safe
       indefinitely. Just use this.
```

---

## 6. Asymmetric Encryption

```
   ⭐ TWO KEYS: public encrypts / verifies, private decrypts / signs.
     Solves key distribution — but ~1000× SLOWER than symmetric.

   ⭐ THEREFORE: NOBODY ENCRYPTS BULK DATA WITH RSA.
     The universal pattern is HYBRID ENCRYPTION:
       1. Generate a random symmetric key
       2. Encrypt the DATA with the symmetric key (fast)
       3. Encrypt the SYMMETRIC KEY with the public key (small)
       4. Send both
     ⭐ This is what TLS, PGP, and every encrypted file format do.
```

```
   ┌──────────────┬──────────────────────────────────────────────┐
   │ RSA          │ ⚠️ Needs 2048-bit minimum, 3072+ preferred.   │
   │              │ Slow, large keys, and ⚠️ many footguns        │
   │              │ (raw RSA, PKCS#1 v1.5 padding oracles).      │
   │              │ Use OAEP for encryption, PSS for signing.    │
   ├──────────────┼──────────────────────────────────────────────┤
   │ ⭐ ECC        │ 256-bit ECC ≈ 3072-bit RSA in strength.      │
   │ (P-256,      │ Much smaller keys, much faster.              │
   │  Curve25519) │ ⭐ Curve25519/Ed25519 are designed to be      │
   │              │ misuse-resistant — no invalid-curve attacks, │
   │              │ constant-time by construction.               │
   └──────────────┴──────────────────────────────────────────────┘

   ⭐ CHOOSE: X25519 for key exchange, Ed25519 for signatures.
     Use RSA only for compatibility with something old.
```

---

## 7. Key Exchange

```
   ⭐ DIFFIE-HELLMAN — establishing a shared secret over a
     PUBLIC channel, without ever transmitting it

   ┌──────────────────────────────────────────────────────────────┐
   │  Alice                                          Bob          │
   │  picks secret a                                 picks b      │
   │  sends g^a mod p  ──────────────────▶                        │
   │                   ◀──────────────────  sends g^b mod p       │
   │  computes (g^b)^a                    computes (g^a)^b        │
   │         = g^(ab)                            = g^(ab)         │
   │                    ⭐ SAME SHARED SECRET                      │
   │                                                              │
   │  ⭐ An eavesdropper sees g^a and g^b but cannot compute       │
   │    g^(ab) — that's the discrete logarithm problem.           │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐⭐ EPHEMERAL DH (DHE / ECDHE) → FORWARD SECRECY

   Generate a FRESH keypair for every session and discard it
   afterward.

   ⭐ CONSEQUENCE: if the server's long-term private key is
     stolen tomorrow, all previously recorded traffic remains
     undecryptable — because the session keys never existed
     anywhere except in memory, and are gone.

   ⚠️ WITHOUT forward secrecy (static RSA key exchange), an
     attacker who records traffic today and steals the key in
     five years can decrypt ALL of it retroactively.

   ⭐ This is why TLS 1.3 REMOVED non-forward-secret key
     exchange entirely. It's mandatory now.
```

```
   ⚠️ RAW DH IS VULNERABLE TO MITM

   Diffie-Hellman gives you a shared secret with SOMEBODY —
   it does not tell you WHO. An attacker in the middle can run
   DH separately with each party.

   ⭐ THEREFORE DH MUST BE AUTHENTICATED — by a signature over
     the exchange (TLS), or by pre-shared identity keys (Signal).
     This is precisely what certificates are for.
```

---

## 8. Digital Signatures

```
   SIGN:    signature = Sign(private_key, hash(message))
   VERIFY:  Verify(public_key, message, signature) → true/false

   ⭐ You sign the HASH, not the message — signatures operate on
     fixed-size inputs, and hashing makes the scheme work for
     messages of any length.
```

```
   ⭐ ALGORITHMS
     Ed25519  ⭐ Fast, small (64-byte signatures), misuse-resistant,
              deterministic. The default choice.
     ECDSA    Widely supported (used in TLS certificates).
              ⚠️ Requires a unique random nonce per signature —
              see the PS3 disaster below.
     RSA-PSS  Use instead of PKCS#1 v1.5 for new systems.

   ⚠️⚠️ THE ECDSA NONCE CATASTROPHE
     ECDSA requires a fresh random k for each signature. Sony
     used a CONSTANT k in the PlayStation 3, which let anyone
     algebraically recover the master private key from two
     signatures. The console was permanently broken.

     ⭐ Ed25519 derives k DETERMINISTICALLY from the key and the
       message, so this entire failure mode cannot occur.
       That's a good illustration of "misuse-resistant design."
```

---

## 9. TLS

```
   ⭐ TLS 1.3 HANDSHAKE — 1 round trip

   Client                                        Server
     │ ClientHello                                  │
     │   + supported cipher suites                  │
     │   + ⭐ key_share (guesses the group and       │
     │     sends its ephemeral public key already)  │
     │ ─────────────────────────────────────────────▶│
     │                                              │
     │                     ServerHello              │
     │                       + key_share            │
     │                     {EncryptedExtensions}    │
     │                     {Certificate}            │
     │                     {CertificateVerify} ⭐    │
     │                     {Finished}               │
     │ ◀─────────────────────────────────────────────│
     │ {Finished}                                   │
     │ ⭐ + application data (immediately)           │
     │ ─────────────────────────────────────────────▶│

   { } = already encrypted

   ⭐ CertificateVerify is the crucial step: the server SIGNS
     the handshake transcript with its private key, proving it
     actually possesses the key its certificate claims. Without
     it, anyone could present someone else's certificate.
```

```
   ⭐ WHAT TLS 1.3 IMPROVED OVER 1.2

   ✅ 1 RTT instead of 2 (and 0-RTT resumption)
   ✅ ⭐ Removed ALL legacy cipher suites: no RC4, no 3DES, no
     CBC modes, no MD5/SHA-1 signatures, no static RSA
   ✅ ⭐ Forward secrecy MANDATORY
   ✅ Only AEAD ciphers
   ✅ Encrypts more of the handshake (certificate included)

   ⭐ THE DESIGN LESSON: the biggest security win came from
     REMOVING options. Every configurable legacy algorithm was
     a downgrade attack waiting to happen (FREAK, Logjam, POODLE
     were all "negotiate down to something broken" attacks).
```

```
   ⚠️ 0-RTT RESUMPTION IS REPLAYABLE
     Data sent in the first flight can be replayed by an
     attacker, because there's no round trip to prove freshness.
     ⭐ Only use 0-RTT for IDEMPOTENT requests. Never for a
       POST that transfers money.
```

```
   mTLS — MUTUAL TLS
     Both sides present certificates. ⭐ The standard for
     service-to-service authentication in a service mesh —
     it replaces "trust based on network location" with
     "trust based on cryptographic identity," which is the
     core of a zero-trust architecture.
```

---

## 10. PKI and Certificates

```
   ⭐ THE CHAIN OF TRUST

   ┌──────────────────────────────────────────────────────────────┐
   │  ROOT CA        ⭐ self-signed, in the OS/browser trust store │
   │      │ signs      (kept OFFLINE in an HSM)                   │
   │      ▼                                                       │
   │  INTERMEDIATE CA   ⭐ your server MUST send this              │
   │      │ signs         (compromise doesn't burn the root)      │
   │      ▼                                                       │
   │  LEAF certificate for api.example.com                        │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY INTERMEDIATES EXIST: the root's private key is too
     valuable to keep online. It signs a small number of
     intermediates, then goes back in the safe. If an
     intermediate is compromised, it can be revoked without
     invalidating every certificate in the world.
```

```
   ⚠️⚠️ THE #1 TLS MISCONFIGURATION: MISSING INTERMEDIATE

   Symptom: "works in Chrome, fails from curl / Java / Go."

   ⭐ WHY: browsers cache intermediates from previous sites and
     many fetch a missing one via the AIA extension. Programmatic
     clients do neither, so validation fails.

   Diagnose:  openssl s_client -connect host:443 -showcerts
   Fix:       concatenate the intermediate into your served chain
```

```
   VALIDATION LEVELS
     DV  Domain Validated — you control the domain. ⭐ Free
         (Let's Encrypt), automated, and cryptographically
         identical to the others.
     OV  Organization Validated
     EV  Extended Validation — ⚠️ browsers REMOVED the green bar
         because studies showed users didn't notice it. Little
         practical value now.

   ⭐ DV with automated renewal is the right default. The
     security comes from the cryptography, not the paperwork.
```

```
   ⚠️ REVOCATION IS THE WEAK POINT OF PKI

   CRL     Certificate Revocation List — large, cached, often
           stale
   OCSP    Online status check — ⚠️ a privacy leak (the CA learns
           every site you visit) and a latency cost, and most
           browsers FAIL OPEN if it's unreachable — which makes
           it nearly useless against an active attacker
   ⭐ OCSP STAPLING  The server fetches and attaches a signed,
           timestamped status. Fixes privacy and latency.
   ⭐ SHORT-LIVED CERTIFICATES  The real answer. A 90-day (or
           now 6-day) certificate with automated renewal makes
           revocation largely unnecessary — the exposure window
           is bounded by expiry.
```

```
   CERTIFICATE TRANSPARENCY
     ⭐ All publicly trusted certificates must be logged to
       public append-only logs. You can monitor for certificates
       issued for YOUR domain that you didn't request — which
       detects CA compromise and mis-issuance.
     ⭐ Set up CT monitoring for your domains. It's free and it
       catches an attack class nothing else does.
```

---

## 11. Key Management

```
   ⭐ KEY MANAGEMENT IS HARDER THAN THE CRYPTOGRAPHY, AND IT'S
     WHERE REAL SYSTEMS FAIL.

   THE LIFECYCLE
   ┌──────────────────────────────────────────────────────────────┐
   │ GENERATE   in a secure environment, from a CSPRNG            │
   │ STORE      ⭐ HSM / KMS ideally; never in code or config      │
   │ DISTRIBUTE securely, to the minimum set of consumers         │
   │ USE        least privilege, and audit every use              │
   │ ROTATE     ⭐ on a schedule AND on any suspected compromise   │
   │ REVOKE     when compromised                                  │
   │ DESTROY    securely, when retention no longer requires it    │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ ENVELOPE ENCRYPTION — the pattern every cloud KMS uses

   ┌──────────────────────────────────────────────────────────────┐
   │  MASTER KEY (in KMS/HSM — ⭐ NEVER leaves)                    │
   │       │ encrypts                                             │
   │       ▼                                                      │
   │  DATA ENCRYPTION KEY (DEK) — one per file/record/tenant      │
   │       │ encrypts                                             │
   │       ▼                                                      │
   │  YOUR ACTUAL DATA                                            │
   │                                                              │
   │  Store the ENCRYPTED DEK alongside the ciphertext.           │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY THIS IS THE STANDARD PATTERN
     • Only small DEKs cross the KMS boundary — bulk data is
       encrypted locally at full speed
     • ⭐ Rotating the master key only requires re-encrypting the
       DEKs, not re-encrypting petabytes of data
     • Per-tenant or per-record DEKs give fine-grained
       cryptographic isolation and revocation
     • The master key never exists outside the HSM
```

```
   ⭐ KEY ROTATION — support it from day one

   Store a KEY ID with every ciphertext:
     { "kid": "v3", "nonce": "...", "ct": "..." }

   → Decrypt with the key the ciphertext names.
   → Encrypt new data with the current key.
   → Re-encrypt old data lazily, or in a background job.

   ⚠️ Retrofitting rotation into a system that assumed one
     eternal key is genuinely painful. The key ID field costs
     nothing now and saves an enormous migration later.
```

---

## 12. Applied Patterns

### Encrypting data at rest

```python
# ⭐ Envelope encryption with a cloud KMS
import boto3, os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

kms = boto3.client("kms")

def encrypt(plaintext: bytes, key_id: str, context: dict) -> dict:
    # ⭐ KMS generates the DEK and returns it both plain and encrypted
    resp = kms.generate_data_key(
        KeyId=key_id, KeySpec="AES_256",
        EncryptionContext=context,       # ⭐ authenticated context —
    )                                    #   binds the key to a tenant/purpose
    dek_plain, dek_encrypted = resp["Plaintext"], resp["CiphertextBlob"]

    nonce = os.urandom(12)
    ct = AESGCM(dek_plain).encrypt(nonce, plaintext, None)
    del dek_plain                        # ⭐ minimize time in memory

    return {"dek": dek_encrypted, "nonce": nonce, "ct": ct, "kid": key_id}
```

```
   ⭐ ENCRYPTION CONTEXT is underused and valuable. It's
     additional authenticated data bound to the key operation,
     so a DEK encrypted for tenant A cannot be decrypted while
     claiming to be tenant B — even by someone with KMS access.
```

### Searchable encrypted data

```
   ⚠️ THE FUNDAMENTAL TENSION: properly encrypted data is
     indistinguishable from random, so you cannot index or
     search it.

   OPTIONS, WITH HONEST TRADEOFFS
   ┌──────────────────────────────────────────────────────────────┐
   │ BLIND INDEX      Store HMAC(key, normalized_value) alongside │
   │                  the ciphertext. ⭐ Enables EXACT-MATCH       │
   │                  lookup only. ⚠️ Leaks equality — identical   │
   │                  values produce identical indexes.           │
   ├──────────────────────────────────────────────────────────────┤
   │ DETERMINISTIC    Same plaintext → same ciphertext.           │
   │ ENCRYPTION       ⚠️ Same leak, plus enables frequency         │
   │                  analysis. Use only where the leak is        │
   │                  acceptable.                                 │
   ├──────────────────────────────────────────────────────────────┤
   │ ⚠️ ORDER-PRESERVING  Enables range queries. ⚠️⚠️ Leaks a LOT   │
   │                  — often enough to recover plaintext.        │
   │                  Generally avoid.                            │
   ├──────────────────────────────────────────────────────────────┤
   │ ⚠️ HOMOMORPHIC    Compute on ciphertext. Mathematically       │
   │                  beautiful, still impractically slow for     │
   │                  most workloads.                             │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ THE PRAGMATIC ANSWER  Encrypt at the storage layer         │
   │                  (transparent database encryption), and      │
   │                  protect access with authorization rather    │
   │                  than trying to search ciphertext.           │
   └──────────────────────────────────────────────────────────────┘
```

### Signing webhooks and API requests

```python
# ⭐ Sign the RAW BODY plus a timestamp
def sign(secret: bytes, timestamp: int, raw_body: bytes) -> str:
    payload = f"{timestamp}.".encode() + raw_body
    return hmac.new(secret, payload, hashlib.sha256).hexdigest()

def verify(secret, header, raw_body) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    ts, sig = int(parts["t"]), parts["v1"]
    if abs(time.time() - ts) > 300:                    # ⭐ replay window
        return False
    expected = sign(secret, ts, raw_body)
    return hmac.compare_digest(expected, sig)          # ⭐ constant-time
```

```
   ⭐ THREE DETAILS THAT MATTER

   1. Sign the RAW BYTES, before any JSON parsing. Re-serializing
      changes whitespace and key order and breaks verification.
   2. Include a TIMESTAMP inside the signed payload, with a
      tolerance window — otherwise a captured request can be
      replayed forever.
   3. ⭐ compare_digest, not ==. A normal comparison returns
      early on the first differing byte, which leaks the correct
      signature one byte at a time.
```

---

## 13. Attacks Worth Knowing

```
   ⭐ TIMING ATTACK
     Execution time leaks secret data. String comparison
     returning early on mismatch lets an attacker recover a
     token byte by byte.
     ⭐ FIX: constant-time comparison for ALL secret comparisons.

   ⭐ PADDING ORACLE
     The system reveals whether decryption padding was valid —
     via an error message, a status code, or ⚠️ even a timing
     difference. That single bit of feedback is enough to
     decrypt everything.
     ⭐ FIX: AEAD. Authenticate before decrypting, and return
       an identical error for every failure.

   ⭐ REPLAY ATTACK
     Capture a valid message, send it again later.
     ⭐ FIX: nonces, timestamps with a window, or sequence numbers.

   ⭐ DOWNGRADE ATTACK
     Force negotiation to a weaker algorithm (FREAK, Logjam,
     POODLE).
     ⭐ FIX: remove legacy options entirely — which is exactly
       what TLS 1.3 did.

   ⭐ LENGTH EXTENSION
     hash(secret‖msg) can be extended without the secret.
     ⭐ FIX: HMAC.

   ⭐ NONCE REUSE
     Catastrophic in GCM and CTR. In GCM it leaks the auth key
     and permits forgery.
     ⭐ FIX: XChaCha20 with random 192-bit nonces, or rigorous
       counter discipline.

   ⚠️ SIDE CHANNELS
     Power analysis, cache timing, Spectre-class speculation
     leaks. Mostly the library author's problem — but a reason
     to prefer constant-time-by-design primitives like
     ChaCha20 and Ed25519.
```

---

## 14. Post-Quantum

```
   ⭐ THE THREAT — be precise about what breaks

   SHOR'S ALGORITHM breaks anything based on factoring or
   discrete logs:
     ⚠️ RSA · ECC · Diffie-Hellman → COMPLETELY BROKEN by a
       sufficiently large quantum computer

   GROVER'S ALGORITHM gives a quadratic speedup on brute-force
   search:
     AES-128 → effectively 64-bit security
     ⭐ AES-256 → effectively 128-bit. Still fine.
     ⭐ SHA-256 → still fine.

   ⭐ SO: symmetric crypto and hashes survive by doubling key
     sizes. ASYMMETRIC crypto needs entirely new mathematics.
```

```
   ⚠️⚠️ "HARVEST NOW, DECRYPT LATER" IS THE REASON THIS MATTERS
     TODAY, not in twenty years.

   An adversary recording encrypted traffic today can decrypt
   it once quantum computers arrive. ⭐ So any data with a long
   confidentiality lifetime — health records, state secrets,
   long-lived identity data — is ALREADY at risk.
```

```
   ⭐ THE NIST-STANDARDIZED ALGORITHMS (2024)

   ML-KEM (Kyber)        key encapsulation — replaces ECDH
   ML-DSA (Dilithium)    signatures — replaces ECDSA/Ed25519
   SLH-DSA (SPHINCS+)    hash-based signatures — ⭐ conservative
                         backup, relies only on hash security
   FN-DSA (Falcon)       compact signatures

   ⭐ WHAT'S HAPPENING NOW: HYBRID deployment. TLS is rolling
     out X25519+ML-KEM together, so the connection is secure
     if EITHER algorithm holds. Chrome and Cloudflare already
     do this by default.

   ⭐ WHAT TO DO TODAY
     • Use AES-256 and SHA-256+ (already quantum-resistant enough)
     • ⭐ Inventory where you use RSA/ECC and how long that data
       must stay confidential — that inventory IS the migration plan
     • Build CRYPTO AGILITY: version your formats, store
       algorithm identifiers, and make swapping primitives a
       configuration change rather than a rewrite
```

---

## 15. Interview Section

<details>
<summary><b>Q1. Why shouldn't you write your own crypto?</b></summary>

The usual answer is that primitives are extraordinarily hard to implement correctly — constant-time execution, side-channel resistance, correct handling of edge cases. All true, and it's why nobody should write their own AES.

But the more important point is that almost nobody does write their own AES. What people actually do is invent their own *protocol* — a custom way of combining primitives. Encrypt-then-sign versus sign-then-encrypt, a homemade token format, a bespoke key derivation scheme.

That's where real systems break. The primitives are solid; the composition is where the bugs live. Getting the order of operations wrong in authenticated encryption, or forgetting to authenticate the associated data, or reusing a nonce across replicas — none of those require breaking AES.

So the practical rule is to use a high-level, misuse-resistant library — libsodium, Tink, Fernet — where the API doesn't offer you the footguns. Or a well-established protocol like TLS or the Signal Protocol.

And crypto failures are silent. Bad encryption produces output that looks exactly like good encryption. There's no test that fails, no error, no symptom — until someone reads your data.
</details>

<details>
<summary><b>Q2. Explain forward secrecy.</b></summary>

Forward secrecy means compromising a long-term private key does not let an attacker decrypt past sessions.

It comes from ephemeral Diffie-Hellman. Both parties generate a fresh keypair per session, derive a shared secret, and discard the ephemeral keys afterward. The session key never existed anywhere except memory and is gone.

The consequence is what matters. Without it — with static RSA key exchange — an attacker who records your traffic today and obtains the server's private key in five years can decrypt every recorded session retroactively. With forward secrecy, that key is useless against past traffic.

That's why TLS 1.3 removed non-forward-secret key exchange entirely rather than making it configurable. It's now mandatory.

The related nuance is that Diffie-Hellman alone is vulnerable to a man-in-the-middle — it gives you a shared secret with *somebody*, not with a verified identity. So the exchange must be authenticated, which is what the server's certificate and the CertificateVerify signature provide.
</details>

<details>
<summary><b>Q3. Encryption vs hashing vs encoding?</b></summary>

Encoding is not security at all — base64 and URL encoding are reversible by anyone and exist for transport compatibility. Base64 appearing in a security discussion is usually a red flag.

Hashing is one-way. You can verify a value matches without storing the value, which is why it's used for passwords and integrity checking. It has no key and cannot be reversed.

Encryption is two-way with a key. You encrypt to keep something confidential and decrypt it later.

The decision rule: if you need the original value back, encrypt. If you only ever need to check whether a supplied value matches, hash.

Two important refinements. Encryption alone doesn't provide integrity — with CBC or CTR mode an attacker can flip bits in the ciphertext and predictably flip bits in the plaintext without the key, which is why AEAD modes exist and why you should always use authenticated encryption.

And general-purpose hashes are wrong for passwords. SHA-256 is designed to be fast; a GPU does billions per second. Passwords need a slow, memory-hard KDF like Argon2id, where the memory requirement is what defeats parallel hardware.
</details>

<details>
<summary><b>Q4. What is AEAD and why does it matter?</b></summary>

Authenticated Encryption with Associated Data — encryption and integrity protection in a single primitive. AES-GCM and ChaCha20-Poly1305 are the common ones.

It matters because encryption without authentication is genuinely dangerous, and the danger isn't obvious. With CBC or CTR mode, an attacker who can modify ciphertext can make predictable changes to the plaintext without knowing the key. And systems that reveal whether decryption succeeded — through an error message, a status code, or even a timing difference — leak enough information for a padding oracle attack to decrypt everything.

AEAD prevents both by verifying the authentication tag before decrypting, so tampered ciphertext is rejected outright.

The associated data part is useful and underused: data that's authenticated but not encrypted. Routing headers, record IDs, version numbers — things that must not be tampered with but don't need to be hidden.

The critical operational rule is nonce uniqueness. Reusing a nonce with GCM doesn't just leak two plaintexts — it leaks the authentication subkey, letting an attacker forge arbitrary messages. That's a total break. XChaCha20-Poly1305 with its 192-bit nonce lets you use random nonces safely and removes the entire class of bug, which is why I'd prefer it where available.
</details>

<details>
<summary><b>Q5. Walk me through a TLS handshake.</b></summary>

In TLS 1.3 it's one round trip. The client sends a ClientHello with supported cipher suites and — crucially — a key share, meaning it guesses the likely group and already includes its ephemeral Diffie-Hellman public key.

The server responds with its own key share, so both sides can immediately derive the shared secret, and the rest of the server's flight is already encrypted: extensions, its certificate, a CertificateVerify, and Finished.

CertificateVerify is the step worth calling out. The server signs the handshake transcript with the private key corresponding to its certificate. Without it, anyone could present someone else's certificate — possession of the private key is what actually authenticates the server.

The client validates the certificate chain against its trust store, checks the hostname, checks validity dates, sends its own Finished, and can send application data immediately.

TLS 1.3's improvements came mostly from removal — no RC4, no 3DES, no CBC modes, no static RSA, no MD5 or SHA-1 signatures. Every legacy option was a downgrade attack waiting to happen; FREAK, Logjam, and POODLE were all "negotiate down to something broken" attacks. The security win came from eliminating choices rather than adding features.

One caveat: 0-RTT resumption data is replayable, since there's no round trip to prove freshness. It should only carry idempotent requests.
</details>

<details>
<summary><b>Q6. How would you encrypt data at rest in a database?</b></summary>

Envelope encryption with a cloud KMS. A master key lives in the KMS or an HSM and never leaves it. For each record or tenant, you ask the KMS to generate a data encryption key, which it returns both in plaintext and encrypted under the master key. You encrypt the data locally with the DEK at full speed, store the encrypted DEK alongside the ciphertext, and discard the plaintext DEK from memory.

Two properties make this the standard pattern. Only small keys cross the KMS boundary, so bulk data encryption isn't rate-limited by an external service. And rotating the master key only requires re-encrypting the DEKs, not re-encrypting petabytes of data.

I'd use per-tenant DEKs for cryptographic isolation, and pass an encryption context — additional authenticated data binding the key operation to a tenant and purpose — so a DEK for tenant A can't be decrypted while claiming to be tenant B.

Critically, I'd store a key ID with every ciphertext from day one. Retrofitting rotation into a system that assumed one eternal key is genuinely painful, and the field costs nothing.

The thing to be honest about is that this protects against stolen backups and disk theft, not against a compromised application — the application necessarily has decryption access. For that you need authorization and audit logging, not more encryption.
</details>

<details>
<summary><b>Q7. What's a timing attack?</b></summary>

An attack where execution time leaks information about secret data.

The canonical example is comparing a secret with a normal string comparison. Most implementations return as soon as they find a differing byte, so a wrong first byte returns faster than a wrong tenth byte. An attacker measures the response time, determines the first byte, then the second, and recovers the whole token in linear rather than exponential time.

The fix is constant-time comparison — `hmac.compare_digest` in Python, `crypto.timingSafeEqual` in Node — which always examines every byte.

This applies to any secret comparison: API keys, HMAC signatures, session tokens, password hash outputs.

A related case worth mentioning is user enumeration in login. If you skip password hashing when the user doesn't exist, the response is measurably faster, so an attacker can enumerate valid accounts even with identical error messages. The fix is verifying against a dummy hash so the work is the same either way.

Over a network, jitter makes timing attacks harder but not impossible — researchers have demonstrated them across the internet with enough samples. It's not a theoretical concern.
</details>

<details>
<summary><b>Q8. Should we be worried about quantum computers?</b></summary>

For asymmetric cryptography, yes — with a specific timeline consideration that makes it relevant now rather than later.

Shor's algorithm completely breaks RSA, ECC, and Diffie-Hellman. Grover's gives only a quadratic speedup on brute-force search, so AES-256 remains effectively 128-bit secure and SHA-256 is fine. Symmetric crypto and hashes survive by doubling key sizes; asymmetric needs entirely new mathematics.

The reason it matters today is "harvest now, decrypt later." An adversary recording encrypted traffic today can decrypt it once quantum computers arrive. So any data with a long confidentiality lifetime — health records, state secrets, long-lived identity data — is already at risk regardless of when the hardware exists.

NIST standardized replacements in 2024: ML-KEM for key exchange, ML-DSA for signatures, with SPHINCS+ as a conservative hash-based backup.

What's happening in practice is hybrid deployment — TLS rolling out X25519 combined with ML-KEM, so the connection is secure if either algorithm holds. Chrome and Cloudflare already do this by default.

Practically, I'd focus on crypto agility: version your formats, store algorithm identifiers with ciphertexts, and make swapping primitives a configuration change rather than a rewrite. The organizations that struggle with this migration will be the ones that hardcoded an algorithm choice a decade ago.
</details>

---

## 16. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                    CRYPTOGRAPHY — ONE PAGE                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ Don't roll your own crypto — AND DON'T ROLL YOUR OWN PROTOCOL.    ║
║   The primitives are fine; COMPOSITION is where systems break.       ║
║   Use libsodium / Tink / Fernet — misuse-resistant by design.        ║
╠══════════════════════════════════════════════════════════════════════╣
║ 4 PROPERTIES: confidentiality · integrity · authenticity ·           ║
║   ⭐ non-repudiation (SIGNATURE only — a MAC can't give it)           ║
║ ⚠️⚠️ ENCRYPTION ALONE ≠ INTEGRITY → always use AEAD                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ RANDOM: secrets/crypto.randomBytes — ⚠️ NEVER random/Math.random     ║
║ HASH: SHA-256/BLAKE3 ✅ · ⚠️ MD5/SHA-1 BROKEN                         ║
║   ⚠️ hashes are for INTEGRITY. Passwords need Argon2id (MEMORY-HARD  ║
║     is what defeats GPUs)                                            ║
║   ⭐ HMAC not hash(secret+msg) — length extension attack              ║
╠══════════════════════════════════════════════════════════════════════╣
║ SYMMETRIC: ⭐ AES-GCM or ChaCha20-Poly1305 (AEAD)                     ║
║   ⚠️⚠️ ECB NEVER · ⚠️⚠️ NONCE REUSE IN GCM = TOTAL BREAK (leaks the    ║
║     auth key → forgery). ⭐ XChaCha20's 192-bit nonce → random is safe║
║ ASYMMETRIC: ~1000× slower → ⭐ ALWAYS HYBRID (sym key for data,       ║
║   asym for the key). X25519 exchange · Ed25519 signatures            ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ FORWARD SECRECY (ECDHE): stealing the long-term key does NOT       ║
║   decrypt past traffic. Mandatory in TLS 1.3.                        ║
║ ⚠️ Raw DH is MITM-able → must be authenticated (that's what certs do)║
║ TLS 1.3: 1-RTT · CertificateVerify PROVES key possession ·           ║
║   ⭐ its biggest win was REMOVING legacy options (downgrade attacks)  ║
║   ⚠️ 0-RTT is REPLAYABLE → idempotent requests only                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ PKI: ⚠️ missing INTERMEDIATE = "works in Chrome, fails from curl"     ║
║   revocation is PKI's weak point → ⭐ short-lived certs + automation  ║
║   ⭐ monitor Certificate Transparency logs for your domains           ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ ENVELOPE ENCRYPTION: master key in KMS → DEK per record → data     ║
║   rotating the master only re-encrypts DEKs, not the data            ║
║   ⭐ STORE A KEY ID with every ciphertext from day one                ║
╠══════════════════════════════════════════════════════════════════════╣
║ ATTACKS: ⭐ timing (→ constant-time compare) · padding oracle (→AEAD) ║
║   replay (→ timestamp+window) · downgrade (→ remove legacy) ·        ║
║   length extension (→HMAC) · nonce reuse                             ║
║ SIGN WEBHOOKS: HMAC over timestamp + ⭐ RAW BYTES, constant-time cmp  ║
╠══════════════════════════════════════════════════════════════════════╣
║ QUANTUM: Shor BREAKS RSA/ECC/DH · Grover only halves symmetric →     ║
║   ⭐ AES-256 and SHA-256 are fine                                     ║
║   ⚠️ "harvest now, decrypt later" makes long-lived secrets at risk    ║
║     TODAY → hybrid X25519+ML-KEM, and build CRYPTO AGILITY           ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Network Security →](network-security.md) · **Related:** [AppSec](appsec.md) · [Linux & Networking](../06-cloud-devops/linux-networking.md)
