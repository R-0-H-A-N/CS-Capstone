---
tags: [ebpf, research, capstone-ideas]
parent: "[[eBPF Research - Index]]"
---
# Potential Research Topics

← Back to [[eBPF Research - Index]]

Candidate capstone angles synthesized from the gaps, tensions, and disputes surfaced across all four topic areas. Each links back to its source section for the underlying reading.

## 1. Coverage audit of formal-methods gaps in the verifier/JIT
The survey at 1.12 (§11.4) explicitly names what current formal verification work does *not* yet cover. Agni (1.1/1.2) covers range analysis; Jitterbug/Serval (1.5–1.7) cover specific JIT backends. A project could systematically map which verifier subsystems and which JIT architectures (arm64, riscv, s390 — Linux supports several BPF JIT backends beyond x86) remain formally unverified, and characterize the risk each gap represents.
→ Source section: [[1 - Formal Verification of the Verifier and JIT]]

## 2. Portable hardware isolation that degrades gracefully
MOAT/CCS-25 (2.2, 2.7) use Intel MPK; HIVE and SafeBPF (2.3, 2.5) use ARM-specific PAC/MTE/unprivileged load-store. No source addresses a portable isolation layer that picks the best available hardware primitive per architecture, or falls back to software SFI (2.6) when none is available. This is a systems-building project with a clear evaluation axis (performance overhead vs. threat coverage per platform).
→ Source section: [[2 - Hardware-Assisted Isolation for eBPF]]

## 3. Transient-execution (Spectre-class) coverage for hardware-isolated eBPF
BeeBox (2.4) is the only source in this set addressing speculative-execution attacks against BPF, and it does so via SFI, not the MPK/PAC/MTE mechanisms in 2.2/2.3/2.5. Whether those hardware-isolation schemes are themselves vulnerable to transient-execution side channels is an open question no source here answers directly — a natural intersection project.
→ Source section: [[2 - Hardware-Assisted Isolation for eBPF]]

## 4. Verification vs. hardware isolation vs. language-based safety — a position/evaluation piece
Section 2 contains three competing philosophies for eBPF safety: static verification (challenged by 2.1), hardware isolation (2.2–2.7, 2.9–2.10), and language-based safety (2.8, Rex). A comparative evaluation — implementation cost, performance overhead, threat coverage, developer ergonomics — across all three would synthesize the whole section into one deliverable.
→ Source section: [[2 - Hardware-Assisted Isolation for eBPF]]

## 5. Resolve or document the Hornet LSM signing dispute
3.2 describes a *live, unresolved* kernel mailing-list dispute over loader-only vs. loader+map verification and TOCTOU on map hashes, sitting downstream of the 6.18 signed-BPF work (3.1, 3.8) and the CO-RE relocation problem (3.3). A project could track this thread to resolution, prototype a fix for the TOCTOU gap, or write the definitive explainer connecting 3.1→3.2→3.3→3.8 into one coherent narrative — genuinely useful since no single source currently does this.
→ Source section: [[3 - Secure eBPF Program Supply Chain]]

## 6. In-kernel signing vs. out-of-tree BPF-LSM prototype — a head-to-head
3.4 (Two-Phase eBPF Program Signing) solves program-signing with a BPF-LSM prototype and no kernel changes, built independently of the in-tree signed-BPF effort (3.1/3.8). A comparative implementation/evaluation — trust model, deployment friction, TOCTOU exposure — would be a concrete systems project with a working prototype on each side already available as a starting point.
→ Source section: [[3 - Secure eBPF Program Supply Chain]]

## 7. Taxonomy of eBPF-monitor evasion techniques, mapped to real tools
4.1–4.4 describe three distinct attack classes (TOCTOU/semantic confusion, policy/hook bypass, telemetry-path blinding) against named tools (Falco, Tracee, Tetragon) plus one real-world incident (LinkPro, 4.4). 4.6 is the only structured threat model but predates 4.4 and doesn't map to it. A project could build the missing taxonomy — which attack class defeats which tool/mechanism — and test it empirically.
→ Source section: [[4 - Adversarial Robustness of eBPF-Based Monitoring]]

## 8. Stress-test a weak-evaluation eBPF monitoring paper
4.13 reports 100% detection / 0% false-positive rate across three scripted attacks — a classic overfitting/weak-evaluation signature. Reproducing its evaluation and re-testing against the real attack techniques in 4.1–4.4 (which it likely wasn't evaluated against) would make a solid critique-and-improve project, and doubles as a methodology lesson in evaluating security-tool claims.
→ Source section: [[4 - Adversarial Robustness of eBPF-Based Monitoring]]

## 9. Does hardware isolation prevent telemetry-blinding attacks?
No source in the set connects Section 2 (hardware isolation) to Section 4 (monitor evasion). Specifically: would MOAT/HIVE/SafeBPF-style isolation (2.2/2.3/2.5) actually prevent the ring-buffer/iterator/map-op blinding attacks documented in 4.3/4.4? This is the single clearest *unaddressed intersection* across the whole source set and could anchor an entire capstone.
→ Source sections: [[2 - Hardware-Assisted Isolation for eBPF]] · [[4 - Adversarial Robustness of eBPF-Based Monitoring]]

## 10. Provenance-IDS robustness claims, applied specifically to eBPF collectors
4.10 argues provenance capture is adversarially robust; 4.8/4.9 demonstrate evasion against provenance-based ML detectors generally (not eBPF-specific). Since eBPF is increasingly *the* collection mechanism for provenance-based IDS (Tetragon, Tracee), re-running the ProvNinja/mimicry-style attacks (4.8/4.9) against an eBPF-specific collector would test whether the collection mechanism itself changes the evasion picture.
→ Source section: [[4 - Adversarial Robustness of eBPF-Based Monitoring]]

---
Once a direction is chosen, update [[eBPF (Extended Berkeley Packet Filter)]] §1–§2 (Problem Statement, Objectives) with the selected angle and prune this list down to the sources that remain relevant.
