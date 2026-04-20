# ML-Powered-Encrypted-Traffic-Fingerprinting-System
Architected ML-Powered Encrypted Traffic Fingerprinting System achieving 94.7% precision in detecting C2 beaconing and data exfiltration within TLS Flows. Integrated Zeek, ELK Stack and Ensemble ML models with sub-second detection latency, demonstrating capability to detect 'invisible threats' where traditional signature-based systems fail. 

# Encrypted Traffic Threat Detection — ML-Based C2 Beaconing Classifier

[![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Precision](https://img.shields.io/badge/Precision-94.7%25-brightgreen.svg)]()
[![Dataset](https://img.shields.io/badge/Dataset-CICIDS2017-orange.svg)]()

> Detect Command-and-Control (C2) beaconing hidden inside encrypted HTTPS/TLS traffic — without decrypting a single packet.

---

## The Problem

Modern malware increasingly tunnels C2 communication over HTTPS to blend in with legitimate traffic. Traditional signature-based detection fails entirely once traffic is encrypted. Deep Packet Inspection (DPI) is blocked by TLS 1.3. Security teams are effectively flying blind.

This project answers: **can we detect C2 beaconing using only TLS handshake metadata and traffic behavioural patterns — with no decryption?**

**Answer: Yes. 94.7% precision.**

---

## How It Works

Rather than inspecting packet payloads, the model extracts **behavioural fingerprints** from:

- TLS handshake metadata (cipher suites, extensions, SNI patterns)
- Flow-level timing features (inter-arrival times, periodicity scores)
- Packet size distributions (entropy, variance, burst patterns)
- JA3/JA3S fingerprints for client-server TLS profiling
- Connection frequency and destination reputation signals

These features are fed into a **Random Forest classifier** trained on labelled benign and C2 traffic flows.

```
Raw PCAP → Zeek → Feature Extraction → ML Model → Alert / Benign
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     TRAFFIC INGESTION                        │
│          Zeek Network Monitor (Live / PCAP replay)           │
└────────────────────────┬────────────────────────────────────┘
                         │ conn.log, ssl.log, x509.log
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   FEATURE ENGINEERING                        │
│   TLS Metadata · Flow Timing · JA3 Fingerprints · Entropy   │
└────────────────────────┬────────────────────────────────────┘
                         │ Feature vectors (numpy arrays)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   ML CLASSIFICATION                          │
│        Random Forest (scikit-learn) — 150 estimators        │
│        Trained on CICIDS2017 + malware-traffic-analysis.net  │
└────────────────────────┬────────────────────────────────────┘
                         │ Prediction + confidence score
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      ALERTING                                │
│     SIEM Export (QRadar/Splunk) · JSON output · CLI alert   │
└─────────────────────────────────────────────────────────────┘
```

---

## Results

| Metric | Score |
|---|---|
| **Precision** | **94.7%** |
| Recall | 91.2% |
| F1 Score | 92.9% |
| False Positive Rate | 2.3% |
| Avg Inference Time | 12ms per flow |

Tested against: Cobalt Strike beaconing, Metasploit HTTPS reverse shells, Sliver C2, and benign enterprise HTTPS traffic.

---

## Quick Start

### Prerequisites

```bash
# System dependencies
sudo apt-get install zeek python3-pip

# Python dependencies
pip install -r requirements.txt
```

### Run on a PCAP file

```bash
# Step 1: Extract features using Zeek
zeek -r your_capture.pcap scripts/extract_features.zeek

# Step 2: Run ML classifier
python detect.py --input conn.log --output results.json

# Step 3: View alerts
cat results.json | python visualise.py
```

### Live capture mode

```bash
sudo python detect.py --interface eth0 --threshold 0.85
```

---

## Project Structure

```
encrypted-traffic-detection/
├── data/
│   ├── sample_pcaps/          # Sample PCAPs for testing (benign + C2)
│   └── feature_sets/          # Pre-extracted feature CSVs
├── models/
│   └── rf_classifier.pkl      # Trained Random Forest model
├── scripts/
│   ├── extract_features.zeek  # Zeek feature extraction script
│   ├── feature_engineer.py    # Python feature pipeline
│   └── train.py               # Model training script
├── detect.py                  # Main detection script
├── visualise.py               # Alert visualisation
├── requirements.txt
└── README.md
```

---

## Training Your Own Model

```bash
# Download CICIDS2017 dataset
wget https://www.unb.ca/cic/datasets/ids-2017.html

# Run training pipeline
python scripts/train.py \
  --data data/feature_sets/combined_flows.csv \
  --output models/rf_classifier.pkl \
  --estimators 150 \
  --test-split 0.2

# Output: Precision, Recall, F1, Confusion Matrix
```

---

## Detection Rules (Sigma)

This project also ships companion Sigma detection rules for SIEM integration:

- `rules/c2_beaconing_periodic.yml` — detects regular interval connections
- `rules/tls_ja3_anomaly.yml` — flags rare JA3 fingerprints
- `rules/high_entropy_dns.yml` — DNS-over-HTTPS C2 detection

Convert to your SIEM format:
```bash
sigmac -t splunk rules/c2_beaconing_periodic.yml
sigmac -t qradar rules/c2_beaconing_periodic.yml
```

---

## Datasets Used

| Dataset | Source | Usage |
|---|---|---|
| CICIDS2017 | University of New Brunswick | Benign + attack flows |
| malware-traffic-analysis.net | Public PCAPs | Real C2 samples |
| Custom captures | Lab environment | Cobalt Strike, Sliver |

All training data is either publicly available or generated in an isolated lab. No client or production data used.

---

## Roadmap

- [ ] Add LSTM-based temporal model for longer sequence detection
- [ ] Suricata rule export
- [ ] Docker deployment with live Zeek integration
- [ ] REST API endpoint for SOAR integration
- [ ] Support for QUIC/HTTP3 traffic analysis

---

## Author

**Charudatta Padhye**

Network Security | Detection Engineering | ML for Security

[LinkedIn](https://www.linkedin.com/in/charudattapadhye/) 

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*If this project helped you, please give it a star. It helps other practitioners find it.*
