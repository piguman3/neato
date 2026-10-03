# NEATO `crypto` API Specification

Extension: `core`

Version: 2

---

A NEATO environment must provide the global `crypto` table exactly as provided by the NEET
Computers API. These functions only compute on data and do not touch the "hardware" of the computer, so they do not
need any protection layer.

`crypto` keeps the NEET Computers API unchanged, including how it reports its own failures, so it is the one part of
`core` that does not use the codes in [errors.md](../common/errors.md).

```c
crypto
  AES
    Encrypt
    GenerateKeyFromPassword
    GenerateSalt
    GenerateKey
    GenerateIv
    Decrypt
  Base64
    Decode
    Encode
  SecureRNG
    GetRandomFromMin
    GetRandomUpTo
    GetRandom
    GetRandomBetween
  RSA
    Encrypt
    Decrypt
    Verify
    Sign
    GenerateKeyPair
  Hash
    MD5
    SHA256
```
