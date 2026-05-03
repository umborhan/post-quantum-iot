# Adaptive Post-Quantum Authentication for SDN-Managed Smart-Home IoT

**FALCON-SDIoT: Federated Adaptive Lattice-based Cryptographic Orchestration and Network Security for SDN-managed IoT**

[![Python](https://img.shields.io/badge/Python-3.10-blue)](https://www.python.org/)
[![Notebook](https://img.shields.io/badge/Jupyter-Notebook-orange)](https://jupyter.org/)
[![Post--Quantum](https://img.shields.io/badge/Post--Quantum-ML--KEM%20%7C%20ML--DSA-green)](#post-quantum-key-management-pq-km)
[![Federated Learning](https://img.shields.io/badge/Federated-Learning-purple)](#hierarchical-federated-device-classification-hfl-dc)
[![Formal Verification](https://img.shields.io/badge/Formal%20Verification-ProVerif%202.05-red)](#formal-verification)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](LICENSE)

> **Paper:** Adaptive Post-Quantum Authentication for SDN-Managed Smart-Home IoT  
> **Authors:** Sameera, Uddin Md. Borhan, Arif Raza, Qianqian Liu, Kashif Sharif  
> **Project short name:** FALCON-SDIoT  
> **Important naming note:** FALCON-SDIoT is the name of this framework. It is not the NIST FALCON signature scheme. The manuscript uses ML-KEM for key encapsulation and ML-DSA for signatures.

---

## Overview

<p align="justify">
This repository provides a research prototype for <b>adaptive post-quantum authentication in SDN-managed smart-home IoT networks</b>. The code follows the paper design while keeping the README language implementation-focused and paraphrased for GitHub documentation. The project studies a smart-home setting where cameras, locks, plugs, sensors, appliances, hubs, and other IoT devices must be registered, classified, authenticated, monitored, and rekeyed without exposing raw household traffic to a central service.
</p>

<p align="justify">
FALCON-SDIoT joins four components into one security loop: <b>PQ-KM</b> for post-quantum key management, <b>HFL-DC</b> for privacy-preserving federated device classification, <b>LB-ZKA</b> for lattice-based proof-of-possession re-authentication, and <b>TSAR</b> for trust-score adaptive reclassification and rekeying. Together, these modules connect behavioral evidence with cryptographic action under an SDN-managed control plane.
</p>

<p align="justify">
The repository is intended for paper reproducibility, protocol understanding, figure generation, and future extension toward production-grade smart-home IoT security systems.
</p>

> **Prototype note:** The notebook uses portable Python interfaces to simulate the ML-KEM and ML-DSA workflow. This keeps the code easy to run across machines while preserving protocol roles, key-delivery paths, accounting boundaries, and evaluation logic. For deployment, replace these simulation classes with certified or audited post-quantum libraries such as liboqs, PQClean, or vendor-supported ML-KEM/ML-DSA modules.

---

## Why This Work Matters

<p align="justify">
Smart-home IoT security needs more than one-time device registration. A practical system must support quantum-aware authentication, protect household traffic privacy, react to device behavior changes, and convert suspicious behavior into concrete cryptographic response. Alerts and SDN flow updates are useful, but high-risk behavior should also trigger stronger keys, fresh rekeying, or delayed relaxation to prevent fast attacker recovery.
</p>

<p align="justify">
FALCON-SDIoT addresses this need by linking classification, trust monitoring, proof-of-possession, and key renewal into one SDN-managed architecture. Instead of treating authentication, traffic intelligence, and policy enforcement as separate tasks, the framework makes them part of one adaptive security lifecycle.
</p>

---

## System Architecture

<p align="center">
  <img src="./figures/sd-iot.png" width="90%" alt="FALCON-SDIoT SDN-managed smart-home IoT architecture"/>
</p>

<p align="justify">
The system is organized around five cooperating layers. Smart-home IoT devices hold identity material, manufacturer credentials, proof keys, and active transport keys. The MUD profile server provides device-type behavior priors for cold-start classification. The edge gateway verifies certificates, runs local classification, checks recurring proofs, computes trust scores, and forwards management messages. The SDN controller and federated-learning aggregator maintain device classes, issue transport keys, install flow rules, aggregate protected updates, and handle rekeying. Local or cloud services receive encrypted device traffic protected by the active class-dependent key.
</p>

| Layer | Role |
|---|---|
| Smart-home IoT devices | Hold identity, device type, manufacturer certificate, signing key, lattice proof key, and active transport key. |
| MUD profile server | Provides device-type behavior priors used for cold-start classification. |
| Edge gateway | Verifies certificates, runs local classification, checks recurring proofs, computes trust scores, and forwards control messages. |
| SDN controller / FL aggregator | Maintains device classes, issues transport keys, installs flow rules, aggregates protected federated updates, and handles rekeying. |
| Local/cloud services | Receive encrypted device traffic protected by the active class-dependent key. |

<p align="justify">
The controller maps each device class to a symmetric key strength. High-risk or hardened devices receive the strongest key, medium-risk devices receive an intermediate key, and lower-risk stable devices use a lighter key while remaining under monitoring.
</p>

| Class | Typical meaning | Transport key length |
|---|---|---:|
| A | High-risk or hardened devices | 256 bits |
| B | Medium-risk devices | 192 bits |
| C | Lower-risk stable devices | 128 bits |

---

## Post-Quantum Key-Management Flow

<p align="center">
  <img src="./figures/pq-km.png" width="62%" alt="FALCON-SDIoT post-quantum key-management and rekeying flow"/>
</p>

<p align="justify">
PQ-KM manages device onboarding and key renewal. The manufacturer binds device identity, MAC address, verification key, lattice public key, and device type into a certificate. During onboarding, the device verifies signed ephemeral KEM public-key material before sending its registration payload. The edge gateway checks the certificate and device signature, HFL-DC assigns an initial security class, and the controller creates a class-dependent transport key with the related SDN policy.
</p>

<p align="justify">
After key delivery, temporary KEM secrets, wrapping keys, and session secrets are erased. Later, TSAR-triggered class changes call the same managed rekeying logic. This design separates heavier onboarding from recurring authentication, so the repeated session path remains lightweight.
</p>

---

## Main Modules

### 1. Post-Quantum Key Management (PQ-KM)

<p align="justify">
PQ-KM registers devices, assigns class-dependent keys, and rekeys devices when their security class changes. It uses ML-KEM-style encapsulation for post-quantum shared-secret establishment and ML-DSA-style signatures for device certificates and authenticated ephemeral public keys. The module derives HKDF-based wrapping keys with role-separated context labels, maps each security class to AES-128, AES-192, or AES-256 transport keys, and erases temporary onboarding material after the session.
</p>

**Implementation objects used in the notebook:**

```text
ManufacturerCA
KyberKEM
DilithiumSign
PQ_KM
CU.sha3()
CU.hkdf()
CU.enc()
CU.dec()
```

---

### 2. Hierarchical Federated Device Classification (HFL-DC)

<p align="justify">
HFL-DC classifies IoT devices into operational security classes without centralizing raw household traffic. The prototype uses 27 smart-home IoT device types, 10 synthetic traffic features, MUD-guided cold-start priors, non-IID edge partitions, gradient clipping, Gaussian differential privacy, Multi-Krum filtering, and federated averaging over selected updates.
</p>

**Federated training summary:**

```text
local traffic at edge
        |
        v
local model update
        |
        v
clip update and add Gaussian noise
        |
        v
Multi-Krum outlier filtering
        |
        v
weighted global aggregation
        |
        v
updated device-class model
```

---

### 3. Lattice-Based Proof-of-Possession Authentication (LB-ZKA)

<p align="justify">
LB-ZKA re-authenticates a registered device by proving possession of the certified lattice secret without revealing a reusable secret. The device owns a lattice proof key pair, and the lattice public key is bound to the manufacturer certificate. Each proof is linked to a verifier nonce and timestamp, so old transcripts are rejected through freshness and nonce checks.
</p>

**Implementation objects used in the notebook:**

```text
LB_ZKA.keygen()
LB_ZKA.prove()
LB_ZKA.verify()
LB_ZKA.authenticate()
```

---

### 4. Trust-Score Adaptive Reclassification and Rekeying (TSAR)

<p align="justify">
TSAR monitors device behavior after onboarding and triggers class changes with cryptographic consequences. It observes a six-feature traffic slice, uses Isolation Forest evidence for anomaly scoring, smooths short-term noise through an exponential moving average, and applies a policy that hardens immediately when trust becomes low. Relaxation is delayed until benign behavior persists for a hold window, which helps prevent attackers from using short benign bursts to quickly reduce protection.
</p>

**TSAR feature slice:**

```text
mean_iat_ms
byte_count_pm
tcp_ratio
dst_port_entropy
conn_count_pm
tls_ratio
```

---

## Repository Layout

Use the following repository structure:

```text
post-quantum-iot-main/
├── figures/
│   ├── pq-km.png          <- post-quantum onboarding, key delivery, erasure, and rekeying flow
│   └── sd-iot.png         <- SDN-managed smart-home IoT architecture
│
├── graphs/
│   ├── accuracy.png       <- centralized and federated classification accuracy
│   ├── confusion.png      <- HFL-DC or final classifier confusion matrix
│   ├── energy.png         <- energy-cost comparison
│   ├── hfl.png            <- HFL-DC convergence / robustness behavior
│   ├── performance.png    <- latency, energy, or end-to-end comparison
│   └── tsar.png           <- trust-score and anomaly response trace
│
├── post_quantum_iot.ipynb <- executable paper-aligned prototype
├── README.md              <- GitHub documentation
└── LICENSE                <- project license, if included
```

---

## Requirements

Recommended environment:

- Ubuntu 20.04 / 22.04 or a recent Linux distribution
- Python 3.10
- Jupyter Notebook or JupyterLab
- NumPy
- Pandas
- SciPy
- scikit-learn
- Matplotlib
- Seaborn
- cryptography
- XGBoost
- ProVerif 2.05 for symbolic verification models, if you reproduce the formal-verification part

Install the Python environment:

```bash
conda create -n falcon-sdiot python=3.10 -y
conda activate falcon-sdiot

pip install \
  numpy pandas scipy scikit-learn matplotlib seaborn \
  cryptography xgboost jupyter
```

Optional ProVerif installation:

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

### 2. Create the environment

```bash
conda create -n falcon-sdiot python=3.10 -y
conda activate falcon-sdiot
pip install -q numpy pandas scipy scikit-learn matplotlib seaborn cryptography xgboost jupyter
```

### 3. Run the notebook

```bash
jupyter notebook post_quantum_iot.ipynb
```

Open the notebook and run all cells from top to bottom.

### 4. Optional: export notebook to Python script

```bash
jupyter nbconvert --to script post_quantum_iot.ipynb
python post_quantum_iot.py
```

---

## Expected Pipeline

<p align="justify">
A complete run should generate the synthetic smart-home IoT traffic benchmark, train centralized baseline models, train HFL-DC with non-IID edge partitions, apply differential privacy and Multi-Krum aggregation, evaluate classification behavior, execute manufacturer certificate and PQ-KM registration logic, run LB-ZKA recurring proof-of-possession authentication, simulate TSAR behavior traces with rekeying decisions, and save figures and tables.
</p>

Expected dataset setting from the paper-aligned prototype:

```text
Samples:        24,300
Device types:   27
Features:       10 traffic features
Classes:        A, B, C
```

Generated visual files should appear in:

```text
figures/pq-km.png
figures/sd-iot.png

graphs/accuracy.png
graphs/confusion.png
graphs/energy.png
graphs/hfl.png
graphs/performance.png
graphs/tsar.png
```

---

## Key Results Reported by the Paper Prototype

<p align="justify">
The following values summarize the reported paper-aligned evaluation. Small differences can occur when libraries, seeds, or execution environments change. The TSAR false-positive value is only for the evaluated synthetic scenarios and should not be treated as a general deployment guarantee.
</p>

| Result area | Reported value / behavior |
|---|---|
| Synthetic benchmark size | 24,300 samples |
| Device taxonomy | 27 smart-home device types |
| HFL-DC macro-accuracy | 0.845 under differential privacy |
| Byzantine robustness | 1.3 percentage-point degradation with one Byzantine edge |
| Recurring authentication latency | 26.2% lower than SDN-IoT under the same accounting boundary |
| Recurring authentication energy | 31.8% lower than SDN-IoT under the same accounting boundary |
| Formal verification | All modeled ProVerif queries return SAFE |
| TSAR false-positive result | 0.0% on the evaluated scenario-bounded traces |

---

## Graphs and Visual Outputs

### Classification Accuracy

<p align="center">
  <img src="./graphs/accuracy.png" width="70%" alt="Classification accuracy comparison"/>
</p>

<p align="justify">
This graph compares centralized models and the federated device classifier. Centralized training can use all raw traffic directly, while HFL-DC keeps data at the edge and sends only protected updates.
</p>

---

### Confusion Matrix

<p align="center">
  <img src="./graphs/confusion.png" width="65%" alt="Confusion matrix"/>
</p>

<p align="justify">
The confusion matrix summarizes how the final classifier assigns traffic samples to the three security classes.
</p>

---

### HFL-DC Learning Behavior

<p align="center">
  <img src="./graphs/hfl.png" width="75%" alt="HFL-DC convergence and robustness behavior"/>
</p>

<p align="justify">
This figure shows how the federated classifier behaves across training rounds under privacy noise and robust aggregation.
</p>

---

### TSAR Trust Response

<p align="center">
  <img src="./graphs/tsar.png" width="75%" alt="TSAR trust-score adaptive response"/>
</p>

<p align="justify">
This graph visualizes trust-score changes, anomaly behavior, hardening decisions, and delayed relaxation logic during the evaluated traces.
</p>

---

### Performance Comparison

<p align="center">
  <img src="./graphs/performance.png" width="75%" alt="Performance comparison"/>
</p>

<p align="justify">
This figure reports the recurring authentication performance comparison under the selected accounting boundary.
</p>

---

### Energy Comparison

<p align="center">
  <img src="./graphs/energy.png" width="75%" alt="Energy comparison"/>
</p>

<p align="justify">
This graph highlights energy costs across the evaluated authentication schemes or protocol stages.
</p>

---

## Notebook Code Tour

```text
post_quantum_iot.ipynb
├── Imports, configuration, random seeds, and plotting setup
├── Device taxonomy, traffic profiles, MUD priors, and class mapping
├── Timing, energy, and protocol-accounting constants
├── Differential-privacy accounting helper
├── Cryptographic helper utilities
├── Manufacturer certificate authority
├── ML-KEM-style interface simulation
├── ML-DSA-style interface simulation
├── PQ-KM onboarding and transport-key distribution
├── LB-ZKA proof generation and verification
├── HFL-DC federated training, clipping, noise, Multi-Krum, and prediction
├── TSAR trust scoring, hardening, relaxation, and rekeying simulation
├── Synthetic dataset generation
├── Centralized baseline model training
├── Figure generation
├── Table printing
└── Main execution pipeline
```

---

## Reproducibility Notes

<p align="justify">
Use the same Python version and package versions when comparing numbers with the manuscript. Keep deterministic seeds fixed where the notebook defines them. XGBoost may introduce small numeric differences across platforms. Differential privacy adds controlled noise, so exact round-level values may vary. For paper figures, save generated outputs into the <code>graphs/</code> and <code>figures/</code> directories shown above. The current benchmark is synthetic and curated for controlled evaluation; real deployment needs validation on broader household traffic traces.
</p>

---

## Formal Verification

<p align="justify">
The manuscript reports ProVerif 2.05 verification for the modeled onboarding, rekeying, and proof-binding logic. The verification model focuses on symbolic protocol properties such as authentication, secrecy, replay resistance, and binding checks.
</p>

Expected verification summary:

```text
Modeled symbolic security queries: SAFE
```

<p align="justify">
Use ProVerif outputs as protocol-model evidence, not as proof that the Python prototype is production hardened.
</p>

---

## Security and Deployment Notes

<p align="justify">
This repository is a research prototype. Before using similar logic in a real deployment, replace the simulated ML-KEM and ML-DSA classes with certified implementations, use hardware-backed key storage on gateways or controllers, add secure boot and firmware integrity checks, harden the SDN controller, and validate TSAR thresholds on real traffic traces from diverse devices and vendors.
</p>

Additional deployment points:

- Consider redundant or threshold-based controller designs.
- Compress or fragment LB-ZKA proof payloads for low-rate links such as IEEE 802.15.4 and 6LoWPAN.
- Revisit differential-privacy budgets for long training schedules.
- Add real device attestation before allowing sensitive class transitions.

---

## Troubleshooting

### `ModuleNotFoundError: No module named 'xgboost'`

Install XGBoost:

```bash
pip install xgboost
```

<p align="justify">
If the notebook contains an import fallback, most non-XGBoost logic can still run, but the full baseline comparison is best reproduced with XGBoost installed.
</p>

---

### `cryptography` installation fails

Upgrade packaging tools and reinstall:

```bash
python -m pip install --upgrade pip setuptools wheel
pip install cryptography
```

---

### Figures are not displayed in GitHub

Check that file names match the README exactly:

```text
figures/pq-km.png
figures/sd-iot.png
graphs/accuracy.png
graphs/confusion.png
graphs/energy.png
graphs/hfl.png
graphs/performance.png
graphs/tsar.png
```

<p align="justify">
Linux paths are case-sensitive, so <code>pq-km.png</code> and <code>PQ-KM.png</code> are different files.
</p>

---

### Figures are not generated by the notebook

Create the output directories and rerun the figure cells:

```bash
mkdir -p figures graphs
```

---

### ProVerif is not found

Add the ProVerif binary location to the shell path:

```bash
export PATH=$PATH:/path/to/proverif2.05
proverif -version
```

---

### Results differ slightly from the paper

<p align="justify">
Small differences can occur because of package versions, random seeds, hardware, and platform-specific numeric behavior. To record your environment, run the following command:
</p>

```bash
python --version
pip freeze > requirements_lock.txt
```

---

## Limitations

<p align="justify">
The traffic benchmark is synthetic and supports controlled analysis, but it does not replace broad real-world smart-home validation. The proof payload is larger than classical or symmetric-only authentication messages, so constrained links may require fragmentation. The edge gateway is semi-trusted in the current system model. Differential-privacy loss accumulates across federated rounds. TSAR results are scenario-bounded and should be retested under longer attacks, firmware changes, and multi-vendor traffic. The Python cryptographic interfaces are for research reproducibility and should not be used as production post-quantum cryptography.
</p>



---

## License

<p align="justify">
This project is released for academic and research use. See <code>LICENSE</code> for details if included in the repository.
</p>

---

## Acknowledgements

<p align="justify">
This repository accompanies the FALCON-SDIoT research project on adaptive post-quantum authentication for SDN-managed smart-home IoT. The implementation is designed for reproducibility, protocol analysis, visualization, and future extension toward deployable post-quantum smart-home security systems.
</p>
