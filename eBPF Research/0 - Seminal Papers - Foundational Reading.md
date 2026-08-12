---
tags: [ebpf, research, seminal, foundations, verification, reading-list]
parent: "[[eBPF Research - Index]]"
status: active
source_count: 13
---
# 0 — Seminal Papers & Foundational Reading

← Back to [[eBPF Research - Index]]

## Purpose
Notes [[1 - Formal Verification of the Verifier and JIT]] through [[4 - Adversarial Robustness of eBPF-Based Monitoring]] cover the *current* literature — the papers you would cite as related work. This note covers the **ancestry**: the papers those papers cite. It is the citation base for a verification-flavoured capstone.

Two things it does that the other notes do not:
1. **Fills a real gap.** PREVAIL (0.6) — the abstract-interpretation alternative to the in-kernel verifier, and the basis of the eBPF-for-Windows verifier — appears nowhere in the four topic notes despite being the most directly relevant non-Linux verification effort in existence. The 1990s safe-kernel-extension line (0.2–0.5) is likewise absent, even though it defines the three competing safety philosophies that [[Potential Research Topics#4. Verification vs. hardware isolation vs. language-based safety — a position/evaluation piece]] proposes to compare.
2. **Gives a spine.** Entries already in the corpus are cross-referenced by their existing IDs (1.1, 1.5, …) rather than re-listed, so this reads as a reading order, not a second copy of the bibliography.

Quality tags follow the [[eBPF Research - Index]] legend (`[PR]` peer-reviewed, `[TECH]`, `[GREY]`, `[PRIM]`).
**Every URL below was fetched and returned HTTP 200 on 2026-07-29.** Where no open PDF was found, that is stated rather than guessed at.

---

## A. Origin

### 0.1 McCanne & Jacobson — *The BSD Packet Filter: A New Architecture for User-level Packet Capture* `[PR]`
USENIX Winter 1993 (LBL technical report, Dec 1992)
→ **[open]** https://www.tcpdump.org/papers/bpf-usenix93.pdf

**Summary.** Introduces the register-based packet-filter VM that everything called "BPF" descends from. Two 32-bit registers, a 16-slot scratch memory, read-only packet access, and — the design decision that matters most for your project — **no backward branches**, so the filter is a DAG and termination is structural rather than proven. The paper's argument is a performance one (filter in the kernel, avoid copying uninteresting packets to userspace), but the safety property it bought almost incidentally is the one the modern verifier has spent thirty years trying to preserve while relaxing.

**Research directions.** The historical baseline for any "how did the verifier's job get so hard?" framing — cBPF's safety was free because the language was too weak to be unsafe; eBPF's is expensive because the language is not. That trade-off curve is the through-line of [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]]. Also useful context for 0.13 (KFlex), which argues the curve is badly positioned today.

> ⚠️ [[eBPF Fundamentals]] §History dates this to 1992. Both are defensible — LBL tech report 1992, USENIX conference paper Winter 1993 — but cite the 1993 USENIX version.

---

## B. The safe-kernel-extension ancestry
These four papers, all 1993–1996, established the three strategies for running untrusted code in a kernel. Every modern eBPF safety paper in your corpus is a descendant of one of them. Reading them together is the cheapest way to see that Section 2's "competing philosophies" framing is a thirty-year-old argument, not a new one.

### 0.2 Wahbe, Lucco, Anderson & Graham — *Efficient Software-Based Fault Isolation* `[PR]`
SOSP 1993
→ **[open]** https://cs155.stanford.edu/papers/sfi.pdf
→ ACM DL (paywalled) https://dl.acm.org/doi/10.1145/168619.168635

**Summary.** The original SFI paper. Rather than proving an extension safe, *rewrite* it: confine each untrusted module to a "fault domain" — an aligned address-space segment — and instrument every store and indirect jump so the address is masked into that segment before use. Safety becomes a runtime property enforced by inserted instructions, at a reported overhead of a few percent, and the checker only has to verify that the instrumentation is present, not that the program is correct.

**Research directions.** Direct ancestor of the SFI work at 2.6 and of BeeBox (2.4). The "enforce, don't prove" split it introduced is exactly the axis [[Potential Research Topics#2. Portable hardware isolation that degrades gracefully]] proposes to build on — SFI is the *software fallback* in that project when no hardware primitive (MPK/PAC/MTE) is available, so this is the paper defining what that fallback costs. Reading it before 2.6 tells you which of 2.6's overheads are inherent to SFI and which are eBPF-specific.

### 0.3 Necula & Lee — *Safe Kernel Extensions Without Run-Time Checking* `[PR]`
OSDI 1996 · introduced Proof-Carrying Code · ACM SIGOPS Hall of Fame, 2006
→ **[open] PDF** https://www.eecs.umich.edu/courses/eecs588.w09/papers/pcc_osdi96.pdf
→ Landing page https://www.usenix.org/conference/osdi-96/safe-kernel-extensions-without-run-time-checking

**Summary.** The opposite bet from 0.2. The extension arrives carrying a machine-checkable *proof* that it satisfies the kernel's safety policy; the kernel runs a small, trusted proof checker instead of a large, trusted analysis. Trust is concentrated in the checker and the policy encoding, and there is zero runtime overhead because nothing is instrumented. The paper's demonstration was network packet filters — literally the eBPF use case, thirty years early.

**Research directions.** This is the intellectual parent of 1.9 (*Prove It to the Kernel*, SOSP 2025), whose title is a near-quotation. The interesting modern question PCC frames sharply: today's verifier is simultaneously the analysis *and* the trusted base, which is why Agni (1.1) has to verify the verifier at all. A PCC-shaped eBPF would not need Agni. Worth using as the framing device for [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]] — "what is in the TCB, and why is it that big?" is a better organising question for a coverage audit than a subsystem checklist.

### 0.4 Bershad, Savage, Pardyak, Sirer, Fiuczynski, Becker, Chambers & Eggers — *Extensibility, Safety and Performance in the SPIN Operating System* `[PR]`
SOSP 1995
→ **[open]** https://courses.cs.duke.edu/spring15/cps210/readings/spin-sosp95.pdf
→ Mirror https://www.cs.columbia.edu/~nieh/teaching/e6118_s00/papers/p267-bershad.pdf
→ ACM DL https://dl.acm.org/doi/10.1145/224056.224077

**Summary.** The third strategy: **language-based safety**. Extensions are written in Modula-3, a type-safe language, and the compiler plus a trusted linker enforce safety, so no separate verifier or runtime instrumentation is needed. Fine-grained interfaces are exported by the kernel and dynamically linked against, giving extension performance close to native at the cost of requiring a specific toolchain and trusting the compiler.

**Research directions.** The direct ancestor of Rex (2.8) — Rust-based, compiler-enforced eBPF — and the third leg of [[Potential Research Topics#4. Verification vs. hardware isolation vs. language-based safety — a position/evaluation piece]]. SPIN also supplies the strongest historical *counter-evidence* for that comparison: language-based extensibility lost to the verifier approach commercially despite winning technically, and understanding why (toolchain lock-in, compiler TCB, no support for adversarially-supplied bytecode) is a genuine contribution a comparison paper could make rather than just tabulating overheads.

### 0.5 Seltzer, Endo, Small & Smith — *Dealing with Disaster: Surviving Misbehaved Kernel Extensions* `[PR]`
OSDI 1996 · the VINO system
→ **[open] PDF** https://users.soe.ucsc.edu/~sbrandt/221/Papers/Kernel/seltzer-osdi96.pdf
→ Landing page https://www.usenix.org/conference/osdi-96/dealing-disaster-surviving-misbehaved-kernel-extensions
→ Author's copy (HTML) https://www.seltzer.com/assets/publications/Dealing-with-Disaster-Surviving-Misbehaving-Kernel-Extensions.htm
→ PostScript https://www.usenix.org/legacy/publications/library/proceedings/osdi96/full_papers/seltzer/seltzer.ps

**Summary.** The pragmatist's answer, and the one most often forgotten: accept that some extensions will misbehave, and make misbehaviour *survivable*. VINO uses **software fault isolation** (0.2) as its safety mechanism, but pairs it with a **lightweight transaction system** to handle what SFI cannot — resource hoarding, infinite loops, deadlock. Each extension ("graft") gets its own heap and stack; the kernel logs every state change a graft makes so that a misbehaving graft can be spontaneously aborted and its effects undone. Safety becomes a recovery property, not a static one.

**Research directions.** The missing fourth philosophy in your Section 2 framing. Nothing in the current corpus addresses *recovery* — everything is prevention (verify, isolate, or type-check). Given that Section 4's whole premise is that adversaries defeat eBPF-based monitors in practice, "what does the system do after safety fails?" is an under-occupied position, and VINO is the paper that stakes it out. Concretely: this is a strong angle to fold into [[Potential Research Topics#9. Does hardware isolation prevent telemetry-blinding attacks?]] — hardware isolation contains a compromised program, but says nothing about restoring the telemetry stream it blinded.

---

## C. The modern verifier line

### 0.6 Gershuni, Amit, Gurfinkel, Narodytska, Navas, Rinetzky, Ryzhyk & Sagiv — *Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions* (PREVAIL) `[PR]`
PLDI 2019 · **the single most important omission from the existing corpus**
→ **[open] PDF** https://vbpf.github.io/assets/prevail-paper.pdf
→ Project page https://vbpf.github.io/
→ Code https://github.com/vbpf/ebpf-verifier
→ ACM DL https://dl.acm.org/doi/10.1145/3314221.3314590

**Summary.** A ground-up replacement for the in-kernel verifier, built on **abstract interpretation** rather than the kernel's hand-rolled symbolic execution over a fixed path budget. Because it uses a proper abstract domain (Zone/Octagon-family numerical domains over registers and memory) with widening, PREVAIL handles **loops of arbitrary trip count** — the single largest expressivity restriction in the Linux verifier — while remaining scalable. It is verifier-as-a-library, running in userspace, and is the verifier used by **eBPF for Windows**.

**Research directions.** This is the paper your capstone most needs and does not currently have. It reframes almost every angle in [[Potential Research Topics]]:
- **A precision/soundness comparison with a real baseline.** 1.2 (SAS 2025) asks a precision question about the *kernel's* abstract operators; PREVAIL is an independently-designed answer to the same question. A head-to-head — which programs does each accept, where does each lose precision, at what analysis cost — is a well-scoped, empirically tractable capstone with both artifacts publicly available.
- **A second implementation to differential-test against.** Two independent verifiers for the same bytecode language is the ideal setup for differential fuzzing; 1.8 (Veritas/SpecCheck, SOSP 2025) fuzzes against a *specification* oracle, whereas PREVAIL gives you an *implementation* oracle. Disagreements are either kernel unsoundness or PREVAIL imprecision, and both are publishable.
- **Cross-platform verification.** PREVAIL is the load-bearing component of eBPF-for-Windows, so the "is verification portable?" question has a concrete testbed rather than being hypothetical.
- Strengthens [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]]: a coverage audit that ignores the existence of a formally-grounded alternative verifier is incomplete.

### 0.7 Vishwanathan, Shachnai, Narayana & Nagarakatte — *Sound, Precise, and Fast Abstract Interpretation with Tristate Numbers* `[PR]`
CGO 2022 · **Distinguished Paper** · = **1.3** in [[1 - Formal Verification of the Verifier and JIT]]
→ **[open] PDF** https://arxiv.org/pdf/2105.05398
→ Abstract https://arxiv.org/abs/2105.05398
→ ACM DL https://dl.acm.org/doi/abs/10.1109/CGO53902.2022.9741267

**Summary.** Formalises the *tnum* (tristate number) abstract domain the Linux verifier uses to track bitwise uncertainty in register values, proves the existing operators sound, shows the kernel's abstract multiplication was needlessly imprecise, and contributes a new algorithm that is both sound and more precise — **now upstream in Linux**. The cleanest available demonstration of the "formal methods → real kernel patch" pipeline.

**Research directions.** Note 1.3 records this as *"Direct link not retrieved"* — **that gap is now closed; update 1.3 with the arXiv link above.** As a base paper it is the best short worked example of the methodology (specify an abstract domain, prove soundness in an SMT solver, find imprecision, upstream a fix) that a capstone in this area would imitate on a different verifier subsystem — pointer tracking and the range-analysis operators outside tnum being the obvious untouched targets for [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]].

### 0.8 Vishwanathan, Shachnai, Narayana & Nagarakatte — *Verifying the Verifier: eBPF Range Analysis Verification* (Agni) `[PR]`
CAV 2023 · = **1.1** — full entry in [[1 - Formal Verification of the Verifier and JIT]]
→ **[open]** https://people.cs.rutgers.edu/~sn349/papers/agni-cav2023.pdf

**Summary.** Automatically extracts the verifier's range-analysis logic from the actual kernel C source into SMT formulae and checks soundness of every abstract operator against its concrete semantics — per kernel version, so it can be re-run as the verifier evolves. It found real unsound operators upstream. This is the state of the art for "verifying the verifier."

**Research directions.** The natural *baseline to extend*: Agni covers range analysis, so the contribution shape is "apply the Agni methodology to a subsystem Agni doesn't cover." See 1.2 for the precision follow-up and [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]] for the scoping.

### 0.9 Nelson, Van Geffen, Torlak & Wang — *Specification and Verification in the Field: Applying Formal Methods to BPF JIT Compilers in the Linux Kernel* (Jitterbug) `[PR]`
OSDI 2020 · = **1.5** — full link set (artifact, slides) in [[1 - Formal Verification of the Verifier and JIT]]
→ **[open]** https://www.usenix.org/system/files/osdi20-nelson.pdf

**Summary.** Moves the trust boundary one stage later: even a sound verifier is worthless if the JIT mistranslates verified bytecode into native code. Jitterbug defines a per-architecture specification of JIT correctness and verifies real Linux BPF JIT backends against it, finding and fixing genuine upstream bugs. Notably, it is designed for *ongoing* use by kernel developers, not a one-shot proof.

**Research directions.** The verifier-vs-JIT distinction is the cleanest way to scope a capstone that would otherwise be unbounded. Coverage across Linux's JIT backends (arm64, riscv, s390, powerpc) is uneven — the auditable gap that [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]] is built on.

### 0.10 Nelson, Bornholt, Gu, Baumann, Torlak & Wang — *Scaling Symbolic Evaluation for Automated Verification of Systems Code with Serval* `[PR]`
SOSP 2019 (Best Paper + Distinguished Artifact) · = **1.6**
→ Project + PDF https://unsat.cs.washington.edu/projects/serval/
→ ACM DL https://dl.acm.org/doi/10.1145/3341301.3359641

**Summary.** The framework Jitterbug is built on. Serval makes automated symbolic evaluation of real systems code tractable by having developers write lightweight symbolic profiles for interpreters, letting an SMT solver do the rest. Read it as *method*, not as a BPF result — it is the toolkit answer to "how would I actually verify a piece of kernel code in one semester?"

**Research directions.** Methodological. If your capstone verifies anything, Serval (or Rosette beneath it) is a plausible vehicle, and this paper is the honest account of where symbolic evaluation stops scaling — useful for setting a defensible scope.

---

## D. The counter-position, and orientation

### 0.11 Jia, Sahu, Oswald, Williams, Le & Xu — *Kernel Extension Verification is Untenable* `[PR]`
HotOS 2023 · = **2.1** in [[2 - Hardware-Assisted Isolation for eBPF]]
→ **[open]** https://people.cs.vt.edu/djwillia/papers/hotos23-untenable.pdf

**Summary.** The position paper arguing the whole enterprise above is a dead end: as eBPF's expressivity grows, the verifier's complexity and TCB grow with it, and static verification cannot keep pace. Proposes moving safety enforcement elsewhere — hardware isolation, or a safe language. Short, sharp, and deliberately provocative.

**Research directions.** Read *after* 0.2–0.5 and the recognition is immediate: this is 0.3 (PCC) vs. 0.2 (SFI) vs. 0.4 (SPIN), re-fought with modern hardware. That framing is itself a contribution and is the strongest available spine for [[Potential Research Topics#4. Verification vs. hardware isolation vs. language-based safety — a position/evaluation piece]] — a comparison grounded in thirty years of the argument rather than three.

### 0.12 *The eBPF Runtime in the Linux Kernel* (survey) `[PR-preprint]`
arXiv 2410.00026 · = **1.12** · §11.4 is the formal-methods gap statement
→ **[open]** https://arxiv.org/pdf/2410.00026

**Summary.** The orientation survey. Broad coverage of the runtime — verifier, JIT, maps, helpers, attach points — with a section that explicitly enumerates what formal methods do *not* yet cover. If you read one thing before anything else, read this, then come back here.

**Research directions.** §11.4 is a ready-made gap list; [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]] is essentially "turn §11.4 into a systematic audit."

### 0.13 Dwivedi, Iyer & Kashyap — *Fast, Flexible, and Practical Kernel Extensions* (KFlex) `[PR]`
SOSP 2024 · **not currently in the corpus**
→ **[open] PDF** — [rs3lab.github.io/…/dwivedi:kflex.pdf](https://rs3lab.github.io/assets/papers/2024/dwivedi:kflex.pdf) *(colon in the path; use the markdown link, a bare URL may truncate)*
→ ACM DL https://dl.acm.org/doi/10.1145/3694715.3695950
→ LPC 2024 talk https://lpc.events/event/18/contributions/1943/
→ Lab publications https://rs3lab.github.io/pages/publications.html

**Summary.** Separates *kernel safety* (the kernel must not be corrupted) from *extension correctness* (the extension should do what it claims), and enforces each with a mechanism suited to it rather than making the verifier do both. The result is materially more expressive extensions at low runtime overhead, backward-compatible with existing eBPF programs, with several mechanisms already upstreamed into mainline Linux.

**Research directions.** The constructive reply to 0.11 — it accepts "verification alone is untenable" without abandoning verification. Because parts are upstream, it is one of very few papers here you can evaluate against a *shipping* kernel rather than a research prototype. Slots directly into [[Potential Research Topics#4. Verification vs. hardware isolation vs. language-based safety — a position/evaluation piece]] as a fourth data point, and is the closest modern relative of 0.5's "make failure survivable" stance.

---

## Suggested reading order
Roughly 100 pages; the four ancestry papers are short and old-school-readable.

1. **0.12** — survey, for orientation (skim; read §11.4 properly)
2. **0.1** — where the VM came from and why it was safe for free
3. **0.2 → 0.3 → 0.4 → 0.5** — the four strategies, in order. Read these consecutively; the payoff is entirely in the contrast
4. **0.11** — the modern restatement of that same argument. Should now feel familiar
5. **0.6** — PREVAIL: the road not taken by Linux, and your best differential-testing partner
6. **0.7 → 0.8** — tnum then Agni: method, then state of the art
7. **0.9** (and 0.10 for method) — the JIT half of the trust boundary
8. **0.13** — the current synthesis attempt

## Actions this note suggests for the rest of the folder
- [x] Update **1.3** in [[1 - Formal Verification of the Verifier and JIT]] with the arXiv link from 0.7 — done 2026-08-07.
- [x] Add **PREVAIL (0.6)** to Section 1 as a first-class source — done 2026-08-07, as 1.18.
- [x] Add **KFlex (0.13)** to Section 2 — done 2026-08-07, as 2.11.
- [ ] Consider a **fifth safety philosophy — recovery/rollback (0.5)** — in Section 2's framing, currently absent.

## 2026-08-07 addendum — the field moved
Four sources surfaced in a fresh sweep, all added to [[1 - Formal Verification of the Verifier and JIT]] as 1.13–1.17 (and [[2 - Hardware-Assisted Isolation for eBPF]] as 2.12):
- **State Embedding (1.13, OSDI'24)** — a *dynamic/self-consistency* oracle for finding verifier bugs, distinct from every static/SMT method in this note. Same author (Hao Sun) as the next item.
- **BCF / "proof-based verifier enhancement" (1.15, LKML RFC, Nov 2025, still unresolved)** — proof-carrying code for eBPF, proposed as an actual kernel patch series. This is **0.3 (Necula & Lee, PCC) happening in real time** — read 0.3's research-directions note ("a PCC-shaped eBPF would not need Agni") and then read 1.15; the prediction and the patch series are close to a direct match.
- **The Rutgers path-pruning grant (1.16)** — confirms, from primary sources, that path pruning is the field's own leading group's named *open* soundness gap as of the most recent update found (May 2025 talk; no resolution located as of 2026-08-07).
- **Heimdall (2.12, arXiv 2605.25411, 2026)** — verified migration of legacy eBPF to Rust; not yet read in full, flagged for triage.

---
See also: [[eBPF Research - Index]] · [[Potential Research Topics]] · [[Author Pages to Watch]] · [[eBPF Fundamentals]]

Compiled: 2026-07-29 · all links verified live on that date
