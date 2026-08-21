---
tags: [ebpf, research, motivation, verifier, impact]
parent: "[[eBPF Research - Index]]"
status: active
---
# Motivation — Why "Verifying the Verifier" Matters

← Back to [[eBPF Research - Index]]

A justification note for [[1 - Formal Verification of the Verifier and JIT]]: is verifier/JIT soundness an *actual* problem worth a capstone's attention, who does it impact, is that impact at real scale, and is it already being actively worked on (i.e. is there room left for a student project)? Synthesized from external CVE/impact evidence gathered 2026-08-21, cross-checked against the source list already in Section 1.

## Is this an actual problem?

Yes — it is a documented, recurring vulnerability class, not a theoretical concern.

- **A dedicated empirical study exists for exactly this question.** A CRiSIS 2025 paper analyzed 249 eBPF-related CVEs (2014–April 2025) — CWE tags, CVSS severity, kernel-version mapping, time-to-patch — to characterize where in the eBPF subsystem vulnerabilities cluster. `[PR]`
  → https://link.springer.com/chapter/10.1007/978-3-032-20732-6_14
- **A second, independent tally** counted 56 eBPF-related CVEs from 2010–2023 and found the majority originated in the verifier specifically — the component meant to be the *sole* safety guarantee for untrusted code reaching the kernel. **Primary source located 2026-08-21**: the figure is Figure 1 of *Unleashing Unprivileged eBPF Potential with Dynamic Sandboxing* — the earlier ⚠️ on this line is now resolved. `[TECH]` ⚠️ *arXiv preprint, not peer-reviewed.*
  → https://arxiv.org/pdf/2308.01983
- **Two further independent counts corroborate the shape, not the exact number**: *SafeBPF* reports 28 of 41 eBPF vulnerabilities (~68%) as memory-safety related (https://arxiv.org/pdf/2409.07508, `[TECH]`), and *SoK: Challenges and Paths Toward Memory Safety for eBPF* (IEEE S&P 2025) finds the **verifier is the most vulnerable component of the subsystem by CVE count** (http://www.nebelwelt.net/files/25Oakland.pdf, `[PR]`). ⚠️ The four studies (249-CVE, 56-CVE, 41-CVE, SoK) use different windows and inclusion criteria — cite them as *converging* evidence, and never sum or cross-compare their raw numbers.
- **Concrete exploit chain, not just an abstract gap**: CVE-2023-2163 — incorrect verifier state-pruning marked unsafe code paths as safe, yielding arbitrary kernel read/write and container escape. This is the exact failure mode that [[1 - Formal Verification of the Verifier and JIT#1.16 Narayana & Nagarakatte (Rutgers) — Verified Path Exploration for eBPF Static Analysis (eBPF Foundation grant) + LSFMM+BPF 2025 talk|1.16]] names as the field's own leading group's identified open frontier — the CVE record shows that gap is not hypothetical, it has already been exploited.
  → https://community.extremenetworks.com/t5/exos-switch-engine-announcements/sa-2025-092-ebpf-verifier-branch-pruning-linux-kernel-cve-2023/ba-p/120605
- **The pattern is structural, repeating across a decade**: CVE-2017-16995 (pointer mismanagement), CVE-2018-18445 / CVE-2019-7308 (bounds-checking), CVE-2021-3490 (ALU32 bounds), CVE-2021-4204 (missing bounds check in `bpf_ringbuf_submit`), CVE-2022-23222 (input validation), CVE-2023-2163, CVE-2023-39191 (dynamic pointer validation), continuing into CVE-2026-63864 and CVE-2026-31413. As eBPF gains capability, verifier complexity grows, a new bug class surfaces, gets patched, and the cycle repeats.
- **The eBPF Foundation's own commissioned audit confirms the concern from inside the ecosystem, not just from outside researchers.** NCC Group's independent review (already logged as [[1 - Formal Verification of the Verifier and JIT#1.11 NCC Group — eBPF Verifier Code Review, v1.0|1.11]]) found a path to arbitrary kernel memory read/write via `find_equal_scalars`, and explicitly recommended disabling unprivileged eBPF by default to shrink the attack surface. `[TECH]`
  → https://www.nccgroup.com/media/4lilthtf/ncc_group_nccgroup_e015561_report_2024-11-11_v10.pdf
- **The Linux Foundation / eBPF Foundation publicly framed this as a live risk**, pairing a formal threat model with the NCC audit release. `[PRIM]`
  → https://www.linuxfoundation.org/press/threat-model-and-independent-verifier-audit-examine-the-security-of-ebpf

**Nuance**: most of these CVEs require `CAP_BPF` or unprivileged-eBPF-loading enabled — this is not "any anonymous user, any kernel." But in multi-tenant containerized environments (the dominant eBPF deployment context), a large population of tenants legitimately hold BPF-adjacent privileges, so the *practical* exposure is broader than "requires root."

## Who does it impact?

1. **Cloud/hyperscale operators running eBPF at the kernel trust boundary in multi-tenant environments** — Meta, Cloudflare, Google, Microsoft, Alibaba, Apple, LinkedIn, Datadog, DoorDash, Walmart, Sky are all named production users (ebpf.io case studies). A verifier soundness bug here is a **tenant-isolation break**: the blast radius is every co-located workload sharing that kernel, not just the attacker's own process.
2. **Security/observability vendors whose product *is* the verifier's trust guarantee** — Cilium/Tetragon, Falco (CNCF's de facto runtime-security standard), Datadog's agent, Pixie. If the verifier can be fooled, the monitoring layer built on top of it inherits a false sense of assurance and becomes itself exploitable — directly relevant to [[4 - Adversarial Robustness of eBPF-Based Monitoring]].
3. **Kernel maintainers and downstream distro/vendor security teams**, who must triage and backport verifier CVEs across a wide LTS matrix — e.g. Extreme Networks' own advisory for CVE-2023-2163 shows propagation as far as network switch OSes, well outside "cloud."
4. **End users of any of the above, transitively** — anyone on a Kubernetes cluster, CDN, or SaaS platform whose provider relies on eBPF for networking or security is exposed if a verifier bug is weaponized, without ever touching eBPF themselves.

## Is that impact actually at scale?

- **Adoption is broad and mainstream**, not niche: Cilium alone lists 500+ contributing companies (Google, Cisco, Red Hat, Datadog, Cloudflare, SUSE among them) and is a CNCF-graduated CNI standard; Falco is the de facto CNCF runtime-security tool. eBPF underpins default networking/observability/security stacks in major managed Kubernetes offerings.
  → https://ebpf.io/case-studies/ · https://ebpf.io/applications/
- **The CVE severity profile is high**: the recurring outcome across the CVE list above is not information disclosure but arbitrary kernel memory read/write and full privilege escalation/container escape — the ceiling outcome for a kernel vulnerability class.
- **Industry consolidation signals it's treated as strategically important, not a fringe concern**: Cisco absorbed Isovalent (Cilium's primary commercial maintainer) into its Security Cloud platform — a large security vendor betting on eBPF as core infrastructure, which raises the stakes of any unresolved verifier-soundness question rather than lowering them.

Verdict: yes — the affected population is large (major cloud/security vendors and their downstream tenants), the failure mode is severe (kernel compromise), and the technology is trending toward more centrality, not less.

## Is it already being actively worked on? (room left for a capstone)

Yes, on multiple fronts simultaneously — summarized here, detailed in [[1 - Formal Verification of the Verifier and JIT]]:

- **Static formal verification of the verifier's own logic** — Agni ([[1 - Formal Verification of the Verifier and JIT#1.1 Vishwanathan, Shachnai, Narayana & Nagarakatte — Verifying the Verifier: eBPF Range Analysis Verification (Agni)|1.1]]) and tnum (1.3) prove range-analysis soundness; Rutgers holds an active eBPF Foundation grant (1.16, awarded Aug 2024) specifically targeting the path-pruning gap that CVE-2023-2163 exploited — **as of the most recent public update (May 2025 talk), this specific gap is confirmed still open**, no completion announcement found as of 2026-08-07.
- **JIT correctness** — Jitterbug/Serval (1.5–1.7) cover specific architectures, unevenly (arm64/riscv/s390 coverage is likely thinner than x86 — flagged as an open angle in [[1 - Formal Verification of the Verifier and JIT#Open questions / angles surfaced by this section|Section 1's open questions]]).
- **Dynamic/testing-based oracles as a complementary strategy** — State Embedding (1.13), specification-based fuzzing (1.8), CEGIS-synthesized unsound-operator corpora (1.17).
- **A structurally different fix, sidestepping verifier soundness entirely** — proof-carrying code via BCF (1.15), a live, contested Nov 2025 kernel RFC still under maintainer scrutiny as of the most recent reply found.
- **Independent, non-academic auditing** — NCC Group's 2024 review (1.11), commissioned by the eBPF Foundation itself.

**Where the room is**: no source in the corpus compares these five distinct bug-finding/soundness strategies (SMT proof of hand-written operators, an independent second verifier via differential testing, spec-oracle fuzzing, self-consistency dynamic testing, proof-carrying refinement) against a shared corpus to characterize which bug classes each catches and which it structurally cannot — flagged in Section 1 as the strongest concrete, buildable gap as of 2026-08-07. Combined with the still-open path-pruning soundness problem (1.16) and uneven JIT-architecture coverage, there is real, unclaimed territory here rather than a solved problem.

## What changed in 2026 — the ground is moving under this topic

This is the most decision-relevant material in this note, and none of it existed when Section 1 was compiled. At the **BPF track of LSFMM+BPF 2026**, the subsystem's own leadership began openly re-drawing the verifier's trust boundary. Three sessions matter:

- **Alexei Starovoitov — "BPF in the agentic era"** (LWN, 2026-06-03) `[PRIM]`. He splits the verifier into **two distinct jobs — catching programmer mistakes, and acting as a security boundary — and argues only the second genuinely requires the kernel**. The mistake-catching half can move to userspace (Rust, or the verifier run under user-mode Linux as a feedback-loop accelerator). The security half is non-negotiable: BPF programs must never crash the kernel, "that's what BPF stands for," and userspace cannot be trusted. Concrete proposals: make any program that clears the Rust compiler also clear the verifier; **add run-time checks where static verification falls short**; remove verifier limits wholesale (six-argument cap, indirect calls, stack size, call depth, the one-million-instruction ceiling, long jumps); and **"widening" for loops — generalize verifier state after a few iterations, trading static precision for inserted runtime checks**. The framing motivation is that LLM coding agents give up on BPF, because the write-test-fix loop is slow and the errors "are insane."
  → https://lwn.net/Articles/1075067/
  **Why this matters to you**: every item on that list *adds* capability and complexity to code sitting inside the kernel TCB, and widening explicitly *trades away* the static precision that Agni-style proofs are about. If this lands, the soundness question does not shrink — it **changes shape**, from "is the range analysis sound?" to "is the runtime-check insertion *complete*?" Nobody has asked the second question yet. It is being decided now, not in 2020.

- **Eduard Zingerman — loop verification with scalar evolution** (LWN, 2026-06-09) `[PRIM]`. To stop the verifier walking loops iteration-by-iteration into the instruction limit, this builds a **dominator tree** to find loop headers/back edges/latches, derives iteration bounds from the registers feeding the latch condition, and **symbolically executes loop bodies**, innermost-outward for nesting. Status: **prototype, "some improvements and some regressions"**; spilled/restored registers unsupported; John Fastabend already raised an **ordering problem when an inner loop mutates an outer loop's variables, and the fix is acknowledged but unimplemented**. An LWN commenter put the thesis of your entire topic in one line: it is "pretty brave" to put what amounts to an **optimizing compiler inside the trusted computing base**.
  → https://lwn.net/Articles/1076121/
  **Why this matters to you**: this is the single most valuable finding here. A brand-new, complex, safety-critical verifier analysis is *in prototype right now*, with an acknowledged unfixed soundness-shaped defect, and **zero formal-methods literature attached to it**. On a crowded frontier, being early beats being clever.

- **Kumar Kartikeya Dwivedi (Meta) — domain-specific invariants via Verus** (LWN, 2026-08-10 — eleven days before this note) `[PRIM]`. Argument: the verifier prevents kernel-invariant violations but does not keep the *system usable*. Evidence from production: **at Meta, the sched_ext watchdog ejects a scheduler once or twice a month**, and silent performance regressions are worse because they never fail outright. Proposal: a minimal Rust wrapper around a subsystem interface, annotated with Verus proofs checked at compile time, rustc emits BPF bytecode, and the in-kernel verifier still runs as normal. Reception **mixed** — Starovoitov was unconvinced two parallel static-analysis pipelines are operable; the session ended unresolved.
  → https://lwn.net/Articles/1087069/
  **Why this matters to you**: it shows the community's live argument has moved *past* "is the verifier sound" to "what must be layered on top of it." Verifier soundness is now treated as necessary-but-insufficient by the people who maintain it — which strengthens your motivation rather than weakening it, but changes how you should frame the contribution.

**Net read**: the "verifier is a static safety proof" model is being actively diluted — toward runtime checks, toward Rust-by-construction, toward out-of-kernel proof. A capstone that assumes the 2023-era framing will be arguing against a moving target. A capstone that *interrogates the new framing* is arguing at the front of the field.

## Verifier acceptance ≠ program correctness (a second motivation axis)

Section 1's framing is verifier *unsoundness* — bad programs accepted. Two 2026 preprints open an independent axis: programs the verifier correctly accepts can still be **security-relevant broken**, which matters if a pure soundness proof turns out to be too heavy for a capstone.

- **Heimdall** (arXiv 2605.25411, 2026-05-25) `[TECH]` ⚠️ *preprint, not peer-reviewed.* Catalogs **six classes of source-level defect that compile cleanly and pass the verifier** (initialization discipline, schema consistency, error handling). It reports **previously undisclosed information leaks in ten open-source eBPF programs**, where ring-buffer or stack-resident event records retain readable remnants of *earlier traced events* — including repeated kernel-text return addresses the authors say are **sufficient to recover the KASLR slide on every event**. Its remedy is migration rather than a stronger verifier: LLM-driven translation of libbpf C to Aya Rust with per-program equivalence established by symbolic execution + Z3, yielding 96 proven-equivalent translations of 102 programs (94.1%).
  → https://arxiv.org/abs/2605.25411
- **bpfix / "Characterizing and Bridging the Diagnostic Gap in eBPF Verifier Rejections"** (arXiv 2607.02748, 2026-07-02) `[TECH]` ⚠️ *preprint.* The complementary failure: rejection messages report *where verification stopped, not where the program lost the proof*. Empirically, over 235 reproduced rejections, **47% return only `EINVAL`**, and a single error string can map to **as many as nine distinct root causes**; 10 of 12 root causes are eBPF-specific. Ships a 12-root-cause taxonomy, the 235-rejection dataset, and a **75-task LLM repair benchmark**.
  → https://arxiv.org/abs/2607.02748 · tool: https://github.com/eunomia-bpf/bpfix

Both ship **datasets you can reuse rather than build** — the scarce resource for a one-semester project.

## What you can actually build on (artifacts, not just papers)

A literature review tells you what is known; this tells you what you can run. Every item below is code or data you could have on disk in week one.

- **Agni** — https://github.com/bpfverif/agni · MIT, 212 commits, Z3 + jsoncpp + libelf, pinned to **llvm-14/clang-14**. Pipeline: extract abstract semantics from verifier C → generate/check `gen` and `sro` verification conditions → synthesize an eBPF program witnessing any unsoundness. Covers 64/32-bit ALU + jump operators plus `BPF_SYNC` (which maps to `reg_bounds_sync`, the precision-propagation step, and must be encoded separately alongside whatever operator you verify). Domains: u64, s64, tnum, u32, s32. Also mirrored at Zenodo (doi 10.5281/zenodo.7931901).
  ⚠️ **Correction worth internalizing**: the widely-repeated figures showing Agni's solver time exploding (v6.4 taking weeks, v6.5–6.8 timing out) come from the **CAV-era** results. The repo's current README runs against `bpf-next` with `--kernver 6.10`, and there was a follow-up LPC 2024 talk, *"Agni: Fast Formal Verification of the Verifier's Range Analysis,"* explicitly about scaling. **Do not propose "make Agni scale" as your contribution without first checking `reproduce-results/` and the issue tracker (7 open issues / 5 open PRs as of 2026-08-21)** — that gap may already be closed.
- **ebpf-verifier-bugs** — https://github.com/bpfverif/ebpf-verifier-bugs · **67 synthesized proof-of-concept eBPF programs** for range-analysis bugs Agni found, plus a `bpf_test_tool` submodule that compiles, runs, and prints the truncated verifier log. This is a **ready-made ground-truth corpus** for the comparative oracle study flagged as Section 1's strongest gap. Cost: it assumes you build and boot a **v5.8-era kernel** (commit `bcf87687`) in a VM.
- **PREVAIL** — the eBPF-for-Windows verifier, a genuinely **independent second implementation**: abstract interpretation with a join operator and fixpoint computation over bounded loops, rather than Linux's path enumeration + pruning. It accepts programs Linux rejects (notably loops, and programs that blow Linux's one-million-state limit) and historically *missed* checks Linux performs (eBPF-to-eBPF calls, packet reallocation, 32-bit subregister tracking). Two verifiers, two different unsound directions — that asymmetry is the differential-testing opportunity.
  → https://pchaigno.github.io/ebpf/2023/09/06/prevail-understanding-the-windows-ebpf-verifier.html · https://archive.fosdem.org/2025/schedule/event/fosdem-2025-6453-performance-evaluation-of-the-linux-kernel-ebpf-verifier/ `[GREY]`
- **bpf_conformance (206 tests) + DiffSpec** (arXiv 2410.04249) `[TECH]` — existing cross-runtime harness spanning Linux/libbpf, userspace uBPF, and eBPF-for-Windows via bpf2c, using the IETF-standardized BPF ISA as the spec. **Crucial limitation for you**: it tests *runtime/ISA execution* semantics; **verifier accept/reject divergence has only been compared qualitatively**. ⚠️ *Search-derived; read the harness directly before building a proposal on it.*
- **BpfChecker** (CCS 2024) `[PR]` — differential testing across userspace runtimes (Windows eBPF as PREVAIL + uBPF compiled userspace), capturing state in both JIT and interpreter modes; the authors themselves note design disparities limit generalization to the kernel runtime.
  → https://yajin.org/papers/CCS2024_BpfChecker.pdf

**Budget honestly**: nearly all of the above needs a kernel build + VM loop (qemu / virtme-ng), a pinned LLVM, and Z3. That is your first fortnight and it produces no novelty — plan for it rather than discovering it in week six. See [[Running and Testing eBPF - Practical Approaches]].

## Funding, venues, and the people — the practical surface

- **eBPF Foundation Academic Research Grant** — https://ebpf.foundation/funding-opportunities/research-fund/ (fetched 2026-08-21). Up to **$50,000**, paid as an unrestricted gift to the host university. **You cannot apply as a student — the applicant must be full- or part-time faculty and serve as PI** — but the program page states funds "ideally cover graduate or post-graduate students' employment/tuition and other research costs." **"Formal verification of the verifier and JITs" is named explicitly in the topics of interest**, alongside verifier scalability/maintainability, safely enabling unprivileged eBPF, and in-kernel JIT improvements. Deliverables include two blog posts; proposals are 2 pages + 1-page budget + CVs, and are **not treated as confidential**. The **2026 cycle is closed; recipients are announced 2026-09-01**. Prior cycles: 2024 = five × $50k from 25 proposals across 20 universities (one of which is the Rutgers path-pruning grant, [[1 - Formal Verification of the Verifier and JIT#1.16 Narayana & Nagarakatte (Rutgers) — Verified Path Exploration for eBPF Static Analysis (eBPF Foundation grant) + LSFMM+BPF 2025 talk|1.16]]); 2025 = two × $50k from 27 proposals across 23 institutions — **Michigan's EPASS ("verifier-cooperative instrumentation," runtime enforcement *extending* static verification, Ryan Huang)** and UC Riverside's eBPF power governors.
  **Action**: if a supervising professor is willing to be PI, this topic is pre-endorsed by the funder. Watch 2026-09-01 for who won, then check whether it collides with your angle.
- **eBPF '26 — 4th Workshop on eBPF and Kernel Extensions** · https://ebpf.github.io/2026/cfp.html (fetched 2026-08-21). **The workshop moved from SIGCOMM to ACM SOSP '26** — Prague, **2026-09-29**. Two tracks: **6-page research papers** and **2-page poster abstracts explicitly for "early-stage findings, position papers, and works that are still in progress."** Double-blind; accepted work goes to the ACM DL; unlimited pages for references. **The 2026 deadline (2026-06-26 AoE) has passed** — so **eBPF '27's poster track is your realistic on-ramp**, and it is a genuinely low-barrier one: a capstone-scale result is exactly what that track exists for. Solicited topics include "improved methods for verifying eBPF program safety" and "security consequences of kernel extensions."
- **Linux Plumbers Conference 2026 eBPF Track** · Prague, **2026-10-05 to 10-07** — a week after eBPF '26, in the same city, so one trip covers both. 30-minute talks; technical committee is Starovoitov, Borkmann, Nakryiko, and Martin Lau. This is where verifier design is *actually argued*, and where 1.16-style work gets its first public airing.
- **The people currently holding this ground** (see also [[Author Pages to Watch]]): **Rutgers** — Nagarakatte, Narayana, Harishankar Vishwanathan, Matan Shachnai; note their **March 2026 tristate-stepping algorithm upstreamed into the mainline `bpf` tree**, a **SAS 2025** paper comparing the precision of the verifier's abstract operators, and a June 2025 upstreamed precision patch. **Meta** — Starovoitov, Zingerman, Dwivedi. **Michigan** — Ryan Huang (EPASS). **eunomia-bpf / Dan Williams / Yusheng Zheng** — bpfix and the diagnostic-gap dataset.

## Honest counter-arguments before you commit

1. **The frontier is crowded and has an incumbent.** Rutgers holds funding, a multi-year head start, upstream commit access, and the flagship tool. Do **not** propose "verify the range analysis." Propose something adjacent to what they are *not* doing.
2. **The ecosystem's own direction partly undercuts "prove the verifier sound."** Leadership's stated plan is more runtime checks, Rust-by-construction, and possibly moving mistake-catching out of the kernel entirely. Frame the work so it **survives that shift** — asking whether the *runtime-check* story is complete is a better bet than proving a static analysis that may be widened away.
3. **Reproduction cost is the most common way this class of project fails.** Z3 solve times, kernel builds, VM harnesses, LLVM version pinning. A capstone that spends ten weeks getting someone else's artifact to run has produced nothing.
4. **The privilege caveat is real and reviewers will raise it** — keep the `CAP_BPF` nuance above stated plainly rather than buried.
5. **⚠️ Provenance**: items sourced from search summaries rather than direct fetch are flagged inline (DiffSpec/bpf_conformance specifics, the 2024/2025 grant proposal counts, the LPC 2024 Agni scaling talk, Rutgers' upstreaming dates). Verify each before it enters a written proposal.

## Concretely, what a capstone-shaped question looks like *now*

Three openings, ordered by how unclaimed they are as of 2026-08-21:

1. **Soundness of the new loop machinery — scalar evolution + widening — while it is still a prototype.** Earliest, least crowded, directly upstream-relevant, and there is already a named unfixed ordering issue (inner loop mutating outer loop variables) to anchor against. Risk: a moving target, and you must track `bpf-next`.
2. **Systematic accept/reject differential testing of the Linux verifier against PREVAIL.** Two independent implementations with *known, opposite* incompleteness profiles; the harness scaffolding largely exists (bpf_conformance, DiffSpec); nobody has done the **verifier-outcome** study systematically, only runtime-semantics comparisons.
3. **The comparative oracle study already flagged in [[1 - Formal Verification of the Verifier and JIT#Open questions / angles surfaced by this section|Section 1's open questions]]** — which bug classes each of the five strategies catches and which it structurally cannot — now materially easier than when it was written, because a shared corpus exists that did not before: **67 Agni PoCs + 235 reproduced rejections + a 75-task repair benchmark**.

## To triage into [[1 - Formal Verification of the Verifier and JIT]]

New sources introduced here that belong in the numbered list (append at the next free ID, per vault convention — do not renumber):
Starovoitov LSFMM+BPF 2026 "BPF in the agentic era" `[PRIM]` · Zingerman scalar-evolution loop verification `[PRIM]` · Dwivedi Verus domain-invariants `[PRIM]` · Heimdall `[TECH]` · bpfix / diagnostic gap `[TECH]` · SoK: Memory Safety for eBPF (IEEE S&P 2025) `[PR]` · SafeBPF `[TECH]` · Dynamic Sandboxing (the 56-CVE primary) `[TECH]` · DiffSpec `[TECH]` · BpfChecker (CCS 2024) `[PR]` · FOSDEM 2025 verifier performance comparison `[GREY]`.

## Related
- [[1 - Formal Verification of the Verifier and JIT]] — full source list this note draws on
- [[Potential Research Topics#1. Coverage audit of formal-methods gaps in the verifier/JIT]] — the capstone angle this motivation note supports
- [[Running and Testing eBPF - Practical Approaches]] — the build/VM cost of every artifact listed above
- [[Author Pages to Watch]] — the groups named in §"Funding, venues, and the people"
- [[eBPF Research - Index]]

---
Compiled: 2026-08-21 · Substantially expanded 2026-08-21 with the LSFMM+BPF 2026 landscape, buildable artifacts, funding/venue surface, and counter-arguments.
