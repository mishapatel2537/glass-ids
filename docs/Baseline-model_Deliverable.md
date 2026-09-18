# GlassIDS: Baseline Intrusion Detection Model 🛡️

## 🎯 Objective

I wanted to build a baseline multiclass network intrusion detection model on CIC-IDS2017, something I could use as a benchmark for explainability work and tuning later on.

## 📊 Data

- **Source:** CIC-IDS2017 (Kaggle mirror of the pruned `MachineLearningCSV` release)
- **Cleaning:** I dropped about 2,867 rows with NaN/inf values, removed duplicate flows, fixed a mangled-dash label corruption in the Web Attack labels, and stripped whitespace from the column headers.
- **Final size:** 2,520,798 flows, down from 2,830,743 before cleaning. That ~310K row drop lines up with CIC-IDS2017's well-documented duplicate-flow issue, so it's not a sign I lost data by mistake.
- **Classes:** BENIGN plus 14 attack types, and severely imbalanced, ranging from BENIGN (2,095,057) all the way down to Heartbleed (11), Web Attack-SQL Injection (21), and Infiltration (36).
- **Features:** 78 flow-level statistics from CICFlowMeter (packet counts/sizes, inter-arrival times, TCP flags, duration, byte rates, etc.), no packet payload content.

## 🧹 Feature Selection

I deliberately left out two columns beyond the obvious label columns:

- **`is_attack`**: a binary column I derived directly from `Label` during cleaning. Including it would be direct label leakage, since the model would basically be reading the answer off a renamed copy of itself.
- **`Destination Port`**: technically a legitimate feature in this dataset, but it near-perfectly identifies FTP-Patator (port 21) and SSH-Patator (port 22). If I kept it, the model could shortcut those two classes with a port lookup instead of learning actual traffic behavior, which would work against the explainability goals I have for this project (a "the model looked at the port number" explanation isn't a useful security insight). After training, I found dropping it didn't hurt Patator detection at all, both classes still scored 0.998+ on precision and recall, which told me the behavioral signal alone was enough. 🎯

## ✂️ Train/Test Split Methodology

CIC-IDS2017's attacks are day-exclusive by design (the DoS family only shows up in Wednesday's capture, Web Attacks only in Thursday morning's, Patator attacks only in Tuesday's, and so on). I tried and rejected two temporally-ordered split strategies before landing on the one I actually used:

1. **Whole-day cutoff** (train on most capture days, test on the rest): I rejected this because it would've left most attack classes with zero test support, since most attacks only happen on capture days that would fall entirely into train.
2. **80/20 chronological split within each source file**: I rejected this too, since several attacks (DoS Hulk, FTP-Patator, all three Web Attacks, Infiltration) run in narrow time windows within their capture day and ended up landing entirely on one side of the cutoff no matter where I drew it (for example, DoS Hulk ended up 100% train, Heartbleed 100% test).

**What I actually used:** a stratified random split (80/20, `stratify=y`), which guarantees every class shows up proportionally in both partitions. This matches common practice in published CIC-IDS2017 baselines.

**Known limitation:** a stratified random split can let highly correlated flows from the same attack burst land on both sides of the split, which might inflate performance a bit compared to true generalization on attacks the model hasn't seen. I accepted this trade-off deliberately, after confirming a leakage-safe temporal split just isn't achievable on this dataset without giving up per-class evaluability entirely.

## 🤖 Model

- **Algorithm:** I used XGBoost with a `multi:softprob` objective (15-class multiclass).
- **Imbalance handling:** `compute_sample_weight(class_weight="balanced")`, the multiclass equivalent of `scale_pos_weight`, which only works for binary classification.
- **Hyperparameters:** `n_estimators=300`, `max_depth=6`, `learning_rate=0.1`, `tree_method="hist"`.

## 📈 Results

### Headline Metrics

- **Macro ROC-AUC (one-vs-rest):** 0.9999
- **Attack-detection recall** (any attack type vs. BENIGN, collapsed to binary): 0.9998

> I'm intentionally not reporting overall accuracy as a headline number here. The test set is about 84% BENIGN, so a classifier that just predicted BENIGN every single time would score ~84% accuracy while catching zero attacks. That number would look great and mean almost nothing.

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

† Support under 10, so I don't trust these precision/recall numbers much (a single misclassification swings them dramatically) and wouldn't read them as representative of real performance.

## 🔍 Key Findings & Limitations

1. **Bot has low precision (0.494) despite high recall (0.992).** I found 397 BENIGN flows misclassified as Bot, versus only 387 true positives, so more than half of everything flagged as Bot is actually normal traffic. This is a direct, expected result of the `balanced` sample weighting: Bot is a small class (1,948 training rows), so the loss function was told to prioritize catching it, and the model traded away precision for recall in response. In a real deployment I think this would cause real alarm fatigue on Bot-labeled alerts. 🚨

2. **Web Attack Brute Force and Web Attack XSS get confused with each other roughly half the time in both directions** (67 of 130 actual XSS attacks predicted as Brute Force, and 51 of 294 actual Brute Force attacks predicted as XSS). I think this comes down to a feature-representation ceiling rather than a tuning problem: flow-level statistics (packet size, timing, byte counts) can't see HTTP payload content, and that's exactly what separates a login brute-force attempt from a script-injection attempt. Fixing this would need packet-payload features (deep packet inspection), which is outside the scope of a NetFlow-based baseline.

3. **Heartbleed, Infiltration, and Web Attack-SQL Injection only have 2, 7, and 4 test examples respectively**, way too few for their metrics to be trustworthy. I'm reporting them for completeness, not as evidence these attack types are reliably detected.

4. **The stratified random split I used carries a documented leakage risk** from correlated flows within the same attack burst landing on both sides of the split (see the Train/Test Split section above). I did try a stricter, leakage-safe temporal split, but found it wasn't feasible on this dataset without giving up per-class evaluability.