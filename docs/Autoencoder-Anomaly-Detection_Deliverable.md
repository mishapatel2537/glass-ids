# GlassIDS: Autoencoder Anomaly Detection 🔬

## 🎯 Objective

The self-generated attack testing phase showed something important: a supervised detector's strong benchmark numbers don't guarantee it recognizes attacks it wasn't explicitly trained to label, XGBoost scored near-perfectly on PortScan and SSH-Patator within CIC-IDS2017 itself, then missed all three of my live, self-generated versions of those attack types completely. I wanted to test a fundamentally different approach in response: an autoencoder, trained only on benign traffic, with no attack labels at all. Instead of learning "what does this specific attack look like," it only ever learns "what does normal look like," and flags anything that doesn't reconstruct well. The question this phase answers: does that unsupervised approach catch anything the supervised baseline missed, and does it hold up on self-captured traffic any better than XGBoost did?

## 🧠 Approach

This is a plain (dense) autoencoder, not an LSTM one. LSTM autoencoders are for sequential data where order across time matters. My data is one row per flow, already summarized into 77 independent numeric features, same shape as every other model in this project, so there's no sequence for a recurrent architecture to exploit.

**Architecture:** `77 → 64 → 32 → 16 → 32 → 64 → 77`, a 16-unit bottleneck in the middle. The bottleneck is the entire mechanism: squeezing 77 numbers down to 16 and back forces the network to keep only what's common enough across benign traffic to be worth compressing. An attack flow structurally different from that learned compression doesn't pass through the bottleneck cleanly, and that reconstruction gap is the anomaly signal.

**Training data: benign only, with its own scaler.** I pulled only `BENIGN` rows out of the same stratified train split used throughout this project, then split those into a 15% validation slice. I deliberately fit a fresh `StandardScaler` on this benign-only training data rather than reusing the classifiers' scaler, since fitting on attack-inclusive data would distort what "normal" even means before training starts.

**Why train, validation, and test aren't all the same distribution.** Train and validation are both benign-only, because validation's whole job is to give an honest, uncorrupted read on the training objective itself, if attack rows diluted that average, early stopping could trigger on a misleading signal. Test, by contrast, is deliberately the original full mixed CIC split, because the actual question this model exists to answer, can reconstruction error separate normal from abnormal, can't be tested on benign-only data at all.

**Training:** same early-stopping discipline as the Day 8 PyTorch models, patience of 5 epochs, best-checkpoint-and-restore via `state_dict`. Training stopped at epoch 26, with the best validation loss (0.0089 MSE, scaled features) actually reached at epoch 21.

**Threshold:** the 95th percentile of reconstruction error on the benign training data itself. This is a deliberate, stated choice, not a fitted or provably optimal one, it explicitly accepts roughly a 5% false-positive rate on benign traffic as the cost of catching subtler anomalies.

## 📈 Results: CIC-IDS2017 Held-Out Test Set

False positive rate on BENIGN test rows: **4.98%**, almost exactly matching the 5% the threshold was built to produce, confirming the pipeline works correctly before looking at anything else.

| Class | Detection Rate | Flows |
|---|---|---|
| Web Attack - SQL Injection † | 0.000 | 4 |
| SSH-Patator | 0.000 | 644 |
| FTP-Patator | 0.001 | 1,186 |
| PortScan | 0.012 | 18,139 |
| Web Attack - XSS | 0.031 | 130 |
| Bot | 0.036 | 390 |
| Web Attack - Brute Force | 0.044 | 294 |
| DDoS | 0.639 | 25,603 |
| DoS slowloris | 0.648 | 1,077 |
| DoS GoldenEye | 0.733 | 2,057 |
| DoS Hulk | 0.902 | 34,569 |
| DoS Slowhttptest | 0.967 | 1,046 |
| Heartbleed † | 1.000 | 2 |
| Infiltration † | 1.000 | 7 |

† Support under 10, not trustworthy numbers, same caveat applied throughout this project's other tiny-sample classes.

## 🔍 Key Finding 1: A Clean Split Along Attack "Loudness," Not Attack Type

The large-sample classes split cleanly into two groups, not a gradient. Quiet, low-volume attacks (SSH-Patator, FTP-Patator, PortScan, the Web Attacks, Bot) are barely detected at all, under 5%. Volumetric DoS/flood attacks (DDoS, slowloris, GoldenEye, Hulk, Slowhttptest) are strongly detected, 64 to 97%.

The mechanism is straightforward once you think about what an autoencoder can actually measure: it has no labels, so it only ever learns "typical magnitude." A DoS flood is, almost by definition, numerically extreme, huge packet counts, huge byte rates, durations sitting near CICFlowMeter's own timeout ceiling, so it reconstructs badly and gets flagged easily. A single scan probe or one failed login attempt is numerically unremarkable, even though its intent is malicious, what actually makes it suspicious is a cross-flow pattern (many probes, many attempts) that a model scoring one flow in isolation simply can't see.

This is genuinely complementary to the baseline, almost a mirror image of it: XGBoost scored 0.989 precision on PortScan and 0.998 on SSH-Patator, exactly where this autoencoder is weakest. That's a real argument for running both models rather than either alone.

## 📈 Results: Self-Captured Attacks and My Own Benign Traffic

| Category | Flows | Flagged |
|---|---|---|
| Port scan (attack) | 65,535 | 100% |
| SSH brute-force (attack) | 14 | 100% |
| Slow-DoS, sustained (attack) | 1,000 | 100% |
| Slow-DoS, probes (attacker tool traffic) | 41 | 100% |
| My own control-plane traffic (benign) | 4 | 100% |

## 🔍 Key Finding 2: Self-Captured Traffic Breaks the Model Entirely, for an Identifiable Reason

A uniform 100% across every category, including my own benign control sessions, was suspicious enough on its own that I didn't take it at face value. I confirmed first that it wasn't a repeat of an earlier bug (the feature-order mismatch that hit the XGBoost scoring), the column alignment checked out cleanly. The real cause only showed up by comparing per-feature reconstruction error on individual rows, not the averaged per-row number.

A handful of columns have almost zero variance in CIC's benign training data (`RST Flag Count`, `Fwd Avg Bulk Rate`, `Fwd PSH Flags`, and similar), since ordinary benign flows rarely show these at all, this matches a packet-level finding from earlier in the project: a clean connection closes via `FIN, ACK`, not `RST`. My self-captured traffic genuinely does carry real values on these columns, a SYN scan legitimately generates RST packets, and my `http.server`-logging benign session's sustained small-chunk transfers genuinely trip the bulk-rate logic. Those values land many standard deviations outside what training ever saw.

The actual failure is architectural, not a data bug: because every layer of a dense autoencoder is a matrix multiplication mixing all 77 inputs together, one catastrophically extreme input distorts the reconstruction of *every* other output, not just its own. The evidence for this is direct: features with completely ordinary scaled values still showed squared errors in the hundreds to hundreds of millions, because the network's shared weights were already thrown off by one or two unrelated extreme inputs. Reconstruction error here isn't cleanly measuring "is this row anomalous," it's measuring "did any single input blow up the whole forward pass," which can't discriminate attack from benign once a flow sits far enough outside the training distribution on even one near-zero-variance column.

## 📊 Summary

| Evaluation | False Positive Rate | What It Shows |
|---|---|---|
| CIC-IDS2017 held-out test | 4.98% | A real, graded result: strong on volumetric DoS/flood attacks (64 to 97%), weak on quiet attacks (under 5%), genuinely complementary to XGBoost |
| Self-captured attacks + own benign traffic | n/a | Not a usable detection-rate result; the model collapsed on both attack and benign traffic from this source alike, see Key Finding 2 |

## ⚠️ Limitations

* The CIC-IDS2017 result is a genuine finding; the self-captured-traffic result is not usable as a detection claim at all, since the model failed on benign traffic from the same source just as completely as on attack traffic.
* The 95th-percentile threshold is a deliberate, stated choice, not fitted or optimized against any criterion.
* Only one architecture and bottleneck size was tried, no hyperparameter search.
* Heartbleed and Infiltration's 100% detection rates rest on 2 and 7 flows respectively and aren't trustworthy on their own, same caveat as every other tiny-sample class in this project.
* This model was evaluated in isolation, not yet combined with XGBoost or the agentic layer, despite the complementary-strengths finding above suggesting real value in doing so.

## 🔭 Future Scope

* **Feature robustness fix:** drop, clip, or separately handle the handful of near-zero-variance training columns (`RST Flag Count`, `Fwd Avg Bulk Rate`, `Fwd PSH Flags`, and similar) responsible for the self-captured-traffic collapse, then retrain. This would likely restore genuine discrimination on self-captured traffic instead of the current uniform flag-everything result.
* **Per-feature error weighting or clipping**, so one extreme input can't distort the reconstruction of unrelated features the way it currently does.
* **A simple ensemble or voting scheme combining XGBoost and this autoencoder**, given how cleanly their strengths and weaknesses turned out to complement each other on CIC-IDS2017's own test set.
