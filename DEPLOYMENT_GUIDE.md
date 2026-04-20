# Deployment Guide — Encrypted Traffic Threat Detection

This guide covers three deployment modes: local testing on a PCAP file, live network monitoring on a Linux host, and Docker-based deployment for production or team use.

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Python | 3.10+ | 3.11 recommended |
| Zeek | 6.0+ | For feature extraction |
| RAM | 4 GB minimum | 8 GB recommended for live mode |
| OS | Ubuntu 22.04 / Debian 11 | Other Linux distros supported |

---

## Option 1 — Local / PCAP Analysis (Quickest Start)

### Step 1: Clone the repository

```bash
git clone https://github.com/charudatta-padhye/encrypted-traffic-detection.git
cd encrypted-traffic-detection
```

### Step 2: Install dependencies

```bash
# Install Zeek (Ubuntu)
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' \
  | sudo tee /etc/apt/sources.list.d/security:zeek.list
sudo apt-get update && sudo apt-get install zeek

# Install Python packages
pip install -r requirements.txt
```

**requirements.txt contains:**
```
scikit-learn==1.4.0
pandas==2.2.0
numpy==1.26.0
pyshark==0.6
scapy==2.5.0
joblib==1.3.2
rich==13.7.0   # for CLI output formatting
```

### Step 3: Analyse a PCAP

```bash
# Extract features with Zeek
zeek -r sample.pcap scripts/extract_features.zeek

# Run detection
python detect.py --input conn.log --output results.json --threshold 0.80

# Inspect results
python visualise.py --input results.json
```

**Output example:**
```
[ALERT] Flow 192.168.1.45 → 185.220.101.47:443
  Confidence : 0.94
  JA3        : 769,47-53-5-10-49196...,0-23-65281
  Reason     : Periodic beaconing (interval: 60.1s ± 1.2s), rare JA3 fingerprint
  MITRE      : T1071.001 (Application Layer Protocol: Web Protocols)

[BENIGN] Flow 192.168.1.45 → 142.250.80.46:443
  Confidence : 0.02 (benign)
```

---

## Option 2 — Live Network Monitoring

Run on a Linux host with access to a network interface (span port, tap, or inline).

### Step 1: Verify interface access

```bash
ip link show
# Use the interface that receives the traffic you want to monitor
# Common: eth0, ens3, bond0
```

### Step 2: Start Zeek in live mode

```bash
# Start Zeek on your interface (requires root or CAP_NET_RAW)
sudo zeek -i eth0 scripts/extract_features.zeek &
```

### Step 3: Start the detection loop

```bash
# Watches conn.log for new flows and classifies in near-real-time
sudo python detect.py \
  --interface eth0 \
  --threshold 0.85 \
  --output-mode syslog \
  --siem-host 192.168.1.100 \
  --siem-port 514
```

**Output modes:**
- `--output-mode json` — writes `results.json` (default)
- `--output-mode syslog` — sends CEF-formatted alerts to your SIEM
- `--output-mode stdout` — prints alerts to terminal

### Step 4: SIEM integration

**QRadar (Log Source — Syslog):**
```
Log Source Type  : Universal DSM
Protocol         : Syslog
Listen Port      : 514
Log Source Identifier: encrypted-traffic-detector
```

**Splunk (Universal Forwarder):**
```bash
# Add to inputs.conf
[monitor:///path/to/results.json]
index = security
sourcetype = encrypted_traffic_ml
```

---

## Option 3 — Docker Deployment (Recommended for Teams)

### Step 1: Build the image

```bash
docker build -t encrypted-traffic-detector:latest .
```

### Step 2: Run with PCAP volume mount

```bash
docker run --rm \
  -v /path/to/pcaps:/data/pcaps \
  -v /path/to/output:/data/output \
  encrypted-traffic-detector:latest \
  --input /data/pcaps/capture.pcap \
  --output /data/output/results.json
```

### Step 3: Run in live mode (host networking required)

```bash
docker run --rm \
  --network host \
  --cap-add NET_RAW \
  encrypted-traffic-detector:latest \
  --interface eth0 \
  --threshold 0.85
```

---

## Configuration Reference

All options can be set via `config.yaml` or CLI flags. CLI flags override config.

```yaml
# config.yaml
model:
  path: models/rf_classifier.pkl
  threshold: 0.85           # Alert above this confidence score
  
features:
  ja3_enabled: true         # Include JA3 fingerprinting
  timing_window: 300        # Seconds to analyse for beaconing periodicity
  
output:
  mode: json                # json | syslog | stdout
  siem_host: null           # Set if using syslog output
  siem_port: 514

alerts:
  mitre_mapping: true       # Include MITRE ATT&CK technique IDs in output
```

---

## Tuning the Threshold

The `--threshold` controls the confidence score above which a flow is flagged:

| Threshold | Precision | Recall | Recommended for |
|---|---|---|---|
| 0.70 | 89.2% | 95.1% | Threat hunting (high recall) |
| **0.85** | **94.7%** | **91.2%** | **General use (default)** |
| 0.95 | 98.1% | 74.3% | Low-noise SOC alerting |

Lower the threshold when hunting. Raise it for production SIEM alerting to reduce analyst fatigue.

---

## Troubleshooting

**Zeek not found:**
```bash
export PATH=$PATH:/opt/zeek/bin
# Add to ~/.bashrc for persistence
```

**Permission denied on interface:**
```bash
sudo setcap cap_net_raw,cap_net_admin=eip $(which zeek)
sudo setcap cap_net_raw=eip $(which python3)
```

**Model file missing:**
```bash
# Retrain from scratch
python scripts/train.py --data data/feature_sets/combined_flows.csv
```

---

## Testing the Installation

```bash
# Runs against included test PCAP (contains known C2 sample)
python detect.py --input data/sample_pcaps/test_c2.pcap --test-mode

# Expected output: 3 ALERTS, 47 BENIGN, Precision check PASS
```

---

*For issues, open a GitHub Issue. Include your Zeek version, OS, and the full error output.*
