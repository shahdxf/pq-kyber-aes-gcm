# Design notes (AES-256-GCM)

## Construction

```
(shared_secret, kem_ct) = ML-KEM-768.Encaps(public_key)
key   = HKDF-SHA256(ikm = shared_secret, salt = none,
                    info = "pq-kyber-aes-gcm/v1" | "aes-256-gcm" | kem_ct, length = 32)
nonce = 12 random bytes
aad'  = header(magic|version|cipher_id) | kem_ct | caller_aad
ct    = AES-256-GCM(key).encrypt(nonce, plaintext, aad')
```

This is the standard KEM/DEM pattern. The KEM establishes a fresh key per message; the AEAD protects the data.

## Why bind the cipher and the KEM ciphertext?

ML-KEM already provides IND-CCA2 security, so binding the KEM ciphertext is not strictly required for the KEM's own
guarantees. It is cheap defence in depth: the derived key is specific to one encapsulation and one cipher, and the
authenticated header stops an attacker from altering the version or cipher id without detection.

## Why a fresh key per message makes random nonces safe

Random 96-bit nonces become risky only when many messages share one key (birthday bound around 2^32 messages).
Here every message uses a new encapsulation, hence a new key, so each key sees exactly one nonce.

## Why AES-256-GCM?

AES-GCM is the most widely deployed AEAD and is very fast on CPUs with AES instructions (most laptops, phones and servers). Its weakness is that fast, constant-time AES in plain software needs hardware support; on hardware without it, implementations are slower or harder to make side-channel safe.

AES-256-GCM is a modern AEAD with a 256-bit key, 96-bit nonce and 128-bit tag, and has no known practical weakness in this use.

## Errors

`decrypt` raises one generic `DecryptionError("decryption failed")` for every failure after parsing (bad tag, wrong key,
wrong AAD, malformed KEM data). Distinct messages would leak which check failed. Format problems found while parsing
(bad magic, unknown version, other cipher id) raise `FormatError`, which is not secret-dependent.

## What this does not do

- **Sender authentication.** Encryption to a public key proves nothing about who sent the message. A real system adds
  signatures (for example ML-DSA) or an authenticated key exchange.
- **Forward secrecy.** The recipient key is long-lived. Ratcheting would add per-message ephemeral keys.
- **Classical hybrid.** A deployment worried about ML-KEM's youth would combine it with X25519 and feed both secrets to HKDF.
- **Side-channel resistance.** `kyber-py` is pure Python; use a native, constant-time ML-KEM for real systems.
