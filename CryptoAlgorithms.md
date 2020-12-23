Crypto Algorithms
=================

## Cryptographic primitives

### Password-based key derivation function (PKDF)

Let `MasterKey` be the Argon2id result of the password with

* output length: 32 bytes
* iterations: 20
* memory usage: 64 MiB
* salt: `00000000000000000000000000000000` (16 bytes)

and derive the subkeys with

* algorithm: Blake2b
* output length: 32 bytes
* message: _empty string_
* key: the `MasterKey`
* personalization: `ascii("CHATv001")` plus 8 bytes of zero-padding to the right (16 bytes)
* salt: `??000000000000000000000000000000` (16 bytes), where `??` is one subkey-dependent byte

Subkey salt values
* LoginKey: `0x01`
* PasswordVerificationKey: `0x02`
* EncryptionPrivkeyEncryptingKey: `0x03`

### Cryptographic hash

* BLAKE2b

Known use cases: symmetric key to ID.

### Fingerprints

* A BLAKE2b-128 hash, hex encoded

### Symmetric authenticated encryption

* IETF ChaCha20-Poly1305

The algorithm identifier "chacha20poly1305-ietf-nonce12prefixed" means
ChaCha20-Poly1305 (IETF variant) with a 12 bytes nonce that is prefixed
to the ciphertext.

### Signatures

* Ed25519

### Public-key encryption

* libsodium's sealed boxes

## Signatures

Signatures are generated using Devices' Keypairs. They are stored as a string in the form `<deviceId>,base64(<rawSignature>)`.

### Device idOwnerIdSignature

```
rawSignature = ed25519(utf8(
    id + "|" + ownerId
))
```

### ConversationPermission signature

```
rawSignature = ed25519(utf8(
    conversationId + "|" +
    conversationKeyId + "|" +
    base64(conversationKey) + "|" +
    ownerId + "|" +
    creatorId + "|" +
    validFrom
))
```

### Authentication LoginKey signature

In this case, the message to be signed is not constructed from a string but from binary data directly.

```
rawSignature = ed25519(loginKey)
```
