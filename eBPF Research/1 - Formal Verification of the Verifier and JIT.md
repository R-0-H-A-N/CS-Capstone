---
tags: [ebpf, research, formal-verification, verifier, jit]
parent: "[[eBPF Research - Index]]"
status: active
source_count: 18
---
# 1 — Formal Verification of the Verifier and JIT

← Back to [[eBPF Research - Index]]

## Theme
Can the in-kernel eBPF verifier — and the JIT that compiles verified bytecode to native code — themselves be proven correct? This section covers the state of the art in mechanically verifying the range-analysis logic of the verifier (Agni, tnum) and the correctness of BPF JIT compilers (Jitterbug, Serval), plus two independent reviews of how far that formal coverage actually reaches.

**Start here:** 1.12 (survey, for orientation), then 1.1 (Agni — state of the art).

## Sources

### 1.1 Vishwanathan, Shachnai, Narayana & Nagarakatte — *Verifying the Verifier: eBPF Range Analysis Verification* (Agni) `[PR]`
CAV 2023
→ **[open]** https://people.cs.rutgers.edu/~sn349/papers/agni-cav2023.pdf

### 1.2 Vishwanathan et al. — *Comparing the Precision of Abstract Operators in the eBPF Verifier* `[PR]`
SAS 2025 · Agni follow-up, precision rather than soundness
→ **[open]** https://people.cs.rutgers.edu/~santosh.nagarakatte/papers/agni-sas-2025.pdf

### 1.3 Vishwanathan et al. — *Sound, Precise, and Fast Abstract Interpretation with Tristate Numbers* `[PR]`
CGO 2022 · **Distinguished Paper** · the tnum paper; now upstream in Linux
→ **[open]** https://arxiv.org/pdf/2105.05398 · Abstract https://arxiv.org/abs/2105.05398
→ ACM DL https://dl.acm.org/doi/abs/10.1109/CGO53902.2022.9741267

### 1.4 Bhat & Shacham — *Formal Verification of the Linux Kernel eBPF Verifier Range Analysis* `[TECH]`
May 2022
→ **[open]** https://sanjit-bhat.github.io/assets/pdf/ebpf-verifier-range-analysis22.pdf

### 1.5 Nelson, Van Geffen, Torlak & Wang — *Specification and Verification in the Field: Applying Formal Methods to BPF JIT Compilers in the Linux Kernel* (Jitterbug) `[PR]`
OSDI 2020
→ **[open] PDF** https://www.usenix.org/system/files/osdi20-nelson.pdf
→ Landing page https://www.usenix.org/conference/osdi20/presentation/nelson
→ Project page https://unsat.cs.washington.edu/projects/jitterbug/
→ Artifact (Docker) https://github.com/uw-unsat/jitterbug/tree/osdi20-artifact
→ LPC 2020 slides (good 20-min orientation) https://homes.cs.washington.edu/~lukenels/slides/2020-08-28-lpc.pdf

### 1.6 Nelson et al. — *Scaling Symbolic Evaluation for Automated Verification of Systems Code with Serval* `[PR]`
SOSP 2019 (best paper)
→ Project + PDF links https://unsat.cs.washington.edu/projects/serval

### 1.7 Van Geffen, Nelson, Dillig, Wang & Torlak — *Synthesizing JIT Compilers for In-Kernel DSLs* `[PR]`
CAV 2020
→ PDF linked from https://homes.cs.washington.edu/~lukenels

### 1.8 *eBPF Misbehavior Detection: Fuzzing with a Specification-Based Oracle* (Veritas / SpecCheck) `[PR]`
SOSP 2025
→ **[open]** https://nebelwelt.net/files/25SOSP.pdf

### 1.9 *Prove It to the Kernel: Precise Extension Analysis via Proof-Guided Abstraction Refinement* `[PR]`
SOSP 2025
→ ACM DL (may be paywalled) https://dl.acm.org/doi/10.1145/3731569.3764796

### 1.10 *Enhanced eBPF Verification and eBPF-based Runtime Safety Protection* (TrustedST) `[TECH]`
LangSec Workshop, IEEE S&P 2024 · sponsor-funded, treat with caution
→ **[open]** https://langsechq.gitlab.io/spw24/papers/LangSec2024-Enhanced-eBPF-verification-and-runtime-safety-protection-Paper.pdf

### 1.11 NCC Group — *eBPF Verifier Code Review*, v1.0 `[TECH]`
11 Nov 2024 · commissioned by the eBPF Foundation
→ **[open]** https://www.nccgroup.com/media/4lilthtf/ncc_group_nccgroup_e015561_report_2024-11-11_v10.pdf

### 1.12 *The eBPF Runtime in the Linux Kernel* (survey) `[PR-preprint]`
arXiv 2410.00026 · **start here** — §11.4 is the formal-methods gap statement
→ **[open]** https://arxiv.org/pdf/2410.00026

### 1.13 Sun & Su (ETH Zürich) — *Validating the eBPF Verifier via State Embedding* `[PR]`
OSDI 2024, pp. 615–628 · **self-consistency / dynamic testing oracle** — the counterpart to Agni's static SMT approach
→ **[open] PDF** https://www.usenix.org/system/files/osdi24-sun-hao.pdf *(⚠️ returned HTTP 403 on direct fetch 2026-08-07 — use the landing page)*
→ Landing page https://www.usenix.org/conference/osdi24/presentation/sun-hao
→ Talk (YouTube) https://www.youtube.com/watch?v=sASFr_9uZ7Y

**Summary.** Embeds concrete program states into an eBPF program as correctness checks, then lets the verifier itself validate whether it approximated those states correctly — a self-consistency oracle that needs no independent specification (contrast with 1.1's SMT extraction or 1.8's spec-based fuzzing). ⚠️ *The widely-repeated headline claim — 15 previously-unknown logic bugs found in one month, including two exploitable local-privilege-escalation bugs — comes from secondary summaries (search results, conference recap), not a verified read of the paper body; the primary PDF 403'd on fetch. Confirm the exact figures against the landing page or talk before citing them in writing.*

**Research directions.** Same author (Hao Sun) is now behind 1.15 (BCF) — read together, they trace one researcher's arc from *finding* verifier bugs via testing to *proposing a kernel mechanism* that would make the verifier's own soundness less load-bearing. Strengthens [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]]: a coverage audit that only inventories *static* verification methods (Agni, tnum, PREVAIL) is missing the dynamic half of the toolbox.

### 1.14 Zheng, Ji, Tao, Gao, Su, Zhang, Quinn & Williams — *Characterizing and Bridging the Diagnostic Gap in eBPF Verifier Rejections* (bpfix) `[PR-preprint]`
arXiv 2607.02748, 2026
→ **[open]** https://arxiv.org/pdf/2607.02748 · Code https://github.com/eunomia-bpf/bpfix

**Summary.** Not a soundness paper — a *diagnosability* one. Studies 235 real verifier rejections and finds 47% return only `EINVAL`, with one error string mapping to as many as nine distinct root causes; argues the verifier's reported failure point is often disconnected from where the underlying safety proof actually broke down. bpfix reconstructs that proof lifecycle from the verifier log to localize the true failure point, and the paper shows this materially improves LLM-based auto-repair success rates (+11–21 points over the raw log).

**Research directions.** Adjacent rather than central to "verifying the verifier" — this is about verifier *usability*, not correctness. Useful primarily as a source of empirical failure-mode taxonomy (root-cause categories) that a soundness-focused audit could reuse or cite as a related-but-distinct thread. Co-authored by Dan Williams, also a co-author of 2.1.

### 1.15 Hao Sun — *[PATCH RFC 00/17] bpf: Introduce proof-based verifier enhancement* (BCF — eBPF Certificate Framework) `[PRIM]`
LKML/bpf-next, RFC posted 6 Nov 2025 · **active, unresolved discussion as of Nov 2025**
→ Thread https://lkml.org/lkml/2025/11/6/949 · Maintainer reply (Starovoitov) https://lkml.org/lkml/2025/11/14/2048 · Follow-up https://lkml.org/lkml/2025/11/15/169

**Summary.** Kernel-side proof-carrying-code for eBPF: on a blocking verifier check, the kernel exports a refinement condition to userspace, an external (untrusted, solver-grade) tool computes a proof, and a small in-kernel checker (~5k LOC) validates the proof and resumes analysis — without turning the verifier itself into an SMT solver. Author claims 403 of 512 previously-rejected real-world programs now verify. Alexei Starovoitov's response (14 Nov 2025) pressed on the evaluation methodology (discrepancy between a ~1500-object-file corpus and the 512 figure cited) rather than rejecting the design outright; discussion was ongoing as of the last reply found (17 Nov 2025).

**Research directions.** This is 0.3 (Necula & Lee, Proof-Carrying Code, 1996) reincarnated almost exactly as that note predicted: *"A PCC-shaped eBPF would not need Agni."* Also the same author as 1.13 — a bug-finder now proposing to route around the class of bug he found. Because it is a live, contested, primary-source discussion (not yet merged, methodology actively questioned), it is a strong candidate for the "track a live kernel dispute to a defensible position" project shape that Topic #5 already uses for the Hornet LSM signing dispute — applied here to verification instead of supply chain.

### 1.16 Narayana & Nagarakatte (Rutgers) — *Verified Path Exploration for eBPF Static Analysis* (eBPF Foundation grant) + LSFMM+BPF 2025 talk `[PRIM]`
Grant awarded Aug 2024, $50,000, covering 2024–2025 · talk: *"Verifying the BPF verifier's path-exploration logic,"* LSFMM+BPF Summit, May 2025
→ Grant record https://ebpf.foundation/funding-opportunities/research-fund/ · Rutgers announcement https://www.cs.rutgers.edu/news-events/news/news-item/professors-santosh-nagarakatte-and-srinivas-narayana-awarded-an-ebpf-foundation-grant
→ LWN coverage of the talk https://lwn.net/Articles/1021825/

**Summary.** Extends Agni to path pruning — the verifier's state-comparison/state-pruning logic, which is both a critical performance optimization (lets the verifier handle programs with combinatorially many static paths) and, per LWN's account of the talk, has "already resulted in at least one security problem." Per the talk, the hard remaining piece is *sound generalization* — formalizing what it means for a pruned path to be a strict behavioral subset of a kept path — and BPF developers at LSFMM received the goal as ambitious but credible. **As of the most recent public update found (the May 2025 talk), this is not solved** — no follow-up blog post or completion announcement was located as of 2026-08-07, despite a second grant-required blog post nominally due 2026-08-01.

**Research directions.** This is the field's own leading group naming path pruning as *the* open soundness gap in the verifier — corroborating, from a primary source, exactly the kind of gap [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]] is built to find. Because Rutgers is actively and publicly working this specific gap with a multi-year head start and the original Agni artifact, a semester capstone should not attempt to *solve* path-pruning soundness — but citing it as the confirmed, still-open frontier is exactly the kind of primary-source grounding a coverage audit needs, and checking for a newer update before finalizing a proposal is cheap due diligence.

### 1.17 Vishwanathan, Shachnai, Narayana & Nagarakatte — *Automatic Synthesis of Abstract Operators for eBPF* `[PR]`
eBPF Workshop @ SIGCOMM 2025, Coimbra, Sept 2025 · companion to 1.2
→ **[open]** https://people.cs.rutgers.edu/~sn624/papers/synthesis-ebpf25.pdf

**Summary.** Uses CEGIS (counterexample-guided inductive synthesis) to automatically construct sound *and* more-precise abstract operators than the kernel's hand-written ones — concretely improving on the existing unsigned add/sub operators by exploiting wrap-around semantics. Also produces synthesized unsound operators as a byproduct, usable as a test corpus for other verification/fuzzing tools (including 1.8/1.13-style approaches).

**Research directions.** Same research line as 1.2/1.3/1.7; if a capstone builds a differential-testing or fuzzing harness (per 1.13, 0.6), this paper's synthesized-unsound-operator corpus is a ready-made source of known-bad test cases rather than something to construct from scratch.

### 1.18 Gershuni, Amit, Gurfinkel, Narodytska, Navas, Rinetzky, Ryzhyk & Sagiv — *Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions* (PREVAIL) `[PR]`
PLDI 2019 · promoted here as a first-class Section 1 source per [[0 - Seminal Papers - Foundational Reading]]'s action item — full entry and research directions at 0.6
→ **[open] PDF** https://vbpf.github.io/assets/prevail-paper.pdf · Code https://github.com/vbpf/ebpf-verifier

## Open questions / angles surfaced by this section
- §11.4 of 1.12 explicitly names what is *not yet* formally covered — a natural gap to build a capstone contribution around.
- Agni (1.1) verifies range-analysis soundness; 1.2 asks a *precision* question instead of soundness — is there a similar precision-vs-soundness split worth exploring for other verifier subsystems (e.g. pointer tracking, tnum arithmetic in 1.3)?
- Jitterbug/Serval (1.5–1.7) cover JIT correctness on specific architectures — coverage across all Linux-supported JIT backends (arm64, riscv, s390) is likely uneven and worth auditing.
- 1.8 and 1.9 (both SOSP 2025) are very recent and adjacent — fuzzing-with-oracle vs. proof-guided abstraction refinement — worth a compare/contrast note once both are read.
- 1.11 (NCC Group review) is an independent, non-academic audit — useful as a "what do practitioners think is still broken" cross-check against the academic verification papers.
- **The oracle landscape, laid out end to end**: SMT proof of hand-written operators (1.1/0.8), an independently-designed second verifier for differential testing (1.18/0.6), specification-oracle fuzzing (1.8), self-consistency dynamic testing (1.13), and now proof-carrying refinement that sidesteps the question of verifier soundness entirely (1.15). No source in this corpus compares these five oracle strategies against a shared corpus (e.g. kernel `selftests/bpf`) to characterize which bug classes each catches and which it structurally cannot — this is the strongest concrete, buildable gap in the section as of 2026-08-07.

See also: [[Potential Research Topics]] · [[Author Pages to Watch]]
