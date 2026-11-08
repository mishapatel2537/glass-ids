# GlassIDS: Self-Generated Attack Testing 🧪

## 🎯 Objective

I wanted to test something the baseline model's own strong CIC-IDS2017 numbers can't answer on their own: does a detector trained on a 2017 public dataset actually recognize attacks it's never seen, generated independently in my own lab in 2026? A model can score very well on a benchmark it was trained and tested on and still fail completely on traffic it's never encountered, which is exactly the traffic it would face in the real world. This document reports what happened when I actually tested that gap instead of assuming it away.

## 🧪 Lab Setup

Two VirtualBox VMs on an isolated internal network (`glassids-lab`), fully separated from my real LAN: Kali Linux as the attacker (`192.168.56.10`) and a disposable Ubuntu Server target (`192.168.56.11`). I generated three attacks from Kali, each captured with `tcpdump` and verified in Wireshark before being trusted:

1. **Port scan**: `nmap -sS -p-`, a SYN scan across all 65,535 ports
2. **SSH brute-force**: `hydra` against a short custom wordlist, with the real target password embedded partway through rather than first or last
3. **Slow-rate DoS**: `slowhttptest -H` (Slowloris-style), run against a throwaway `python3 -m http.server`

## 🏷️ Ground-Truth Labeling

Unlike CIC-IDS2017, these captures have no `Label` column, so I had to build ground truth myself. The obvious shortcut, "flows from the attacker's IP are the attack," doesn't actually work here: my own control-plane traffic (SSH'ing in to start `tcpdump`, or to run the HTTP server) also came from Kali's IP in every single capture, so source IP alone can't separate attack from benign.

I used `tshark`'s conversation view plus Wireshark's packet-level inspection (TCP flags, handshake shape, Follow Stream) to separate each capture properly:

- **Port scan**: one long-duration, real-data flow on port 22 is my own control session; everything else (near-instant half-open SYN flows across every port) is the attack.
- **Brute-force**: the one flow present from the very start of the capture is my control session; `hydra`'s cluster of connections, which all start later in a tight burst, is the attack.
- **Slow-DoS**: two separate benign flows on port 22, not one. A short one from typing the `tcpdump` command, and a much longer one I didn't expect going in: `http.server` logs every incoming request to its own console, and with ~290 slow connections hammering it for nearly three minutes, that log streamed back over SSH almost the whole time. I kept this as its own category rather than folding it into generic "benign," since it's a realistic pattern (an analyst's session during a live incident) that CIC-IDS2017 doesn't represent at all. On port 8000, I split the attack traffic itself into the ~1,000 sustained slow-trickle flows and ~40 fast periodic connections, which turned out to be `slowhttptest`'s own health-check probes, kept separate to see whether the model could tell the real slow attack apart from the attacker's own fast, ordinary-looking probe traffic.

## 🔧 Flow-Extraction Pipeline Issues

Running these captures through the same `cicflowmeter` pipeline I validated earlier surfaced real bugs that never showed up on a pre-made public dataset:

- **Non-deterministic flow counts.** Re-extracting the same slow-DoS capture repeatedly produced a different row count every time (2,464, then 3,264, then 2,750...). I traced this to a background thread in the installed package that evicts flows based on real wall-clock time rather than the timestamps inside the capture itself, meaning the result depended on how fast my machine happened to process the file that particular run, not on anything in the pcap. I fixed it by disabling that thread for offline file processing, where it serves no purpose anyway (it exists for live capture). Confirmed fixed by re-extracting twice in a row and getting identical output, then independently cross-checking that count against `tshark`'s own conversation total, which matched exactly.
- **A unit mismatch I had to apply more broadly than before.** `cicflowmeter` reports `Flow Duration` and every other timing-derived column (IAT, Active, Idle statistics) in seconds, while the model's training data uses microseconds. I'd only spot-checked this for one column during pipeline validation; this time I had to apply it everywhere it actually mattered, or the model would have silently seen a 120-second attack as `120` instead of the `~120,000,000` it learned to recognize.
- **A direction-flip bug specific to the slow-DoS capture.** `capinfos` had already flagged this file as `Strict time order: False`, and it turned out that for two flows, `cicflowmeter` recorded the target as the "source" instead of Kali, because of how it decides direction from whichever packet it processes first. Fixed by matching control-plane flows on either port rather than destination port alone.

## 📈 Results

| Category | Flows | Predicted BENIGN | Detection Rate |
|---|---|---|---|
| Port scan (attack) | 65,535 | 65,535 | **0%** |
| SSH brute-force (attack) | 14 | 14 | **0%** |
| Slow-DoS, sustained (attack) | 1,000 | 1,000 | **0%** |
| Slow-DoS, probes (attacker tool traffic) | 41 | 41 | n/a, see note below |
| My own control-plane traffic (benign) | 4 | 4 | 0 false positives |

I'm not treating the probe row as a detection-rate failure the same way as the other three. These 41 flows are fast, structurally ordinary-looking HTTP connections that `slowhttptest` itself generates as periodic health checks, not the attack's actual signature, so correctly calling them benign would arguably be the right answer, not a miss.

## 🔍 Key Findings & Limitations

1. **Total detection failure across all three self-generated attacks, despite near-perfect scores on these exact same classes within CIC-IDS2017 itself** (PortScan: 0.989 precision / 1.000 recall; SSH-Patator: 0.998 / 0.999, both from the baseline evaluation). This is the central result the whole phase was built to surface: strong benchmark performance on 2017 data doesn't guarantee the model recognizes the same attack categories when they're generated independently, even by the exact tools the dataset itself modeled.

2. **`Init_Win_bytes_forward` (TCP initial window size) has zero variance across all 66,549 self-captured attack flows**, pinned to one fixed value per attack type (1024 for the port scan, 64240 for both brute-force and slow-DoS), while CIC-IDS2017's own training data for these classes shows real variation (standard deviation 7,600 to 14,100, spanning several distinct values). My best explanation: my whole lab is one Kali VM's TCP stack generating every attack, while CIC-IDS2017 was captured across many real, differently-configured machines in 2017. If the model learned to treat this feature's variation as meaningful signal, a single homogeneous attacker machine can't reproduce that signal no matter how realistic the attack behavior itself is. 🎯

3. **Not every feature failed to generalize.** The slow-DoS flows' `Fwd Packets/s` median (0.152) landed close to CIC's own slowloris median (0.151), and `Flow Duration` for those same flows (median ~121s) matched what I'd independently confirmed in Wireshark almost exactly. The gap is concentrated in specific features, `Init_Win_bytes_forward` especially, not a wholesale "nothing matches" story.

4. **The brute-force duration gap has a mundane, non-concerning explanation.** My `hydra` connections ran far shorter (median 2.5s) than CIC's SSH-Patator examples (median ~12s), which lines up with my short custom wordlist and `sshd`'s `MaxAuthTries=6` forcing frequent reconnects. This is a setup difference, not a detection-relevant anomaly.

5. **Only 4 benign flows total in this evaluation**, so the 0% false-positive result carries very low statistical confidence and shouldn't be read as a strong claim in either direction, the same tiny-sample caveat the baseline deliverable already applies to Heartbleed, Infiltration, and SQL Injection.

6. **This lab is a single attacker machine against a single target, both on one physical host.** The flat window-size signature may be an artifact of that homogeneity rather than something inherent to the attack tools themselves. I didn't have the infrastructure to generate the same attacks from multiple distinct source machines or operating systems to test that directly.

7. **I wasn't able to get SHAP's per-flow explanation working against these self-captured flows** (a library-internal feature-metadata mismatch I couldn't resolve after a reasonable amount of effort), so the mechanism behind the misclassifications here is inferred from direct feature-distribution comparisons against training data, not from the model's own stated attribution the way the agentic layer's alert packets are.

8. **This only covers the three attack types tested here.** It says nothing about how well the model would generalize to attack types this phase didn't test.
