# Changelog

## 0.1.0
- ML-KEM-768 + HKDF-SHA256 + AES-256-GCM.
- Versioned wire format; header and KEM ciphertext authenticated.
- `pqkyber-aes` command-line tool (keygen, encrypt, decrypt).
- Test suite covering tamper detection across the whole envelope.
