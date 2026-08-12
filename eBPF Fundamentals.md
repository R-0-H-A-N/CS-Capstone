---
tags: [ebpf, fundamentals, primer, baseline]
related: "[[eBPF (Extended Berkeley Packet Filter)]]"
status: done
---
# eBPF — Fundamentals

← Back to [[eBPF (Extended Berkeley Packet Filter)]]

> **Purpose of this note.** This is the baseline conceptual reference for the capstone: what eBPF *is*, how it works end to end, and what its parts are called. It is deliberately **descriptive**, not argumentative — it explains the mechanism without taking a position on whether the safety model holds. That question is the subject of the research notes in [[eBPF Research - Index]].

---

## 1. The one-paragraph definition

**eBPF is a technology that lets you run small, sandboxed programs inside the operating system kernel — safely, at runtime, without writing a kernel module and without recompiling or rebooting the kernel.** You write a program in a restricted dialect of C (or Rust), compile it to a compact 64-bit bytecode, and hand it to the kernel through the `bpf()` syscall. Before the kernel will accept it, a static analyser called the **verifier** proves the program is safe to run — that it terminates, that it touches only memory it is allowed to touch, and that it calls only functions it is allowed to call. If it passes, a **JIT compiler** translates the bytecode into native machine code, and the program is **attached to a hook** — a specific event inside the kernel, such as a packet arriving on a NIC, a syscall being entered, or a security decision being made. From then on, every time that event fires, your code runs, at native speed, in kernel context.

The short version people use: **eBPF is to the kernel what JavaScript is to the browser.** The browser doesn't let you rewrite its rendering engine, but it does give you an event-driven sandbox in which your code runs alongside it. eBPF is the same bargain, struck with the Linux kernel.

**Name note:** "BPF" originally stood for *Berkeley Packet Filter*, and "eBPF" for *extended* BPF. Neither acronym describes the technology any more — it long ago outgrew packet filtering. In current kernel and standards usage, **eBPF is treated as a proper noun, not an abbreviation** ([RFC 9669](https://www.rfc-editor.org/rfc/rfc9669.html) says as much explicitly). The original 1990s design is now retroactively called **cBPF** (classic BPF) to distinguish it.

---

## 2. The problem eBPF solves

The kernel sits between every application and the hardware. It sees every packet, every syscall, every file open, every process creation. That makes it the ideal place to observe, filter, secure, or reroute system behaviour. It is also the *worst* place to put new code, because kernel code runs with full privilege and no memory protection — a bug is not a crashed process, it is a crashed machine or a compromised one.

Historically you had three bad options:

**Option A — modify the kernel itself.** Upstream your change into Linux. Correct, permanent, and reviewed by the best systems engineers alive. It also takes months to years, and then several more years to reach the distributions your users actually run.

**Option B — write a loadable kernel module.** Fast to iterate and you get full kernel power. You also get full kernel *risk*: a null dereference panics the box. Modules are tied to kernel internals, so they break across versions, and they carry no safety guarantee whatsoever. This is the mechanism behind a meaningful share of the industry's worst outages.

**Option C — do it from userspace.** Safe, portable, easy to debug. But you cannot see what the kernel sees, and getting data out means expensive context switches and copies. Filtering packets in userspace means every packet you intend to drop has already cost you the trip up the stack.

eBPF is a fourth option that takes the good half of each:

| | Kernel module | Userspace daemon | **eBPF** |
|---|---|---|---|
| **Safety** | None — bugs panic the kernel | Process isolation | Verifier-checked before load |
| **Performance** | Native, in-kernel | Context switches + copies | Native (JIT), in-kernel |
| **Deployment** | Load module, often reboot | Trivial | Load at runtime, no reboot |
| **Kernel visibility** | Total | Only what's exported | Broad, via defined hooks |
| **Portability** | Recompile per kernel | High | High, via CO-RE (§9) |
| **Expressiveness** | Unrestricted | Unrestricted | **Restricted** — bounded, no arbitrary memory, limited API |
| **Failure mode** | System crash | Process dies | Program rejected at load, or dropped event |

The trade is explicit and worth stating plainly: **you give up expressiveness to buy safety.** eBPF will refuse to run a program it cannot prove safe, and that refusal is often the hardest part of writing one. Everything in §6 is downstream of that bargain.

---

## 3. The lifecycle, end to end

This is the single most useful thing to internalise. Every eBPF deployment follows the same seven steps.

```
   ┌─────────────────────── USERSPACE ────────────────────────┐
   │                                                          │
   │  1. Write        restricted C  /  Rust                   │
   │        │                                                 │
   │        ▼  clang -target bpf                              │
   │  2. Compile      → ELF object containing BPF bytecode    │
   │                     + BTF type information               │
   │        │                                                 │
   │        ▼  libbpf / ebpf-go / Aya                         │
   │  3. Load         bpf(BPF_PROG_LOAD, ...)  ──────────┐    │
   │                                                     │    │
   │  7. Interact     read maps, poll ring buffer  ◄──┐  │    │
   └──────────────────────────────────────────────────┼──┼────┘
                                                      │  │
   ┌─────────────────────── KERNEL ───────────────────┼──┼────┐
   │                                                  │  ▼    │
   │  4. Verify       static analysis — terminates?  safe     │
   │                  memory in bounds? API allowed?          │
   │                          │                               │
   │                     reject ◄──┴──► accept                │
   │                                     │                    │
   │  5. JIT compile  BPF bytecode → native x86-64/arm64/...  │
   │                                     │                    │
   │  6. Attach       XDP · tc · kprobe · tracepoint ·        │
   │                  fentry · LSM · cgroup · struct_ops      │
   │                                     │                    │
   │                              ┌──────▼──────┐             │
   │                    EVENT ───►│ your program│───► maps ───┘
   │                              └─────────────┘   (shared state)
   └──────────────────────────────────────────────────────────┘
```

**1 — Write.** Source is C restricted to what the verifier can reason about, annotated with section names (`SEC("xdp")`, `SEC("kprobe/do_sys_openat2")`) that tell the loader which program type and attach point each function is destined for. Rust is a first-class alternative via Aya.

**2 — Compile.** Clang/LLVM has a BPF backend (`-target bpf`). Output is a standard ELF object file whose sections contain BPF bytecode, map definitions, and **BTF** (see below).

**3 — Load.** A userspace loader library parses the ELF, creates the maps, applies relocations, and calls `bpf(BPF_PROG_LOAD, ...)`. Loading requires privilege (§12).

**4 — Verify.** The kernel's verifier statically analyses the bytecode and either accepts or rejects it. This is the safety gate and the subject of §6.

**5 — JIT.** The in-kernel JIT compiler translates verified bytecode into native instructions for the host architecture. Linux ships JIT backends for x86-64, arm64, riscv, s390, powerpc, and others. (An interpreter exists as a fallback but is disabled on most modern configurations.)

**6 — Attach.** The program is bound to a hook. Attachment is what makes it *live*.

**7 — Run and communicate.** The program executes on every occurrence of its event. It shares state with userspace — and with other eBPF programs — through **maps** (§7).

### What one actually looks like

The whole model fits in about twenty lines. This program fires on every process execution and streams the PID and command name to userspace:

```c
/* execsnoop.bpf.c — the kernel side.
   Build: clang -O2 -g -target bpf -c execsnoop.bpf.c -o execsnoop.bpf.o */

#include "vmlinux.h"                   /* every kernel type, generated from BTF */
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "GPL";  /* required — many helpers are GPL-only */

struct event {                          /* what we hand to userspace */
    __u32 pid;
    char  comm[16];
};

struct {                                /* a MAP: 256 KB shared ring buffer  →§7 */
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} events SEC(".maps");

SEC("tp/sched/sched_process_exec")      /* the HOOK: a static tracepoint  →§11 */
int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
{
    struct event *e;

    e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e)                             /* the VERIFIER REJECTS the program     */
        return 0;                       /* without this NULL check        →§6   */

    e->pid = bpf_get_current_pid_tgid() >> 32;        /* a HELPER call     →§8  */
    bpf_get_current_comm(&e->comm, sizeof(e->comm));  /* another one            */

    bpf_ringbuf_submit(e, 0);           /* commit; userspace can now read it     */
    return 0;
}
```

Five things to notice, because they are the whole model in miniature:

1. **`SEC(...)` annotations are the wiring.** `SEC("tp/sched/sched_process_exec")` is what tells the loader this is a tracepoint program and where to attach it. Nothing else in the file says "run me on exec."
2. **The map is declared, not allocated.** The loader creates it before the program loads; the program only ever refers to it.
3. **`if (!e) return 0;` is not defensive style — it is mandatory.** Remove it and the load *fails*, because the verifier cannot prove the following writes are in bounds. This is §6 made concrete.
4. **Every kernel interaction is a helper call.** There is no `printf`, no `malloc`, no direct kernel function call.
5. **`ctx` is the context pointer** from `r1` — the typed window onto the event (§5).

The userspace side is correspondingly small. `bpftool gen skeleton execsnoop.bpf.o > execsnoop.skel.h` produces a header, and the program becomes roughly: `execsnoop_bpf__open_and_load()` → `execsnoop_bpf__attach()` → poll the ring buffer in a loop, printing each `struct event` as it arrives. Steps 3, 6, and 7 of the diagram, one function call each.

### BTF — the type information that makes the rest work

**BTF (BPF Type Format)** is a compact debug-info format describing the types of everything involved: kernel structs, map key/value types, program function signatures. It is introduced here because three later features are impossible without it.

- Modern kernels are built with BTF for their own data structures, exposed at `/sys/kernel/btf/vmlinux`. `bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h` produces a single header describing every kernel type on *that* machine.
- BTF lets the verifier understand pointer types precisely, which is what enables typed kernel-function calls (**kfuncs**, §8).
- BTF is the mechanism behind **CO-RE** portability (§9).

If you remember one thing: *BTF is how eBPF stopped being kernel-version-specific.*

---

## 4. Where it came from

Worth knowing because the design constraints are historical.

**1992 — cBPF.** Steven McCanne and Van Jacobson publish *The BSD Packet Filter* — a tiny register-based virtual machine that lets userspace push a packet-matching program *into* the kernel, so the kernel discards uninteresting packets before copying them up. Two 32-bit registers (an accumulator `A` and an index `X`), a 16-slot scratch memory (`M[0..15]`) that programs may write, read-only access to the packet itself, and no loops at all. This is the engine underneath `tcpdump` to this day, and it later found a second life powering `seccomp-bpf` syscall filtering — a job it still does, on the *classic* VM (see §11).

**2014 — eBPF.** Alexei Starovoitov (with Daniel Borkmann) rewrites the VM for Linux 3.15+. The changes are foundational: 64-bit registers mapped 1:1 onto real hardware registers so JIT output is efficient; eleven registers instead of two; **maps** for persistent state; **helper functions** for calling into the kernel; and, crucially, decoupling from packets entirely — the same machinery can now attach to tracepoints and kprobes. cBPF programs are transparently translated to eBPF internally.

**2015–2018 — the ecosystem.** XDP lands (2016), pushing programs down to the NIC driver, ahead of the kernel's own networking stack. BCC and later bpftrace make tracing accessible. Cilium proves eBPF can replace iptables at scale for Kubernetes networking.

**2018–2021 — portability and maturity.** BTF and CO-RE arrive, ending the era of shipping a compiler to every production host. BPF trampolines (`fentry`/`fexit`) make tracing dramatically cheaper than kprobes. BPF LSM makes eBPF a security-enforcement mechanism, not merely an observational one.

**2021–present — institutionalisation.** The eBPF Foundation forms under the Linux Foundation (Meta, Google, Isovalent, Microsoft, Netflix, Red Hat). Microsoft ships eBPF for Windows. The IETF publishes **[RFC 9669](https://www.rfc-editor.org/rfc/rfc9669.html)** (October 2024, D. Thaler), standardising the instruction set independently of Linux — which matters for hardware offload and for non-Linux implementations. `sched_ext` lets eBPF programs implement CPU schedulers.

---

## 5. The virtual machine and instruction set

eBPF's runtime model is small enough to describe completely.

**Registers.** Eleven 64-bit registers, `r0`–`r10`, with a fixed calling convention chosen to map cleanly onto x86-64 and arm64 ABIs so that JIT compilation is close to a direct translation:

| Register | Role |
|---|---|
| `r0` | Return value from helper calls; **exit code of the program** |
| `r1`–`r5` | Arguments to helper/function calls (`r1` also holds the program's context pointer on entry) |
| `r6`–`r9` | Callee-saved — survive a helper call |
| `r10` | **Read-only** frame pointer for the stack |

That `r10` is read-only is not a detail — it means a program cannot forge a stack pointer.

**Stack.** A fixed **512 bytes** per program. This is small, and it is the reason eBPF code passes data through maps rather than large local buffers.

**Instruction encoding.** Fixed-width **64-bit (8-byte)** instructions: an 8-bit opcode, 4-bit destination register, 4-bit source register, 16-bit signed offset, 32-bit signed immediate. One exception: a "wide" 16-byte form exists for loading 64-bit immediates. Roughly a hundred opcodes covering ALU (32- and 64-bit), load/store, jump/branch, atomics, call, and exit.

**Context.** On entry, `r1` points to a **context structure** whose type depends on the program type: `struct xdp_md` for XDP, `struct __sk_buff` for tc, `struct pt_regs` for kprobes, and so on. This is the program's window onto the event.

**Return value.** The meaning of `r0` at exit is also program-type-specific: for XDP it is a verdict (`XDP_PASS`, `XDP_DROP`, `XDP_TX`, `XDP_REDIRECT`); for LSM, zero or a negative errno that *denies the operation*; for most tracing programs it is simply ignored.

**Where it runs.** In kernel context, on the CPU that took the event, typically with preemption disabled and always without the ability to sleep — unless it is explicitly a *sleepable* program (a category added later for hooks where blocking is permissible, e.g. certain LSM and tracing attach points).

---

## 6. The verifier

The verifier is the component that makes the whole arrangement tenable. It is a static analyser living in `kernel/bpf/verifier.c`, and it runs at load time, before a single instruction executes.

### What it actually does

It performs an **abstract interpretation** of the program, walking every reachable path from entry and maintaining, for each register at each instruction, a symbolic description of what that register could hold:

- **Type** — is this a scalar, a pointer to the stack, a pointer to map value, a pointer to packet data, a pointer to a kernel struct, or possibly-NULL?
- **Range** — what is the proven minimum and maximum value? Signed and unsigned bounds are tracked separately.
- **Tristate numbers (`tnum`)** — a bitwise abstraction recording, per bit, whether it is known-0, known-1, or unknown. This is what lets the verifier reason precisely about masks and shifts.

From that state it enforces the safety properties:

1. **Termination.** Originally by refusing loops entirely (a DAG check). Modern verifiers support **bounded loops** — loops the analyser can prove terminate — plus explicit bounded-iteration constructs (`bpf_loop()`, open-coded iterators).
2. **Memory safety.** Every load and store must be provably in bounds. Reading packet data requires an explicit comparison against `data_end` *first*; the verifier tracks that comparison and only then permits the access. Uninitialised stack reads are rejected.
3. **Pointer discipline.** Pointers may not be leaked to userspace, converted to scalars, or arbitrarily arithmetic'd. A map lookup returns a *maybe-NULL* pointer, and the verifier will not permit dereferencing it until the program has explicitly checked for NULL.
4. **API restriction.** Only helper functions and kfuncs permitted for that program type may be called, with argument types matched against their signatures.
5. **Complexity budget.** Path exploration is capped, with state pruning to merge equivalent states. A program that is *safe but too hard to prove safe* is rejected.
6. **Speculative-execution hardening.** Additional analysis and instruction rewriting to mitigate Spectre-class attacks, since an attacker-supplied program running in kernel context is an unusually convenient gadget source.

If verification succeeds, the verifier may also rewrite the program — inlining certain helper calls, patching in bounds masks, and applying **constant blinding** on hardened configurations to prevent JIT-spraying.

### What this feels like in practice

The verifier rejecting your program is the normal condition of eBPF development, not an exceptional one. Its error messages take the form of a full instruction-by-instruction register-state dump, and reading them fluently is a genuine skill. The common causes are: dereferencing a map lookup without a NULL check; touching packet bytes without a `data_end` bounds check; a loop whose bound the verifier can't establish; blowing the 512-byte stack; and exceeding the complexity budget.

> **Positioning note.** The verifier is where eBPF's safety guarantee is concentrated: it is a large, evolving, hand-written analyser in C, and everything downstream trusts it. Whether that concentration of trust is sound, whether the verifier can be *formally proven* correct, and what defence-in-depth would look like if it can't, are exactly the open questions the capstone research covers — see [[1 - Formal Verification of the Verifier and JIT]] and [[2 - Hardware-Assisted Isolation for eBPF]]. This note describes the mechanism; those notes evaluate it.

---

## 7. Maps — how eBPF holds state

An eBPF program is a stateless function invoked on an event. **Maps** are how it gets state, and they are as central to the model as the verifier.

A map is a kernel-resident key/value store, created before the program loads, accessed from eBPF via helpers (`bpf_map_lookup_elem`, `bpf_map_update_elem`, `bpf_map_delete_elem`) and from userspace via the `bpf()` syscall or by memory-mapping. Maps therefore serve four distinct purposes at once:

- **Persistence** across invocations of the same program
- **Communication** between kernel and userspace, in both directions
- **Sharing** between different eBPF programs
- **Configuration** — userspace writes policy into a map, the program reads it on each event

Maps are reference-counted objects; they can be pinned into the BPF filesystem (`/sys/fs/bpf`) so they outlive the process that created them.

### The main map types

| Category | Types | Use |
|---|---|---|
| **General** | `HASH`, `ARRAY`, `LRU_HASH`, `PERCPU_HASH`, `PERCPU_ARRAY` | Counters, state tables, config. Per-CPU variants avoid cross-CPU contention entirely — the standard trick for high-rate counting. |
| **Streaming to userspace** | `RINGBUF`, `PERF_EVENT_ARRAY` | Pushing events out. See below. |
| **Specialised lookup** | `LPM_TRIE` | Longest-prefix match — i.e. CIDR/routing lookups. |
| **Queues** | `QUEUE`, `STACK` | FIFO/LIFO without keys. |
| **Program references** | `PROG_ARRAY` | Targets for **tail calls** (§10). |
| **Nesting** | `ARRAY_OF_MAPS`, `HASH_OF_MAPS` | Atomically swappable map sets. |
| **Networking objects** | `SOCKMAP`, `SOCKHASH`, `DEVMAP`, `CPUMAP`, `XSKMAP` | Redirect targets — sockets, devices, CPUs, AF_XDP queues. |
| **Storage attached to objects** | `SK_STORAGE`, `INODE_STORAGE`, `TASK_STORAGE`, `CGRP_STORAGE` | Per-socket/inode/task/cgroup local storage with the kernel handling lifetime — hugely convenient for security tooling. |

### Getting data out: ring buffer vs perf buffer

Two mechanisms, and choosing correctly matters.

- **`PERF_EVENT_ARRAY` (perf buffer)** — the older approach. One independent ring per CPU. Consequences: memory scales with core count, events can be reordered relative to each other across CPUs, and a busy CPU can drop events while another CPU's buffer sits empty.
- **`RINGBUF` (BPF ring buffer)** — the modern default. A single multi-producer/single-consumer buffer shared across all CPUs. It preserves ordering, uses memory far more efficiently, and supports a *reserve/commit* pattern (`bpf_ringbuf_reserve()` → fill in place → `bpf_ringbuf_submit()`) that avoids a copy and lets the program discard an event after inspecting it.

Prefer `RINGBUF` unless you need per-CPU independence or must support kernels that predate it.

---

## 8. Helpers and kfuncs — the kernel API

An eBPF program cannot call arbitrary kernel functions; that would defeat the entire safety argument. It reaches the kernel through two curated interfaces.

**Helper functions** are a stable, numbered ABI defined by the kernel, each with a fixed signature the verifier checks. Which helpers are available depends on program type — a tracing program can read arbitrary kernel memory, a networking program cannot. A representative sample:

| Helper | Purpose |
|---|---|
| `bpf_map_lookup_elem` / `update_elem` / `delete_elem` | Map access |
| `bpf_probe_read_kernel` / `bpf_probe_read_user` | Safely read memory that might fault |
| `bpf_ktime_get_ns` | Monotonic timestamp |
| `bpf_get_current_pid_tgid` / `_uid_gid` / `_comm` | Current task identity |
| `bpf_get_stackid` / `bpf_get_stack` | Capture a stack trace (the basis of continuous profiling) |
| `bpf_perf_event_output` / `bpf_ringbuf_output` | Emit an event to userspace |
| `bpf_redirect` / `bpf_redirect_map` | Steer a packet elsewhere |
| `bpf_trace_printk` | Debug printing to the trace pipe |
| `bpf_tail_call` | Jump to another program (§10) |
| `bpf_spin_lock` / `bpf_spin_unlock` | Guard a map value |

**kfuncs (BPF kernel functions)** are the newer mechanism: ordinary kernel functions explicitly *exported* to BPF via `BTF_KFUNCS_START`/`BTF_ID_FLAGS` annotations. Because BTF gives the verifier full type information, kfuncs can take and return typed kernel pointers, which helpers largely cannot. The trade is stability: helpers are a committed ABI, whereas **kfuncs carry no stability guarantee and can change between kernel releases**. The kernel community has clearly signalled kfuncs as the direction of travel — new functionality is increasingly exposed this way rather than as numbered helpers.

Modern kfuncs also underpin things helpers never could: acquiring and releasing refcounted kernel objects with verifier-enforced pairing, dynamic pointers (`bpf_dynptr`), and open-coded iterators.

---

## 9. CO-RE and portability

**The problem.** eBPF tracing programs read kernel data structures directly. But `struct task_struct` has a different layout on every kernel build — field offsets shift with version and with config options. A program compiled against one kernel's headers reads garbage on another.

**The old answer (BCC).** Ship Clang/LLVM and the kernel headers to every production machine and compile the program at runtime, on the target. This works, and it is why BCC tools historically pulled in hundreds of megabytes of dependencies, burned CPU and memory at startup, and failed outright on hosts without matching kernel headers.

**The modern answer — CO-RE ("Compile Once, Run Everywhere").** Four pieces working together:

1. **BTF in the kernel** — the running kernel publishes its own type layout at `/sys/kernel/btf/vmlinux`.
2. **`vmlinux.h`** — a single generated header giving you every kernel type, so you no longer build against kernel source headers at all.
3. **Compiler relocations** — instead of emitting a hardcoded offset, Clang emits a *CO-RE relocation record* ("this access is to field `pid` of `struct task_struct`") alongside the instruction.
4. **Loader-side patching** — at load time, libbpf reads the target kernel's BTF, resolves each relocation to that kernel's actual offset, and rewrites the bytecode before submitting it to the verifier.

The result is a single small ELF binary that runs correctly across a wide range of kernel versions, with no compiler and no headers on the target. CO-RE also provides feature-detection constructs (`bpf_core_field_exists()`, `bpf_core_type_exists()`) so a program can adapt to structural differences rather than merely tolerate offset shifts.

> One consequence worth flagging, because it recurs in the capstone research: **CO-RE relocation means the bytecode the verifier sees is not byte-identical to the bytecode that was compiled and shipped.** That complicates any scheme that wants to cryptographically sign eBPF programs — see [[3 - Secure eBPF Program Supply Chain]].

---

## 10. Program composition and limits

**BPF-to-BPF function calls.** Programs can be split into multiple functions ("subprograms") rather than force-inlined into one blob. The verifier analyses each and then checks the whole call graph. Two constraints apply:

- **Call depth** is capped at **8 frames** (`MAX_CALL_FRAMES`).
- **The 512-byte stack is a budget for the entire call chain, not per frame.** The verifier sums each subprogram's stack usage down the deepest path and rejects the program if the total exceeds `MAX_BPF_STACK`. Mixing subprograms with tail calls tightens this further — a caller in that situation is limited to 256 bytes, because the worst case would otherwise stack 33 tail calls' worth of frames.

**Tail calls.** `bpf_tail_call(ctx, &prog_array, index)` transfers control to another eBPF program, replacing the current one — it does **not** return. This is how large logic gets decomposed past size limits, and how dispatch tables are built (e.g. index a `PROG_ARRAY` by protocol number). Because the stack frame is replaced rather than stacked, tail calls are cheap, but the chain is capped: **the tail-call limit is 33** (the kernel constant `MAX_TAIL_CALL_CNT` was itself renamed from 32 to 33 in 5.16 to match the behaviour, which had always been 33 — a fine example of why these numbers are worth checking rather than recalling).

**Size and complexity limits.** These two are routinely conflated and are genuinely different:

- **`BPF_MAXINSNS` = 4096 instructions** — the program-size ceiling for **unprivileged** programs.
- **`BPF_COMPLEXITY_LIMIT_INSNS` = 1,000,000** — the verifier's budget for *instructions processed during analysis*. On modern kernels, **privileged** programs may also be up to 1M instructions in size, so for privileged loads the binding constraint is usually the verifier's exploration budget rather than raw program length. A 200-instruction program with heavy branching can exhaust the complexity limit; a million NOPs will not.

Both limits were raised together by a single 2019 commit (`c04c0d2b968a`); see §16 for the version, which is worth confirming before you cite it.

---

## 11. Program types and attach points

The taxonomy of *where* eBPF can hook is effectively the map of what eBPF can do. Three broad domains plus a growing "other".

### Networking

Ordered roughly by how early in the packet's life they run:

| Hook | Position | Notes |
|---|---|---|
| **XDP** | NIC driver, before an `sk_buff` is allocated | Earliest and fastest possible point. Verdicts: `PASS`, `DROP`, `TX`, `REDIRECT`, `ABORTED`. Basis of DDoS scrubbing (Cloudflare, Meta's Katran) — dropping a packet here costs almost nothing. Some NICs support hardware offload. |
| **tc / clsact** | Traffic control layer, ingress **and** egress | Operates on a fully-formed `sk_buff`, so richer metadata is available; can mangle, redirect, and shape. The workhorse for container networking (Cilium). |
| **socket filter** | On a socket | The direct descendant of cBPF's original job. |
| **cgroup hooks** | `cgroup_skb`, `cgroup_sock`, `cgroup_sockopt`, `cgroup_sysctl`, `cgroup_device` | Per-cgroup network and resource policy — the primitive behind container-scoped network policy. |
| **sockops / sk_msg / sk_skb** | Socket lifecycle and message layer | TCP tuning by connection, and socket-to-socket redirect for accelerated proxying (bypassing the stack between two local sockets). |
| **lwt, netfilter, flow_dissector** | Lightweight tunnels, netfilter integration, protocol parsing | More specialised placements. |

### Tracing and observability

| Hook | What it attaches to |
|---|---|
| **kprobe / kretprobe** | Any kernel function entry/return, dynamically. Maximum flexibility, no stability guarantee, moderate overhead. |
| **fentry / fexit** | Same targets, via **BPF trampolines** — substantially lower overhead than kprobes, and `fexit` sees both arguments *and* return value in one program (kretprobes require stashing args in a map). Prefer these where available. |
| **tracepoint / raw_tracepoint** | Static instrumentation points the kernel maintains deliberately. Stable-ish interface; raw variants skip argument marshalling for speed. |
| **uprobe / uretprobe** | Userspace function entry/return — instrumenting applications and libraries without modifying them. |
| **USDT** | Statically-defined userspace tracepoints compiled into applications. |
| **perf_event** | Sampling on hardware/software perf events — the basis of continuous CPU profiling and flame graphs. |
| **BPF iterators** | Walk kernel data structures (tasks, sockets, map contents) and produce output like a synthetic `/proc` file. |

### Security

| Hook | Role |
|---|---|
| **BPF LSM** | Attaches to Linux Security Module hooks. A non-zero (negative errno) return **denies** the operation. This is what makes eBPF an *enforcement* mechanism, not just an observational one — and it is the principal eBPF security hook. |

> **`seccomp` is not an eBPF hook.** This trips people up constantly, so state it plainly: `seccomp-bpf` syscall filters run on the **classic** BPF VM, not the extended one. There is no `BPF_PROG_TYPE_SECCOMP` in mainline. Proposals to move seccomp onto eBPF go back to 2015 and have been raised repeatedly since, but security objections — chiefly that eBPF's much larger attack surface is a poor fit for a mechanism whose entire purpose is to be safely usable by unprivileged processes — have kept them out of the kernel. seccomp shares BPF's *name and heritage*, not its modern runtime.

### Other / structural

- **`struct_ops`** — implement a kernel operations struct in eBPF. Notably: pluggable **TCP congestion control** algorithms, and **`sched_ext`**, which lets an eBPF program act as the CPU scheduler.
- **HID-BPF** — fix up misbehaving USB/Bluetooth input devices without a driver patch.
- **`cgroup_iter`, `kfunc`-driven infrastructure, and a steadily growing list** — the set of hooks is not fixed and expands nearly every release.

---

## 12. Privileges and the operational security model

Loading eBPF is a privileged operation, and the details matter for any threat model.

- **Historically:** `CAP_SYS_ADMIN` — effectively root — for everything.
- **Since 5.8:** capabilities were split so you no longer need full root. **`CAP_BPF`** covers core map and program operations; **`CAP_PERFMON`** adds tracing/profiling; **`CAP_NET_ADMIN`** adds networking program types. Meaningful improvement, but note that `CAP_BPF` + `CAP_PERFMON` together are close to root in practice — an attacker with them can read arbitrary kernel memory.
- **Unprivileged eBPF** (loading with no capabilities at all) existed and was constrained to a small subset, but it proved to be a fertile source of local privilege-escalation vulnerabilities. It is **disabled by default** on essentially all modern distributions via `kernel.unprivileged_bpf_disabled`.
- **BPF token** is a newer delegation mechanism: it allows a privileged process to hand a scoped, filesystem-mediated capability to load specific kinds of BPF objects to a less privileged one — the motivating case being containers that need BPF without granting the container root on the host.
- **Hardening knobs:** `net.core.bpf_jit_harden` enables constant blinding to frustrate JIT-spray attacks; `bpf_jit_enable` controls JIT compilation; `bpf_stats_enabled` exposes per-program runtime accounting.
- **Program signing.** Work on cryptographically signing eBPF programs so the kernel can verify provenance before loading has been in flight in the kernel community. Because this is live, contested, and version-specific, it is documented with sources in [[3 - Secure eBPF Program Supply Chain]] rather than summarised here.

---

## 13. Why it is fast

The performance story has four independent components, and it is worth separating them:

1. **No context switches.** The program runs where the event happens, in kernel context. Compare a userspace agent, which pays a transition (and often a copy) per event.
2. **JIT to native code.** After verification, the bytecode becomes real machine instructions. There is no interpreter in the hot path on modern configurations, and the register model was designed to make JIT output near-optimal.
3. **Early placement.** XDP runs before the kernel allocates an `sk_buff`. Dropping a packet there skips the entire networking stack — the reason eBPF-based DDoS mitigation scales to line rate on commodity hardware.
4. **Filtering at the source.** Even when you *do* ship data to userspace, you ship only what survived in-kernel filtering and aggregation. A histogram computed in a per-CPU map and read once per second replaces a firehose of individual events.

Per-CPU maps deserve a specific mention: for counters and aggregates they eliminate cross-CPU cache-line contention entirely, with userspace summing the per-CPU values at read time. This is the standard pattern for high-rate accounting.

---

## 14. Limitations and honest caveats

A baseline note that only lists strengths is not a baseline note.

- **The verifier is the development experience.** Correct programs get rejected because they are hard to *prove* correct. Fighting the verifier is the dominant cost of writing non-trivial eBPF, and error messages are dense register-state dumps.
- **Restricted execution.** 512-byte stack. No unbounded loops. No sleeping outside explicitly sleepable program types. No arbitrary memory access. No floating point.
- **Constrained API.** Only helpers and kfuncs. Not a general-purpose programming environment, and deliberately so.
- **Kernel-version fragmentation.** CO-RE solved *data layout* portability. It did not solve *feature* portability — which hooks, helpers, kfuncs, and map types exist still varies by version, so real tooling does runtime feature probing (`bpftool feature`, libbpf's probe API).
- **Debugging is awkward.** No debugger attaches to a running eBPF program. You have `bpf_trace_printk`, maps you inspect from outside, `bpftool prog dump xlated/jited`, and the verifier log. Effective, but indirect.
- **Observability gaps are real.** Events *can* be dropped under load (ring buffer full), and there are inherent race conditions when reading userspace memory that may change between the hook firing and the read — the TOCTOU class of problems.
- **eBPF is itself an attack surface.** It is a JIT compiler and an execution engine reachable, under some configurations, by less-than-fully-trusted code — historically a productive source of CVEs. And eBPF used *defensively* becomes a target: an adversary aware of an eBPF monitor can attempt to evade or blind it. Both threads are covered in [[4 - Adversarial Robustness of eBPF-Based Monitoring]].
- **Not a silver bullet.** For sustained heavy computation, complex parsing, or anything needing rich libraries, userspace remains correct. eBPF's sweet spot is *small decisions made very often, close to the event.*

---

## 15. Toolchain and ecosystem

### Development

| Tool | What it is |
|---|---|
| **libbpf** | The canonical C loader library, maintained in-tree. Handles ELF parsing, map creation, CO-RE relocation, and generates **skeletons** — a header giving your userspace program typed access to every map and program. The modern default. |
| **BCC** | Python/Lua/C++ frontend with runtime compilation, plus a large library of ready-made tools (`execsnoop`, `biolatency`, `tcpconnect`, …). Heavier at runtime; increasingly the tools are being ported to libbpf + CO-RE. |
| **bpftrace** | A high-level, awk-like DSL for ad-hoc tracing. One-liners for exploratory work: `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'`. The right first reach for interactive investigation. |
| **bpftool** | Kernel-shipped introspection CLI — list, inspect, dump, load, attach, pin, dump BTF, probe features. Indispensable for debugging. |
| **ebpf-go** (Cilium) | Pure-Go library — no cgo, embeds bytecode into the binary. Dominant in the cloud-native ecosystem. |
| **Aya** | Pure-Rust, for both the eBPF side and the userspace side, no libbpf dependency. |
| **libbpf-rs** | Rust bindings over libbpf, if you prefer the C library's semantics. |
| **LLVM/Clang** | The compiler with the BPF backend. GCC also has a BPF target, though LLVM remains dominant. |

### Notable deployed systems

- **Networking:** **Cilium** (Kubernetes CNI and service mesh, largely replacing iptables/kube-proxy), **Katran** (Meta's L4 load balancer), **Calico** (eBPF dataplane), Cloudflare's L4Drop.
- **Observability:** **Pixie** (auto-instrumented Kubernetes observability), **Parca**/**Pyroscope** (continuous profiling), **Hubble** (network flow visibility), Netflix's flame-graph tooling, **Coroot**, **Grafana Beyla**.
- **Security:** **Falco** (CNCF runtime threat detection), **Tetragon** (Cilium's runtime enforcement), **Tracee** (Aqua), plus commercial EDR increasingly built on eBPF.

### Beyond Linux

- **eBPF for Windows** — Microsoft's implementation, reusing the upstream verifier work (via PREVAIL) and JIT, targeting source and bytecode compatibility.
- **Userspace runtimes** — **uBPF** (C), **rbpf** (Rust), **bpftime** (userspace eBPF with uprobe acceleration). Useful for embedding the eBPF execution model outside a kernel entirely.
- **Standardisation** — [RFC 9669](https://www.rfc-editor.org/rfc/rfc9669.html) fixes the instruction set as an IETF Standards Track document, decoupling the ISA from any one implementation. This is what makes hardware offload and independent implementations viable propositions.

---

## 16. Version-gated features — check before citing

Everything in this table depends on kernel version. **Verify against your target kernel before relying on any row in written work**; features also vary by distribution backport, and this note is a summary, not a source.

| Feature | Introduced (approx.) | Note |
|---|---|---|
| eBPF core (64-bit VM, maps, helpers) | 3.15–3.18 | |
| XDP | 4.8 | |
| BTF | 4.18 | |
| 1M complexity limit / 1M-instruction privileged programs | **5.2 or 5.3 — unresolved** | Commit `c04c0d2b968a`, authored April 2019; sources disagree on the release. Confirm via `git describe --contains` before citing. Unprivileged remains 4096 (`BPF_MAXINSNS`). |
| Bounded loops | 5.3 | Before this, loops were rejected outright |
| BPF trampolines, `fentry`/`fexit` | 5.5 | |
| BPF LSM | 5.7 | |
| BPF ring buffer (`RINGBUF`) | 5.8 | |
| `CAP_BPF` / `CAP_PERFMON` split | 5.8 | |
| Sleepable BPF programs | 5.10 | Initially narrow; expanded since |
| `MAX_TAIL_CALL_CNT` constant corrected 32 → 33 | 5.16 | Effective limit was always 33 |
| `bpf_loop()` helper | 5.17 | |
| Open-coded iterators | 6.4 | |
| `sched_ext` (eBPF CPU schedulers) | 6.12 | |
| Program signing | in flight | See [[3 - Secure eBPF Program Supply Chain]] — do not cite a version from this note |

---

## 17. Glossary

| Term | Meaning |
|---|---|
| **Attach point / hook** | The kernel location an eBPF program binds to; determines when it runs and what context it gets |
| **BTF** | BPF Type Format — compact type/debug information enabling CO-RE, kfuncs, and precise verification |
| **bpffs** | The BPF filesystem, usually `/sys/fs/bpf`, where maps and programs are *pinned* to outlive their creating process |
| **cBPF** | Classic BPF — the original 1992 32-bit, two-register packet-filter VM |
| **CO-RE** | Compile Once, Run Everywhere — BTF-driven load-time relocation for cross-kernel portability |
| **Context** | The per-program-type struct passed in `r1` describing the event |
| **Helper** | A stable, numbered kernel function callable from eBPF |
| **JIT** | Just-In-Time compiler translating verified bytecode into native machine code |
| **kfunc** | A kernel function explicitly exported to eBPF; typed via BTF, **not** ABI-stable |
| **kprobe / uprobe** | Dynamic instrumentation of a kernel / userspace function |
| **Map** | Kernel-resident key/value store: eBPF's state, and its channel to userspace |
| **Perf buffer** | Older per-CPU event-streaming mechanism (`PERF_EVENT_ARRAY`) |
| **Pinning** | Persisting a map or program in bpffs by path |
| **Program type** | The category (`XDP`, `KPROBE`, `LSM`, …) determining allowed hooks, context, helpers, and return semantics |
| **Ring buffer** | Modern shared-across-CPUs event-streaming map (`RINGBUF`) |
| **Skeleton** | libbpf-generated header giving userspace typed handles to a BPF object's maps and programs |
| **Tail call** | Non-returning transfer of control to another eBPF program via `PROG_ARRAY` |
| **tnum** | Tristate number — the verifier's per-bit known-0/known-1/unknown abstraction |
| **Trampoline** | Low-overhead dispatch mechanism behind `fentry`/`fexit` and BPF LSM |
| **Verifier** | The static analyser that proves a program safe before the kernel accepts it |
| **XDP** | eXpress Data Path — the earliest networking hook, in the NIC driver |

---

## 18. Where to go next

**For the capstone specifically:** this note is the *mechanism*. The evaluation of that mechanism — whether the verifier can be trusted, what hardware-backed alternatives exist, how bytecode provenance is established, and how eBPF-based monitors fare against a knowledgeable adversary — is in [[eBPF Research - Index]], with candidate project directions in [[Potential Research Topics]].

**Primary references:**
- [ebpf.io](https://ebpf.io) — the eBPF Foundation's hub; the "What is eBPF?" page is the canonical short introduction
- [docs.ebpf.io](https://docs.ebpf.io) — per-helper, per-map-type, per-program-type reference with version annotations. **The right place to check any specific claim in §16.**
- [docs.kernel.org/bpf](https://docs.kernel.org/bpf/) — in-tree kernel documentation
- [RFC 9669](https://www.rfc-editor.org/rfc/rfc9669.html) — the standardised instruction set
- Brendan Gregg, *BPF Performance Tools* — the practical tracing reference
- Liz Rice, *Learning eBPF* — the best end-to-end introduction in book form
- [nakryiko.com](https://nakryiko.com) — Andrii Nakryiko's blog; the authoritative deep dives on libbpf, CO-RE, BTF, and the ring buffer, written by the person who built much of it
- LWN.net's BPF coverage — for how and why any given feature landed

---

Compiled: 2026-07-25
