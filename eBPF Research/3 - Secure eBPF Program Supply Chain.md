---
tags: [ebpf, research, supply-chain, signing, lsm]
parent: "[[eBPF Research - Index]]"
---
# 3 — Secure eBPF Program Supply Chain

← Back to [[eBPF Research - Index]]

## Theme
Verification and hardware isolation (Sections [[1 - Formal Verification of the Verifier and JIT|1]] and [[2 - Hardware-Assisted Isolation for eBPF|2]]) both assume the *bytecode reaching the kernel* is the bytecode you intended to load. This section is about that assumption: how do you sign, attest, and gate eBPF programs so a compromised loader or build pipeline can't smuggle in something the verifier will happily accept? This topic is almost entirely primary sources (kernel patches, LWN coverage, live mailing-list disputes) rather than peer-reviewed papers — that's the defining characteristic of where this problem currently lives: in-flight kernel engineering, not academia.

## Sources

### 3.1 *Signed BPF programs* — patch series discussion `[PRIM]`
LWN.net, 21 Sep 2025 · the series that became 6.18; documents the libbpf-in-TCB problem
→ https://lwn.net/Articles/1038961/

### 3.2 Boscaccy, B. — *[PATCH v3 0/9] Reintroduce Hornet LSM* `[PRIM]`
linux-kernel, 26 Mar 2026 · the live dispute: loader-only vs. loader+map verification, TOCTOU on map hashes
→ https://lkml.iu.edu/2603.3/05269.html

### 3.3 *eBPF in Kernel Lockdown Mode* (short paper) `[PRIM]`
Linux Plumbers Conference · the clearest statement of the CO-RE-relocation-invalidates-signature problem
→ **[open]** https://lpc.events/event/7/contributions/678/attachments/580/1177/eBPF-in-kernel-lockdown-mode-short-paper.pdf
→ Session page https://lpc.events/event/7/contributions/678/

### 3.4 Wang, C. — *Two-Phase eBPF Program Signing* `[GREY]`
2 Feb 2025 · BPF-LSM prototype, no kernel changes
→ Write-up https://wangcong.org/2025-02-02-two-phase-ebpf-program-signing.html
→ Code https://github.com/congwang/ebpf-2-phase-signing

### 3.5 *BPF token and BPF FS-based delegation* `[PRIM]`
LWN.net · merged Linux 6.9
→ https://lwn.net/Articles/953427/ and https://lwn.net/Articles/959350/

### 3.6 Corbet, J. — *Unprivileged bpf()* `[PRIM]`
LWN, 12 Oct 2015
→ https://lwn.net/Articles/660331/

### 3.7 Corbet, J. — *Reconsidering unprivileged BPF* `[PRIM]`
LWN, 16 Aug 2019
→ https://lwn.net/Articles/796328/

### 3.8 Release documentation for signed BPF in 6.18 `[PRIM]`
→ Kernel Newbies https://kernelnewbies.org/Linux_6.18
→ Phoronix summary https://www.phoronix.com/news/Linux--6.18-BPF

### 3.9 Srinivasan, Naveen & Naveen — *Bomfather* `[PR-preprint]`
arXiv 2503.02097 · ⚠️ **inverse topic** — eBPF *for* supply chain, not supply chain *of* eBPF. Included so you can cite it as a scoping exclusion.
→ **[open]** https://arxiv.org/pdf/2503.02097

## Open questions / angles surfaced by this section
- 3.1 → 3.2 → 3.3 → 3.8 form a chronological arc: the 6.18 signed-BPF patch series, a live follow-on LSM dispute, the CO-RE relocation problem that motivated lockdown-mode work, and the eventual release notes. Reading them in that order tells the whole story of how this feature landed.
- 3.2 is described as a *live dispute* (loader-only vs. loader+map verification, TOCTOU on map hashes) — this is genuinely unresolved kernel engineering as of the source date. A capstone could track this thread to its resolution or propose a position on it.
- 3.4 is a grey-literature prototype (BPF-LSM, no kernel changes) sitting *outside* the upstream signing effort — worth comparing against the in-tree approach for a "what would it take to do this without kernel patches" angle.
- 3.6/3.7 (2015, 2019 LWN pieces) show unprivileged BPF policy swinging back and forth over a decade — useful historical framing for why supply-chain trust became necessary at all.
- 3.9 is flagged explicitly as out of scope (inverse topic) — worth keeping only as a "don't confuse this with—" reference, not a citation.

See also: [[Potential Research Topics]]
