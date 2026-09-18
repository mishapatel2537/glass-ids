# GlassIDS: PyTorch vs. XGBoost Architecture Comparison 🧪

## 🎯 Objective

Test whether the baseline XGBoost model's results, and specifically its approach to class imbalance, generalize to neural network architectures. Two from-scratch PyTorch models (a small MLP and a 1D-CNN) were trained under conditions kept as close to identical as possible to the baseline, so any difference in outcome could be attributed to the model family rather than to different data, features, or preprocessing.

## 📊 Methodology

- **Features and split:** identical to the baseline deliverable — the same 77 features (Destination Port and the derived `is_attack`/`source_file` columns excluded), the same stratified 80/20 train/test split (`random_state=42`), reproduced deterministically rather than saved and reloaded.
- **Validation split (neural models only):** since neural networks need a validation signal for early stopping in a way XGBoost's baseline training didn't, a further 85/15 split was carved out of the training data specifically for the PyTorch models (1,714,142 train / 302,496 validation), leaving the original 504,160-row test set completely untouched until final evaluation. This is a methodological asymmetry worth naming honestly: XGBoost was trained with a fixed hyperparameter set and no validation monitoring, while the neural models got early stopping's benefit of picking their own best checkpoint. If anything, this likely helps the neural models relative to XGBoost, not the other way around, which makes XGBoost's win margin below more meaningful, not less.
- **Feature scaling:** `StandardScaler`, fit on the training data only, applied to validation and test. New requirement specific to neural networks (tree-based splits are scale-invariant, gradient descent is not).
- **Class imbalance handling:** the same "balanced" philosophy across all three models, translated into each framework's native mechanism — `compute_sample_weight("balanced")` for XGBoost, `compute_class_weight("balanced")` fed into `CrossEntropyLoss` for the neural models. On the actual training data used, this produced extreme per-class weights for the rarest classes: Heartbleed at roughly 14,285x, Web Attack-SQL Injection at roughly 8,163x, Infiltration at roughly 4,571x.
- **Gradient clipping:** added to the neural models' training loop specifically because of those extreme weights — a single Heartbleed row in a mini-batch, weighted thousands of times over, could otherwise produce a gradient large enough to destabilize training. Clipping caps the size of any single update regardless of what produced the oversized gradient.
- **Early stopping:** patience of 5 epochs, maximum 50, with the model's best-validation-loss checkpoint saved and reloaded at the end of training, so the final model reflects its best generalizing point rather than wherever training happened to stop.

## 🤖 Models

- **XGBoost:** the existing baseline, `multi:softprob` objective. Full methodology and per-class results in the baseline deliverable doc.
- **MLP:** 77 → 128 → 64 → 15, ReLU activations, no output-layer softmax (handled internally by `CrossEntropyLoss`), Adam optimizer, learning rate 0.001. Early stopping triggered at epoch 11; best checkpoint at epoch 6 (validation loss 0.2178).
- **1D-CNN:** treats the 77 flow features as a single-channel sequence of length 77 (`Conv1d → ReLU → Conv1d → ReLU → MaxPool1d → Linear → Linear`). Included specifically as an architecture-comparison experiment, with the caveat stated upfront: flow features don't have genuine local or spatial structure (there's no real reason feature 12 should be "near" feature 13), so this tests whether a CNN's usual advantage transfers to data it wasn't designed for. Early stopping triggered at epoch 14; best checkpoint at epoch 9 (validation loss 0.2260).

## 📈 Results

### Macro-averaged comparison

| Model | Macro Precision | Macro Recall | Macro F1 | Weighted F1 | Macro ROC-AUC |
|---|---|---|---|---|---|
| XGBoost | 0.859 | 0.934 | **0.888** | 0.998 | 0.9999 |
| CNN | 0.606 | 0.833 | 0.607 | 0.954 | 0.9982 |
| MLP | 0.494 | 0.857 | 0.548 | 0.956 | 0.9980 |

Macro F1 (unweighted across all 15 classes) is the metric to trust here, not accuracy or weighted F1 — both of the latter are dominated by the ~84% BENIGN majority and would look strong even for a model doing poorly on the classes that actually matter.

### MLP per-class results

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| BENIGN | 0.998 | 0.919 | 0.957 | 419,012 |
| Bot | 0.033 | 0.997 | 0.063 | 390 |
| DDoS | 0.993 | 0.992 | 0.992 | 25,603 |
| DoS GoldenEye | 0.624 | 0.998 | 0.768 | 2,057 |
| DoS Hulk | 0.962 | 0.987 | 0.974 | 34,569 |
| DoS Slowhttptest | 0.632 | 0.996 | 0.773 | 1,046 |
| DoS slowloris | 0.774 | 0.959 | 0.857 | 1,077 |
| FTP-Patator | 0.448 | 0.997 | 0.618 | 1,186 |
| Heartbleed † | 0.667 | 1.000 | 0.800 | 2 |
| Infiltration † | 0.065 | 0.857 | 0.121 | 7 |
| PortScan | 0.940 | 0.998 | 0.968 | 18,139 |
| SSH-Patator | 0.043 | 0.977 | 0.083 | 644 |
| Web Attack - Brute Force | 0.108 | 0.905 | 0.193 | 294 |
| Web Attack - SQL Injection † | 0.008 | 0.250 | 0.016 | 4 |
| Web Attack - XSS | 0.120 | 0.023 | 0.039 | 130 |

### CNN per-class results

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| BENIGN | 0.999 | 0.932 | 0.964 | 419,012 |
| Bot | 0.036 | 0.995 | 0.070 | 390 |
| DDoS | 0.912 | 0.999 | 0.954 | 25,603 |
| DoS GoldenEye | 0.912 | 0.997 | 0.952 | 2,057 |
| DoS Hulk | 0.854 | 0.996 | 0.920 | 34,569 |
| DoS Slowhttptest | 0.813 | 0.995 | 0.895 | 1,046 |
| DoS slowloris | 0.861 | 0.988 | 0.920 | 1,077 |
| FTP-Patator | 0.875 | 0.991 | 0.929 | 1,186 |
| Heartbleed † | 1.000 | 0.500 | 0.667 | 2 |
| Infiltration † | 0.070 | 1.000 | 0.131 | 7 |
| PortScan | 0.739 | 0.999 | 0.850 | 18,139 |
| SSH-Patator | 0.342 | 0.932 | 0.501 | 644 |
| Web Attack - Brute Force | 0.137 | 0.901 | 0.238 | 294 |
| Web Attack - SQL Injection † | 0.038 | 0.250 | 0.067 | 4 |
| Web Attack - XSS | 0.500 | 0.023 | 0.044 | 130 |

† Support under 10: not statistically reliable, reported for completeness only.

## 🔍 Key Findings

1. **XGBoost decisively outperforms both neural architectures, and the gap is almost entirely in precision, not recall.** Recall is comparably high across all three models (0.83–0.93 macro), but macro precision drops from 0.859 (XGBoost) to 0.606 (CNN) to 0.494 (MLP). Both neural models catch attacks about as reliably as XGBoost does; they're just far noisier about it, generating many more false positives per true positive.

2. **The mechanism: the identical "balanced" weighting recipe does not transfer cleanly across model families.** XGBoost builds its model incrementally, correcting residual errors tree by tree, so extreme sample weights influence which mistakes get prioritized without ever destabilizing the whole model at once. A neural network trained by gradient descent does one continuous joint optimization over the entire weighted loss surface, so an extreme per-class weight (like Heartbleed's ~14,285x) reshapes the global decision boundary directly. Bot's precision, for example, collapsed to 0.033 (MLP) and 0.036 (CNN), compared to XGBoost's already-weak but far better 0.494.

3. **Gradient clipping solved training stability but not calibration** — two genuinely separate problems. Both neural models trained with smooth, non-spiking loss curves despite the extreme weights, confirming clipping did its job. But the *converged* decision boundary was still heavily skewed toward over-predicting rare classes. Clipping controls how violently a model updates; it doesn't control what the optimization eventually converges to.

4. **The CNN beat the MLP on macro F1 (0.607 vs. 0.548) despite having a worse validation loss (0.2260 vs. 0.2178) during training.** Validation loss is a smooth, continuous measure of average prediction confidence; precision and recall are discrete, threshold-based measures of the actual classification decisions. The CNN recovered real precision on several classes the MLP badly mishandled (SSH-Patator: 0.043 → 0.342, FTP-Patator: 0.448 → 0.875, DoS GoldenEye: 0.624 → 0.912), at some cost elsewhere (DDoS: 0.993 → 0.912, PortScan: 0.940 → 0.739). The practical lesson: a lower validation loss doesn't guarantee better downstream classification decisions, especially under heavy class reweighting.

5. **The Web Attack - XSS to Brute Force confusion is structural, not one run's artifact.** 122 of 130 real XSS attacks were misclassified as Brute Force in both the MLP and the CNN, independently trained architectures landing on the same number. This confirms the same feature-representation ceiling identified in the XGBoost baseline: flow-level statistics can't see the HTTP payload content that actually distinguishes these two attack types, so no amount of architecture change fixes it.

## ⚠️ Limitations

- The validation-split asymmetry noted in Methodology (neural models got early stopping's benefit of picking their best epoch, XGBoost didn't) means this comparison likely gives the neural models a slight edge they wouldn't otherwise have, making their loss here more notable rather than less.
- A follow-up experiment with a gentler weighting scheme (square-root-dampened rather than literal inverse-frequency) was considered as a way to test whether the neural models' precision collapse could be fixed. It was deliberately not run: the miscalibration under identical "balanced" weighting is itself the finding this comparison was designed to surface, not a bug to optimize away.
