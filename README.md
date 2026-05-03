# Adaptive Post-Quantum Authentication for SDN-Managed Smart-Home IoT
**FALCON-SDIoT · IEEE TDSC 2026 Paper-Aligned Implementation**

[![Python](https://img.shields.io/badge/Python-3.10-blue)](https://python.org)
[![Notebook](https://img.shields.io/badge/Jupyter-Notebook-orange)](https://jupyter.org)
[![Cryptography](https://img.shields.io/badge/Cryptography-PQ--Ready-green)](https://cryptography.io)
[![Federated Learning](https://img.shields.io/badge/Federated-Learning-purple)](#hierarchical-federated-device-classification-hfl-dc)
[![Formal Verification](https://img.shields.io/badge/ProVerif-2.05-red)](https://bblanche.gitlabpages.inria.fr/proverif/)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](LICENSE)

> **Sameera, Uddin Md. Borhan, Arif Raza, Qianqian Liu, Kashif Sharif**  
> Shenzhen University · Columbia University · Beijing Institute of Technology  
> Corresponding author: **Kashif Sharif**  
> Project name: **FALCON-SDIoT** — *Federated Adaptive Lattice-based Cryptographic Orchestration and Network Security for SDN-managed IoT*

---

## Overview

<p align="justify">
This repository contains the paper-aligned implementation of <b>Adaptive Post-Quantum Authentication for SDN-Managed Smart-Home IoT</b>. The project implements <b>FALCON-SDIoT</b>, a unified smart-home IoT security framework that combines post-quantum onboarding and rekeying, privacy-preserving federated device classification, certified lattice-based proof-of-possession authentication, and trust-score adaptive reclassification under an SDN-managed control plane.
</p>

<p align="justify">
The implementation is provided as a single executable notebook, <code>post_quantum_iot.ipynb</code>, which reproduces the complete prototype pipeline: synthetic IoT traffic generation, centralized baseline training, hierarchical federated learning with differential privacy and Multi-Krum aggregation, post-quantum key-management simulation, lattice proof-of-possession authentication, TSAR trust simulation, result-table printing, and figure generation.
</p>

FALCON-SDIoT addresses four coupled security challenges in smart-home IoT:

1. <b>Quantum-resilient authentication:</b> Device onboarding and rekeying are modeled around ML-KEM-style encapsulation and ML-DSA-style certificate-backed signatures.
2. <b>Private traffic intelligence:</b> Raw smart-home traffic remains local at edge gateways; only clipped and noised federated model updates leave the edge.
3. <b>Certified proof-of-possession:</b> Re-authentication proves possession of a certified lattice secret without exposing reusable device secrets.
4. <b>Adaptive cryptographic response:</b> Trust-score changes trigger class-dependent hardening, relaxation, and rekeying instead of only producing alerts or SDN flow-rule updates.

> **Important implementation note:** The notebook provides a paper-aligned executable prototype. For portability, the ML-KEM and ML-DSA interfaces are simulated using standard Python cryptography primitives while preserving the protocol roles, message flow, accounting boundaries, and security-control logic. For production-grade deployment, replace the simulation classes with NIST/PQ library implementations such as liboqs, PQClean, or vendor-certified ML-KEM/ML-DSA modules.

---

## Unified System Model

<p align="center">
  <img src="./figures/FALCON_SDIoT.png" width="90%" alt="FALCON-SDIoT five-layer SDN smart-home IoT architecture"/>
</p>

<p align="justify">
FALCON-SDIoT operates across five layers: smart-home IoT devices, the MUD profile server, the edge gateway, the SDN controller/federated-learning aggregator, and cloud or local services. Devices hold manufacturer-certified identities and lattice proof keys. The edge verifies device credentials, runs HFL-DC inference, checks proof-of-possession transcripts, evaluates trust, and relays management messages. The controller assigns security classes, issues class-dependent transport keys, and installs SDN policies.
</p>

---

## Architecture

```text
Smart-home IoT device d
  Stores:
    ID_d, MAC_d, device type μ_d
    ML-DSA-style signing key pair (vpk_d, ssk_d)
    lattice PoP key pair (t_d, s_d)
    manufacturer certificate cert_d

        |
        |  certificate-backed registration
        v

ManufacturerCA
  cert_d = Sign_msk(ID_d, MAC_d, vpk_d, t_d, μ_d, exp_d)
  verify(cert_d) binds:
    device identity + MAC + signing key + lattice public key + device type

        |
        v

PQ_KM registration and transport-key delivery
  Phase 0: verify signed ephemeral KEM public keys τ_e and τ_c
  Phase 1: D -> E: encrypted registration payload P1
  Phase 2: E verifies cert_d and σ_d
  Phase 3: E -> C: class and identity binding P2
  Phase 4: C samples class-dependent transport key k_d^(0)
  Phase 5: C -> E -> D: encrypted delivery and SDN policy installation
  Erasure: ephemeral KEM secrets, wrapping keys, and session secrets deleted

        |
        +------------------------------+
        |                              |
        v                              v

HFL_DC                         LB_ZKA recurring authentication
  Synthetic IoT traffic          R_q = Z_q[X]/(X^n + 1)
  27 device types                t_d = A s_d mod q
  10 traffic features            stmt_d = (ID_d, η_auth, τ_auth)
  MUD cold-start priors          π_d = Prove(pp, t_d, s_d, stmt_d)
  Dirichlet non-IID split        Verify(pp, t_d, stmt_d, π_d)
  DP clipping + Gaussian noise   single-use verifier nonce
  Multi-Krum aggregation         timestamp window Δ_auth = 30 s
  FedAvg global update

        |
        v

TSAR trust-score adaptive reclassification and rekeying
  x_d = [mean_iat_ms, byte_count_pm, tcp_ratio,
         dst_port_entropy, conn_count_pm, tls_ratio]
  Isolation Forest anomaly score -> EMA trust score T_d(t)
  if T_d(t) < θ↑:
      Harden(ℓ), reset φ, trigger immediate rekey
  if T_d(t) > θ↓ for W_hold:
      Relax(ℓ), reset φ, trigger class-dependent rekey
  otherwise:
      keep current security class
```

---

## Method Details

This section explains how the paper modules are mapped into the implementation.

### 1. Device Taxonomy, Traffic Profiles, and MUD Priors

**Notebook section:** `§1 DEVICE TAXONOMY, PROFILES, MUD PRIORS`  
**Core objects:** `DEVICE_TYPES`, `CLASS_MAP`, `PROFILES`, `MUD_PRIORS`, `TSAR_IDX`

<p align="justify">
The prototype defines a smart-home taxonomy with 27 IoT device types mapped into three operational security classes. Class A contains high-risk or privacy-sensitive devices such as cameras and locks. Class B contains medium-risk devices such as switches and hubs. Class C contains lower-risk commodity appliances. Each device type is assigned a synthetic traffic profile over ten features, and a six-feature slice is used by TSAR for trust monitoring.
</p>

```python
TSAR features:
[
  mean_iat_ms,
  byte_count_pm,
  tcp_ratio,
  dst_port_entropy,
  conn_count_pm,
  tls_ratio
]
```

MUD-guided cold-start priors encode expected class likelihoods before sufficient live traffic is available:

```python
P(A/B/C | camera) = (0.90, 0.08, 0.02)
P(A/B/C | switch) = (0.10, 0.85, 0.05)
P(A/B/C | kettle) = (0.03, 0.05, 0.92)
```

<p align="justify">
The MUD prior is used as a bootstrap regularizer and interpretability signal. It is not treated as a universal accuracy-improvement mechanism; the final classifier still learns from traffic features under federated training.
</p>

---

### 2. Timing, Energy, and Accounting Boundaries

**Notebook section:** `§2 TIMING & ENERGY CONSTANTS`  
**Related outputs:** `TABLE III`, `TABLE IV`, `fig5_performance.png`, `fig6_energy.png`

<p align="justify">
The notebook separates one-time onboarding cost from recurring authentication cost. This is important because post-quantum onboarding carries larger certificate, KEM, and signature costs, while recurring proof-of-possession authentication is the operation most frequently executed during normal smart-home operation.
</p>

```text
Total Registration = Kyber-768 KeyGen + Kyber-768 Encaps + Dilithium3 Sign
                   = 9.43 + 8.11 + 6.71
                   = 24.25 ms

Total Authentication = LB-ZKA Prove + LB-ZKA Verify + AES-256-GCM Enc + SHA3-256
                     = 1.82 + 2.11 + 0.89 + 0.12
                     = 4.94 ms
```

The session-level energy model is:

```text
E_total = E_crypto + E_radio + E_edge + E_controller + E_idle
```

---

### 3. Differential Privacy Accounting

**Notebook section:** `§3 FIX-1: RÉNYI DP COMPOSITION`  
**Function:** `renyi_dp_budget()`

<p align="justify">
HFL-DC applies Gaussian differential privacy to clipped client updates and then reports the composed privacy budget using Rényi differential privacy accounting. The evaluation uses a per-round budget of ε=1.2 and δ=10⁻⁵ over 10 rounds. The reported total budget is approximately (4.35, 10⁻⁵), which should be interpreted as moderate privacy rather than a tight privacy guarantee.
</p>

```python
sigma_DP = C_clip * sqrt(2 * ln(1.25 / delta_r)) / epsilon_r
```

With `C_clip = 1`, `epsilon_r = 1.2`, and `delta_r = 1e-5`, the notebook obtains:

```text
σ_DP ≈ 4.037
```

---

### 4. Cryptographic Utilities

**Notebook section:** `§4 CRYPTOGRAPHIC UTILITIES`  
**Class:** `CU`  
**Methods:** `sha3()`, `hkdf()`, `enc()`, `dec()`

The `CU` class centralizes common cryptographic helper routines:

```python
CU.sha3(data)      # SHA3-256 digest
CU.hkdf(secret, info, length)  # HKDF key derivation
CU.enc(key, plaintext, aad)    # AES-GCM encryption
CU.dec(key, ciphertext, aad)   # AES-GCM decryption
```

<p align="justify">
This keeps the protocol implementation readable and makes domain separation explicit through the HKDF context string, for example <code>D2E</code>, <code>E2C</code>, and <code>C2D</code>.
</p>

---

### 5. Manufacturer Certificate Authority

**Notebook section:** `§5 MANUFACTURER CA`  
**Class:** `ManufacturerCA`  
**Methods:** `issue()`, `verify()`

<p align="justify">
The manufacturer certificate authority binds each device identity to its public verification material. The certificate includes the device identity, MAC address, verification key, lattice public key, device type, and expiration metadata. This binding is essential because LB-ZKA must prove possession of a secret whose public key is manufacturer-certified rather than attacker-substituted.
</p>

```python
cert_d = Sign_msk(ID_d, MAC_d, vpk_d, t_d, μ_d, exp_d)
```

Implementation role:

```text
ManufacturerCA.issue(...)
  -> produces cert_d

ManufacturerCA.verify(cert_d)
  -> checks signature and extracts identity-bound public material
```

---

### 6. ML-KEM and ML-DSA Interface Simulation

**Notebook sections:**  
`§6 ML-KEM SIMULATION (FIPS 203)`  
`§7 ML-DSA SIMULATION (FIPS 204)`

**Classes:** `KyberKEM`, `DilithiumSign`  
**Methods:** `keygen()`, `encaps()`, `decaps()`, `sign()`, `verify()`

<p align="justify">
The notebook implements ML-KEM-style and ML-DSA-style interfaces to reproduce the paper protocol flow. The class names and message flow follow the post-quantum design, while the executable prototype uses Python cryptography primitives for portability. This allows the notebook to validate protocol sequencing, key wrapping, replay protection, message sizes, timing accounting, and class-dependent rekeying logic without requiring a platform-specific PQC build.
</p>

```python
pk_e, sk_e = KyberKEM.keygen(level="768")
c_de, K_de = KyberKEM.encaps(pk_e)
K_de_prime = KyberKEM.decaps(sk_e, c_de)

vpk_d, ssk_d = DilithiumSign.keygen(level="3")
sigma_d = DilithiumSign.sign(ssk_d, M1)
ok = DilithiumSign.verify(vpk_d, M1, sigma_d)
```

---

### 7. Post-Quantum Key Management (PQ-KM)

**Notebook section:** `§8 PQ-KM — Algorithm 1`  
**Class:** `PQ_KM`  
**Method:** `register()`

<p align="justify">
PQ-KM implements certificate-backed registration and transport-key distribution under SDN control. It verifies manufacturer certificates, checks device signatures, establishes wrapping keys using KEM-derived shared secrets, assigns the initial security class, delivers the class-dependent transport key, installs the SDN policy, and erases ephemeral state after completion.
</p>

#### Registration flow

```text
1. Edge and controller generate ephemeral KEM key pairs.
2. Edge signs pk_e || ν_e; controller signs pk_c || ν_c.
3. Device verifies signed ephemeral KEM public key from edge.
4. Device builds M1 = (ID_d || cert_d || μ_d || ν_d).
5. Device signs M1 and encapsulates to edge.
6. Edge decapsulates, decrypts P1, verifies cert_d and σ_d.
7. Edge classifies the device and forwards identity/class binding to controller.
8. Controller registers (ID_d, vpk_d, t_d, ℓ_0(d)).
9. Controller samples k_d^(0) with length κ(ℓ_0(d)).
10. Controller installs SDN policy and delivers encrypted key material.
11. Device installs k_d^(0).
12. Device, edge, and controller erase ephemeral secrets.
```

Class-dependent transport key lengths:

| Security class | Typical devices | Transport key |
|---|---|---:|
| A | cameras, locks, health/safety devices | AES-256 |
| B | switches, hubs, medium-risk controllers | AES-192 |
| C | kettles, bulbs, low-risk appliances | AES-128 |

---

### 8. Lattice-Based Proof-of-Possession Authentication (LB-ZKA)

**Notebook section:** `§9 LB-ZKA`  
**Class:** `LB_ZKA`  
**Methods:** `keygen()`, `prove()`, `verify()`, `authenticate()`

<p align="justify">
LB-ZKA provides recurring proof-of-possession after onboarding. The device proves knowledge of the certified lattice secret <code>s_d</code> without revealing a reusable secret. Each proof is bound to a verifier-issued nonce and timestamp, which prevents replay of an old proof transcript.
</p>

Core relation:

```text
R_q = Z_q[X] / (X^n + 1)
t_d = A s_d mod q,  ||s_d||_∞ ≤ η
```

Authentication interface:

```python
stmt_d = (ID_d, eta_auth, tau_auth)
pi_d = LB_ZKA.prove(pp, t_d, s_d, stmt_d)
ok = LB_ZKA.verify(pp, t_d, stmt_d, pi_d)
```

The notebook validates the expected behavior:

```text
LB-ZKA valid:             True
Proof size:               ~720 bytes
Tampered proof rejected:  True
```

Implementation detail:

```text
The polynomial ring multiplication uses exact integer negacyclic convolution.
This avoids floating-point rounding errors that appeared in an earlier FFT-based version.
```

---

### 9. Hierarchical Federated Device Classification (HFL-DC)

**Notebook section:** `§10 HFL-DC`  
**Class:** `HFL_DC`  
**Methods:** `_partition()`, `_mud_augment()`, `_mkrum()`, `_dp_clip_noise()`, `train()`, `predict()`

<p align="justify">
HFL-DC maps traffic observations to security classes while keeping raw traffic at edge gateways. The global model is trained over non-IID edge partitions. Each edge trains locally, clips its update, adds Gaussian noise for differential privacy, and sends only the protected update to the aggregator. Multi-Krum removes outlier updates before FedAvg aggregation.
</p>

Pipeline:

```text
Dataset X, y
  |
  v
Dirichlet non-IID partition across N_edge = 6 gateways
  |
  v
Local model update Δ_i
  |
  v
Clip: Δ_i / max(1, ||Δ_i||_2 / C_clip)
  |
  v
Noise: + N(0, σ_DP² I)
  |
  v
Multi-Krum score:
  score_i = sum nearest ||Δ_i - Δ_j||²
  |
  v
Retain m = N_edge - f - 2 updates
  |
  v
FedAvg update of global model w_g
```

Evaluation setting:

```text
N_edge = 6
f = 1 Byzantine edge
m = 3 retained updates
Dirichlet α = 0.5
Rounds = 10
Per-round DP: ε = 1.2, δ = 1e-5
```

---

### 10. Trust-Score Adaptive Reclassification and Rekeying (TSAR)

**Notebook section:** `§11 TSAR`  
**Class:** `TSAR`  
**Methods:** `register()`, `_anom()`, `_ema()`, `_policy()`, `simulate()`, `tpr()`, `fpr()`, `latency()`

<p align="justify">
TSAR converts behavior monitoring into cryptographic action. It tracks each device's trust score over sliding windows. When trust falls below the hardening threshold, the device is immediately moved to a stronger security class and PQ-KM rekeys it. When behavior becomes benign, relaxation is delayed until a hold window is satisfied, preventing attackers from quickly spoofing benign behavior to reduce protection.
</p>

Policy rule:

```text
if T_d(t) < θ↑:
    (ℓ_t, φ_t) = (Harden(ℓ_{t-1}), 0)
elif T_d(t) > θ↓ and (φ + 1) W_slide ≥ W_hold:
    (ℓ_t, φ_t) = (Relax(ℓ_{t-1}), 0)
elif T_d(t) > θ↓:
    (ℓ_t, φ_t) = (ℓ_{t-1}, φ + 1)
else:
    (ℓ_t, φ_t) = (ℓ_{t-1}, 0)
```

TSAR evaluation scenarios:

| Scenario | Expected behavior |
|---|---|
| Camera hijack / port scan | Detect anomaly, harden class, trigger rekey |
| Plug high-rate anomaly | Detect anomaly, harden class, trigger rekey |
| Benign firmware update | Avoid false positive |
| Adversarial spoofing | Prevent premature relaxation through hold-window logic |

---

## Notebook Code Tour

```text
post_quantum_iot.ipynb
├── §0  Imports & setup
│   ├── installs/loads cryptography, scipy, xgboost, sklearn, matplotlib, seaborn
│   └── configures deterministic seeds and plotting defaults
│
├── §1  Device taxonomy, profiles, MUD priors
│   ├── defines 27 smart-home IoT device types
│   ├── maps devices to classes A/B/C
│   ├── defines 10-feature traffic profiles
│   └── defines 6-feature TSAR slice
│
├── §2  Timing and energy constants
│   ├── per-operation latency and energy
│   ├── recurring-session accounting
│   └── comparator rows for performance tables
│
├── §3  Differential privacy accounting
│   └── renyi_dp_budget()
│
├── §4  Cryptographic utilities
│   └── CU.sha3(), CU.hkdf(), CU.enc(), CU.dec()
│
├── §5  Manufacturer CA
│   └── ManufacturerCA.issue(), ManufacturerCA.verify()
│
├── §6  ML-KEM-style interface
│   └── KyberKEM.keygen(), encaps(), decaps()
│
├── §7  ML-DSA-style interface
│   └── DilithiumSign.keygen(), sign(), verify()
│
├── §8  PQ-KM
│   └── PQ_KM.register()
│
├── §9  LB-ZKA
│   ├── LB_ZKA.keygen()
│   ├── LB_ZKA.prove()
│   ├── LB_ZKA.verify()
│   └── LB_ZKA.authenticate()
│
├── §10 HFL-DC
│   ├── HFL_DC._partition()
│   ├── HFL_DC._mud_augment()
│   ├── HFL_DC._mkrum()
│   ├── HFL_DC._dp_clip_noise()
│   ├── HFL_DC.train()
│   └── HFL_DC.predict()
│
├── §11 TSAR
│   ├── TSAR.register()
│   ├── TSAR._anom()
│   ├── TSAR._ema()
│   ├── TSAR._policy()
│   ├── TSAR.simulate()
│   ├── TSAR.tpr()
│   ├── TSAR.fpr()
│   └── TSAR.latency()
│
├── §12 Dataset generation
│   └── generate_dataset()
│
├── §13 Centralized baselines
│   └── train_baselines()
│
├── §14 Figure generation
│   ├── fig1_accuracy()
│   ├── fig2_confusion()
│   ├── fig3_hfl()
│   ├── fig4_tsar()
│   ├── fig5_performance()
│   └── fig6_energy()
│
├── §15 Table printers
│   └── print_all_tables()
│
└── §16 Main pipeline
    └── main()
```

---

## Repository Layout

```text
post-quantum-iot-main/
├── figures/
│   ├── FALCON_flow.png              <- post-quantum registration/rekeying flow
│   └── FALCON_SDIoT.png             <- five-layer SDN smart-home IoT architecture
│
├── graphs/
│   ├── fig1_accuracy.png            <- centralized vs HFL-DC classification accuracy
│   ├── fig2_hfldc_confusion.png     <- HFL-DC confusion matrix
│   ├── fig2b_stacking_confusion.png <- centralized stacking confusion matrix
│   ├── fig3_hfl.png                 <- HFL-DC convergence / robustness curve
│   ├── fig4_tsar.png                <- TSAR trust-score trace
│   ├── fig5_performance.png         <- recurring authentication latency and energy
│   ├── fig6_energy.png              <- per-operation energy comparison
│   └── gitee                        <- placeholder or generated artifact
│
├── post_quantum_iot.ipynb           <- full paper-aligned executable prototype
└── README.md                        <- repository documentation
```

---

## Requirements

Recommended environment:

- Ubuntu 20.04 / 22.04
- Python 3.10
- Jupyter Notebook or JupyterLab
- scikit-learn
- scipy
- numpy
- pandas
- matplotlib
- seaborn
- cryptography
- xgboost
- ProVerif 2.05 for symbolic verification models

Install Python dependencies:

```bash
conda create -n falcon python=3.10 -y
conda activate falcon

pip install \
  numpy pandas scipy scikit-learn matplotlib seaborn \
  cryptography xgboost jupyter
```

Optional ProVerif setup:

```bash
sudo apt-get update
sudo apt-get install -y ocaml wget tar make gcc

wget https://bblanche.gitlabpages.inria.fr/proverif/proverif2.05.tar.gz
tar -xzf proverif2.05.tar.gz
cd proverif2.05
./build
./proverif -version
```

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/umborhan/post-quantum-iot.git
cd post-quantum-iot-main
```

### 2. Create and activate the environment

```bash
conda create -n falcon python=3.10 -y
conda activate falcon
pip install -q cryptography xgboost scikit-learn scipy matplotlib seaborn pandas numpy jupyter
```

### 3. Run the notebook

```bash
jupyter notebook post_quantum_iot.ipynb
```

Then run all notebook cells from top to bottom.

### 4. Optional: convert notebook to a script

```bash
jupyter nbconvert --to script post_quantum_iot.ipynb
python post_quantum_iot.py
```

Expected console header:

```text
======================================================================
  FALCON-SDIoT | IEEE TDSC | Paper-Aligned Implementation
  XGBoost: True | Python: 3.10.x
======================================================================
```

---

## Expected Pipeline Output

The main pipeline runs nine stages:

```text
[STEP 1/9]  Generating synthetic IoT traffic dataset
[STEP 2/9]  Training centralized baseline classifiers
[STEP 3/9]  HFL-DC with Multi-Krum, MUD priors, and differential privacy
[STEP 4/9]  Federated evaluation
[STEP 5/9]  ManufacturerCA + PQ-KM registration
[STEP 6/9]  LB-ZKA recurring proof-of-possession authentication
[STEP 7/9]  TSAR trust-score adaptive reclassification and rekeying
[STEP 8/9]  Generating figures
[STEP 9/9]  Printing paper tables
```

Expected dataset summary:

```text
[DATA] 24300 samples × 10 features | A=9000 B=8100 C=7200
Train=19440  Test=4860
```

Expected generated graph files:

```text
graphs/fig1_accuracy.png
graphs/fig2_hfldc_confusion.png
graphs/fig2b_stacking_confusion.png
graphs/fig3_hfl.png
graphs/fig4_tsar.png
graphs/fig5_performance.png
graphs/fig6_energy.png
```

---

## Key Results

### Classification and Federated Learning

| Method | Accuracy | F1 | Precision | Recall | Privacy |
|---|---:|---:|---:|---:|---|
| Centralized SDN-IoT Stacking | 0.9006 | 0.9003 | 0.9005 | 0.9006 | None |
| HFL-DC Round 1 + MUD | 0.8556 | 0.8544 | 0.8664 | 0.8556 | (ε,δ)-DP |
| HFL-DC Round 5 + MUD | 0.8632 | 0.8625 | 0.8742 | 0.8632 | (ε,δ)-DP |
| HFL-DC Round 10 + MUD | 0.8580 | 0.8567 | 0.8691 | 0.8580 | (ε,δ)-DP |
| HFL-DC Round 10 + 1 Byzantine edge | 0.8516 | — | — | — | Multi-Krum + DP |

<p align="justify">
HFL-DC preserves raw-traffic locality and remains close to centralized classification performance while adding differential privacy and Byzantine-resilient aggregation.
</p>

### Key Management and Authentication

| Metric | Value |
|---|---:|
| Registration latency | 24.25 ms |
| Recurring authentication latency | 4.94 ms |
| LB-ZKA prove time | 1.82 ms |
| LB-ZKA verify time | 2.11 ms |
| Proof/session-control payload | ~720 bytes |
| Tampered proof rejected | True |
| κ(A/B/C) | 256 / 192 / 128 bits |

### End-to-End Performance Comparison

| Scheme | IoT (ms) | Edge (ms) | Total (ms) | Energy (mJ) | Memory | Comm. | PQ |
|---|---:|---:|---:|---:|---:|---:|---|
| Simple-XOR | 8.038 | 10.737 | 18.775 | 234.67 | 40 | 79 | No |
| SKICAP | 17.200 | 8.200 | 25.400 | 281.13 | 198 | 341 | No |
| EPAW | 18.300 | 50.200 | 68.500 | 700.11 | 150 | 228 | No |
| SDN-IoT | 6.600 | 7.880 | 14.480 | 180.88 | 34 | 294 | No |
| PILIKE | 8.120 | 9.340 | 17.460 | 218.22 | 52 | 216 | Yes |
| **FALCON-SDIoT** | **4.880** | **5.810** | **10.690** | **123.40** | **42** | **720** | **Yes** |

<p align="justify">
The recurring authentication path reduces total latency by 26.2% and energy by 31.8% relative to the SDN-IoT baseline under the same system-level accounting boundary. The communication payload is larger because the recurring proof transcript is post-quantum and proof-based.
</p>

### TSAR Detection and Rekeying

| Scenario | TPR | FPR | Latency | Response |
|---|---:|---:|---:|---|
| Camera hijack / port scan | 100% | 0.0% | ~3 windows | C→A hardening + rekey |
| Plug high-rate anomaly | 100% | 0.0% | ~4 windows | C→B hardening + rekey |
| Firmware update benign trace | N/A | 0.0% | N/A | Correctly ignored |
| Adversarial spoof trace | N/A | 0.0% | N/A | Relaxation blocked |
| Overall evaluated trace | 100% | 0.0% | 3.1 min | Scenario-bounded |

> The 0.0% false-positive result is bounded to the evaluated synthetic scenarios. It should not be interpreted as a deployment-wide guarantee.

---

## Figures and Visual Results


### Post-Quantum Key-Management Flow

<p align="center">
  <img src="./figures/FALCON_flow.png" width="90%" alt="FALCON-SDIoT post-quantum key-management flow"/>
</p>

<p align="justify">
The flow diagram describes registration, controller issuance, key delivery, erasure, and rekeying. It highlights the separation between heavyweight onboarding and lightweight recurring authentication.
</p>

### Classification Accuracy

<p align="center">
  <img src="./graphs/fig1_accuracy.png" width="70%" alt="Classification accuracy comparison"/>
</p>

<p align="justify">
This graph compares centralized baselines against HFL-DC. Centralized models achieve slightly higher accuracy because they can see all raw traffic centrally. HFL-DC trades a small utility drop for privacy-preserving and Byzantine-robust training.
</p>

### Confusion Matrices

<p align="center">
  <img src="./graphs/fig2_hfldc_confusion.png" width="45%" alt="HFL-DC confusion matrix"/>
  &nbsp;
  <img src="./graphs/fig2b_stacking_confusion.png" width="45%" alt="Stacking baseline confusion matrix"/>
</p>

<p align="justify">
The left confusion matrix shows HFL-DC performance under federated training. The right matrix shows the centralized stacking baseline. These plots help identify which device classes are easiest or hardest to separate under privacy-preserving training.
</p>

### HFL-DC Convergence

<p align="center">
  <img src="./graphs/fig3_hfl.png" width="75%" alt="HFL-DC convergence"/>
</p>

<p align="justify">
The convergence figure reports federated learning behavior across rounds, including clean training, MUD-guided cold start, and the Byzantine-edge scenario handled through Multi-Krum aggregation.
</p>

### TSAR Trust Trace

<p align="center">
  <img src="./graphs/fig4_tsar.png" width="75%" alt="TSAR trust-score trace"/>
</p>

<p align="justify">
The TSAR figure visualizes trust evolution over time. Sudden anomaly evidence lowers trust, triggers immediate hardening, and invokes rekeying. Relaxation requires sustained benign behavior through the hold-window counter.
</p>

### Authentication Performance

<p align="center">
  <img src="./graphs/fig5_performance.png" width="80%" alt="Authentication latency and energy comparison"/>
</p>

<p align="justify">
This graph compares recurring authentication latency and energy against classical and post-quantum prior schemes. FALCON-SDIoT obtains lower recurring latency and energy than the SDN-IoT baseline under the stated accounting boundary, while paying a larger communication cost for post-quantum proof material.
</p>

### Per-Operation Energy

<p align="center">
  <img src="./graphs/fig6_energy.png" width="75%" alt="Per-operation energy"/>
</p>

<p align="justify">
The energy graph separates expensive one-time onboarding primitives from recurring session primitives. This supports the design choice of using heavier post-quantum credential checks during onboarding while keeping recurring proof-of-possession relatively lightweight.
</p>

---

## Generated Tables

The notebook prints paper-style tables directly in the console:

```text
TABLE III  — Operational Costs
TABLE IV   — End-to-End Performance Comparison
TABLE V    — HFL-DC Federated Classification Accuracy
TABLE VI   — TSAR Detection Performance
```

To save console output:

```bash
python post_quantum_iot.py 2>&1 | tee falcon_run_log.txt
```

---

## Formal Verification with ProVerif

The notebook includes a ProVerif setup and model-generation section.

### Install ProVerif

```bash
sudo apt-get install -y ocaml
wget https://bblanche.gitlabpages.inria.fr/proverif/proverif2.05.tar.gz
tar -xzf proverif2.05.tar.gz
cd proverif2.05
./build
export PATH=$PATH:$(pwd)
proverif -version
```

### Example run

```bash
proverif proverif_models_v2/Q1_transport_key_secrecy.pv
```

The formal verification section models the protocol-level security queries, including transport-key secrecy, authentication, replay protection, and proof-binding properties. The paper-aligned result reports all modeled symbolic queries as `SAFE`.

---

## Reproducing the Main Results

### Run the full notebook

```bash
jupyter notebook post_quantum_iot.ipynb
```

### Or run as script

```bash
jupyter nbconvert --to script post_quantum_iot.ipynb
python post_quantum_iot.py 2>&1 | tee run_log.txt
```

### Check generated figures

```bash
ls graphs/
```

Expected:

```text
fig1_accuracy.png
fig2_hfldc_confusion.png
fig2b_stacking_confusion.png
fig3_hfl.png
fig4_tsar.png
fig5_performance.png
fig6_energy.png
```

---

## Important Configuration Values

| Parameter | Value | Meaning |
|---|---:|---|
| Number of samples | 24,300 | Synthetic IoT benchmark size |
| Device types | 27 | Smart-home device categories |
| Traffic features | 10 | Classification feature vector |
| TSAR features | 6 | Trust-monitoring feature vector |
| Train/test split | 19,440 / 4,860 | 80/20 split |
| Federated edges | 6 | Edge gateways |
| Byzantine budget | 1 | Malicious/noisy edge update |
| Multi-Krum retained updates | 3 | `m = N_edge - f - 2` |
| Dirichlet α | 0.5 | Non-IID partition strength |
| DP ε per round | 1.2 | Per-round privacy parameter |
| DP δ | 1e-5 | Failure probability |
| DP σ | ~4.037 | Gaussian noise scale |
| Rounds | 10 | HFL-DC training rounds |
| κ(A) | 256 bits | High-security transport key |
| κ(B) | 192 bits | Medium-security transport key |
| κ(C) | 128 bits | Low-security transport key |
| TSAR hold window | 60 min | Delay before relaxation |
| LB-ZKA proof payload | ~720 bytes | Recurring proof/session-control payload |

---

## Troubleshooting

### `ModuleNotFoundError: No module named 'xgboost'`

Install the missing package:

```bash
pip install xgboost
```

The notebook can still run most logic without XGBoost if the import fallback is enabled, but the full baseline table is best reproduced with XGBoost installed.

### `cryptography` installation fails

Upgrade pip and wheel:

```bash
python -m pip install --upgrade pip setuptools wheel
pip install cryptography
```

### ProVerif command not found

Add the ProVerif build directory to your `PATH`:

```bash
export PATH=$PATH:/path/to/proverif2.05
proverif -version
```

### Figures are not saved

Make sure the `graphs/` directory exists:

```bash
mkdir -p graphs
```

Then rerun the figure-generation cells or the complete notebook.

### Results differ slightly between runs

The notebook sets deterministic seeds where possible, but small differences may occur because of library versions, XGBoost behavior, or platform-specific numeric routines. Record your environment with:

```bash
python --version
pip freeze > requirements_lock.txt
```

### Communication overhead looks larger than classical schemes

This is expected. FALCON-SDIoT uses a post-quantum proof-of-possession transcript, so the recurring proof payload is larger than ECC-style or symmetric-only messages. The design trades communication size for post-quantum proof binding, replay resistance, and adaptive rekeying.

### Can this be deployed directly on constrained devices?

The current repository is a research prototype. For deployment, replace the simulated PQ interfaces with certified ML-KEM/ML-DSA implementations, use hardware-backed key storage on gateways, compress or fragment proof transcripts for IEEE 802.15.4/6LoWPAN links, and validate on real smart-home traffic traces.

---

## Limitations

<p align="justify">
The current prototype uses a curated synthetic IoT traffic benchmark. This enables controlled evaluation but does not replace broad validation on real-world smart-home traces. The edge gateway is semi-trusted in the current model, so stronger hardware-backed key isolation or direct controller-to-device delivery should be considered for deployment. The differential-privacy budget accumulates across rounds, and the reported total budget is moderate rather than tight. The recurring LB-ZKA proof payload is approximately 720 bytes, which exceeds a single IEEE 802.15.4 frame and requires fragmentation on constrained links. The TSAR 0.0% false-positive result is limited to the evaluated scenario traces and should not be interpreted as a universal guarantee.
</p>

---



---

## Acknowledgements

<p align="justify">
This work is associated with the FALCON-SDIoT research project on adaptive post-quantum authentication for SDN-managed smart-home IoT. The implementation is designed to support paper reproducibility, protocol understanding, figure generation, and future extension toward production-grade post-quantum smart-home security prototypes.
</p>
# post-quantum-iot
Adaptive Post-Quantum Authentication for SDN-Managed Smart-Home IoT
