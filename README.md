# SelfVerifier: Signed-Only Execution Framework

Self-verifying PyTorch models with ML-DSA-44 signatures. Blocks pickle-based RCE attacks by enforcing signature verification at load time.

> **If you use this code or build upon this work, please cite our paper:**
>
> Sunwoo Lee, Hyuk Lim, and Seunghyun Yoon, "Signed-Only Execution for Third-Party Pre-Trained Models in AI Platforms", *IEEE International Conference on Big Data (IEEE BigData)*, Macau, December 2025.

## Problem: Model Supply Chain Attacks

Modern AI platforms routinely load third-party pre-trained models using `torch.load()`, which relies on Python's pickle serialization. This creates a critical vulnerability:

- **Pickle-based RCE**: PyTorch's pickle deserialization enables arbitrary code execution via the `__reduce__()` hook
- **Supply chain poisoning**: Malicious models can execute remote payloads, exfiltrate data, or install backdoors during model loading
- **Insufficient defenses**: Existing malware scans and heuristic checks fail to detect embedded malicious code

This threat is documented in CVE-2025-32434 and demonstrated in recent research on AI model supply chain attacks.

## Solution: Signed-Only Execution

Our framework implements a defense-in-depth approach:

1. **Pre-deployment Validation**: Models undergo security checks (format whitelisting, operator restrictions, sandboxed test loading)
2. **Cryptographic Signing**: Validated models are signed using ML-DSA-44 (NIST FIPS 204) post-quantum signatures
3. **Runtime Enforcement**: Signature verification is automatically triggered during `torch.load()`, blocking unsigned or tampered models

## Key Features

- **Automatic Verification**: Signature verification happens transparently during `torch.load()` via the `__reduce__()` hook - no API changes required
- **Post-Quantum Security**: Uses ML-DSA-44 (CRYSTALS-Dilithium) providing NIST Level 2 security against quantum adversaries
- **Tamper Detection**: Any modification to model bytes causes SHA-256 hash mismatch and signature verification failure
- **Bypass Prevention**: All loading methods trigger verification; attackers cannot skip the security check
- **Re-save Attack Protection**: Returns `SelfVerifier` object instead of raw model, preserving cryptographic metadata across saves

## How It Works

### Self-Verifying Model Format

```
signed_model.pt (ZIP container)
├── data.pkl
│   ├── model_data_bytes    # Serialized model weights
│   ├── signature           # ML-DSA-44 signature (2420 bytes)
│   ├── public_key          # ML-DSA-44 public key (1312 bytes)
│   └── __reduce__()        # Auto-verification hook
```

### Verification Flow

When `torch.load('signed_model.pt')` is called:

1. PyTorch unpickles the `SelfVerifier` object
2. The `__reduce__()` hook automatically invokes `_verify_and_restore()`
3. SHA-256 hash is computed over `model_data_bytes`
4. ML-DSA-44 signature is verified against the computed hash
5. If verification fails → `ValueError` raised, load aborted
6. If verification succeeds → `SelfVerifier` object returned (preserving signature for re-save protection)

```python
# Loading a signed model (verification is automatic!)
import torch

verifier = torch.load('signed_model.pt')  # Auto-verifies signature
model = pickle.loads(verifier.model_data_bytes)  # Extract model
```

## Installation

### Prerequisites

- Python 3.8+
- PyTorch
- PQClean ML-DSA-44 library (compiled as `libmldsa44.so`)

### Build ML-DSA Library

```bash
cd ML-DSA
./setting.sh
```

This compiles the PQClean ML-DSA-44 implementation into a shared library.

## Usage

### Generate Signing Keys

```python
from mldsa44_binding import keypair

public_key, secret_key = keypair()

with open('ml_dsa_public.key', 'wb') as f:
    f.write(public_key)
with open('ml_dsa_secret.key', 'wb') as f:
    f.write(secret_key)
```

### Sign a Model

```bash
cd about-pickle_internal
python self_verifying_secure.py create model.pt signed_model.pt ml_dsa_secret.key ml_dsa_public.key
```

Or programmatically:

```python
from self_verifying_secure import create_self_verifying_model

result = create_self_verifying_model(
    model_path='model.pt',
    secret_key_path='ml_dsa_secret.key',
    public_key_path='ml_dsa_public.key',
    output_path='signed_model.pt'
)
```

### Load a Signed Model

```python
import torch
import pickle

# Automatic signature verification during load
verifier = torch.load('signed_model.pt', weights_only=False)

# Extract the verified model
model = pickle.loads(verifier.model_data_bytes)

# Use the model
output = model(input_data)
```

### Verify a Model (Manual Check)

```bash
python self_verifying_secure.py verify signed_model.pt
```

## Project Structure

```
SelfVerifier/
├── about-pickle_internal/          # Main implementation (internal signature)
│   ├── self_verifying_secure.py    # Core signing and verification logic
│   ├── mldsa44_binding.py          # Python bindings to ML-DSA-44 C library
│   ├── test_all_models.py          # Comprehensive test suite
│   ├── libmldsa44.so               # Compiled ML-DSA library
│   ├── ml_dsa_secret.key           # Secret signing key
│   ├── ml_dsa_public.key           # Public verification key
│   ├── models/                     # Source models for testing
│   ├── models_defense/             # Signed models
│   └── models_attack/              # Tampered models for testing
└── ML-DSA/                         # PQClean ML-DSA-44 source
    ├── crypto_sign/                # ML-DSA implementation
    ├── common/                     # Common utilities
    └── setting.sh                  # Build script
```

## Security Guarantees

### Protected Against

| Threat | Protection |
|--------|------------|
| Pickle RCE | Verification occurs before model deserialization |
| File tampering | SHA-256 hash mismatch triggers signature failure |
| Signature forgery | ML-DSA-44 security (128-bit classical, quantum-resistant) |
| Bypass attempts | All loading methods trigger `__reduce__()` hook |
| Load-modify-save attacks | SelfVerifier object preserves signature across saves |

### Not Protected Against

- **Semantic backdoors**: Malicious weights designed to misbehave on specific inputs
- **Training-time poisoning**: Attacks occurring before signing
- **Key compromise**: Stolen secret keys allow signing arbitrary models
- **Pre-signing supply chain attacks**: Malicious models signed by trusted parties

## Performance

Cryptographic overhead is negligible compared to I/O and serialization:

| Model Size | Hash Time | Sign/Verify Time | Total Overhead |
|------------|-----------|------------------|----------------|
| 87 MB      | ~61 ms    | < 1 ms           | < 0.1% |
| 2.5 GB     | ~1.7 s    | < 1 ms           | < 0.01% |
| 12.4 GB    | ~8.5 s    | < 1 ms           | < 0.001% |

File size overhead: ~3.7 KB (signature + public key + wrapper metadata)

## Running Tests

```bash
cd about-pickle_internal

# Run comprehensive test suite
python3 test_all_models.py
```

This tests:
1. Normal model loading (baseline)
2. Normal + malicious model (attack demonstration)
3. Signed model loading (should succeed)
4. Signed + tampered model (should be blocked)

## Design Philosophy

This framework exploits the same pickle mechanism that enables RCE attacks. The `__reduce__()` hook that allows arbitrary code execution is precisely what makes automatic verification possible:

```
Pickle RCE possible → __reduce__() works → SelfVerifier works
```

This creates a fundamental coupling: as long as PyTorch's pickle-based serialization remains vulnerable to RCE, our defense mechanism remains effective. The defense and the attack surface are existentially linked.

## Publication

This work has been published at:

> **Sunwoo Lee, Hyuk Lim, and Seunghyun Yoon**, "Signed-Only Execution for Third-Party Pre-Trained Models in AI Platforms", *IEEE International Conference on Big Data (IEEE BigData)*, Macau, December 2025.

If you use this work, please cite:

```bibtex
@inproceedings{lee2025selfverifier,
  title={Signed-Only Execution for Third-Party Pre-Trained Models in AI Platforms},
  author={Lee, Sunwoo and Lim, Hyuk and Yoon, Seunghyun},
  booktitle={IEEE International Conference on Big Data (IEEE BigData)},
  year={2025},
  address={Macau}
}
```

## References

- [CVE-2025-32434](https://nvd.nist.gov/vuln/detail/CVE-2025-32434) - PyTorch arbitrary code execution vulnerability
- [NIST FIPS 204](https://csrc.nist.gov/pubs/fips/204/final) - ML-DSA (Module-Lattice-Based Digital Signature Algorithm)
- Wang et al., "Model Supply Chain Poisoning" (2025)
- Jiang et al., "An Empirical Study of Pre-Trained Model Reuse in the Hugging Face Deep Learning Model Registry" (2022)
- Sood et al., "Malicious Models in the Software Supply Chain" (2025)

## Authors

- Sunwoo Lee
- Hyuk Lim 
- Seunghyun Yoon

Korea Institute of Energy Technology (KENTECH)

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
