# GlassIDS 🛡️

### Explainable, Self-Tested Intrusion Detection & Autonomous Response System

**Status:** 🚧 Work in Progress

GlassIDS is a machine-learning-based Network Intrusion Detection System (IDS) designed to detect malicious network traffic, explain why traffic was flagged, and eventually respond to threats automatically.

The project is being developed to go beyond simply training a model on a public dataset. It combines network traffic analysis, machine learning, explainability, controlled real-world testing, and an agentic response layer.

## 🔄 What GlassIDS Does

The planned system follows this general pipeline:

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
Response
      ↓
Decision Log
```

The detector will work primarily with network-flow features rather than analyzing individual packets directly.

## 🛠️ Technologies

The project currently uses or plans to use:

* **Python** for the main development
* **tcpdump & Wireshark** for network traffic capture and inspection
* **CIC-IDS2017** as the primary public dataset
* **CICFlowMeter / Zeek** for extracting flow features
* **XGBoost** as the initial machine-learning baseline
* **PyTorch** for a deep-learning model
* **SHAP** for model explainability
* **Kali Linux** for controlled security testing
* **Anthropic / Claude API** for the agentic decision layer
* **nftables** for firewall-based responses

## 📌 Current Progress

### ✅ Completed

The networking, data-preparation, modeling, and explainability stages are done:

* Development and security-testing environment set up
* Network traffic captured and inspected
* Basic packet and flow concepts understood
* CIC-IDS2017 obtained and explored
* Flow-feature data cleaned, with missing/invalid values checked and class imbalance examined
* Baseline XGBoost model trained and evaluated across all 15 traffic classes
* Full baseline methodology and results documented: see [`docs/Baseline-model_Deliverable.md`](docs/Baseline-model_Deliverable.md)
* PyTorch MLP and 1D-CNN trained and evaluated against the baseline under identical features, split, and class-imbalance handling
* Full architecture-comparison methodology and findings documented: see [`docs/PyTorch-XGBoost_Comparison.md`](docs/PyTorch-XGBoost_Comparison.md)
* SHAP-based explanations generated for all three models, covering both known confusion patterns (Bot vs. BENIGN false positives, Web Attack subtype overlap) with global and local (single-alert) explanations
* Full SHAP methodology and findings documented: see [`docs/SHAP_Explainability.md`](docs/SHAP_Explainability.md)

### 🔄 In Progress

* Initial self-extracted feature-extraction pipeline, validated against the dataset's existing feature data

### 🗺️ Planned

With the modeling and explainability work done, the project moves toward the agentic layer next, with live attack-generation testing deliberately saved for last:

* Connecting SHAP explanations to an LLM decision agent
* Allowing the agent to choose between block, rate-limit, log-only, or escalate
* Connecting appropriate responses to `nftables`
* Recording decisions and reasoning in an auditable log
* Setting up the controlled attack-generation environment (target VM and Kali networking) and testing against Kali-generated attacks (port scans, brute-force attempts, slow denial-of-service traffic)
* Measuring detection and false-positive performance on previously unseen traffic

Additional planned work includes an autoencoder-based anomaly detector, a comparison with Suricata, security scanning of the project code, and a threat-model section.

## 🧪 Testing Approach

An important part of GlassIDS is that it will not be evaluated only on the public dataset.

A disposable target virtual machine will be used for controlled security testing. Kali Linux will generate traffic such as port scans, SSH brute-force attempts, and slow denial-of-service traffic.

The generated traffic will then be captured and passed through the same feature-extraction and detection pipeline.

This allows the project to investigate whether a model trained on older public data can recognize attacks generated independently in the lab.

## 🔍 Explainability

GlassIDS is designed to make its predictions understandable rather than treating the model as a black box.

SHAP is used to identify which network-flow features contributed to a prediction, both globally (what matters across many predictions) and locally (why one specific alert fired).

For example, an alert should eventually be able to show:

```text
Prediction: Malicious

Important contributing features:
- Flow duration
- Packet count
- Packet size
- TCP flags

Confidence: High
```

These explanations will also become part of the evidence supplied to the response agent.

## 🤖 Autonomous Response

The final stage of the project introduces an LLM-based decision layer.

Instead of automatically blocking every high-confidence alert, the agent will receive the model's prediction, confidence, flow information, and SHAP explanation and decide what response is appropriate.

Possible responses are:

* **Block**
* **Rate-limit**
* **Log-only**
* **Escalate for human review**

The reasoning and resulting action will be recorded so that the complete path from detection to response can be reviewed later.

## 📍 Current Status

GlassIDS is **actively under development**.

The baseline XGBoost model, the PyTorch architecture comparison, and SHAP-based explainability across all three models are complete, including a mechanistic explanation for both known confusion patterns and one worked local example of a false positive (full write-ups for all three in `docs/`). 🎯

Currently moving on to the LLM-based agentic decision layer. The controlled attack-generation and validation testing has been deliberately deferred to the end of the project, since it needs its own infrastructure and doesn't block the explainability or agentic work above.