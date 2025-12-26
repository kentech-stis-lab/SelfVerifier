# about-pickle_internal

Self-verifying PyTorch model implementation with internal ML-DSA-44 signatures. This directory contains the core implementation for signed-only execution framework.

## Prerequisites

### 1. Create `models/` directory and download models

Before running the tests, download the following models from HuggingFace:
```bash
mkdir -p models
```

| File | Size | Model | Source |
|------|------|-------|--------|
| `small_model.pt` | 87 MB | all-MiniLM-L6-v2 | [sentence-transformers/all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) |
| `medium_model.pt` | 2.5 GB | OPT-1.3B | [facebook/opt-1.3b](https://huggingface.co/facebook/opt-1.3b) |
| `large_model.pt` | 12.4 GB | OPT-6.7B | [facebook/opt-6.7b](https://huggingface.co/facebook/opt-6.7b) |

**Download script:**
```python
import torch
from transformers import AutoModel, AutoModelForCausalLM

# Small model
model = AutoModel.from_pretrained("sentence-transformers/all-MiniLM-L6-v2")
torch.save(model, "models/small_model.pt")

# Medium model
model = AutoModelForCausalLM.from_pretrained("facebook/opt-1.3b")
torch.save(model, "models/medium_model.pt")

# Large model
model = AutoModelForCausalLM.from_pretrained("facebook/opt-6.7b")
torch.save(model, "models/large_model.pt")
```

### 2. Build ML-DSA library
```bash
cd ../ML-DSA/crypto_sign/ml-dsa-44/1_clean
make
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD
```

> **Note:** `models_defense/` and `models_attack/` directories are automatically generated when running the test scripts.



## Directory Structure

```
about-pickle_internal/
├── models/                  # Source models (unsigned)
├── models_defense/          # Signed models (protected)
├── models_attack/           # Tampered models (for testing)
├── data/                    # Log files and test data
├── uploads/                 # Attack demo upload directory
└── __pycache__/             # Python bytecode cache
```

## Directories

### `models/`
Original unsigned PyTorch model files used as source for signing.

| File | Size | Model | Source |
|------|------|-------|--------|
| `small_model.pt` | 87 MB | all-MiniLM-L6-v2 | [sentence-transformers/all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) |
| `medium_model.pt` | 2.5 GB | OPT-1.3B | [facebook/opt-1.3b](https://huggingface.co/facebook/opt-1.3b) |
| `large_model.pt` | 12.4 GB | OPT-6.7B | [facebook/opt-6.7b](https://huggingface.co/facebook/opt-6.7b) |

### `models_defense/`
Signed models with ML-DSA-44 cryptographic signatures. These models are protected against tampering.

| File | Description |
|------|-------------|
| `small_normal.pt` | Unsigned small model (baseline) |
| `small_signed.pt` | Signed small model (protected) |
| `medium_normal.pt` | Unsigned medium model (baseline) |
| `medium_signed.pt` | Signed medium model (protected) |
| `large_normal.pt` | Unsigned large model (baseline) |
| `large_signed.pt` | Signed large model (protected) |

### `models_attack/`
Tampered models with injected malicious payloads for security testing. Used to verify that the defense mechanism correctly blocks attacks.

| File | Description |
|------|-------------|
| `small_normal_malicious.pt` | Unsigned model + malicious payload (attack succeeds) |
| `small_signed_malicious.pt` | Signed model + malicious payload (attack blocked) |
| `medium_normal_malicious.pt` | Unsigned model + malicious payload (attack succeeds) |
| `medium_signed_malicious.pt` | Signed model + malicious payload (attack blocked) |
| `large_normal_malicious.pt` | Unsigned model + malicious payload (attack succeeds) |
| `large_signed_malicious.pt` | Signed model + malicious payload (attack blocked) |

### `data/`
Log files and test scenario data.

| File | Description |
|------|-------------|
| `cnc_log.txt` | Command and control server log (attack demo) |

### `uploads/`
Directory for files uploaded during attack demonstration.

| File | Description |
|------|-------------|
| `attack_demo.sh` | Copy of attack script for upload demo |

## Files

### Core Implementation

| File | Description |
|------|-------------|
| `self_verifying_secure.py` | Main implementation of self-verifying model class with `SelfVerifier` and `_verify_and_restore()` function |
| `secure_signature.py` | Alternative length-prefix format signature implementation with `SecureSignedModel` class |
| `mldsa44_binding.py` | Python ctypes bindings to ML-DSA-44 C library (`keypair()`, `sign()`, `verify()`) |

### Cryptographic Keys

| File | Size | Description |
|------|------|-------------|
| `ml_dsa_secret.key` | 2,560 bytes | ML-DSA-44 secret key for signing |
| `ml_dsa_public.key` | 1,312 bytes | ML-DSA-44 public key for verification |
| `libmldsa44.so` | 57 KB | Compiled ML-DSA-44 shared library (PQClean) |

### Testing and Demo

| File | Description |
|------|-------------|
| `test_all_models.py` | Comprehensive test suite for all model sizes and attack scenarios |
| `server.py` | HTTP server for attack demonstration (serves malicious scripts) |
| `attack_demo.sh` | Demonstration script executed during pickle RCE attack |
| `test_normal.pt` | Small test model (unsigned) |
| `test_signed.pt` | Small test model (signed) |

### Output Files

| File | Description |
|------|-------------|
| `output.txt` | Test execution output log |
| `terminal_output.txt` | Terminal output capture |

## Usage

### Sign a Model

```bash
python3 self_verifying_secure.py create models/small_model.pt signed.pt ml_dsa_secret.key ml_dsa_public.key
```

### Verify a Model

```bash
python3 self_verifying_secure.py verify signed.pt
```

### Start Attack Demo Server

```bash
python3 server.py
```


### Run Security Tests

```bash
python3 test_all_models.py
```



## Test Scenarios

The test suite validates four scenarios for each model size:

1. **Normal model + torch.load()** - Baseline loading (succeeds)
2. **Normal + Malicious + torch.load()** - Attack demonstration (RCE executes)
3. **Signed model + torch.load()** - Protected loading (succeeds with verification)
4. **Signed + Malicious + torch.load()** - Attack blocked (signature verification fails)
