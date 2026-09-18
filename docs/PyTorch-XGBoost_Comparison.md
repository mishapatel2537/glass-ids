# GlassIDS: PyTorch vs. XGBoost Architecture Comparison 🧪

## 🎯 Objective

I wanted to test whether my baseline XGBoost model's results, and especially how it handled class imbalance, would actually carry over to neural networks. So I built two PyTorch models from scratch, a small MLP and a 1D-CNN, and trained them under conditions as close to identical to the baseline as I could manage. That way, if the results came out different, I could point to the model family as the actual reason, not some other difference in data, features, or preprocessing.

## 📊 Methodology

- **Features and split:** I used the exact same 77 features as the baseline (Destination Port and the derived `is_attack`/`source_file` columns excluded), and the same stratified 80/20 train/test split (`random_state=42`). I reproduced this deterministically instead of saving and reloading it.
- **Validation split (neural models only):** neural networks need a validation signal for early stopping in a way the baseline's XGBoost training didn't, so I carved out an extra 85/15 split from the training data just for the PyTorch models (1,714,142 for training, 302,496 for validation), and left the original 504,160-row test set completely untouched until final evaluation. I want to flag this honestly: XGBoost was trained with a fixed hyperparameter set and no validation monitoring, while the neural models got to pick their own best epoch through early stopping. If anything, this probably helps the neural models more than it helps XGBoost, which makes XGBoost's win below more convincing, not less.
- **Feature scaling:** I used `StandardScaler`, fit only on the training data, then applied to validation and test. This is a new step compared to the baseline, since tree splits don't care about feature scale but gradient descent does.
- **Class imbalance handling:** I kept the same "balanced" philosophy across all three models, just translated into whatever mechanism each framework actually uses: `compute_sample_weight("balanced")` for XGBoost, `compute_class_weight("balanced")` fed into `CrossEntropyLoss` for the neural models. On the actual training data, this gave some pretty extreme per-class weights for the rarest classes: Heartbleed came out to roughly 14,285x, Web Attack-SQL Injection around 8,163x, and Infiltration about 4,571x.
- **Gradient clipping:** I added this to the neural models' training loop specifically because of those extreme weights. A single Heartbleed row in a mini-batch, weighted thousands of times over, could otherwise produce a gradient big enough to blow up training. Clipping just caps how large any one update can be, no matter what caused the oversized gradient.
- **Early stopping:** patience of 5 epochs, max of 50, saving the model's best validation-loss checkpoint and reloading it at the end of training. That way the final model reflects its best point during training, not wherever it happened to stop.

## 🤖 Models

- **XGBoost:** the same baseline model from before, `multi:softprob` objective. Full methodology and per-class results are in the baseline deliverable doc.
- **MLP:** 77 → 128 → 64 → 15, ReLU activations, no softmax on the output layer since `CrossEntropyLoss` already handles that internally, Adam optimizer with a learning rate of 0.001. Early stopping kicked in at epoch 11, with the best checkpoint at epoch 6 (validation loss 0.2178).
- **1D-CNN:** treats the 77 flow features as a single-channel sequence of length 77 (`Conv1d → ReLU → Conv1d → ReLU → MaxPool1d → Linear → Linear`). I included this mainly as an experiment, and I want to be upfront that flow features don't really have the local or spatial structure CNNs are built for (there's no real reason feature 12 should be "near" feature 13). So this was more a test of whether a CNN's usual advantage shows up anyway on data it wasn't designed for. Early stopping triggered at epoch 14, best checkpoint at epoch 9 (validation loss 0.2260).

## 📈 Results

### Macro-averaged comparison

| Model | Macro Precision | Macro Recall | Macro F1 | Weighted F1 | Macro ROC-AUC |
|---|---|---|---|---|---|
| XGBoost | 0.859 | 0.934 | **0.888** | 0.998 | 0.9999 |
| CNN | 0.606 | 0.833 | 0.607 | 0.954 | 0.9982 |
| MLP | 0.494 | 0.857 | 0.548 | 0.956 | 0.9980 |

Macro F1 (averaged evenly across all 15 classes, not weighted by how common they are) is the number I'd actually trust here, not accuracy or weighted F1. Both of those get dominated by the ~84% BENIGN majority and would still look fine even if the model was doing badly on the classes that actually matter.

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

† Support under 10, not statistically reliable, reported for completeness only.

## 🔍 Key Findings

1. **XGBoost clearly wins, and almost all of the gap is in precision, not recall.** Recall stayed pretty similar across all three models (0.83 to 0.93 macro), but macro precision dropped from 0.859 (XGBoost) to 0.606 (CNN) to 0.494 (MLP). Both neural models catch attacks about as reliably as XGBoost does, they're just a lot noisier about it, throwing out way more false positives per true positive.

2. **My best explanation: the same "balanced" weighting recipe doesn't transfer cleanly across model families.** XGBoost builds itself up tree by tree, correcting residual errors as it goes, so extreme sample weights just change which mistakes get prioritized without ever destabilizing the whole model at once. A neural network trained with gradient descent is doing one continuous optimization over the entire weighted loss surface, so an extreme weight like Heartbleed's ~14,285x directly reshapes the whole decision boundary. You can see this in Bot's precision, which collapsed to 0.033 in the MLP and 0.036 in the CNN, compared to XGBoost's already-not-great-but-still-much-better 0.494.

3. **Gradient clipping fixed training stability but not calibration, and I think those are genuinely two separate problems.** Both neural models trained with smooth, non-spiking loss curves, so clipping clearly did what it was supposed to. But the model they actually converged to was still heavily skewed toward over-predicting rare classes. Clipping controls how big of an update can happen in one step, it doesn't control what the model eventually settles on.

4. **One thing I didn't expect: the CNN beat the MLP on macro F1 (0.607 vs. 0.548) even though its validation loss was worse (0.2260 vs. 0.2178) during training.** I think this comes down to validation loss being a smooth measure of how confident the model was on average, while precision and recall are about the actual discrete decisions it made. The CNN recovered real precision on a few classes the MLP handled badly (SSH-Patator went from 0.043 to 0.342, FTP-Patator from 0.448 to 0.875, DoS GoldenEye from 0.624 to 0.912), though it lost a bit elsewhere (DDoS dropped from 0.993 to 0.912, PortScan from 0.940 to 0.739). Lesson for myself here: a lower validation loss doesn't automatically mean better real classification decisions, especially once you're doing heavy class reweighting.

5. **The Web Attack - XSS to Brute Force mixup looks like a real structural issue, not something that just happened in one run.** 122 of the 130 actual XSS attacks got misclassified as Brute Force in both the MLP and the CNN, two separately trained models landing on the exact same number. That backs up what I found in the XGBoost baseline too: flow-level stats just can't see the HTTP payload content that actually tells these two attack types apart, so changing the architecture doesn't fix it.

## ⚠️ Limitations

- The validation-split asymmetry I mentioned in the methodology (the neural models got to pick their best epoch through early stopping, XGBoost didn't) probably gives the neural models a small edge they wouldn't otherwise have. That makes their loss here more meaningful, not less.
- I also considered trying a gentler weighting scheme (something like square-root-dampened instead of straight inverse-frequency) to see if I could fix the neural models' precision problem. I decided not to run that experiment. The whole point of this comparison was to see whether the same weighting approach transfers across model families, and it clearly doesn't, so that miscalibration is the actual finding here, not a bug I should try to engineer away.