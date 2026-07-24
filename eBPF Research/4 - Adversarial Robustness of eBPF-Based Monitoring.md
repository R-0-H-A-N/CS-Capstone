---
tags: [ebpf, research, security-monitoring, evasion, rootkits]
parent: "[[eBPF Research - Index]]"
---
# 4 — Adversarial Robustness of eBPF-Based Monitoring

← Back to [[eBPF Research - Index]]

## Theme
Sections [[1 - Formal Verification of the Verifier and JIT|1]]–[[3 - Secure eBPF Program Supply Chain|3]] are about trusting eBPF programs *going into* the kernel. This section flips the question: once eBPF is used defensively (Falco, Tetragon, Tracee, provenance-based IDS), how robust is that monitoring against an adversary who knows it's there? Split into 4A — grey-literature attacks aimed specifically at eBPF monitors — and 4B — peer-reviewed literature on evading the broader class of provenance-based ML/IDS detectors, which eBPF tooling is now converging with.

## 4A — Attacks on eBPF monitors specifically (grey literature)

### 4.1 Guo & Zeng — *Phantom Attack: Evading System Call Monitoring* `[GREY]`
DEF CON 29, Aug 2021 · TOCTOU (v1) + semantic confusion (v2); Falco and Tracee affected
→ Slides https://media.defcon.org/DEF%20CON%2029/DEF%20CON%2029%20presentations/Rex%20Guo%20Junyuan%20Zeng%20-%20Phantom%20Attack%20-%20%20Evading%20System%20Call%20Monitoring.pdf
→ PoC code https://github.com/rexguowork/phantom-attack
→ Video https://av.tib.eu/media/54228

### 4.2 Form3 Offensive Security — *Bypassing eBPF-based Security Enforcement Tools* `[GREY]`
Tetragon TracingPolicy kprobe bypass, worked example
→ https://www.form3.tech/blog/engineering/bypassing-ebpf-tools

### 4.3 0xMatheuZ — *Breaking eBPF Security: How Kernel Rootkits Blind Observability Tools* `[GREY]`
Telemetry-path attacks: iterators, ring/perf buffers, map ops
→ https://matheuzsecurity.github.io/hacking/ebpf-security-tools-hacking/

### 4.4 Synacktiv — *LinkPro: eBPF Rootkit Analysis* `[GREY, high quality]`
Oct 2025 · real incident, AWS-hosted infra; `getdents` + `sys_bpf` hooking, XDP magic-packet C2
→ https://www.synacktiv.com/en/publications/linkpro-ebpf-rootkit-analysis

### 4.5 Virus Bulletin 2025 — *Unmasking the Unseen: Modern Linux Rootkits and Detection* `[GREY]`
Taxonomy incl. eBPF and io_uring rootkits
→ **[open]** https://www.virusbulletin.com/uploads/pdf/conference/vb2025/papers/Unmasking-the-unseen-a-deep-dive-into-modern-Linux-rootkits-and-their-detection.pdf

### 4.6 Kelly, Callaghan & Martin — *eBPF Security Threat Model* `[TECH]`
ControlPlane / Linux Foundation · attack trees; the only structured threat model located
→ **[open]** https://www.linuxfoundation.org/hubfs/eBPF/ControlPlane%20—%20eBPF%20Security%20Threat%20Model.pdf

### 4.7 SPiCa `[GREY — unreviewed code]`
Cross-view eBPF rootkit detection, NMI canary, `.bss`-hidden globals. ⚠️ Widely miscited as research; it is a GitHub project with no published evaluation.
→ https://github.com/0xKirisame/SPiCa

## 4B — Peer-reviewed adjacent literature (evasion of provenance IDS)

### 4.8 Mukherjee, Wiedemeier, Wang et al. — *Evading Provenance-Based ML Detectors with Adversarial System Actions* (ProvNinja) `[PR]`
USENIX Security '23
→ **[open] PDF** https://www.usenix.org/system/files/usenixsecurity23-mukherjee.pdf
→ Landing page https://www.usenix.org/conference/usenixsecurity23/presentation/mukherjee
→ Slides https://www.usenix.org/system/files/sec23_slides_mukherjee.pdf

### 4.9 Goyal, A. et al. — *Sometimes, You Aren't What You Do: Mimicry Attacks against Provenance Graph Host IDS* `[PR]`
NDSS '23 ⚠️ *direct link not retrieved — search the NDSS 2023 programme*

### 4.10 Han, Pasquier et al. — *Provenance-based Intrusion Detection: Opportunities and Challenges* `[PR]`
TaPP 2018 · **the counterpoint** — argues provenance capture is adversarially robust
→ **[open]** https://www.usenix.org/system/files/conference/tapp2018/tapp2018-paper-han.pdf

### 4.11 *Marlin: Knowledge-Driven Analysis of Provenance Graphs* `[PR-preprint]`
arXiv 2403.12541 · §2.2 is a compact survey of PIDS evasion techniques
→ **[open]** https://arxiv.org/pdf/2403.12541

### 4.12 Yu, Y.-C. et al. — *Kernel-level Hidden Rootkit Detection Based on eBPF* (HKRD) `[PR]`
Computers & Security, Jun 2025 · DOI 10.1016/j.cose.2025.104582
→ https://www.sciencedirect.com/science/article/abs/pii/S0167404825002718 *(paywalled)*

### 4.13 Syairozi & Arizal — *Comparative Analysis of eBPF-Based Runtime Security Monitoring Tools* `[PR — weak]`
RITECH 2025, SCITEPRESS ⚠️ reports 100% detection / 0% FPR across three scripted attacks; useful as a **critique target**, not as evidence
→ **[open]** https://www.scitepress.org/Papers/2025/142727/142727.pdf

## Open questions / angles surfaced by this section
- 4.1–4.4 collectively describe three distinct attack surfaces against eBPF monitors: TOCTOU/semantic confusion (4.1), policy/hook bypass (4.2), and telemetry-path blinding (4.3, 4.4 in the wild). A capstone taxonomy that maps *which* eBPF-based tools are vulnerable to *which* class would be genuinely new — 4.6 is the only existing structured threat model and doesn't map to real incidents like 4.4.
- 4.10 is the explicit counterpoint to 4.8/4.9 — "provenance capture is adversarially robust" vs. two papers demonstrating evasion. That tension, especially once eBPF is the collection mechanism rather than generic provenance capture, is underexplored.
- 4.13 is flagged as a critique target: a 100%-detection/0%-FPR result across three scripted attacks is a textbook overfitting/weak-evaluation red flag — reproducing and stress-testing this tool's claims against 4.1–4.4's attack techniques could itself be a project.
- 4.7 (SPiCa) is unreviewed code frequently miscited as research — worth explicitly noting in any lit review to avoid propagating the miscitation.
- No source here directly tests whether the *hardware isolation* techniques in [[2 - Hardware-Assisted Isolation for eBPF]] would prevent the telemetry-blinding attacks in 4.3/4.4 — that intersection is unaddressed by the current source set.

See also: [[Potential Research Topics]]
