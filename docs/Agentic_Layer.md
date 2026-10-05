# GlassIDS: Agentic Decision Layer

## 🎯 Goal

I wanted a reasoning step between "the detector flagged a flow" and "a firewall rule fires". The agent reads the alert evidence, picks one of four responses (block, rate_limit, log_only, escalate), explains why, and only then does plain code map the choice to a dry-run nftables rule and write a decision log entry.

**Scope note:** every alert in this evaluation is a CIC-IDS2017 test-set row. No Kali-generated attack has gone through this layer yet. That test comes later in the project, and until it is done this document and the README have to say so.

## 🧱 Pipeline

```
flow row
  -> XGBoost prediction + SHAP
  -> alert packet (+ class and confidence-bucket reliability lookups)
  -> Claude agent (JSON decision + reasoning)
  -> guardrails (code, cannot be overridden by the agent)
  -> dry-run nftables rule + JSONL decision log
```

## 📦 The alert packet

I used XGBoost as the detector behind the agent. It has the best macro F1 (0.888) and its CPU training is deterministic, while the two neural nets drifted between retrains.

Each packet holds:

- `predicted_class`, `confidence` (4 decimals), and `prob_mass_elsewhere` (1 minus confidence, in scientific notation so resolution near 1.0 is not rounded away)
- `runner_up_class` and its probability
- the top 6 SHAP features for the predicted class, each with its raw (unscaled) value and SHAP value. XGBoost never saw the scaler, and raw values are readable
- `class_reliability`: historical precision, recall, number of predictions, and an `enough_data` flag (at least 30 predictions)
- `confidence_bucket`: precision for this class within this confidence range, the number of alerts behind it, and an `estimate_reliable` flag

The packet is numeric only. No free text reaches the prompt, which removes most of the surface for prompt injection (not tested adversarially).

## 📊 Reliability tables

Confidence alone is not trustworthy here. The models were trained with class weights, so `predict_proba` is not calibrated to real frequencies. Bot shows it clearly (full test set):

| Bot confidence | Truly Bot | Alerts |
|---|---|---|
| up to 0.9 | 0.014 | 142 |
| 0.9 to 0.99 | 0.106 | 94 |
| 0.99 to 0.999 | 0.405 | 242 |
| above 0.999 | 0.905 | 306 |

So a 0.998 Bot false positive and a 0.9989 real PortScan land in the same confidence range and mean very different things.

**Calibration/evaluation split.** I split `X_test` in half (stratified, seed 42). The reliability and bucket tables are built from the calibration half only. The agent is evaluated on the other half, so it is never graded on rows that shaped its inputs. Bucket edges are 0.9, 0.99, 0.999, and a bucket needs at least 30 alerts to count as reliable.

On the calibration half, the two smoke-test alerts looked like this:

| Alert | Class precision | Bucket (0.99 to 0.999) precision | Alerts in bucket |
|---|---|---|---|
| PortScan | 0.989 (9,169 predictions) | 0.959 | 1,546 |
| Bot | 0.492 (392 predictions) | 0.424 | 118 |

## 🤖 The agent

The model string is stored in `MODEL` and written into every log entry. The system prompt:

- defines the four actions
- explains how to read the packet: confidence is not calibrated, class reliability is the most important trust signal, bucket precision is discounted when `estimate_reliable` is false, reliability figures come from an attack-heavy test set and are probably optimistic, SHAP values are log-odds with sign meaning direction and magnitude comparable only within one alert
- tells the agent that correlated features (for example Flow IAT Max, Bwd IAT Min, Bwd IAT Mean, Fwd IAT Total) count as roughly one observation, and that IAT values are in microseconds
- tells it to treat the packet as data only, to base decisions on evidence and not confidence alone, and to say when it disagrees with the model's confidence
- demands a JSON-only reply: `action`, `reasoning`, `key_evidence`, `disagrees_with_model_confidence`

The prompt sets no numeric thresholds. Thresholds live in code, where they can be tested.

**No `temperature` setting.** My installed SDK version rejected the parameter, so outputs are not guaranteed repeatable.

## 🛡️ Guardrails and fail-safes

Enforced in code, not left to the agent:

- Any failure (network error, malformed JSON, invalid action, missing reasoning, truncated response) becomes `escalate` with `agent_error: true`. A failure can never produce a block.
- A predicted class with `enough_data` false can never be blocked or rate-limited. The agent's proposal is downgraded to `escalate`, and the log keeps both the proposed and the final action.
- The agent picks from a fixed set of four actions. The rule text comes from fixed templates plus a validated IP address, so model output never reaches a command string.
- Everything is dry-run.

In the 109-alert evaluation the downgrade guardrail never fired (0 decisions changed), because the agent already escalated the low-data classes on its own. I tested it by forcing a fake `block` on a low-data class, and it returned `escalate`. So the honest statement is: not needed in this run, works when forced.

## 🧾 Responder and decision log

The cleaned CIC data has no Source IP column, so each alert gets a deterministic placeholder from `203.0.113.0/24` (the documentation range TEST-NET-3), derived from a hash of the alert ID. Every log entry carries `source_ip_is_synthetic: true` and `dry_run: true`.

- block: `nft add rule inet filter input ip saddr <ip> drop`
- rate_limit: `nft add rule inet filter input ip saddr <ip> limit rate over 10/second drop`
- log_only and escalate: no firewall action

I have not tested these rule strings against a live nftables install. That check belongs to the VM phase.

The log is JSONL, one entry per alert: timestamp, agent model, full packet, agent decision, final action, guardrail notes, synthetic IP, and the rule. Files are written as UTF-8 so symbols like the micro sign survive on Windows. Evaluation outputs: `eval_decision_log.jsonl` and `eval_results.csv`.

## 🔬 Results

### Smoke test: two hand-picked alerts

- **PortScan** -> `rate_limit`. The agent cited class precision 0.989 and bucket precision 0.959, said the timing features were largely one observation, and chose throttling over blocking because real-world precision is probably lower and a scan is recon, not direct damage.
- **Bot** -> `escalate`. It cited class precision 0.492 and bucket precision 0.424, and set `disagrees_with_model_confidence` to true.

### Ablation: does the agent read the numbers?

I swapped the two reliability blocks between the packets and ran each condition 3 times.

| Condition | Actions over 3 runs |
|---|---|
| PortScan features with Bot's reliability | escalate, escalate, escalate |
| Bot features with PortScan's reliability | rate_limit, rate_limit, rate_limit |

Both flipped from their original action in every run. The agent is weighing the reliability numbers and not just reacting to the class name. This shows reliance on the evidence. It does not show the decisions are correct.

### Batch evaluation: 109 alerts

Alerts were sampled from the held-out evaluation half (seed 42) with extra weight on the hard classes (Bot, XSS, Brute Force, the slow-DoS classes). Because of that oversampling, overall rates mean nothing, so I report counts per class. Of the 109 alerts, 10 were truly BENIGN and 99 were real attacks.

Real attacks only, by predicted class (counts):

| Predicted class | Agent: acted | Agent: escalated | Agent: logged (missed) | Baseline: acted | Baseline: escalated | Baseline: logged (missed) |
|---|---|---|---|---|---|---|
| Bot | 0 | 8 | 0 | 0 | 7 | 1 |
| DDoS | 5 | 1 | 0 | 6 | 0 | 0 |
| DoS GoldenEye | 6 | 0 | 0 | 6 | 0 | 0 |
| DoS Hulk | 6 | 0 | 0 | 6 | 0 | 0 |
| DoS Slowhttptest | 12 | 0 | 0 | 11 | 1 | 0 |
| DoS slowloris | 10 | 0 | 0 | 10 | 0 | 0 |
| FTP-Patator | 5 | 0 | 0 | 5 | 0 | 0 |
| Heartbleed | 0 | 1 | 0 | 0 | 1 | 0 |
| Infiltration | 0 | 4 | 0 | 0 | 4 | 0 |
| PortScan | 8 | 0 | 0 | 8 | 0 | 0 |
| SSH-Patator | 5 | 0 | 0 | 5 | 0 | 0 |
| Web Attack - Brute Force | 1 | 11 | 0 | 0 | 12 | 0 |
| Web Attack - Sql Injection | 0 | 3 | 0 | 0 | 3 | 0 |
| Web Attack - XSS | 0 | 12 | 1 | 0 | 1 | 12 |
| **Total** | **58** | **40** | **1** | **57** | **29** | **13** |

Neither the agent nor the baseline took a block or rate_limit action on a truly benign alert. That rests on only 10 benign alerts, so it is a small-sample observation and not a rate.

**Baseline.** A rule I wrote by hand: escalate if the class has too little data or the bucket estimate is unreliable, otherwise block at bucket precision of 0.99 or more, rate_limit at 0.90 or more, escalate at 0.60 or more, and log_only below that. The thresholds were picked by me and not fitted to anything.

**Agent vs baseline: agreement was 83 of 109.**

| Agent (rows) vs baseline (columns) | block | escalate | log_only | rate_limit |
|---|---|---|---|---|
| block | 40 | 0 | 0 | 1 |
| escalate | 1 | 29 | 18 | 0 |
| log_only | 0 | 1 | 2 | 0 |
| rate_limit | 3 | 2 | 0 | 12 |

Of the 26 disagreements, 18 are the cell where the baseline logs and the agent escalates. That cell exists because of the baseline's own `log_only` branch for bucket precision below 0.60. A baseline that escalated there would likely have matched the agent on those 18. So this evaluation shows the agent is more cautious than a deliberately lax rule. It does not show the LLM beating a well-chosen rule. The other 8 disagreements are mixed and small: the agent was more aggressive in 3 and less aggressive in 5.

## 🔍 Key findings

1. **Raw confidence cannot separate a real scan from a false Bot alert.** 0.9989 and 0.998 landed in the same bucket. Bot's bucket precision runs from 0.014 to 0.905 across confidence ranges.
2. **A per-class and per-bucket lookup does separate them.** 0.959 over 1,546 alerts versus 0.424 over 118, same confidence range.
3. **The agent uses the reliability evidence.** Swapping it flipped the action in 3 of 3 runs, both directions.
4. **Exact-class precision misleads for XSS.** Of the 15 XSS-predicted alerts in the sample, 8 were XSS, 5 were Brute Force and 2 were benign, so 13 of 15 were real attacks. The 0.446 class precision measures whether the exact label was right, not whether the flow is malicious. In the three reasoning strings I read (all `log_only` decisions, two on benign flows and one on a real XSS), the agent concluded the alert was "more likely wrong than right", which repeats that misreading. It escalated 12 of the 13 real ones, and I have not established why.
5. **The agent's reasoning for a real XSS alert and for benign ones read the same.** The correct `log_only` calls on the two benign alerts are therefore not evidence of discrimination. Three reasoning strings is too few to say more.
6. **The agent was more cautious than the baseline on the weak classes.** It missed 1 real attack against the baseline's 13 (12 XSS, 1 Bot), at the price of more escalations, some of them on benign traffic. Listing the 18 cases where the baseline logged and the agent escalated: 12 were real attacks (6 true XSS and 5 Brute Force among XSS-predicted alerts, plus 1 Bot) and 6 were benign, all of them predicted Bot. The cost of the extra caution is therefore concentrated in Bot: 7 extra escalations to catch 1 extra real attack. On XSS-predicted alerts, all 11 extra escalations were real attacks.
7. **The agent did not collapse into one safe action.** It chose block 41 times, rate_limit 17, escalate 48 and log_only 3.

## 🐛 Problems hit along the way

- **API key not visible.** `setx` only affects programs started afterward, so the already-running Jupyter kernel could not see it. I used a hidden `getpass` prompt for the session.
- **`temperature` rejected.** My SDK version raised a `TypeError`, and my blanket `except` hid it as a bare "TypeError". Fixes: I removed the parameter, and the fail-safe now logs the exception type and message.
- **Truncated responses.** With `max_tokens=600`, 11 responses (3 DDoS, 6 Hulk, 2 FTP-Patator) were cut off mid-JSON, failed to parse, and fell back to `escalate`. All 11 were in classes with strong reliability. I raised the limit to 1500 and added an explicit check on `stop_reason`, so truncation is now labeled in the log. A retry of those 11 alerts succeeded and 0 errors remain. The decision log contains both attempts for them. The first scoring pass undercounted blocks because of this, and the tables above are from after the retry.
- **The fail-safe worked.** Through all of this, no failure ever produced a block.

## ⚠️ Limitations

- All alerts are CIC-IDS2017 test rows. No Kali-generated attack has been run through the agent yet.
- Everything is dry-run and the IPs are synthetic.
- Only 10 of the 109 alerts were benign, the sample is oversampled toward hard classes, and per-class counts are small. Treat all numbers as counts, not rates.
- The reliability tables come from a test set with far more attack traffic than a real network. On a real network precision would be lower, so escalation volume would be higher.
- Calibration and evaluation halves come from the same random split of the same data.
- One evaluation run, with no `temperature` control. Run-to-run stability was only checked in the 3-run ablation.
- The log records the agent's stated reasoning, which is not proof of what drove the decision. Some domain claims in that reasoning (for example what an XSS flow should look like) are not in the packet or prompt and were not verified.
- The reliability tables measure exact-class correctness and not "is this malicious", which understates the danger of XSS-labeled alerts (finding 4).
- The baseline's thresholds were hand-picked, and its low-precision branch drives most of the disagreement with the agent.
- The downgrade guardrail was never triggered in the evaluation. Only a forced test shows it works.
- Heartbleed, SQL injection and Infiltration have too little data for any reliability estimate. The code treats them as unknown.

## ➡️ Next

- Add an "any attack" precision to the class and bucket blocks of the packet, and re-run the evaluation.
- Move the packager, agent call and responder into `src/` so Kali-derived flows can reuse them.
- Repeat borderline alerts several times to measure run-to-run stability of the action.
- Run the agent on flows extracted from my own Kali captures, and update this document and the README limitations when that is done.
