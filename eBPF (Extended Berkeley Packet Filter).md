---
tags: [ebpf, capstone, home]
status: active
---
# eBPF (Extended Berkeley Packet Filter) - Capstone Project

## Project Overview
**Working Title**: [Insert specific project title here]  
**Project Lead**: Rohan Kalia  
**Start Date**: [Date]  
**Expected Completion**: [Date]

---

## 1. Problem Statement & Motivation

### What Problem Are We Solving?
[Describe the real-world problem or limitation that your eBPF project addresses. Why does this matter?]

### Why eBPF?
[Explain why eBPF is the right approach for this problem. What advantages does it offer?]

---

## 2. Project Objectives

### Primary Goals
- [x] Goal 1: [Be specific and measurable]
- [ ] Goal 2: [Be specific and measurable]
- [ ] Goal 3: [Be specific and measurable]

### Success Criteria
[Define how you'll measure success. What does "done" look like?]

---

## 3. Scope & Constraints

### What's In Scope
- [Feature/component 1]
- [Feature/component 2]
- [Feature/component 3]

### What's Out of Scope
- [Explicitly list what you're NOT doing]

### Technical Constraints
- [Performance requirements, kernel version compatibility, hardware requirements, etc.]

---

## 4. Key Concepts & Background

### eBPF Fundamentals
- **What it is**: an in-kernel virtual machine that runs sandboxed bytecode inside the Linux kernel without changing kernel source or loading a kernel module. Programs are checked by a static **verifier** (range analysis, pointer tracking, loop bounds) and then **JIT-compiled** to native code for near-native performance.
- **Safety model in flux**: the traditional safety story rests almost entirely on the verifier. Recent work (2.1, *Kernel Extension Verification is Untenable*) argues this is increasingly unsustainable as the verifier's surface grows, motivating two responses: **hardware-assisted isolation** (Intel MPK, ARM PAC/MTE, software fault isolation) and **language-based safety** (Rex — compiler-enforced guarantees instead of/alongside verification).
- **Trust doesn't stop at the verifier**: the bytecode reaching the kernel has to be the bytecode you intended to load. This is the **supply-chain** problem — signing, attestation, and gating (BPF token/FS delegation, the signed-BPF work landing in kernel 6.18, CO-RE relocation invalidating signatures) — currently being fought out on kernel mailing lists more than in academia.
- **Use cases**: networking/packet processing (XDP, tc), observability and tracing (kprobes, tracepoints, perf/ring buffers), and security monitoring (Falco, Tetragon, Tracee — provenance-style runtime intrusion detection).
- **Deployed eBPF is itself an attack surface**: once used defensively, eBPF-based monitors face documented evasion techniques — TOCTOU/semantic confusion, hook/policy bypass, and telemetry-path blinding — with at least one real-world incident (the LinkPro eBPF rootkit).
- Full source-level detail on all of the above lives in [[eBPF Research - Index]]; see especially [[1 - Formal Verification of the Verifier and JIT]] (verifier/JIT trust), [[2 - Hardware-Assisted Isolation for eBPF]] (isolation responses), [[3 - Secure eBPF Program Supply Chain]] (signing/attestation), and [[4 - Adversarial Robustness of eBPF-Based Monitoring]] (evasion of deployed monitors).

### Related Technologies
- **Toolchain**: libbpf (user-space loading/skeletons), BCC (BPF Compiler Collection), bpftool, LLVM/Clang (BPF backend), CO-RE
- **Hooks & attach points**: XDP, tc, kprobes/kretprobes, tracepoints, LSM hooks, perf/ring buffers
- **Deployed security tooling**: Falco, Tetragon, Tracee
- **Hardware isolation primitives**: Intel MPK, ARM PAC, ARM MTE, software fault isolation (SFI)
- **Kernel trust mechanisms**: BPF token, BPF FS-based delegation, Linux Security Modules (LSM), Hornet LSM (in-flight)

---

## 5. Architecture & Design Approach

### High-Level Architecture
[Describe the overall structure:
- eBPF kernel program(s)
- User-space components
- Data structures and communication mechanisms]

### Technical Stack
- **Language(s)**: [e.g., C, Rust, Go, etc.]
- **Tools**: [e.g., libbpf, LLVM, Clang]
- **Kernel Version**: [Minimum supported kernel]

---

## 6. Project Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Research & Planning | [Weeks] | Design document, proof-of-concept |
| Development Phase 1 | [Weeks] | [First milestone] |
| Development Phase 2 | [Weeks] | [Second milestone] |
| Testing & Optimization | [Weeks] | Performance benchmarks, test suite |
| Documentation & Demo | [Weeks] | Final documentation, demonstration |

---

## 7. Deliverables

- [ ] Source code (well-commented and version-controlled)
- [ ] Technical documentation
- [ ] Performance benchmarks/results
- [ ] Test suite with coverage metrics
- [ ] Demonstration/proof-of-concept
- [ ] Final report

---

## 8. References & Resources

### Recommended Reading
- [[eBPF Fundamentals]] — baseline primer: what eBPF is, the load→verify→JIT→attach lifecycle, the VM and instruction set, the verifier, maps, helpers/kfuncs, CO-RE, hooks, limitations, toolchain, and a glossary. Start here if any term below is unfamiliar.
- [[Running and Testing eBPF - Practical Approaches]] — the *practical* counterpart to the research: how to get a Linux kernel running on this machine, five approach options (bpftrace/BCC, libbpf+CO-RE, Rust-Aya/Go, userspace bpftime, cloud/bare metal) with the languages each requires, a benchmarking-and-testing methodology, and a feasibility map of which [[Potential Research Topics]] angles this hardware can and cannot support.
- [[eBPF Research - Index]] — full research hub: formal verification, hardware isolation, program supply chain, and adversarial robustness of eBPF monitoring, with ~44 sources and candidate project angles in [[Potential Research Topics]]
  - [[1 - Formal Verification of the Verifier and JIT]] — can the verifier/JIT be proven correct? (Agni, tnum, Jitterbug, Serval)
  - [[2 - Hardware-Assisted Isolation for eBPF]] — hardware-backed defense-in-depth (MPK, PAC, MTE, SFI) and the language-based alternative (Rex)
  - [[3 - Secure eBPF Program Supply Chain]] — signing, attestation, and gating bytecode before it reaches the kernel
  - [[4 - Adversarial Robustness of eBPF-Based Monitoring]] — how robust is eBPF-based monitoring (Falco, Tetragon, Tracee) against an adversary who knows it's there?
  - [[Author Pages to Watch]] — researcher pages worth bookmarking for new work not yet indexed

### Tools & Libraries
- **libbpf** — user-space library for loading/managing BPF programs (skeletons, CO-RE)
- **BCC** (BPF Compiler Collection) — Python/Lua front-end for writing BPF tools
- **bpftool** — kernel-shipped introspection/debugging CLI for BPF objects
- **LLVM/Clang** — compiles restricted-C to BPF bytecode
- **Falco / Tetragon / Tracee** — eBPF-based runtime security monitoring and enforcement (relevant if the project angle touches Section 4)

### Learning Resources
- [ebpf.io](https://ebpf.io) — eBPF Foundation's community hub and docs
- *The eBPF Runtime in the Linux Kernel* (survey, arXiv 2410.00026 — see 1.12 in [[1 - Formal Verification of the Verifier and JIT]]) — good technical primer/orientation before diving into any subtopic
- LWN.net's BPF coverage — historical and in-flight kernel engineering context (signed BPF, BPF token, unprivileged `bpf()` policy history — see [[3 - Secure eBPF Program Supply Chain]])
- Linux Plumbers Conference eBPF track — primary-source talks on in-flight kernel work

---

## 9. Risks & Mitigation

| Risk | Likelihood | Impact | Mitigation Strategy |
|------|------------|--------|---------------------|
| [Risk 1] | | | |
| [Risk 2] | | | |

---

## 10. Notes & Progress

- **2026-07-24**: Research phase compiled — [[eBPF Research - Index]] now holds ~44 sources across four thematic areas (verifier/JIT formal verification, hardware-assisted isolation, program supply chain, adversarial robustness of eBPF monitoring), with ten candidate project angles synthesized in [[Potential Research Topics]]. §4 and §8 above have been filled in from this research. **Next step**: choose a direction from [[Potential Research Topics]] (or propose a new one) so §§1–3, 5–7, 9 can be written against it.
- **2026-07-25**: Added [[eBPF Fundamentals]] — a standalone, deliberately *descriptive* baseline primer on the technology itself (lifecycle, VM/ISA, verifier, maps, helpers/kfuncs, CO-RE, hook taxonomy, privileges, limitations, toolchain, glossary), linked from §8 above. It covers the mechanism; the research notes evaluate it. §4 of this note remains the project's own condensed framing and was left as-is.
- **2026-08-02**: Added [[Running and Testing eBPF - Practical Approaches]] — the implementation-side companion to the literature review: five approach options with per-option language requirements, a four-layer testing/benchmarking methodology (verifier-log metrics, event-loss reporting, null baselines, median+IQR under VM noise), a target-kernel recommendation, and a feasibility map from each of the ten [[Potential Research Topics]] to what this hardware can actually run.
  - **Material scoping finding**: macOS does not expose PKU/OSPKE to guests on the development machine (Intel i5-8259U, x86_64, 8 GB), so **Intel MPK isolation is not practically usable locally**, and being x86 rules out ARM PAC/MTE. ⚠️ This is a claim about what the *host OS exposes*, not a verified claim that the silicon lacks protection keys — see that note's §1.1. Topics **#2, #3, and #9** — the hardware-isolation cluster, including the "clearest unaddressed intersection" flagged in [[eBPF Research - Index]] — therefore require rented bare metal, not just a local VM. Topics **#1, #5, #6, #7, #8, #10** are locally feasible.
  - **Recommended kernel baseline**: **6.18 LTS** (6.18.41 as of this date), since that is where the signed-BPF work behind [[3 - Secure eBPF Program Supply Chain]] (3.1, 3.8) landed and it is now a longterm release.
  - §5 (Technical Stack) above is still an unfilled placeholder — it should be completed from that note's §3–§4 once a direction is chosen.
- **2026-08-07**: Refreshed [[1 - Formal Verification of the Verifier and JIT]] and [[2 - Hardware-Assisted Isolation for eBPF]] with six sources not previously in the corpus (1.13–1.18, 2.11–2.12), and closed three of [[0 - Seminal Papers - Foundational Reading]]'s four outstanding action items. Notably: a **live, unresolved Nov-2025 kernel RFC** (1.15, "BCF") proposing proof-carrying code for eBPF — the modern echo of 0.3 (Necula & Lee, 1996) the vault predicted — and primary-source confirmation (1.16) that **path-pruning soundness is the field's own leading group's named open gap**, not yet solved as of the most recent update found. Direction under discussion: a **coverage audit reframed as an oracle-comparison artifact** — Section 1 now names five distinct verifier bug-finding strategies (SMT proof, differential testing against PREVAIL, spec-oracle fuzzing, self-consistency/state-embedding, proof-carrying refinement) that no source compares against a shared corpus. This satisfies the §7 deliverables (buildable artifact, not just a survey), is locally feasible per [[Running and Testing eBPF - Practical Approaches]] §6 (Topic #1 row), and the literature review's other three sections become motivation/related-work rather than dead weight — verifier trust is the assumption all three others sit on. **Not yet decided**: awaiting sign-off before filling §§1–3 against this direction.

---

**Last Updated**: 2026-08-07