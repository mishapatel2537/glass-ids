# GlassIDS: Suricata Rule-Based Comparison 🔎

## 🎯 Objective

Every detector in this project so far, XGBoost, the PyTorch models, the autoencoder, learns statistical patterns from data. I wanted a genuine "ML vs. traditional rules" data point: how does a signature-based IDS, the kind most real security teams actually deploy, handle the exact same three self-generated attacks that already exposed a real generalization gap in the ML-based detector?

## ⚙️ Setup

Installed Suricata 8.0.8 natively on Windows rather than on Kali, since offline pcap-read mode (`-r <file>`) needs no live-capture driver at all, sidestepping the class of Windows networking problems that complicated earlier phases of this project. Loaded rules via `suricata-update`, pulling the free Emerging Threats Open ruleset (53,001 rules enabled). Ran each of the three self-captured pcaps offline with `-k none` to disable checksum validation, since VirtualBox's checksum offloading can make captured packets look invalid even though the real traffic was fine.

## 🧪 Pass 1: Default Ruleset

With the stock ET Open ruleset, exactly one alert type fired, across all three captures combined: `ET INFO Python SimpleHTTP ServerBanner`, triggered by Attack 3 only. This is a passive signature that fingerprints *what web server software is running*, not an attack-behavior detection, it would fire identically on one innocuous request to that same server. Attacks 1 and 2 produced zero alerts of any kind.

## 🔧 Pass 2: Enabling the Relevant Disabled Signatures

Before concluding anything, I checked whether this reflected a genuinely blind ruleset or just scan/DoS categories sitting disabled by default, a common ET Open convention since those categories are prone to false positives on ordinary traffic.

- **Attack 1 (port scan):** found `SID 2009582 — ET SCAN NMAP -sS window 1024`, disabled by default. Its headline condition, a TCP window size of 1024, exactly matches the earlier finding that every single self-captured port-scan flow carries `Init_Win_bytes_forward = 1024` with zero variance. Enabled it.
- **Attack 2 (SSH brute-force):** found `SID 2001219 — ET SCAN Potential SSH Scan` was already enabled.
- **Attack 3 (slow-DoS):** found `SID 2016033 — ET SCAN Simple Slowloris Flooder` was already enabled, but reading its actual logic showed it requires an HTTP POST request with a specific Content-Length and a missing User-Agent header. `slowhttptest -H` (Slowloris mode) generates GET requests that do carry a User-Agent, a direct structural mismatch with what this rule is actually written to detect.

Re-ran all three captures with the updated ruleset.

## 📈 Results

No change. Attacks 1 and 2 still produced zero alerts, despite enabling a rule whose headline condition (`window 1024`) is satisfied by literally every flow in Attack 1's capture. Attack 3 still only produced the same passive `ServerBanner` match; the Slowloris-specific rule never fired, consistent with the GET-vs-POST mismatch identified above.

## 🔍 Key Finding

Signature-based detection missed all three attacks completely, even once the conceptually correct rule was identified and enabled. That's a genuinely different failure mode than the ML detector's, and together they make a sharper point than either result alone:

- **XGBoost (ML) failed from distributional shift.** It learned the right attack categories, but this lab's traffic sits outside the statistical range the model learned "attack" to look like (the `Init_Win_bytes_forward` zero-variance finding, documented in the self-generated-attack deliverable).
- **Suricata (rules) failed from exact-match brittleness.** A rule can target the right attack category and still require byte-level conditions, a precise TCP-option fingerprint, a specific HTTP method-and-header combination, that a different tool version or a differently-configured attack run doesn't reproduce exactly. `SID 2009582` matching on window size alone wasn't enough; whatever else its content match checks for, this nmap run's crafted packets evidently didn't satisfy it.

Both detection paradigms generalized poorly to independently-generated attack traffic, for philosophically different reasons, exactly the gap this project's self-testing phase was built to expose. Neither "train a better model" nor "turn on more rules" is a complete answer by itself.

## ⚠️ Limitations

- Only the free Emerging Threats Open ruleset was tested; a more actively maintained or commercial ruleset might include less brittle matching logic.
- The exact byte-level reason `SID 2009582` didn't fire, despite matching its headline window-size condition, wasn't root-caused further. The most likely explanation is additional TCP-option or packet-fingerprint conditions in the rule's content match that this nmap version's crafted packets don't reproduce exactly, but this is a reasoned hypothesis, not a confirmed one.
- Suricata's SSH visibility is limited to the unencrypted version-banner exchange; anything requiring visibility into actual authentication success or failure, which is encrypted, is outside what a payload-matching rule can inspect.
- This comparison covers three attack captures against one open ruleset; it isn't a comprehensive evaluation of Suricata as a product.

## 🔭 Future Scope

- Try a more actively maintained or commercial ruleset (for example, Proofpoint ET Pro) to see whether its matching logic is less brittle against current tool versions.
- Capture the exact raw SYN packet nmap sends and diff it byte-by-byte against `SID 2009582`'s content match to confirm the precise condition that isn't being satisfied.
