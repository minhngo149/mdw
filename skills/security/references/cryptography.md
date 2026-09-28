# Cryptography

Security-sensitive cryptographic usage: hashing, encryption, signing, randomness, and the token and
password specifics that depend on them. Extends step 11 of `SKILL.md`. Be precise: report concrete
insecure implementations, not generic algorithm preferences.

## Contents

1. Randomness
2. Password hashing
3. Encryption
4. Signing and MAC
5. Token security
6. What to report

## 1. Randomness

The most common concrete, high-impact crypto bug: a **security token generated from a non-crypto
PRNG**.

| Use | Safe source | Unsafe source |
|---|---|---|
| Session id, reset/verification token, API key, nonce, salt, password | Go `crypto/rand`; Node `crypto.randomBytes`/`randomUUID`; Python `secrets`/`os.urandom`; Java `SecureRandom` | Go `math/rand`; Node `Math.random`; Python `random`; Java `java.util.Random`; any PRNG seeded from time |

Red flags: `math/rand`, `rand.Seed(time.Now()...)`, `Math.random()`, `random.choice`, and any token
built by indexing an alphabet with a non-crypto PRNG. A token from `math/rand` seeded with the
current time is **predictable**: an attacker who knows the approximate generation time can reproduce
the sequence. This is a real finding (`HIGH` for reset/session tokens), not a style nit — but note
the exploit requires reproducing the seed/state, so complexity is moderate, not trivial.

Also check **entropy length**: even from a CSPRNG, a token needs enough bytes (≥128 bits) to resist
guessing. Note short tokens.

UUIDs: v4 from a CSPRNG is fine as an identifier; a UUID is **not** a secret by construction (v1 is
time/MAC-based and predictable). Do not treat a UUID as a security token unless it is CSPRNG-random
and long enough.

## 2. Password hashing

| Acceptable | Not acceptable for passwords |
|---|---|
| bcrypt, scrypt, argon2 (argon2id), PBKDF2 with a high iteration count | MD5, SHA-1, SHA-256/512 alone (fast, brute-forceable), unsalted hashes, reversible encoding, plaintext |

Check the **cost/work factor** (bcrypt cost, argon2 memory/time, PBKDF2 iterations) is reasonable, a
per-password salt is used (built into bcrypt/argon2), and comparison uses the library's verify
function (constant-time). Report a concrete weak choice (e.g. `sha256(password)`), not "consider a
stronger KDF" when bcrypt/argon2 is already used correctly.

## 3. Encryption

| Check | Concern |
|---|---|
| Algorithm and mode | ECB reveals patterns; unauthenticated CBC is malleable. Prefer authenticated modes (AES-GCM, ChaCha20-Poly1305). |
| IV / nonce | must be unique per encryption; a fixed or reused nonce breaks confidentiality/integrity. A hardcoded IV is a concrete finding. |
| Key management | key source and storage (`secrets.md`); hardcoded keys are findings. |
| Integrity | is ciphertext authenticated (AEAD or encrypt-then-MAC)? |
| Custom crypto | hand-rolled encryption/"encoding" (XOR, base64-as-"encryption") is a finding. |

## 4. Signing and MAC

- **HMAC** for webhooks and tokens: correct construction, secret strength (`secrets.md`),
  **constant-time comparison** of the computed vs supplied MAC (a normal `==`/`!=` on the digest is a
  timing side channel — `LOW`, but note it), and signing the exact bytes.
- **Asymmetric signatures** (JWT RS/ES, request signing): verification actually performed, correct
  key used, algorithm handling (`authentication.md`).

## 5. Token security

Bring together randomness (section 1) and lifecycle for every token type:

```text
access token   refresh token   password-reset token   email-verification token
API token   webhook signature
```

For each, check: entropy source and length, expiration, storage (hashed at rest for reset/API
tokens; a token stored in plaintext in the DB is exposed if the DB is), rotation, revocation,
single-use where required (reset, verification), audience/scope, and signature verification. A
reset token that is predictable, long-lived, reusable, stored in plaintext, or logged is a chain of
weaknesses — report the ones present, and combine them in the impact.

## 6. What to report

| Finding | Typical severity |
|---|---|
| Security token from a non-crypto PRNG / time-seeded RNG | `HIGH` (reset/session), moderate complexity |
| Passwords hashed with a fast/unsalted hash or stored plaintext | `HIGH`/`CRITICAL` |
| Hardcoded encryption/signing key or IV | `HIGH` (with `secrets.md`) |
| ECB mode / unauthenticated encryption of sensitive data | `MEDIUM`/`HIGH` |
| Hand-rolled "encryption" | `MEDIUM`/`HIGH` |
| Non-constant-time MAC/secret comparison | `LOW` |
| Short token entropy | `LOW`/`MEDIUM` |

Verify safely in unit tests: assert a token generator uses the crypto source (or show it does not),
check hash cost, check comparison is constant-time. Never weaken crypto in a shared or production
environment to test.
