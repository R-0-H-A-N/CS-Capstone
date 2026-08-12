---
tags: [ebpf, methodology, tooling, environment, benchmarking, capstone]
related: "[[eBPF (Extended Berkeley Packet Filter)]]"
status: done
---
# Running and Testing eBPF — Practical Approaches

← Back to [[eBPF (Extended Berkeley Packet Filter)]]

The research notes in [[eBPF Research - Index]] establish *what* is worth investigating. [[eBPF Fundamentals]] establishes *how eBPF works*. This note covers the third thing neither of them does: **how to actually get eBPF running on the hardware in front of you, in what language, and how to measure it in a way a capstone examiner will accept.**

It is deliberately opinionated about trade-offs and deliberately explicit about which options your current machine forecloses.

---

## 0. The one constraint everything else follows from

**eBPF is a Linux kernel subsystem.** There is no eBPF on macOS. Every approach below is therefore, first and foremost, an answer to *"how do I get a Linux kernel I control?"* — with one genuine exception (Option D, which runs eBPF bytecode entirely in userspace and needs no kernel at all).

A second consequence: **Apple's `clang` ships no BPF backend.** The `clang` at `/usr/bin/clang` (Apple clang 17.0.0) cannot emit BPF bytecode. You need a real upstream LLVM/Clang with the `bpf` target — which is simplest to obtain *inside* the Linux guest, where it is a package-manager install.

---

## 1. Your machine — verified constraints

Probed 2026-08-02. These are not estimates; they are what the hardware reports.

| Property | Value | Consequence |
|---|---|---|
| CPU | Intel Core i5-8259U @ 2.30 GHz (Coffee Lake) | x86_64, **not** Apple Silicon |
| Cores | 4 physical / 8 logical | Kernel builds are slow but viable |
| RAM | **8 GB** | The binding constraint. See below. |
| Disk free | 66 GB of 233 GB | Enough for one VM + one kernel tree, not three |
| OS | macOS 15.7.7 (24G720) | Supports Virtualization.framework |
| CPU security features reported by macOS | SMEP, SMAP | — |
| **PKU / OSPKE** | not reported by macOS ⚠️ *see below — this is weak evidence* | **Intel MPK is not practically usable here** (§1.1) |
| Virtualization tooling installed | *none* (no lima, colima, qemu, multipass, vagrant, VirtualBox, podman) | Every VM route starts with an install |
| Docker | CLI present at `/usr/local/bin/docker`; **daemon not running** | ⚠️ LinuxKit VM kernel version/BTF/BPF-LSM config unverified |
| Toolchain present | Apple clang 17.0.0, python3 3.13.5, gh 2.96.0, GNU Make 3.81 | — |
| Toolchain absent | `rustc`, `cargo`, `go`, `cmake`, upstream `llvm`/`llvm-strip` | — |

### What this rules out locally

Two of the most attractive directions in [[Potential Research Topics]] cannot be reproduced on this laptop:

- **Intel MPK (`pkey_mprotect`, PKRU)** — the isolation primitive used by MOAT and the CCS-25 work (sources 2.2, 2.7). macOS does not report `PKU`/`OSPKE` on this machine, and a hypervisor cannot expose to a guest a feature the host OS has not enabled. **You cannot run MPK-based isolation here**, at native speed or otherwise. QEMU *software* emulation (TCG) can emulate PKU, but at an interpretation penalty that makes any performance number meaningless — fine for correctness, useless for the "overhead vs. threat coverage" evaluation axis those topics hinge on. See §1.1 for exactly how strong this claim is.
- **ARM PAC / MTE** — used by HIVE and SafeBPF (2.3, 2.5). You are on x86. Same story: emulation only.

This is a real scoping constraint, not a detail. Topics **#2**, **#3**, and **#9** all require either cloud/rented hardware with the right CPU, or an explicit reframing of the evaluation to be qualitative/emulated. Decide that consciously rather than discovering it in week nine.

### 1.1 — What the PKU probe actually proves ⚠️

The PKU row above deserves a caveat rather than a flat "your CPU lacks it", because the two are not the same claim.

- **What was measured**: macOS `sysctl` CPU-feature output. That reports the features the *operating system* recognises and exposes — not the full capability of the silicon. macOS makes no use of protection keys, so their absence from `sysctl` is expected regardless of what the die can do.
- **What is therefore *not* established**: whether the i5-8259U's silicon implements PKU. Protection keys shipped broadly on Intel *server* parts (Skylake-SP onward) and only patchily on contemporaneous client/mobile parts; this probe cannot settle which side of that line an eighth-generation U-series part falls on, and no attempt was made to settle it from Intel's own documentation.
- **What *is* established, and is the part that matters**: a guest kernel can only use a CPU feature the host exposes to it. macOS does not enable OSPKE, so `pkey_mprotect()`/PKRU are unavailable to any Linux VM running on this machine under Virtualization.framework or HVF-accelerated QEMU — irrespective of the underlying silicon.

The practical conclusion (no local MPK work at meaningful fidelity) survives intact; only the *reason* changes, from "the hardware lacks it" to "the host OS does not expose it". If you ever need the stronger claim — for a proposal's constraints section, say — verify it against Intel's ARK/SDM entry for the i5-8259U rather than against this probe.

### The 8 GB problem

8 GB is workable but tight. A Linux VM wants 4 GB to be pleasant; a full kernel build wants more, and `-j8` on 4 GB of guest RAM will thrash. Mitigations, in order of preference:

1. **Don't build the kernel.** Most of the work below runs against a *distribution* kernel. Only verifier/JIT modification (Topic #1) genuinely requires building one.
2. Build with `-j4`, not `-j8`, and use `make localmodconfig` to cut the config down from ~5,000 modules to a few hundred.
3. Build in the cloud (Option C), copy the artifact down.

---

## 2. First: what does "test an algorithm with eBPF" mean?

The phrase covers three genuinely different projects with three different toolchains. Pick deliberately — this choice drives everything downstream.

### Meaning A — eBPF as the *subject*: implement an algorithm as an eBPF program
You write the algorithm in restricted C (or Rust/Aya), load it, and measure how it performs *inside the kernel*. Example: a packet-classification or rate-limiting algorithm in XDP, benchmarked at line rate.
**Testing question**: does it verify, does it produce correct output, what is its per-packet/per-event cost?
**Hard part**: the verifier. Loops, unbounded pointer arithmetic, and the instruction limit will reject the naïve version of most textbook algorithms. Making an algorithm *verifiable* is itself a research-grade problem, and a legitimate capstone contribution.

### Meaning B — eBPF as the *instrument*: measure algorithms running elsewhere
You use eBPF to observe some other program (a database, a scheduler, a userspace algorithm) via kprobes/uprobes/tracepoints, and analyse the resulting event stream.
**Testing question**: is your measurement accurate, and what does the instrumentation itself cost?
**Hard part**: observer effect. Your probe overhead has to be small relative to what you are measuring, and you must prove it is.

### Meaning C — eBPF's *machinery* as the subject (this is your capstone)
You are testing the verifier, the JIT, the isolation mechanism, the signing path, or the robustness of an eBPF-based monitor. The "algorithm" under test is the kernel's, not yours.
**Testing question**: is the verifier sound/complete on input class X? Does isolation scheme Y stop attack Z? Does monitor W miss evasion V?
**Hard part**: you need adversarial inputs and a ground truth, not just a benchmark harness.

Every angle in [[Potential Research Topics]] is Meaning C. **But you should still do a Meaning-A or Meaning-B exercise first** — you cannot meaningfully critique the verifier until you have been rejected by it a dozen times.

---

## 3. The approach options

Five routes, ordered by setup cost. They are not mutually exclusive; the recommended sequence in §7 uses three of them.

---

### Option A — bpftrace / BCC in a Linux VM *(lowest cost, fastest first result)*

**What it is.** High-level front-ends that hide the compile-load-attach cycle. `bpftrace` is an awk-like one-liner language; BCC is a Python library that embeds BPF C as a string and compiles it at runtime via LLVM.

**Languages required**
- **bpftrace DSL** — awk-like, learnable in an afternoon. `kprobe:vfs_read { @[comm] = count(); }` is a complete program.
- **Python 3** — for BCC. You already have 3.13.5 on the host; the VM will have its own.
- A *reading* knowledge of restricted C — BCC programs embed C, you just don't have to build it yourself.

**Setup**: install Lima or multipass, `apt install bpftrace bpfcc-tools linux-headers-$(uname -r)`. Under an hour.

**Good for**: getting your hands on real event streams immediately; Meaning B work; the empirical side of Topics **#7**, **#8**, **#10** (you can script attacker behaviour and watch what a monitor does or doesn't see).

**Bad for**: anything needing a stable, distributable artifact. BCC compiles at runtime, needs kernel headers on the target, and is being progressively displaced by CO-RE + libbpf. Do not build a capstone deliverable on BCC alone.

**Caveat**: bpftrace's convenience hides exactly the mechanics (map lifecycle, verifier interaction, relocation) that your research topics are *about*. It is a scouting tool, not a destination.

---

### Option B — libbpf + CO-RE in a Linux VM *(the standard, and the reference answer)*

**What it is.** The canonical modern workflow, and what essentially every source in [[eBPF Research - Index]] assumes. You write the kernel-side program in restricted C, compile it to BPF bytecode with upstream Clang, generate a *skeleton* header with `bpftool gen skeleton`, and write a userspace loader in C that opens/loads/attaches it. CO-RE + BTF make the resulting binary portable across kernel versions without recompilation.

**Languages required**
- **Restricted C** — the kernel side. This is the non-negotiable one. It is C with the verifier as a very opinionated code reviewer: bounded loops only, every pointer dereference must be provably in-range, no arbitrary function calls, a hard instruction limit. If you learn one language for this project, learn this.
- **C (ordinary)** — the userspace loader. Skeletons make this mostly boilerplate.
- **Make / a little shell** — the build is a multi-stage pipeline (`clang -target bpf` → `llvm-strip` → `bpftool gen skeleton` → `cc`).

**Setup**: VM, then `apt install clang llvm libbpf-dev libelf-dev zlib1g-dev bpftool linux-tools-$(uname -r)`. Start from the `libbpf-bootstrap` template rather than assembling the pipeline by hand. ⚠️ *link not verified this pass — github.com/libbpf/libbpf-bootstrap*

**Good for**: everything. Topics **#1**, **#5**, **#6**, **#7** all live here. It is also the only option where the mechanics you are researching are fully exposed to you.

**Bad for**: developer comfort. The error messages are the verifier's, which is to say cryptic. Budget real time for this.

---

### Option C — Rust + Aya, or Go + cilium/ebpf *(better ergonomics, one language instead of two)*

**Rust / Aya.** A pure-Rust eBPF library that deliberately depends on neither libbpf nor BCC — it issues `bpf()` syscalls itself using only the `libc` crate. You write *both* the kernel side and the userspace side in Rust. BTF support is transparent; function-call relocation and global-data maps are supported; async loaders work with tokio or async-std. No C toolchain and no kernel headers are needed to build, and the project claims release builds complete "in a matter of seconds." Combined with BTF and musl you get a self-contained compile-once-run-everywhere binary.
- **Version**: `aya` **0.14.0**, published 2026-06-24, MSRV **1.87.0**, Rust edition 2024, MIT OR Apache-2.0. (crates.io API, verified 2026-08-02.)
- ⚠️ **Pre-1.0.** Fourteen published versions, ~3.0M lifetime downloads, but the API is not frozen. The repo describes itself as supporting "a large chunk of the eBPF API" — which is an honest way of saying *not all of it*. For a capstone with a fixed deadline, pin your version and expect to hit a gap eventually. (github.com/aya-rs/aya, verified 2026-08-02: 2,490 commits, 4.7k stars, 132 open issues.)

**Go / cilium-ebpf.** The other mainstream binding; what Cilium, Tetragon, and much of the cloud-native ecosystem is written in. Kernel side is still restricted C — Go is only the userspace half. ⚠️ *not verified this pass.*

**Languages required**
- *Aya route*: **Rust** (both sides). Non-trivial if you don't already have it — ownership, lifetimes, and `no_std` for the kernel half. But you learn one language, not two.
- *Go route*: **Go** (userspace) **+ restricted C** (kernel). Go is quick to learn; you still need the C.

**Good for**: Topic **#6** (out-of-tree BPF-LSM prototypes are frequently Go). Aya is also the honest choice if the language-based-safety thread (source 2.8, Rex) interests you — it puts you in the right mental model.

**Bad for**: matching the literature. Nearly every paper you'll cite uses C/libbpf. If your contribution is a *comparison* against published work, using a different binding adds a confound you'll have to defend.

---

### Option D — bpftime: userspace eBPF, no kernel required *(the escape hatch)*

**What it is.** A userspace eBPF **runtime and extension framework** — explicitly *not* merely a userspace VM (that component alone is `llvmbpf`). It includes a loader, a verifier, helpers, maps, ufunc support, and event sources. It bypasses the kernel entirely and uses LLVM to accelerate uprobes, USDT, syscall hooks, XDP, and even GPU instrumentation. Dynamic binary rewriting lets it attach to an **already-running process with no restart**. It works with unmodified clang, libbpf, and bpftrace, and supports CO-RE via BTF.

- Published at **OSDI '25** — *"Extending Applications Safely and Efficiently,"* pp. 557–574. MIT licence. (github.com/eunomia-bpf/bpftime, verified 2026-08-02: 354 commits, ~1.5k stars, 96 open issues.)
- **Verifier choice**: PREVAIL in userspace, *or* the kernel verifier. PREVAIL is already in your reading list — see `0 - Seminal Papers` in [[eBPF Research - Index]].
- **JIT backends**: `llvmbpf` (JIT and AOT), the ubpf JIT, or a plain interpreter.
- **Claimed performance**: up to **10× lower uprobe overhead** than kernel uprobes; up to 10× faster than NVbit for GPU instrumentation.
- Hooking derives from frida-gum, zpoline, and `syscall_intercept`; GPU support via a custom eBPF→PTX conversion.

**Languages required**: restricted C (unchanged — it consumes normal BPF bytecode), plus **C++/CMake** if you intend to modify bpftime itself.

**Why it matters to you specifically**: it is the one option that does not require solving the Linux-kernel problem first, and the *verifier-choice* design makes it an unusually direct instrument for verifier research — you can run the same bytecode through PREVAIL and through the kernel verifier and diff the outcomes. That is a genuinely interesting experiment and it is nearly free to set up.

**Stated limits (from the project's own README)** — take these seriously:
- Some kernel helpers and kfuncs are unavailable in userspace.
- **No direct access to kernel structures such as `task_struct`.** This is disqualifying for most Section 4 (monitor-evasion) work, which is precisely about kernel-side observation.
- Safety depends on the *userspace* verifier — a different trust model, which is either a limitation or your thesis, depending on the angle.
- `load` must run before `start`; `bpftime attach` may require root.
- AF_XDP/DPDK integration is marked **(experimental)**.
- ⚠️ No production-readiness claim, and no version number or release date is published on the README.

---

### Option E — Cloud / rented bare metal *(the only route to the hardware-isolation topics)*

**What it is.** Rent a machine with the CPU features your laptop lacks, and/or a kernel you can't easily run locally.

**When you need it**
- **Intel MPK work (Topics #2, #3, #9)** — requires a CPU with `pku`/`ospke`. Broadly: Intel Skylake-SP and later *server* parts, and most modern AMD EPYC/Ryzen. ⚠️ *Verify the exact instance type's flags with `grep -o pku /proc/cpuinfo` before committing money — cloud vendors do not always expose PKU to guests.*
- **ARM PAC/MTE work (Topics #2, #9)** — requires AArch64 hardware. Note that MTE in particular is *not* enabled on all ARM server parts; check before assuming.
- **Line-rate networking benchmarks** — XDP numbers from a laptop VM are not publishable.
- **Kernel builds** — a 16-core cloud box builds a kernel in minutes rather than an hour.

**Languages required**: same as whichever of A–C you run on it, plus enough shell/Terraform/`cloud-init` to make the environment reproducible. **Reproducibility is a grading criterion** — script the provisioning, don't hand-configure.

**Cost note**: bare-metal rental is meaningfully more expensive than a shared VM, but nested virtualization and CPU-feature passthrough on shared instances are unreliable. For anything measuring *hardware* behaviour, insist on bare metal or a dedicated instance.

---

## 4. Languages — what to actually learn, and how much

| Language | Where it's used | Depth needed | Learn it if… |
|---|---|---|---|
| **Restricted C** | Kernel side, all options | **Deep.** This is the core skill. | Always. Non-negotiable. |
| **C (ordinary)** | libbpf userspace loaders | Moderate — skeletons do the heavy lifting | You take Option B |
| **Rust** | Aya, both sides | Deep — `no_std`, ownership | You take Option C-Rust, or care about language-based safety (2.8) |
| **Go** | cilium/ebpf, libbpfgo userspace | Light — Go is small | You take Option C-Go, or want to read/modify Tetragon |
| **Python** | BCC, and *all* your analysis/plotting | Light for BCC; you already know enough | Always, for analysis if nothing else |
| **bpftrace DSL** | One-liners, exploration | Trivial — an afternoon | Always. Cheapest possible win. |
| **C++ / CMake** | Modifying bpftime, PREVAIL | Moderate | You take Option D *and* modify it |
| **Shell / Make** | Build pipelines, provisioning, harnesses | Moderate | Always |

**The honest summary**: **restricted C is mandatory; everything else is a choice.** Rust is the most valuable *additional* language for this specific project — it sits directly on the language-based-safety thread your research already touches — but it is also the most expensive to learn well. If your capstone timeline is tight, C + Python + bpftrace is a complete and defensible stack.

---

## 5. Testing and benchmarking — the part that separates a project from a demo

This is what the literature review can't tell you and what an examiner will press on. Four layers.

### 5.1 Correctness layer

**"Does it verify?" is your first and cheapest test signal.** A rejected program is a failed test with a free explanation attached.

- Get the full verifier log. Via libbpf, raise `log_level` (1 for the standard log, 2 for per-instruction state, 4 for stats). Via `bpftool prog load ... -d`. The log ends with a summary line — `processed N insns (limit 1000000) max_states_per_insn ... total_states ... peak_states ...` — and **those numbers are themselves a metric**, not just diagnostics. For any verifier-adjacent topic, `processed insns`, `total_states`, and verification wall-time are your primary dependent variables.
- **The kernel's own BPF selftests are the canonical harness**: `tools/testing/selftests/bpf` in the kernel tree, principally `test_verifier` (verifier accept/reject cases) and `test_progs` (end-to-end functional tests). If you are doing Topic **#1** (mapping formal-verification gaps), this suite *is* your corpus — it encodes what the kernel developers believe the verifier should accept and reject, and the delta between that and what a formal tool proves is exactly the gap you'd be characterising.
- Write your own accept/reject fixtures alongside it. A verifier-research project's test suite is a set of programs with expected verdicts, not a set of assertions about output.

### 5.2 Functional layer

- **Golden-output tests**: known input → known event stream. Drive with a deterministic workload generator, not ambient system noise.
- **Check for dropped events explicitly.** Ring buffers and perf buffers both silently lose data under pressure. `bpf_ringbuf_query()`, the perf buffer's `lost_cb` callback, and your own drop counters in a map. **A benchmark that doesn't report its loss rate is not reporting its results.** This is the single most common way eBPF measurements are quietly wrong.
- **Negative tests**: does it behave correctly when the map is full, when the target process exits mid-probe, when attached to a PID that never fires?

### 5.3 Performance layer

- **Always measure a null baseline.** An empty program attached at the same hook. Your reported overhead is (instrumented − null), otherwise you are measuring the attach mechanism, not your code.
- **Choose the right unit.** Per-event nanoseconds or cycles for tracing; packets-per-second and cycles-per-packet for XDP/tc; verification time and instruction count for verifier work.
- **Use the kernel's own benchmark runner** where it fits — `tools/testing/selftests/bpf/bench` includes ring-buffer, trigger, and counting benchmarks that are already peer-scrutinised, which is worth more than a harness you wrote yesterday. ⚠️ *Confirm which benchmarks exist in the specific kernel version you target; the set changes.*
- **`perf stat`** for cycles, instructions, cache misses, and branch misses around the hot path.

### 5.4 Statistical rigour — where laptop VMs will betray you

This matters more than usual for you, because you are benchmarking inside a virtual machine on a 4-core mobile CPU with turbo boost.

- **Pin CPUs** (`taskset`), and where possible isolate cores (`isolcpus`) so the scheduler doesn't move your workload mid-run.
- **Disable frequency scaling** in the guest and, if you can, the host — a Coffee Lake mobile part will thermally throttle under sustained load, and your "performance regression" will be your fan.
- **Warm up**, then discard the warm-up. JIT compilation, page faults, and cache population all happen once.
- **Many repetitions; report median and interquartile range, not mean and standard deviation.** Systems latency distributions are not normal — they are long-tailed, and the mean is dominated by the tail.
- **Report the environment with every number**: `uname -r`, distro, LLVM version, whether BTF was present, VM type and vCPU/RAM allocation, and whether it was bare metal. Numbers without this are not reproducible and will be marked down.
- **State the VM caveat explicitly in the writeup.** Timing inside a VM on a shared, thermally-constrained laptop is noisy. Owning that limitation reads as rigour; having an examiner find it reads as carelessness.

### 5.5 For adversarial topics (#7, #8, #9, #10) specifically

Benchmarking is the wrong frame. Your dependent variable is **detection**, not latency.

- Build an **attack corpus** with ground-truth labels — which technique, which expected observable.
- Report a **confusion matrix**, not an accuracy figure. Topic #8 exists precisely because "100% detection / 0% false positives" (source 4.13) is a red flag, and reproducing that claim requires you to be better at evaluation than the paper was.
- **Establish the baseline first**: does the monitor detect the attack *un*evaded? If not, your evasion result is meaningless.
- Run attacks against **each** monitor (Falco, Tetragon, Tracee) — the taxonomy in Topic #7 is a matrix of technique × tool, and the empty cells are the contribution.

---

## 6. Which approach unlocks which research topic

Cross-referenced against [[Potential Research Topics]].

| Topic | Viable locally? | Recommended route |
|---|---|---|
| **#1** Formal-methods gaps in verifier/JIT | ✅ Yes | B (+ kernel source tree; selftests as corpus). Largely analytical — cross-arch JIT claims need QEMU emulation, which is fine here since you're checking *correctness*, not speed. |
| **#2** Portable hardware isolation | ❌ **No** — no PKU exposed to guests (§1.1), x86 only | E (bare metal, per-arch). The most build-heavy topic and the one your laptop least supports. |
| **#3** Transient-execution coverage | ❌ **No** | E. Also needs microarchitectural measurement expertise beyond eBPF itself. |
| **#4** Verification vs. hardware vs. language safety | ⚠️ Partly | Primarily a position/evaluation piece. B + C-Rust locally for the language-safety half; E if you want real isolation numbers. |
| **#5** Hornet LSM signing dispute | ✅ Yes | B, on a **6.18 LTS** kernel (see §8). Mostly reading, tracking, and explaining — light on hardware. |
| **#6** In-kernel signing vs. BPF-LSM head-to-head | ✅ Yes | B, on 6.18 LTS. Two existing prototypes to start from. Strong systems project with a modest hardware bill. |
| **#7** Taxonomy of monitor evasion | ✅ Yes | A then B, in a VM with snapshots. **Snapshot before every attack run.** |
| **#8** Stress-test the weak-evaluation paper | ✅ Yes | A + B. Excellent value: contribution is methodological, hardware demands are minimal. |
| **#9** Does hardware isolation stop telemetry blinding? | ❌ **No** locally | E. Flagged in [[eBPF Research - Index]] as the clearest unaddressed intersection — and the one your hardware most blocks. Budget for cloud if you pick it. |
| **#10** Provenance-IDS attacks vs. eBPF collectors | ✅ Yes | A + B. Needs an ML/analysis pipeline (Python) more than exotic kernels. |

**The pattern is stark**: the topics your machine supports are #1, #5, #6, #7, #8, #10 — the verifier, supply-chain, and adversarial-robustness threads. The topics it blocks are the hardware-isolation cluster (#2, #3, #9) — blocked because macOS does not expose protection keys to a guest (§1.1), not because the silicon has been shown to lack them. If the hardware-isolation angle is where your interest genuinely lies, the honest plan is to price a cloud instance now and confirm PKU is exposed to guests *before* writing the proposal.

---

## 7. A recommended sequence

Not a timeline — an ordering. Each step produces something you can show.

1. **Get a Linux kernel.** Install Lima (`brew install lima`) or multipass, allocate 4 GB / 2 vCPU / 40 GB, boot a recent Ubuntu or Fedora. Verify BTF is present: `ls /sys/kernel/btf/vmlinux`. If it isn't, you have the wrong image — CO-RE and most modern tooling depend on it. *(Lima install docs verified 2026-08-02; QEMU is required only for the QEMU driver, and macOS 13+ hosts can use the `vz` backend. ⚠️ vz behaviour on Intel Macs specifically not verified.)*
2. **Snapshot the VM.** Before anything else. You will break it, and Section 4 work involves deliberately running malicious behaviour.
3. **One afternoon of bpftrace.** `bpftrace -l` to enumerate probes, then a handful of one-liners. Goal: see real kernel events with your own eyes. Cost: hours.
4. **One `libbpf-bootstrap` program, end to end.** Write it, compile it, generate the skeleton, load it, read from a ring buffer. Goal: understand every stage of the pipeline. Cost: a few days, most of it fighting the verifier — which is the point.
5. **Get rejected on purpose.** Write a program with an unbounded loop, then one with an unchecked pointer. Read the full verifier log each time. **This is the single most valuable hour** for anyone whose capstone concerns the verifier: you cannot critique what you have never been refused by.
6. **Build a measurement harness** — null baseline, pinned CPU, many reps, median + IQR, loss-rate reporting. Reuse it for everything after.
7. **Only now choose the topic**, having felt the constraints. Then re-read §6 and provision hardware accordingly.
8. **If verifier-adjacent**: install bpftime and run the same bytecode through PREVAIL and the kernel verifier. Cheap, and a diff between two verifiers is a result.

---

## 8. Kernel version — which to target

Verified against `kernel.org` on **2026-08-02**:

| Series | Version | Note |
|---|---|---|
| mainline | 7.2-rc5 | Don't. |
| stable | 7.1.5 | Latest features, short support window |
| **longterm** | **6.18.41** | ⭐ **Recommended baseline** |
| longterm | 6.12.100, 6.6.147, 6.1.180, 5.15.213, 5.10.262 | Older LTS lines |
| linux-next | next-20260731 | For tracking in-flight work only |

**Target 6.18 LTS.** It carries the **signed-BPF work** that [[3 - Secure eBPF Program Supply Chain]] (sources 3.1, 3.8) is built around, and it is now a longterm release — meaning it is stable, packaged, and will still be supported when you defend. For supply-chain topics (#5, #6) this is not merely convenient; it is the version the work you're citing actually landed in.

Record the exact version in every result you report. "Linux 6.18" is not a specification; "6.18.41, Ubuntu 26.04, BTF present, LLVM 20" is.

---

## 9. Caveats and unverified claims

Per vault convention, flagged rather than smoothed over:

- ⚠️ **Aya is pre-1.0** (0.14.0). API stability is not guaranteed across a multi-month project. Pin the version in `Cargo.toml` and expect at least one gap in API coverage.
- ⚠️ **bpftime publishes no version number or release date** on its README, and makes no production-readiness claim. It is OSDI-published research software. Excellent for experiments; do not build a deliverable's critical path on it without a fallback.
- ⚠️ **Docker Desktop's LinuxKit VM was not inspected** — the daemon was not running at probe time. Its kernel version, BTF availability, and BPF-LSM configuration are unknown. Containers also share the host VM's kernel, so Docker gives you *a* kernel, not one you control. Treat it as unsuitable for kernel-version-sensitive work until verified.
- ⚠️ **The local PKU finding rests on a macOS `sysctl` probe**, which reports OS-enabled features rather than silicon capability. The *practical* conclusion holds (a guest cannot use a feature the host has not enabled), but the note deliberately does **not** claim the i5-8259U's silicon lacks protection keys — that was never verified. See §1.1.
- ⚠️ **PKU availability on cloud instances is not guaranteed** even when the underlying CPU supports it — hypervisors may mask the feature. Verify on the actual instance before committing.
- ⚠️ **Links verified 2026-08-02**: kernel.org release banner, github.com/aya-rs/aya, github.com/eunomia-bpf/bpftime, crates.io API for `aya`, lima-vm.io installation docs.
- ⚠️ **Links not verified this pass**: libbpf-bootstrap, cilium/ebpf, bcc, bpftrace, multipass, UTM. Check before citing.
- ⚠️ The `selftests/bpf/bench` benchmark set changes between kernel versions. Confirm what exists in your target tree rather than assuming.

---

## 10. Related notes

- [[eBPF Fundamentals]] — the mechanism itself. **§15 (Toolchain and ecosystem)** and **§16 (Version-gated features — check before citing)** are the reference for what each tool *is*; this note covers which to pick and why. **§11** covers the hook/attach taxonomy referenced throughout.
- [[eBPF Research - Index]] — the literature. This note is its practical counterpart.
- [[Potential Research Topics]] — the ten candidate angles, mapped to feasibility in §6 above.
- [[eBPF (Extended Berkeley Packet Filter)]] — capstone home note. **§5 (Technical Stack)** should be filled from §3–§4 of this note once a direction is chosen.

---
Compiled: 2026-08-02 · Environment probed on the same date; re-verify §1 if the machine changes.
