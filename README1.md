# hybrid-ids-ips-ml

A real-time hybrid Intrusion Detection and Prevention System (IDS/IPS) built as a graduation project. It combines a deep learning classifier trained on the CICIDS2017 dataset with an unsupervised Isolation Forest anomaly detector, an IOC matching engine, a statistical flow rate guard, an alert correlation engine, and an active IPS enforcement layer — all monitored through a live SOC-style web dashboard.

---

## Overview

Most intrusion detection systems rely on a single detection method, which leads to either too many false positives or missed attacks. This project takes a **defense-in-depth** approach by stacking multiple independent detection layers. Each layer covers the blind spots of the others, so an attack that slips past one engine is likely caught by another.

The system captures live network traffic, reconstructs TCP/UDP flows, extracts features, and runs them through all detection layers simultaneously. Any confirmed threat can trigger an IPS response to block the source.

---

## Detection Layers

### 1. CICIDS2017 Deep Learning Classifier
A neural network model trained on the CICIDS2017 benchmark dataset. It classifies network flows into attack categories including PortScan, Slow HTTP / Slowloris, DoS Hulk, DDoS, Brute Force, and more. Per-class confidence thresholds are applied to each attack type rather than a single global threshold, which reduces false positives for low-confidence classes while staying sensitive for well-represented ones.

### 2. Isolation Forest Anomaly Detector
An unsupervised model trained on the operator's own normal traffic baseline. Any flow that looks statistically unusual compared to that baseline triggers an anomaly alert — even if the CICIDS classifier considers it benign. This layer catches zero-day-style behavior and attack tools the supervised model has never seen.

### 3. IOC Engine
Checks every flow against blacklisted IP addresses and ports loaded from CSV files. Matches are flagged immediately, independent of any ML inference. Whitelisted IPs are excluded from all alerting.

### 4. Flow Rate Statistical Guard
Monitors packets-per-second and bytes-per-second over a sliding window. High-rate floods such as HULK DDoS and UDP floods often produce many short flows that individually look benign to the ML model. The rate guard catches these purely through statistical behavior, with no reliance on learned patterns.

### 5. Alert Correlation Engine
Combines weak signals from the same source IP into escalated, high-confidence alerts. A single ML alert at moderate confidence might be a false positive. The same alert combined with a rate flood from the same IP within 30 seconds is almost certainly a real attack. Correlation rules include:

| Condition | Severity |
|-----------|----------|
| ML alert + IOC blacklist hit from same source | CRITICAL |
| Rate flood + ML alert from same source | CRITICAL |
| Anomaly + ML alert from same source | CRITICAL |
| 10+ unique destination IPs scanned in 60 seconds | HIGH |
| 3+ probability guard alerts from same source | HIGH |

### 6. IPS Enforcement
When a threat is confirmed, the IPS layer can insert firewall rules to block the source IP. It supports a `dry-run` mode (logs what it would block without acting) and an `active` mode (enforces blocks in real time). This makes the system a full IDS/IPS rather than detection-only.

---

## SOC Dashboard

A local web dashboard provides real-time visibility into what the system is detecting.

**Alerts page** — shows all IOC, ML, IPS-triggering, and correlated alerts with:
- Alert timeline chart
- Alert source / engine distribution
- Alert type breakdown
- Top alerting source IPs

**All Logs page** — merges traffic logs with alert records and shows:
- Log timeline
- Protocol distribution
- Top source and destination IPs

The dashboard auto-refreshes every 3 seconds and exposes JSON API endpoints (`/api/alerts`, `/api/traffic`, `/api/summary`) that can be consumed by external tools.

---

## Attack Coverage

| Attack Type | Detection Methods |
|-------------|-------------------|
| Slow HTTP / Slowloris | ML + Anomaly + Correlation |
| HULK DDoS | ML + Rate Guard + Anomaly |
| Port Scan | ML + Correlation |
| Unknown DDoS tools | Rate Guard + Anomaly |
| ICMP Flood | IOC + Rate Guard |
| Blacklisted IPs/Ports | IOC Engine |
| Multi-stage attacks | Correlation → CRITICAL alert |

---

## Project Structure

```
.
├── main.py                        # Entry point — starts packet capture and all engines
├── packet_capture.py              # Live traffic capture using Scapy
├── cicids_live_features.py        # Flow feature extraction (CICIDS2017 format)
├── ml_engine.py                   # CICIDS2017 deep learning classifier
├── anomaly_engine.py              # Isolation Forest anomaly detector
├── ioc_engine.py                  # IOC blacklist/whitelist matching
├── ioc_loader.py                  # Loads IOC CSV files
├── flow_rate_guard.py             # Statistical rate-based flood detection
├── correlation_engine.py          # Multi-signal alert correlation
├── ips_enforcer.py                # IPS firewall rule enforcement
├── alert_writer.py                # Writes alerts to logs/alerts.jsonl
├── traffic_writer.py              # Writes traffic logs to logs/traffic.jsonl
├── dashboard.py                   # SOC web dashboard (FastAPI)
├── models/                        # Trained ML models and normalizer
├── data/                          # Blacklist/whitelist IP and port CSVs
├── docs/                          # Phase guides and technical documentation
├── run_real_life_ids.sh           # Run IDS on a live interface
├── run_dashboard.sh               # Launch the dashboard
└── requirements.txt
```

---

## Requirements

- Python 3.10+
- Linux (tested on Kali Linux)
- Root / sudo access for packet capture and IPS enforcement

Install dependencies:

```bash
pip install -r requirements.txt
```

---

## Running the System

**Start the IDS on a live interface:**

```bash
sudo python main.py --iface eth0 --log-traffic --ips-mode dry-run --ml-mode predict
```

**Start the dashboard (separate terminal):**

```bash
python dashboard.py --host 0.0.0.0 --port 8000
```

Open `http://127.0.0.1:8000` in your browser.

**Run offline self-tests:**

```bash
python offline_ml_portscan_selftest.py
python offline_ml_slow_http_selftest.py
```

---

## Training the Anomaly Detector on Your Own Traffic

```bash
# Step 1: capture normal traffic
sudo python main.py --iface eth0 --ml-mode features-only --alerts logs/normal_flows.jsonl

# Step 2: train the Isolation Forest
python anomaly_engine.py --train --normal-log logs/normal_flows.jsonl --model-out models/isolation_forest.pkl

# Step 3: restart the IDS — model loads automatically
sudo ./run_ml_only_ids.sh eth0
```

---

## Academic Context

This project was developed as a graduation project exploring hybrid network intrusion detection. The core academic contribution is the combination of:

- A **supervised** deep learning classifier (CICIDS2017) for known attack pattern recognition
- An **unsupervised** Isolation Forest trained on environment-specific baseline traffic for anomaly detection
- A **statistical** flow rate guard for high-volume flood detection independent of learned patterns
- A **correlation engine** that fuses weak signals into high-confidence, actionable alerts

This layered architecture demonstrates that defense-in-depth at the detection layer significantly reduces both false negatives (missed attacks) and false positives (alert fatigue) compared to any single detection method alone.
