# GlassIDS: Code Security Scan 🔒

## 🎯 Objective

GlassIDS is a security tool, so I wanted to hold its own code to the same standard I hold the models to: check it instead of assuming it's fine. I ran two static analysis scanners over the project code and mapped what they found to OWASP and CWE categories. Then I reviewed by hand the parts a scanner can't reason about, mainly the LLM response layer. I also wanted a plain statement of what GlassIDS does and doesn't defend against, because a clean scan says something about the code and nothing about the detector's coverage.

## ⚙️ Setup

- **Tools:** Bandit 1.9.4 (a Python security linter) and Semgrep 1.184.0 with the Trail of Bits ruleset, run on Python 3.13.
- **Scope:** all of the project's code lives in Jupyter notebooks under `src/`, and Bandit and Semgrep only read `.py` files. So I converted every notebook to a `.py` file (`security_reports/converted/`) and scanned those copies. Line numbers below refer to the converted files, not the notebook cells.
- **Reports:** both scanners wrote JSON reports to `security_reports/`, which I loaded and tabulated in PowerShell.

## 📊 Results

14 findings in total (Bandit 7, Semgrep 7). **None are High severity**, and neither scanner reported hardcoded secrets, injection, shell execution, or network-exposure issues. 🎯

| Finding | Count | Severity | Location | CWE | OWASP Top 10 (2021) |
|---|---|---|---|---|---|
| Bandit B614: unsafe PyTorch load | 3 | Medium | `shap_explainability.py` L87, L91, L243 | CWE-502 Deserialization of Untrusted Data | A08 Software and Data Integrity Failures |
| Bandit B101: assert used | 4 | Low | `agentic_layer.py` L72; `autoencoder_anomaly_detection.py` L239; `self_captured_attack_evaluation.py` L199, L247 | CWE-703 Improper Check or Handling of Exceptional Conditions | No direct category |
| Semgrep: automatic-memory-pinning | 7 | Warning | `autoencoder_anomaly_detection.py` L85-86; `pytorch_model.py` L97-98, L211-213 | CWE-676 Use of Potentially Dangerous Function (the rule's own tag) | None |

The OWASP column is best-effort where nothing in the Top 10 fits. The memory-pinning rule carries a CWE-676 tag from Semgrep, but it is an audit-subcategory rule rated low likelihood and low impact, and no OWASP category applies. The assert rule has no OWASP category either.

## 🔍 Key Findings

1. **The only Medium-severity finding is real in principle but low risk here.** `torch.load` uses pickle underneath, so loading a tampered file can execute code. All three calls load `mlp_model.pt` and `cnn_model.pt`, which `pytorch_model.py` writes locally with `torch.save(model.state_dict(), ...)`, so the files never come from outside the project and exploiting this needs write access to my own models folder. The CNN is loaded twice because I redefined its class with separate ReLU instances to get `DeepExplainer` working (see the SHAP deliverable). The recommended fix is `torch.load(path, map_location=device, weights_only=True)`, which is compatible because only plain state dicts are ever saved. My installed PyTorch is 2.14.0 (CUDA 12.6 build), and since version 2.6 `weights_only` defaults to `True`, so these calls already load in the restricted mode at runtime. Bandit flags them anyway because the argument isn't written out. Making it explicit documents the intent, keeps the code safe if it is ever run on an older PyTorch, and should clear the finding.

2. **Three of the four asserts are notebook sanity checks, and one is a guard I actually rely on.** Asserts are stripped when Python runs with `-O`, so they should never enforce security-relevant logic. Here, the checks at `agentic_layer.py` L72 (test-set column order), `autoencoder_anomaly_detection.py` L239 (column count) and `self_captured_attack_evaluation.py` L199 (every Attack 3 flow got a label) only gate experiment code, so I'm accepting them. The one at `self_captured_attack_evaluation.py` L247 checks that feature columns match the model's training order before scoring, and its own message says why: a mismatch makes the model misread features silently. That is exactly the bug that hit the XGBoost scoring earlier in the project, so this check protects the integrity of the results. If that code ever runs outside a notebook it should become an explicit `raise`.

3. **The Semgrep warnings are a performance hint, not a vulnerability.** The seven hits line up exactly with the seven `DataLoader(...)` constructions in the project (2 in the autoencoder notebook, 5 in the PyTorch one), none of which sets `pin_memory`. Pinned memory speeds up host-to-GPU transfers and has no confidentiality or integrity impact. Semgrep tags it CWE-676 (audit, low likelihood, low impact), which is a loose fit for what is really a performance hint. An optional tweak is `pin_memory=torch.cuda.is_available()`.

4. **A clean scan mostly means the scanners had little to match against.** Of the 14 findings, only one assert touches the agentic layer, which is the part of the project that carries the most design risk. That gap is what the next section covers.

## 🧐 What the Scanners Can't See

Bandit and Semgrep match known bad code patterns. They don't reason about design. Three things I reviewed by hand:

- **`joblib.load` has the same risk as the flagged `torch.load`, and nothing flagged it.** `scaler.joblib` and `label_encoder.joblib` are loaded in `shap_explainability.py`, `agentic_layer.py` and `self_captured_attack_evaluation.py`. joblib is pickle-based, so this is the same CWE-502 / A08 class with the same mitigation: the files are produced locally and loaded from a folder I control. The XGBoost model is saved as JSON, which doesn't have this problem.

- **Dependencies are not pinned.** The notebooks install SHAP and PyTorch (from the CUDA package index) through inline `pip install` cells that pull whatever version is current, and the other packages they import (pandas, scikit-learn, XGBoost, joblib, the Anthropic SDK and so on) have no pinned version at all (A06:2021, best-effort).

- **The LLM response layer.** The controls I found in the code and the agentic deliverable:
  - the alert packet is numeric only, so no free text from the network reaches the prompt
  - the agent can only choose one of four actions, and any failure (network error, malformed JSON, invalid action, truncated response) becomes `escalate`, so a failure never produced a block
  - a guardrail in code downgrades block and rate-limit to escalate when a class has too little evidence
  - rule text comes from fixed templates plus an IP that passes `ipaddress` validation, so model output never reaches a command string
  - everything is dry-run, with synthetic source IPs

  The relevant categories from the OWASP Top 10 for LLM Applications (2025) are LLM01 Prompt Injection, LLM05 Improper Output Handling and LLM06 Excessive Agency. The controls above address all three by design, but none of it has been tested against adversarially crafted alerts. One gap I noticed: the agent makes one API call per alert with no throttling, so a flood of alerts means a flood of API calls and cost (LLM10 Unbounded Consumption, best-effort). The Anthropic SDK does retry failed requests twice by default (`max_retries` is 2 in my install), which helps with transient errors but is not a rate limit or a cap on total calls. Alert packets also leave the machine for an external API, though they carry only numeric flow features and a synthetic alert ID, not real IPs or payloads.

## 🛡️ What GlassIDS Does and Doesn't Defend Against

The README's threat model states this briefly. Here is the evidence behind each line:

| Claim | Evidence |
|---|---|
| Detects CIC-IDS2017-style attacks very well | Baseline deliverable: attack-detection recall 0.9998, macro F1 0.888 |
| Did not detect this lab's own versions of three attacks | Self-generated attack deliverable: 0% detection for port scan, SSH brute-force and slow-DoS |
| Rules did no better on the same captures | Suricata deliverable: all three missed, even with relevant signatures enabled |
| Autoencoder catches loud attacks, not quiet ones | Autoencoder deliverable: 64 to 97% on volumetric DoS, under 5% on scans and brute force |
| Can't see payload-level attacks | Baseline and SHAP deliverables: XSS and Brute Force share the same flow-level evidence and are confused about half the time |
| Does not enforce anything | Agentic deliverable: dry-run `nftables` rules, synthetic source IPs |
| Agent only evaluated on benchmark rows | Agentic deliverable scope note: no Kali-generated attack has gone through it yet |
| No protection against evasion, poisoning or model tampering | Not tested, and nothing verifies that a saved model file is the one I trained |

## ⚠️ Limitations

- Static analysis only finds known patterns. A clean result is not a security guarantee.
- The scan covered converted copies of the notebooks, not the notebooks themselves, so anything that exists only in saved cell outputs wasn't scanned. The `nftables` rule strings themselves (never tested against a live install) were not covered either.
- The assert finding's CWE comes from Bandit's own report and the memory-pinning finding's CWE comes from Semgrep's rule metadata. Only the OWASP column is best-effort for those two findings, since neither fits the Top 10 cleanly.
- There was no dynamic or adversarial testing. The LLM layer was reviewed by reading the code and the agentic deliverable, not by attacking it.
- I ran each scanner once with its default configuration, so I haven't compared results across tools or rulesets.

## 🔭 Future Scope

- Apply `weights_only=True` to the three `torch.load` calls and re-run Bandit to confirm B614 clears.
- Turn the column-order check in `self_captured_attack_evaluation.py` into an explicit `raise` before that code is reused outside a notebook.
- Record a SHA-256 hash of every saved model artifact (`xgboost_baseline.json`, the `.pt` files, the `.joblib` files) and verify it at load time.
- Pin exact dependency versions (for example from `pip freeze` inside the project's virtual environment) and run `pip-audit` to check for known vulnerable packages.
- Red-team the agent with crafted alerts (flows engineered to look benign or to inflate confidence), and add throttling or batching of agent calls.
- Optional: set `pin_memory` on the `DataLoader`s to clear the Semgrep warnings.
