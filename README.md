# GlassIDS 🛡️

### Explainable, Self-Tested Intrusion Detection & Autonomous Response System

GlassIDS is a machine-learning-based Network Intrusion Detection System (IDS) that detects malicious network traffic, explains why each alert fired, and decides how to respond using an LLM-powered agent.

It goes beyond training a model on a public dataset. It combines network traffic analysis, machine learning, explainability, controlled real-world attack testing, and an auditable agentic response layer in one pipeline.

## ✨ Highlights

* 🔍 **Explainable:** every alert comes with a SHAP breakdown of the features that drove it
* 🧪 **Self-tested:** evaluated on attacks generated in a private lab with Kali Linux, not only on a public benchmark
* 🤖 **Agentic response:** an LLM agent reads the evidence and chooses to block, rate-limit, log, or escalate
* 📜 **Auditable:** every decision, its reasoning, and the evidence behind it are written to a decision log
* ⚖️ **Honest evaluation:** class-imbalance-aware metrics instead of headline accuracy

## 🔄 How It Works

```text
Network Traffic
      ↓
Packet Capture
      ↓
Flow / Feature Extraction
      ↓
Machine Learning Detection
      ↓
SHAP Explanation
      ↓
LLM Decision Agent
      ↓
Firewall Response (nftables)
      ↓
Decision Log
```

The detector works on network-flow features rather than individual packets. A flow summarizes a whole conversation between two machines (source and destination IP, ports, and protocol) into one row of statistics such as duration, packet counts, byte counts, and TCP flag counts.

## 🛠️ Technologies

* **Python** for the main development
* **tcpdump & Wireshark** for network traffic capture and inspection
* **CIC-IDS2017** as the primary public dataset
* **CICFlowMeter / Zeek** for extracting flow features
* **XGBoost** as the machine-learning baseline
* **PyTorch** for the deep-learning models (MLP and 1D-CNN)
* **SHAP** for model explainability
* **Kali Linux** for controlled security testing (nmap, hydra, slowhttptest, hping3)
* **Anthropic / Claude API** for the agentic decision layer
* **nftables** for firewall-based responses
* **Suricata** for comparison against a traditional rule-based IDS
* **Bandit & Semgrep** for security scanning of the project code

## 🧱 Project Components

### 1. Data & Feature Pipeline 📦
CIC-IDS2017 was explored and cleaned: missing and infinite values were handled, duplicates removed, labels checked, and the class imbalance documented. A self-built feature-extraction pipeline was validated against the dataset's existing feature data, so the same pipeline can process traffic captured in the lab.

### 2. Baseline Model 🌲
An XGBoost classifier was trained and evaluated across all 15 traffic classes. Evaluation focuses on recall, ROC-AUC, per-class precision/recall/F1, and full confusion matrices, since accuracy alone is misleading when attack traffic is a small minority.

📄 Full methodology and results: [`docs/Baseline-model_Deliverable.md`](docs/Baseline-model_Deliverable.md)

### 3. Deep Learning Comparison 🧠
A PyTorch MLP and a 1D-CNN were trained and compared against the XGBoost baseline using identical features, the same split, and the same class-imbalance handling, so any difference comes from the modeling approach and not from inconsistent setup.

📄 Full comparison: [`docs/PyTorch-XGBoost_Comparison.md`](docs/PyTorch-XGBoost_Comparison.md)

### 4. Explainability 🔍
SHAP explanations were generated for all three models, with both global views (what matters across many predictions) and local views (why one specific alert fired). The analysis explains two known confusion patterns: Bot traffic versus BENIGN false positives, and overlap between Web Attack subtypes.

📄 Full write-up: [`docs/SHAP_Explainability.md`](docs/SHAP_Explainability.md)

### 5. Self-Generated Attack Testing 🧪
A disposable target virtual machine sits on an isolated lab network with a Kali Linux attacker. Kali generated port scans (nmap), SSH brute-force attempts (hydra), and slow denial-of-service traffic (slowhttptest / hping3). Each attack was captured with tcpdump, passed through the same feature-extraction and detection pipeline, and scored for detection rate and false-positive rate on previously unseen traffic.

This tests whether a model trained on older public data can recognize attacks generated independently in the lab.

### 6. Agentic Decision Layer 🤖
Instead of automatically blocking every high-confidence alert, an LLM agent receives the model's prediction, confidence, flow information, and SHAP explanation. It chooses a response and states its reasoning in plain language before any action is taken.

| Action | What it means |
| --- | --- |
| 🚫 **Block** | Generate an `nftables` rule to drop the traffic |
| 🐢 **Rate-limit** | Generate an `nftables` rule to throttle the traffic |
| 📝 **Log-only** | Record the event with no firewall action |
| 🙋 **Escalate** | Flag for human review, logged with no automatic action |

Every decision, the agent's reasoning, and the evidence behind it are written to a decision log, so the full path from packet to explanation to action can be reviewed later.

### 7. Extended Analysis 🔬
* **Autoencoder:** trained on benign traffic only, using reconstruction error as an anomaly score to flag attack types the classifier has never seen labeled examples of
* **Suricata comparison:** the same captures run through a traditional rule-based IDS, compared against the ML detector's alerts
* **Code security scan:** Bandit and Semgrep run over the project code, with findings mapped to OWASP / CWE categories
* **Threat model:** a plain statement of what GlassIDS does and does not defend against (see below)

## 🔍 Explainability in Action

Every alert can show why it was raised, for example:

```text
Prediction: Malicious

Important contributing features:
- Flow duration
- Packet count
- Packet size
- TCP flags

Confidence: High
```

These explanations are also the evidence supplied to the response agent, so the agent's decision is grounded in the same reasoning a human analyst would see.

## 🧪 Testing Approach

GlassIDS is not evaluated only on the public dataset. A model can score very well on a benchmark it was trained on and still fail on traffic it has never seen, which is exactly what it would face in the real world.

To confront that gap:

1. A disposable target VM is attacked from Kali Linux inside an isolated network
2. The traffic is captured and verified in Wireshark
3. The captures go through the same pipeline as the dataset traffic
4. Detection rate per attack type and false-positive rate on ordinary traffic are reported

## 🛡️ Threat Model

**GlassIDS is designed to detect:**
* Port scans, brute-force login attempts, and slow denial-of-service traffic
* Attack patterns that look anomalous at the flow level

**GlassIDS does not claim to defend against:**
* Encrypted command-and-control traffic with no flow-level anomaly
* Attacks that closely imitate normal traffic
* Payload-level attacks that need deep packet inspection
* Adversaries who deliberately craft traffic to evade the model

The LLM agent adds judgment but is not infallible. Escalation exists so that uncertain cases go to a human instead of triggering an automatic block, and every decision is logged for review.

## ⚠️ Limitations

* The primary training data (CIC-IDS2017) dates from 2017, so newer attack styles may be under-represented
* Lab-generated attacks cover a limited set of attack types
* Firewall actions are generated as `nftables` rules and are not necessarily enforced on a live network
* Class imbalance means rare attack classes are harder to learn and evaluate than common ones

## 🚀 Getting Started

```bash
# Clone the repository
git clone https://github.com/<your-username>/glassids.git
cd glassids

# Create and activate a virtual environment
python -m venv glassids-env
source glassids-env/bin/activate      # Windows: glassids-env\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

To use the agentic decision layer, set your Anthropic API key as an environment variable:

```bash
export ANTHROPIC_API_KEY="your-key-here"
```

## 📁 Repository Structure

```text
glassids/
├── docs/        # Methodology and results write-ups
├── src/         # Pipeline, models, explainability, and agent code
├── data/        # Dataset and extracted features (not tracked)
└── README.md
```

## ⚖️ Responsible Use

All attack traffic was generated only against a disposable virtual machine that I own, inside an isolated lab network. Do not use these techniques against systems you do not own or have explicit permission to test.

## 📚 Documentation

* [`docs/Baseline-model_Deliverable.md`](docs/Baseline-model_Deliverable.md): XGBoost baseline
* [`docs/PyTorch-XGBoost_Comparison.md`](docs/PyTorch-XGBoost_Comparison.md): architecture comparison
* [`docs/SHAP_Explainability.md`](docs/SHAP_Explainability.md): explainability analysis