# ML-DSA Binding Technique

This document describes how ML-DSA (Module-Lattice Digital Signature Algorithm) is bound to machine learning models in the SelfVerifier framework to prevent pickle deserialization attacks.

## Overview

The SelfVerifier framework implements cryptographic binding between ML models and their signatures using ML-DSA-44, a post-quantum digital signature algorithm. This binding ensures that any modification to a signed model—including injection of malicious pickle payloads—is detected before deserialization.

## ML-DSA-44 Parameters

| Parameter | Size |
|-----------|------|
| Public Key | 1,312 bytes |
| Secret Key | 2,560 bytes |
| Signature | 2,420 bytes |

The implementation uses the PQClean reference implementation via ctypes bindings (`libmldsa44.so`).

---

## ctypes Binding to C Library

The `mldsa44_binding.py` module provides Python bindings to the ML-DSA-44 C library using Python's `ctypes` foreign function interface.

### Loading the Shared Library

```python
import ctypes
from ctypes import c_uint8, c_size_t, POINTER

# Locate the shared library relative to the module
LIB_PATH = os.path.join(os.path.dirname(__file__), 'libmldsa44.so')

# Load the C library
_mldsa = ctypes.CDLL(LIB_PATH)
```

The `CDLL` loader maps the shared library into the Python process, making its exported functions callable.

### Defining Constants

Constants from the C header (`api.h`) are replicated in Python:

```python
CRYPTO_PUBLICKEYBYTES = 1312   # Public key size
CRYPTO_SECRETKEYBYTES = 2560   # Secret key size
CRYPTO_BYTES = 2420            # Signature size
```

### Function Prototype Declarations

Each C function requires explicit type declarations for arguments and return values:

#### Key Generation

```c
// C function signature
int PQCLEAN_MLDSA44_CLEAN_crypto_sign_keypair(uint8_t *pk, uint8_t *sk);
```

```python
# Python ctypes declaration
_mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_keypair.argtypes = [
    POINTER(c_uint8),  # pk - output public key buffer
    POINTER(c_uint8)   # sk - output secret key buffer
]
_mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_keypair.restype = ctypes.c_int
```

#### Signature Creation

```c
// C function signature
int PQCLEAN_MLDSA44_CLEAN_crypto_sign_signature(
    uint8_t *sig, size_t *siglen,
    const uint8_t *m, size_t mlen,
    const uint8_t *sk
);
```

```python
# Python ctypes declaration
_mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_signature.argtypes = [
    POINTER(c_uint8),      # sig    - output signature buffer
    POINTER(c_size_t),     # siglen - output signature length
    POINTER(c_uint8),      # m      - input message
    c_size_t,              # mlen   - message length
    POINTER(c_uint8)       # sk     - secret key
]
_mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_signature.restype = ctypes.c_int
```

#### Signature Verification

```c
// C function signature
int PQCLEAN_MLDSA44_CLEAN_crypto_sign_verify(
    const uint8_t *sig, size_t siglen,
    const uint8_t *m, size_t mlen,
    const uint8_t *pk
);
```

```python
# Python ctypes declaration
_mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_verify.argtypes = [
    POINTER(c_uint8),      # sig    - signature to verify
    c_size_t,              # siglen - signature length
    POINTER(c_uint8),      # m      - original message
    c_size_t,              # mlen   - message length
    POINTER(c_uint8)       # pk     - public key
]
_mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_verify.restype = ctypes.c_int
```

### Python Wrapper Functions

#### `keypair()` - Key Generation

```python
def keypair():
    """Generate ML-DSA-44 keypair."""
    # Allocate fixed-size buffers for output
    pk = (c_uint8 * CRYPTO_PUBLICKEYBYTES)()  # 1312 bytes
    sk = (c_uint8 * CRYPTO_SECRETKEYBYTES)()  # 2560 bytes

    # Call C function
    ret = _mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_keypair(pk, sk)

    if ret != 0:
        raise RuntimeError("Key generation failed")

    # Convert ctypes arrays to Python bytes
    return bytes(pk), bytes(sk)
```

**Key Techniques:**
- `(c_uint8 * N)()` creates a fixed-size C array of N bytes
- Arrays are passed by reference automatically (pointer decay)
- `bytes(array)` converts ctypes array to Python bytes

#### `sign(message, secret_key)` - Signature Creation

```python
def sign(message, secret_key):
    """Sign a message with ML-DSA-44."""
    # Validate input
    if len(secret_key) != CRYPTO_SECRETKEYBYTES:
        raise ValueError(f"Invalid secret key size: {len(secret_key)}")

    # Allocate output buffers
    sig = (c_uint8 * CRYPTO_BYTES)()   # Max signature size
    siglen = c_size_t()                 # Actual signature length

    # Convert Python bytes to ctypes arrays
    msg_array = (c_uint8 * len(message)).from_buffer_copy(message)
    sk_array = (c_uint8 * CRYPTO_SECRETKEYBYTES).from_buffer_copy(secret_key)

    # Call C function
    ret = _mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_signature(
        sig,
        ctypes.byref(siglen),  # Pass pointer to siglen
        msg_array,
        len(message),
        sk_array
    )

    if ret != 0:
        raise RuntimeError("Signing failed")

    # Return only the actual signature bytes
    return bytes(sig[:siglen.value])
```

**Key Techniques:**
- `from_buffer_copy()` copies Python bytes into a new ctypes array
- `ctypes.byref()` creates a pointer to a ctypes object (for output parameters)
- `siglen.value` accesses the actual value from `c_size_t`
- Slicing `sig[:siglen.value]` extracts only the valid signature bytes

#### `verify(signature, message, public_key)` - Signature Verification

```python
def verify(signature, message, public_key):
    """Verify an ML-DSA-44 signature."""
    # Validate input
    if len(public_key) != CRYPTO_PUBLICKEYBYTES:
        raise ValueError(f"Invalid public key size: {len(public_key)}")

    # Convert Python bytes to ctypes arrays
    sig_array = (c_uint8 * len(signature)).from_buffer_copy(signature)
    msg_array = (c_uint8 * len(message)).from_buffer_copy(message)
    pk_array = (c_uint8 * CRYPTO_PUBLICKEYBYTES).from_buffer_copy(public_key)

    # Call C function
    ret = _mldsa.PQCLEAN_MLDSA44_CLEAN_crypto_sign_verify(
        sig_array,
        len(signature),
        msg_array,
        len(message),
        pk_array
    )

    # Return value: 0 = valid, non-zero = invalid
    return ret == 0
```

**Key Techniques:**
- Variable-length arrays: `(c_uint8 * len(data))` creates array sized to input
- Boolean conversion: C returns 0 for success, Python returns `True`

### Memory Management

```
Python bytes ──────────────────────────────────────────────► Python bytes
      │                                                            ▲
      │ from_buffer_copy()                              bytes(array)
      ▼                                                            │
ctypes array ──► C function ──► ctypes array (modified in place) ──┘
```

1. **Input Conversion**: `from_buffer_copy()` allocates new C memory and copies Python bytes into it
2. **C Function Call**: C code operates on ctypes arrays directly
3. **Output Conversion**: `bytes()` copies ctypes array back to Python bytes

This ensures:
- Python's immutable bytes are not violated
- C function has writable memory for output
- No memory leaks (ctypes manages allocation/deallocation)

### Error Handling

```python
try:
    _mldsa = ctypes.CDLL(LIB_PATH)
except OSError as e:
    raise RuntimeError(f"Failed to load ML-DSA library: {e}")
```

Common failure cases:
- Library file not found
- Architecture mismatch (32-bit vs 64-bit)
- Missing dependencies (libc, etc.)

### Complete Data Flow Example

```
┌─────────────────────────────────────────────────────────────────┐
│                        Python Layer                              │
├─────────────────────────────────────────────────────────────────┤
│  message = b"Hello"           # Python bytes                     │
│  secret_key = bytes(2560)     # Python bytes                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ msg_array = (c_uint8 * 5).from_buffer_copy(message)     │    │
│  │ sk_array = (c_uint8 * 2560).from_buffer_copy(secret_key)│    │
│  │ sig = (c_uint8 * 2420)()                                │    │
│  │ siglen = c_size_t()                                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼ ctypes FFI call                   │
├─────────────────────────────────────────────────────────────────┤
│                         C Layer                                  │
├─────────────────────────────────────────────────────────────────┤
│  PQCLEAN_MLDSA44_CLEAN_crypto_sign_signature(                   │
│      sig,          // uint8_t* - writes 2420 bytes              │
│      &siglen,      // size_t*  - writes actual length           │
│      msg_array,    // uint8_t* - reads 5 bytes                  │
│      5,            // size_t   - message length                 │
│      sk_array      // uint8_t* - reads 2560 bytes               │
│  );                                                              │
│                              │                                   │
│                              ▼ return 0 (success)                │
├─────────────────────────────────────────────────────────────────┤
│                        Python Layer                              │
├─────────────────────────────────────────────────────────────────┤
│  signature = bytes(sig[:siglen.value])  # Python bytes (2420)   │
└─────────────────────────────────────────────────────────────────┘
```

## Binding Architectures

The framework implements two distinct binding approaches:

### Approach A: Self-Verifying Model (Internal Binding)

This approach embeds the signature and public key inside the pickle structure itself, leveraging Python's `__reduce__()` protocol to trigger automatic verification during deserialization.

#### Structure

```
SelfVerifier Object
├── model_data_bytes: bytes    # Serialized original model
├── public_key: bytes          # ML-DSA-44 public key (1,312 bytes)
└── signature: bytes           # ML-DSA-44 signature (2,420 bytes)
```

#### Signing Process

```
┌─────────────────┐
│  Original Model │
└────────┬────────┘
         │ pickle.dumps(model, protocol=4)
         ▼
┌─────────────────┐
│  model_bytes    │
└────────┬────────┘
         │ SHA-256(model_bytes)
         ▼
┌─────────────────┐
│   model_hash    │
└────────┬────────┘
         │ mldsa_sign(model_hash, secret_key)
         ▼
┌─────────────────┐
│   signature     │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  SelfVerifier(model_bytes,          │
│               public_key,           │
│               signature)            │
└────────┬────────────────────────────┘
         │ torch.save(self_verifier, path)
         ▼
┌─────────────────┐
│  Signed Model   │
│     (.pt)       │
└─────────────────┘
```

#### Verification Process (Automatic on Load)

When `torch.load()` is called, pickle invokes the `__reduce__()` method which returns a callable (`_verify_and_restore`) and its arguments:

```python
def __reduce__(self):
    return (_verify_and_restore, (self.model_data_bytes,
                                   self.public_key,
                                   self.signature))
```

The `_verify_and_restore` function:

1. Computes `SHA-256(model_data_bytes)`
2. Calls `mldsa_verify(signature, computed_hash, public_key)`
3. If verification succeeds: returns the `SelfVerifier` object
4. If verification fails: raises `ValueError` and blocks loading

#### Security Property

The returned `SelfVerifier` object retains the original signature and public key. If an attacker modifies and re-saves the model, the next load will fail verification because the new data won't match the original signature.

---

### Approach B: Length-Prefix Format (External Binding)

This approach uses a custom binary file format with explicit length fields, ensuring cryptographic boundary definition for the signed region.

#### File Format

```
Offset    Size      Field
──────────────────────────────────────────────────
0x00      8 bytes   Magic Header: b'MLDSASIG'
0x08      1 byte    Format Version: 0x04
0x09      8 bytes   signed_region_length (big-endian uint64)
0x11      N bytes   signed_region_bytes ← HASH TARGET
0x11+N    8 bytes   signature_length (big-endian uint64)
0x19+N    M bytes   signature
0x19+N+M  8 bytes   public_key_length (big-endian uint64)
0x21+N+M  K bytes   public_key
```

#### Signed Region Structure

The signed region is a pickled dictionary containing:

```python
{
    'model_data': <original_model_data>,
    'version': '3.0-secure',
    'timestamp': <creation_time>,
    'format': 'length-prefix',
    'hash_algorithm': 'SHA-256',
    'signature_algorithm': 'ML-DSA-44'
}
```

#### Signing Process

```
┌─────────────────┐
│    Model Data   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────┐
│  signed_region = {              │
│    'model_data': model_data,    │
│    'version': '3.0-secure',     │
│    'timestamp': time.time(),    │
│    ...                          │
│  }                              │
└────────┬────────────────────────┘
         │ pickle.dumps(signed_region, protocol=4)
         ▼
┌─────────────────┐
│  signed_bytes   │
└────────┬────────┘
         │ SHA-256(signed_bytes)
         ▼
┌─────────────────┐
│   hash_value    │
└────────┬────────┘
         │ mldsa_sign(hash_value, secret_key)
         ▼
┌─────────────────┐
│   signature     │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Write binary file:                 │
│  [HEADER][VERSION][LEN][DATA]       │
│  [SIG_LEN][SIG][PK_LEN][PK]         │
└─────────────────────────────────────┘
```

#### Verification Process (Before Deserialization)

Critical security feature: verification happens **before** pickle deserialization.

```
┌─────────────────┐
│  Read file      │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  1. Validate magic header           │
│  2. Check format version            │
│  3. Read signed_region_length       │
│  4. Read exactly N bytes            │
│  5. Read signature                  │
│  6. Read public_key                 │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Compute SHA-256(signed_region)     │
│  Verify signature with public_key   │
└────────┬────────────────────────────┘
         │
    ┌────┴────┐
    │ Valid?  │
    └────┬────┘
    YES  │  NO
    ▼    │  ▼
┌────────┐ ┌────────────────┐
│Deserial│ │ Raise Exception│
│  ize   │ │ BLOCK LOADING  │
└────────┘ └────────────────┘
```

#### Security Property

The length-prefix format prevents append attacks. An attacker cannot add malicious data beyond the signed region because:
1. The exact byte count is specified by `signed_region_length`
2. Only those exact bytes are hashed and verified
3. Any appended data is outside the verified boundary and ignored

---

## Binding Comparison

| Aspect | Self-Verifying (Internal) | Length-Prefix (External) |
|--------|---------------------------|--------------------------|
| Verification trigger | Automatic via `__reduce__()` | Explicit binary parsing |
| Verification timing | During deserialization | Before deserialization |
| Format | Standard pickle | Custom binary format |
| Key location | Inside pickle object | At file end |
| Append attack protection | Via hash mismatch | Via length boundary |
| Compatibility | Any pickle loader | Requires custom loader |

## Hash-then-Sign Binding

Both approaches use the hash-then-sign paradigm:

```
Binding = Sign(Hash(Data), SecretKey)
```

Where:
- **Hash**: SHA-256 (256-bit output)
- **Sign**: ML-DSA-44 signature algorithm
- **Data**: Serialized model bytes

This binds the signature to the exact byte representation of the model. Any modification—even a single bit change—produces a different hash, causing signature verification to fail.

## Attack Mitigation

### Pickle RCE Attack Vector

Standard pickle deserialization is vulnerable to arbitrary code execution:

```python
class MaliciousPayload:
    def __reduce__(self):
        return (os.system, ('malicious_command',))
```

An attacker can append this payload to a model file. When loaded with `torch.load()`, the malicious code executes.

### How ML-DSA Binding Prevents This

1. **Signature covers entire model data**: Any appended or modified content changes the hash
2. **Verification before/during deserialization**: Attack payload is detected before execution
3. **Cryptographic guarantee**: Without the secret key, attackers cannot forge valid signatures

```
Original Model Hash:  a1b2c3d4...
Modified Model Hash:  x9y8z7w6...  ← Different!

Signature was created for: a1b2c3d4...
Verification fails for:    x9y8z7w6...

Result: Loading blocked, attack prevented
```

## Key Management

### Key Generation

```python
public_key, secret_key = mldsa44_binding.keypair()
```

### Key Storage

- `ml_dsa_secret.key`: 2,560 bytes (PROTECT THIS - used for signing)
- `ml_dsa_public.key`: 1,312 bytes (can be distributed - used for verification)

### Security Considerations

1. **Secret key protection**: The secret key must be kept confidential. Anyone with the secret key can sign malicious models.
2. **Public key distribution**: The public key can be embedded in models and shared freely.
3. **Key rotation**: If the secret key is compromised, all models signed with it should be considered untrusted.

## Implementation Files

| File | Purpose |
|------|---------|
| `mldsa44_binding.py` | Low-level ctypes bindings to ML-DSA-44 C library |
| `self_verifying_secure.py` | Self-verifying model implementation (Approach A) |
| `secure_signature.py` | Length-prefix format implementation (Approach B) |
| `libmldsa44.so` | PQClean ML-DSA-44 shared library |

## Post-Quantum Security

ML-DSA (formerly known as CRYSTALS-Dilithium) is a NIST-standardized post-quantum signature algorithm. It provides security against both classical and quantum computer attacks, future-proofing the binding mechanism against advances in quantum computing.
