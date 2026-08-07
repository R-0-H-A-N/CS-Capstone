---
tags: [ebpf, research, hardware-isolation, mpk, sfi]
parent: "[[eBPF Research - Index]]"
---
# 2 — Hardware-Assisted Isolation for eBPF

← Back to [[eBPF Research - Index]]

## Theme
If static verification can't be fully trusted (see 2.1's central argument, and the gaps documented in [[1 - Formal Verification of the Verifier and JIT]]), what does defense-in-depth look like using actual hardware isolation primitives — Intel MPK, ARM PAC/MTE/unprivileged load-store, software fault isolation? This section is the "isolation response" to Section 1's verification question.

**Start here:** 2.1 (the motivating argument), then 2.2 or 2.3 (the isolation response).

## Sources

### 2.1 Jia, Sahu, Oswald, Williams, Le & Xu — *Kernel Extension Verification is Untenable* `[PR]`
HotOS '23 · **the motivating paper for this whole section**
→ **[open]** https://people.cs.vt.edu/djwillia/papers/hotos23-untenable.pdf
→ DOI https://dl.acm.org/doi/10.1145/3593856.3595892

### 2.2 Lu, Wang, Wu, He & Zhang — *MOAT: Towards Safe BPF Kernel Extension* `[PR]`
USENIX Security '24, pp. 1153–1170 · Intel MPK
→ **[open] PDF** https://www.usenix.org/system/files/usenixsecurity24-lu-hongyi.pdf
→ Mirror https://cse.sustech.edu.cn/faculty/~zhangfw/paper/moat-usenixsecurity24.pdf
→ Extended arXiv version https://arxiv.org/pdf/2301.13421
→ Landing page https://www.usenix.org/conference/usenixsecurity24/presentation/lu-hongyi
→ Code + project site https://sites.google.com/view/safe-bpf/
→ Talk (17 min) https://www.youtube.com/watch?v=__2WUqcTJjg

### 2.3 Zhang, Wu, Meng, Zhang et al. — *HIVE: A Hardware-assisted Isolated Execution Environment for eBPF on AArch64* `[PR]`
USENIX Security '24, pp. 163–180 · ARM unprivileged load/store + PAC
→ **[open] PDF** https://yinqian.org/papers/sec24a.pdf
→ Landing page https://www.usenix.org/conference/usenixsecurity24/presentation/zhang-peihua

### 2.4 Jin, Gaidis & Kemerlis — *BeeBox: Hardening BPF against Transient Execution Attacks* `[PR]`
USENIX Security '24, pp. 613–630 · SFI vs. speculative execution
→ Landing page (PDF linked there) https://www.usenix.org/conference/usenixsecurity24/presentation/jin-di

### 2.5 Lim, Prasad, Han & Pasquier — *SafeBPF: Hardware-assisted Defense-in-depth for eBPF Kernel Extensions* `[PR]`
CCSW '24 · ARM MTE + software SFI
→ **[open] arXiv** https://arxiv.org/abs/2409.07508 · PDF https://arxiv.org/pdf/2409.07508
→ Author copy https://tfjmp.org/publications/2024-ccsw.pdf
→ ACM DOI 10.1145/3689938.3694781

### 2.6 Lim, Han & Pasquier — *Unleashing Unprivileged eBPF Potential with Dynamic Sandboxing* (SandBPF) `[PR]`
SIGCOMM eBPF Workshop '23 · software-only predecessor to SafeBPF
→ **[open]** https://arxiv.org/pdf/2308.01983

### 2.7 Lu, H. — *Hardware-assisted Memory Isolation* (dissertation abstract) `[PR]`
ACM CCS 2025 · Moat extended: MPK + MMU for unlimited domains
→ https://dl.acm.org/doi/10.1145/3719027.3765566

### 2.8 Jia et al. — *Rex: Closing the Language-Verifier Gap with Safe and Usable Kernel Extensions* `[PR]`
USENIX ATC '25, pp. 325–342 · the *language-based* alternative to hardware isolation
→ **[open] arXiv** https://arxiv.org/abs/2502.18832 · PDF https://arxiv.org/pdf/2502.18832
→ Author copy https://people.cs.vt.edu/djwillia/papers/atc25-rex.pdf
→ Landing page https://www.usenix.org/conference/atc25/presentation/jia
→ Code https://github.com/rex-rs/rex

### 2.9 Patel et al. — NSDI '26 `[PR]` ⚠️ *title unconfirmed*
→ **[open]** https://www.usenix.org/system/files/nsdi26-patel.pdf
Useful chiefly for its related-work map of this section. Verify bibliographic details before citing.

### 2.10 Li, Gu, Xia, Zang & Chen — *Memory Isolation Mechanism of eBPF Based on PKS Hardware Feature* `[PR]`
Journal of Software (China), 2022 ⚠️ *cited secondhand only; original not retrieved*

### 2.11 Dwivedi, Iyer & Kashyap — *Fast, Flexible, and Practical Kernel Extensions* (KFlex) `[PR]`
SOSP 2024 · added per [[0 - Seminal Papers - Foundational Reading]]'s action item (= 0.13, full entry there)
→ **[open] PDF** [rs3lab.github.io/…/dwivedi:kflex.pdf](https://rs3lab.github.io/assets/papers/2024/dwivedi:kflex.pdf) · ACM DL https://dl.acm.org/doi/10.1145/3694715.3695950

**Summary.** Separates *kernel safety* from *extension correctness* and enforces each with a purpose-fit mechanism instead of asking the verifier to do both — a constructive reply to 2.1 that accepts "pure verification is untenable" without abandoning verification entirely. Several mechanisms are already upstreamed into mainline Linux, making it one of the few sources in this set evaluable against a shipping kernel.

### 2.12 — *Heimdall: Formally Verified Automated Migration of Legacy eBPF Programs to Rust* `[PR-preprint]`
arXiv 2605.25411, 2026
→ **[open]** https://arxiv.org/pdf/2605.25411

**Summary.** ⚠️ *Not yet read in full — flagging for triage.* Combines two threads already in this vault: formal verification (Section 1) and language-based safety via Rust (2.8, Rex) — specifically automated, formally-verified translation of existing restricted-C eBPF programs into Rust rather than requiring programs to be authored in a safe language from scratch. Directly relevant if the language-based-safety pole of [[Potential Research Topics#4. Verification vs. hardware isolation vs. language-based safety — a position/evaluation piece]] is pursued: it addresses that comparison's biggest practical objection (existing eBPF is not going to be rewritten by hand).

## Open questions / angles surfaced by this section
- 2.1 is the thesis; 2.8 (Rex) is the counter-thesis — a *language-based* alternative to hardware isolation rather than hardware isolation itself. A capstone could position itself explicitly between these two poles.
- MPK (2.2, 2.7) vs. ARM-specific mechanisms (2.3, 2.5 — PAC/MTE) split along architecture lines. Is there room for a portable/hybrid isolation layer that degrades gracefully depending on available hardware features?
- 2.4 (BeeBox) is the only source addressing *transient execution* (Spectre-class) attacks against BPF — most of this section is silent on speculative-execution threats, which is a real gap.
- 2.5/2.6 (SafeBPF / SandBPF) show a software-only predecessor evolving into a hardware-assisted successor — worth reading as a pair to see what hardware assistance actually bought them (performance? threat coverage? both?).
- 2.9 and 2.10 are the weakest-sourced entries here (unconfirmed title; secondhand citation) — verify before relying on either for a lit review.

See also: [[1 - Formal Verification of the Verifier and JIT]] · [[Potential Research Topics]] · [[Author Pages to Watch]]
