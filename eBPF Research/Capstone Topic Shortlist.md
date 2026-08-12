---
tags: [ebpf, research, capstone-ideas]
parent: "[[eBPF Research - Index]]"
status: active
---
# Capstone Topic Shortlist

← Back to [[eBPF Research - Index]]

A curated shortlist pulled together on 2026-08-07 in response to a request for capstone topics, with emphasis on "verifying the verifier." General angles are condensed from [[Potential Research Topics]]; the verifier-specific angles are new synthesis grounded in [[1 - Formal Verification of the Verifier and JIT]].

## General capstone directions (from across all four sections)

1. **Coverage audit of formal-methods gaps** — map which verifier subsystems and JIT backends (arm64, riscv, s390) remain formally unverified.
2. **Portable hardware isolation** — a layer that picks MPK/PAC/MTE per architecture, falling back to software SFI.
3. **Transient-execution coverage** — do MPK/PAC/MTE-based isolation schemes have their own Spectre-class holes?
4. **Verification vs. hardware isolation vs. language-based safety** — a comparative evaluation piece.
5. **The Hornet LSM signing dispute** — a live, unresolved kernel-mailing-list thread worth tracking to resolution.
6. **Monitor-evasion taxonomy** — mapping attack classes to specific tools (Falco, Tracee, Tetragon).
7. **Does hardware isolation stop telemetry-blinding attacks?** — the clearest unaddressed intersection in the corpus.

Full detail on all of these lives in [[Potential Research Topics]] already — this section only condenses it for side-by-side comparison with the verifier-specific angles below.

## Verifying the verifier — primary interest

Five concrete angles, in ascending order of ambition.

### A. The oracle-comparison study
*The strongest gap flagged in [[1 - Formal Verification of the Verifier and JIT#Open questions / angles surfaced by this section]], dated 2026-08-07.*

Nobody in the corpus runs the five existing "verify the verifier" strategies against a shared corpus, such as the kernel's own `selftests/bpf`:
- SMT proof of hand-written operators — Agni, 1.1
- an independent second verifier for differential testing — PREVAIL, 1.18
- specification-oracle fuzzing — 1.8
- self-consistency dynamic testing via state embedding — 1.13
- proof-carrying refinement — BCF, 1.15

A capstone that characterizes *which bug classes each oracle catches and which it structurally cannot* would be genuinely novel and is buildable: most of these have public artifacts (Agni, PREVAIL, BCF's RFC patches) to start from rather than build from scratch.

### B. Track the BCF (proof-carrying eBPF) dispute to a defensible position
*Source: 1.15.*

Hao Sun's RFC is live and contested as of November 2025 — Starovoitov pushed back specifically on the evaluation methodology (a corpus-size discrepancy between ~1500 object files and the cited 512-program figure). This is nearly identical in shape to the Hornet dispute (general topic 5 above), except applied to verification rather than supply chain. A project could reproduce the disputed 403/512 figure, resolve the discrepancy, and either vindicate or debunk the claim with a defensible methodology — a rare chance to do primary, adjudicable research on an open kernel argument rather than review closed literature.

### C. The unsound-operator fuzzing corpus
*Sources: 1.17 + 1.13.*

1.17's CEGIS synthesis work throws off *known-unsound* abstract operators as a byproduct. Nobody has fed that corpus into 1.13's self-consistency state-embedding approach to see if it catches them. Smaller in scope than A, but has a very clean "does oracle X detect known-bad case Y" evaluation built in.

### D. JIT-side gap, mirroring the verifier-side one
*Sources: 1.5–1.7.*

Jitterbug/Serval established JIT correctness proofs, but coverage across Linux's supported BPF JIT backends (x86, arm64, riscv, s390) looks uneven from the notes gathered so far. A focused audit — or an attempt to extend Serval-style verification to an under-covered backend — is the JIT-side twin of topic A above, and considerably more contained.

### E. Do not attempt: path-pruning soundness
*Source: 1.16. Named to rule out, not pursue.*

Rutgers has a funded, multi-year head start on formalizing sound generalization for the verifier's state-pruning logic, and as of the last check (2026-08-07) it's still unsolved even by them. A semester capstone citing this as the confirmed open frontier is useful grounding; attempting to solve it independently is not a fight worth picking.

## Verified, author-acknowledged open problems (primary-source grounded)

Read directly from the full text of 1.8 (Veritas/SpecCheck, Lyu, Dwivedi, Bourgeat, Payer, Xu & Kashyap, SOSP '25 — fetched and read in full 2026-08-07, not just the abstract) rather than inferred from secondary summaries. These are gaps the paper's own authors name as unsolved, which is stronger footing for a "solve something new" capstone than a gap synthesized from reading between the lines of several sources.

### F. Loop verification via invariant inference, not bounded unrolling
> "SpecCheck verifies eBPF programs with loops using bounded loop unrolling, which is sufficient as a testing oracle... An alternative is automated loop invariant inference or generating eBPF programs based on pre-defined loop invariants. We leave them to future work." — §6.5

Bounded unrolling only approximates loop behavior for the depths tested; it can't soundly reason about `bpf_loop()` or iterator-based programs in general. Building actual loop-invariant inference for a verification oracle — even scoped to common `bpf_loop`/iterator patterns rather than full generality — solves something the paper explicitly leaves open, not something already claimed elsewhere. Highest ambition and highest risk of the three: invariant synthesis is a genuinely hard problem, and scoping it down to stay tractable in a semester is the real design challenge.

### G. Extend kernel helper/kfunc specification coverage (recommended starting point)
> "SpecCheck models all 171 op-codes... except for deprecated instructions... The remaining gap is in kernel helpers, where we deliberately focus on the 50 most frequently used of 455 functions (≈ 11%). As writing kernel function specifications becomes increasingly labor-intensive and the eBPF ecosystem expands, we are exploring scalable automation techniques to efficiently generate the specification for kernel functions with minimal manual effort." — §5

89% of kernel helpers/kfuncs are entirely unspecified in the only holistic specification-based oracle that exists. Extending coverage — manually, or by building the "scalable automation" the authors gesture at but don't build — is concrete, bounded, and reuses a working, open-source, peer-reviewed artifact (`github.com/rs3lab/veritas`) rather than requiring an oracle built from scratch. Deliverable is checkable: N new kernel functions specified, M new bugs found.

### H. A complete formal definition of availability (CIA triad)
> "We are also actively working on a formal definition for availability, the last component of CIA." — §3.3

Confidentiality (SP5) and integrity (SP2/SP4) are proven in the paper; availability (SP1, control-flow/termination safety) is defined informally only, and its soundness relative to the full CIA model is not yet established. Narrower and more theory-flavored than F or G — a good fit if the interest is in the proof side rather than systems-building.

### Bonus: three still-open bugs in the actual mainline verifier
Table 1 of 1.8 lists bugs #10, #11, and #13 as status **"Reported"** — not acked, not fixed, as of the paper's submission (Oct 2025): coarse-grained pointer comparison (#11), inconsistent scalar/pointer arithmetic constraints (#10), and inaccurate arithmetic-instruction tracking (#13). Worth re-checking against current mainline before building on them — they may have moved since — but if still open, a patch that fixes and upstreams one would be a small, unambiguous, verifiable "solved a real problem" deliverable to pair alongside F, G, or H rather than carry a capstone alone.

## Recommendation
**G (extend kfunc/helper specification coverage)** is the safer bet for something finishable and demonstrable in a semester: it reuses a working artifact, targets a gap the authors themselves quantified (89%), and produces a checkable result. **F (loop invariant inference)** is the more exciting, higher-risk alternative if the goal is maximum novelty and the risk of an open-ended research problem is acceptable. A strong shape: **G as the core deliverable, F as a stretch goal** on the same codebase.

(Superseding the earlier recommendation of A/B below, which was written before the primary-source pass on 1.8 — kept for context, not as the current pick.)

**A** is the strongest option among the earlier five if oracle comparison rather than solving a new problem is preferred — most original relative to existing literature, has real artifacts to build on, and produces a result (a bug-class-coverage matrix across oracles) that's citable regardless of outcome. **B** is the stronger alternative for live investigative work over an evaluation harness.

---
Compiled: 2026-08-07, updated 2026-08-07 with primary-source findings from 1.8 — not yet linked from [[eBPF Research - Index]] or folded into [[Potential Research Topics]]; do that once a direction is chosen.
