---
tags: [ebpf, research, formal-verification, verifier, jit]
parent: "[[eBPF Research - Index]]"
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
CGO 2022 · the tnum paper; partially upstreamed into Linux
→ *Direct link not retrieved.* Search DOI via ACM DL or the Rutgers publications page (see [[Author Pages to Watch]]).

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

## Open questions / angles surfaced by this section
- §11.4 of 1.12 explicitly names what is *not yet* formally covered — a natural gap to build a capstone contribution around.
- Agni (1.1) verifies range-analysis soundness; 1.2 asks a *precision* question instead of soundness — is there a similar precision-vs-soundness split worth exploring for other verifier subsystems (e.g. pointer tracking, tnum arithmetic in 1.3)?
- Jitterbug/Serval (1.5–1.7) cover JIT correctness on specific architectures — coverage across all Linux-supported JIT backends (arm64, riscv, s390) is likely uneven and worth auditing.
- 1.8 and 1.9 (both SOSP 2025) are very recent and adjacent — fuzzing-with-oracle vs. proof-guided abstraction refinement — worth a compare/contrast note once both are read.
- 1.11 (NCC Group review) is an independent, non-academic audit — useful as a "what do practitioners think is still broken" cross-check against the academic verification papers.

See also: [[Potential Research Topics]] · [[Author Pages to Watch]]
