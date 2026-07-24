---
tags: [ebpf, research, index, moc]
related: "[[eBPF (Extended Berkeley Packet Filter)]]"
---
# eBPF Security & Verification Research — Index

Hub note for the research gathered around the [[eBPF (Extended Berkeley Packet Filter)]] capstone project. It organizes ~44 sources across four thematic areas of eBPF safety research — what happens *before* a program is trusted (verification, isolation, supply chain) and what happens *after* it's deployed defensively (adversarial robustness of eBPF-based monitoring) — plus a synthesis of candidate project angles the sources themselves don't cover yet.

## How this folder is organized
- Each numbered topic note below reorganizes the source list for that theme (retrieval order, not citation order), preserving quality tags and every URL variant that was found (open PDF, landing page, mirror, code, talk).
- [[Potential Research Topics]] translates gaps, tensions, and unaddressed intersections between sections into ten candidate capstone angles.
- [[Author Pages to Watch]] lists researcher pages worth bookmarking for new work before it's indexed by search engines.

## Quality tag legend
- `[PR]` — peer-reviewed
- `[PRIM]` — primary kernel source (patch, LWN, LPC)
- `[TECH]` — tech report / commissioned review
- `[GREY]` — blog, conference talk, or unreviewed code
- ⚠️ flags — sourcing caveats called out explicitly in the original research pass (unconfirmed titles, secondhand citations, weak evaluations, scope exclusions)

## Topics

### [[1 - Formal Verification of the Verifier and JIT]]
Can the verifier and JIT be proven correct? Agni, tnum, Jitterbug, Serval, plus two independent reviews of how far formal coverage reaches. 12 sources.

### [[2 - Hardware-Assisted Isolation for eBPF]]
If verification can't be fully trusted, what does hardware-backed defense-in-depth look like — MPK, PAC, MTE, SFI — and what's the language-based alternative? 10 sources.

### [[3 - Secure eBPF Program Supply Chain]]
How do you sign, attest, and gate the bytecode reaching the kernel in the first place? Almost entirely primary/in-flight kernel engineering sources. 9 sources.

### [[4 - Adversarial Robustness of eBPF-Based Monitoring]]
Once eBPF is used defensively (Falco, Tetragon, Tracee), how robust is it against an adversary who knows it's there? Split into eBPF-specific attacks (4A) and adjacent provenance-IDS evasion literature (4B). 13 sources.

## Reading spine
If you want the thesis before the breadth, read in this order (~90 pages total):

1. **1.12** — survey, for orientation
2. **2.1** — the argument (verification is untenable)
3. **1.1** — Agni, the state of the art in verification
4. **2.2** or **2.3** — the hardware-isolation response

## Cross-cutting gaps worth noting
- No source connects Section 2 (hardware isolation) to Section 4 (monitor evasion) — does MPK/PAC/MTE-based isolation actually prevent the telemetry-blinding attacks documented in 4.3/4.4? See [[Potential Research Topics#9. Does hardware isolation prevent telemetry-blinding attacks?]].
- Section 3 (supply chain) is almost entirely primary sources, not peer-reviewed literature — the problem is still being fought out on kernel mailing lists (see 3.2), not settled in academia.
- Several sources across all four sections carry explicit sourcing caveats (unconfirmed titles, secondhand citations, weak evaluations) — check the ⚠️ flags in each topic note before citing.

## Related
- [[eBPF (Extended Berkeley Packet Filter)]] — capstone project home note (§8 References & Resources should point here once a direction is chosen)

---
Compiled: 2026-07-24
