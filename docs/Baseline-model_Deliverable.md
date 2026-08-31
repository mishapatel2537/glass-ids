# GlassIDS: Baseline Intrusion Detection Model 🛡️

## 🎯 Objective

Establish a baseline multiclass network intrusion detection model on CIC-IDS2017, providing a benchmark for future explainability work and tuning.

## 📊 Data

- **Source:** CIC-IDS2017 (Kaggle mirror of the pruned `MachineLearningCSV` release)
- **Cleaning:** dropped ~2,867 rows with NaN/inf values, removed duplicate flows, fixed a mangled-dash label corruption in the Web Attack labels, stripped whitespace from column headers
- **Final size:** 2,520,798 flows (down from 2,830,743 pre-cleaning; the ~310K row drop is consistent with CIC-IDS2017's well-documented duplicate-flow issue, not a data-loss bug)
- **Classes:** BENIGN plus 14 attack types, severely imbalanced, ranging from BENIGN (2,095,057) down to Heartbleed (11), Web Attack-SQL Injection (21), and Infiltration (36)
- **Features:** 78 flow-level statistics from CICFlowMeter (packet counts/sizes, inter-arrival times, TCP flags, duration, byte rates, etc.), no packet payload content

## 🧹 Feature Selection

Two columns were deliberately excluded from the feature set beyond the obvious label columns:

- **`is_attack`**: a binary column derived directly from `Label` during cleaning. Including it would be direct label leakage, since the model would essentially be reading the answer off a renamed copy of itself.
- **`Destination Port`**: technically a legitimate feature in this dataset, but it near-perfectly identifies FTP-Patator (port 21) and SSH-Patator (port 22). Keeping it would let the model shortcut those two classes via a port lookup instead of learning actual traffic behavior, which would undermine future explainability work (a "the model looked at the port number" explanation isn't a useful security insight). After training, dropping it turned out not to hurt Patator detection at all: both classes still scored 0.998+ on precision and recall, confirming the behavioral signal alone was enough. 🎯

## ✂️ Train/Test Split Methodology

CIC-IDS2017's attacks are day-exclusive by design (the DoS family only appears in Wednesday's capture, Web Attacks only in Thursday morning's, Patator attacks only in Tuesday's, and so on). Two temporally-ordered split strategies were tried and rejected before landing on the final approach:

1. **Whole-day cutoff** (train on most capture days, test on the rest): rejected, since it would leave most attack classes with zero test support, as most attacks only occur on capture days that would fall entirely in train.
2. **80/20 chronological split within each source file**: rejected, since several attacks (DoS Hulk, FTP-Patator, all three Web Attacks, Infiltration) run in narrow time windows within their capture day and landed entirely on one side of the cutoff no matter where it was drawn (for example, DoS Hulk ended up 100% train, Heartbleed 100% test).

**Final approach:** a stratified random split (80/20, `stratify=y`), which guarantees every class is represented proportionally in both partitions. This matches common practice in published CIC-IDS2017 baselines.

**Known limitation:** a stratified random split can allow highly correlated flows from the same attack burst to land on both sides of the split, which may inflate performance somewhat relative to true generalization on unseen attack instances. This trade-off was accepted deliberately after confirming that a leakage-safe temporal split isn't achievable on this dataset without sacrificing per-class evaluability entirely.

## 🤖 Model

- **Algorithm:** XGBoost, `multi:softprob` objective (15-class multiclass)
- **Imbalance handling:** `compute_sample_weight(class_weight="balanced")`, the multiclass equivalent of `scale_pos_weight`, which only applies to binary classification
- **Hyperparameters:** `n_estimators=300`, `max_depth=6`, `learning_rate=0.1`, `tree_method="hist"`

## 📈 Results

### Headline Metrics

- **Macro ROC-AUC (one-vs-rest):** 0.9999
- **Attack-detection recall** (any attack type vs. BENIGN, collapsed to binary): 0.9998

> Overall accuracy is intentionally not reported as a headline number. The test set is ~84% BENIGN, so a trivial classifier that always predicts BENIGN would score ~84% accuracy while catching zero attacks. Accuracy would look strong here and mean almost nothing.

### Per-Class Results

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| BENIGN | 1.000 | 0.998 | 0.999 | 419,012 |
| Bot | 0.494 | 0.992 | 0.659 | 390 |
| DDoS | 0.999 | 1.000 | 1.000 | 25,603 |
| DoS GoldenEye | 0.987 | 0.999 | 0.993 | 2,057 |
| DoS Hulk | 0.998 | 0.999 | 0.999 | 34,569 |
| DoS Slowhttptest | 0.932 | 0.995 | 0.963 | 1,046 |
| DoS slowloris | 0.988 | 0.991 | 0.989 | 1,077 |
| FTP-Patator | 0.998 | 0.999 | 0.999 | 1,186 |
| Heartbleed † | 1.000 | 1.000 | 1.000 | 2 |
| Infiltration † | 0.875 | 1.000 | 0.933 | 7 |
| PortScan | 0.989 | 1.000 | 0.994 | 18,139 |
| SSH-Patator | 0.998 | 1.000 | 0.999 | 644 |
| Web Attack - Brute Force | 0.767 | 0.816 | 0.791 | 294 |
| Web Attack - SQL Injection † | 0.429 | 0.750 | 0.545 | 4 |
| Web Attack - XSS | 0.428 | 0.477 | 0.451 | 130 |
| **Macro avg** | **0.859** | **0.934** | **0.888** | 504,160 |
| **Weighted avg** | **0.999** | **0.998** | **0.998** | 504,160 |

† Support under 10: these precision/recall values are not statistically reliable (a single misclassification swings them dramatically) and should not be read as representative model performance.

## 🔍 Key Findings & Limitations

1. **Bot has low precision (0.494) despite high recall (0.992).** 397 BENIGN flows were misclassified as Bot, versus 387 true positives, meaning more than half of everything flagged as Bot is actually normal traffic. This is a direct, expected consequence of `balanced` sample weighting: Bot is a small class (1,948 training rows), so the loss function was told to prioritize catching it, and the model traded precision for recall in response. In a real deployment, this would likely cause meaningful alarm fatigue on Bot-labeled alerts. 🚨

2. **Web Attack Brute Force and Web Attack XSS are confused with each other roughly half the time in both directions** (67 of 130 actual XSS attacks predicted as Brute Force, and 51 of 294 actual Brute Force attacks predicted as XSS). This reflects a feature-representation ceiling rather than a tuning problem: flow-level statistics (packet size, timing, byte counts) can't see HTTP payload content, and payload content is exactly what differentiates a login brute-force attempt from a script-injection attempt. Resolving this would require packet-payload features (deep packet inspection), which is out of scope for a NetFlow-based baseline.

3. **Heartbleed, Infiltration, and Web Attack-SQL Injection have too few test examples (2, 7, and 4 respectively) for their metrics to be trustworthy.** They're reported for completeness, not as evidence of reliable detection capability for these attack types.

4. **The stratified random split carries a documented leakage risk** from correlated flows within the same attack burst landing on both sides of the split (see the Train/Test Split section above). A stricter, leakage-safe temporal split was attempted and found infeasible on this dataset without sacrificing per-class evaluability.
