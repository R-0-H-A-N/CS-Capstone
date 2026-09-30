---
tags: [ebpf, reading-notes, learning-ebpf, history, kernel]
parent: "[[Learning eBPF - Index]]"
status: done
---
# Ch 1: What Is eBPF, and Why Is It Important?

← Back to [[Learning eBPF - Index]]

*Book pp. 1–13.* This is the conceptual "why" chapter. It covers eBPF's history, the kernel/user-space split, why changing the kernel the traditional ways is hard, and why eBPF suits cloud-native environments.

**One-line definition (book):** eBPF lets you load custom code into the kernel *dynamically*, changing kernel behaviour without modifying or reconfiguring applications. The book names three headline uses: **performance tracing**, **high-performance networking with built-in visibility**, and **detecting and (optionally) preventing malicious activity**.

---

## 1. History: from BPF to eBPF

| Year | Kernel | Milestone |
|---|---|---|
| 1993 | — | **BSD Packet Filter** paper, McCanne & Van Jacobson (Lawrence Berkeley National Lab). It describes a *pseudomachine* running **filters**: programs that accept or reject a packet, written in a 32-bit assembly-like instruction set |
| 1997 | 2.1.75 | BPF arrives in Linux, used by **tcpdump** for efficient packet capture |
| 2005 | — | **kprobes** exist: traps on almost any kernel instruction, used from kernel modules |
| 2012 | 3.5 | **seccomp-bpf**: BPF decides whether to allow or deny a *syscall*. First step beyond packet filtering (Ch 10) |
| 2014 | 3.18 | **eBPF** (see list below). Official "birthday" is **26 Sep 2014**, when the verifier, bpf() syscall and map patches were accepted |
| 2015 | — | eBPF programs can attach to **kprobes**, which transformed tracing. Networking hooks start appearing |
| 2016 | — | Production use: Brendan Gregg at Netflix ("eBPF brings superpowers to Linux"); **Cilium** announced, the first project to replace the entire container datapath with eBPF |
| 2017 | — | Facebook open-sources **Katran** (L4 load balancer). Every packet to facebook.com since 2017 has gone through eBPF/XDP |
| 2018 | — | eBPF becomes a **separate kernel subsystem**, maintained by Daniel Borkmann (Isovalent) and Alexei Starovoitov (Meta), later joined by Andrii Nakryiko. **BTF** introduced for portability (Ch 5) |
| 2020 | — | **LSM BPF**: programs attach to the Linux Security Module interface. Security becomes the third major use case after networking and observability |

**What the 3.18 "extended" rewrite added:**
- An overhauled instruction set built for **64-bit** machines, with a rewritten interpreter
- **Maps**: data structures shared between BPF programs and user space (Ch 2)
- The **`bpf()` syscall**, through which user space talks to eBPF (Ch 4)
- **Helper functions** (Ch 2, Ch 6)
- The **verifier**, which checks that programs are safe to run (Ch 6)

**Growth:**
- More than 300 kernel developers have contributed.
- The program size limit grew from **4,096 instructions to 1 million *verified* instructions**. Tail calls and function calls make the limit effectively irrelevant.
  - ⓘ The 1M limit applies to privileged loaders. Unprivileged programs are still capped at 4,096.

### Naming
- The acronym is now meaningless, and "eBPF" and "BPF" are used interchangeably.
- The kernel says **BPF**: the `bpf()` syscall, `bpf_` helpers, `BPF_PROG_TYPE_*`.
- Outside the kernel community, **eBPF** has stuck: ebpf.io, the eBPF Foundation.

## 2. The Linux kernel (core mental model)
- The **kernel** is the software layer between applications and hardware. Applications run in unprivileged **user space** and can't touch hardware directly.
- They request kernel services through the **system call (syscall) interface**: file I/O, networking, even memory access. The kernel also coordinates concurrent processes.
- Languages and standard libraries hide syscalls, so few developers notice how much the kernel does. `strace` reveals it:
  - `strace -c echo "hello"` shows **111 syscalls** (openat, mmap, newfstatat, close, execve, …).
- **The key insight for observability:** apps depend on the kernel, so *watching the kernel reveals app behaviour*. For example, intercepting the open-file syscall shows every file any app opens.

## 3. Why not just change the kernel?
**Upstreaming a patch is slow:**
- The kernel is about 30M lines of code (a footnote gives 28.8M for 5.12), so you need deep familiarity.
- The community, and ultimately Linus Torvalds, must accept the change as good for everyone. Only **about ⅓ of patches are accepted**, typically after **3–6 months** (Jiang, Adams & German, 2013).
- Releases come every 2–3 months, but **distributions lag by years**. The book's example: RHEL 8.5 (Nov 2021) shipped kernel **4.18** (Aug 2018). Security patches reach users faster.

**Kernel modules avoid upstreaming but are full kernel programming:**
- A crash takes down the whole machine.
- Beyond crashes there's the *security* question: does the module have exploitable bugs, and do we trust its authors not to be malicious? Kernel code can access everything.
- Distributions are slow to adopt new kernels precisely so the code gets **hardened** through wide use.

**eBPF's answer is the verifier.** A program is loaded **only if it's safe**: per the book, it won't crash the machine, won't lock it up in a hard loop, and won't allow data to be compromised.

## 4. eBPF's strengths
- **Dynamic loading.** Load or unload with no reboot. Once attached, a program fires for **every** instance of the event, including events from processes that were **already running**. It gives instant visibility of everything on the machine, including all containers. New functionality doesn't require every other Linux user to accept the change.
- **Performance.**
  - JIT-compiled to native code, and no expensive kernel↔user transition per event.
  - The XDP paper (Høiland-Jørgensen et al., CoNEXT '18) reports XDP routing **2.5×** faster than the normal kernel, and **4.3×** faster than IPVS for load balancing.
  - **In-kernel filtering**: only the relevant subset of events is sent to user space, which was the original point of BPF.

## 5. eBPF in cloud-native environments
- Containers, Kubernetes, ECS and serverless (Lambda, Fargate) all still run on servers with kernels. **All containers on a node share one kernel**, so one eBPF program sees every pod on that node.
- The resulting "superpowers":
  1. **No application changes or reconfiguration** needed.
  2. Programs **observe pre-existing processes** as soon as they are attached.
- **The sidecar model**, where an instrumentation container is injected into each pod by modifying its YAML, has these downsides:
  1. The pod must **restart** for the sidecar to be added.
  2. If the YAML modification fails (e.g., the deployment is mislabelled so the admission controller skips it), the pod is **silently uninstrumented**.
  3. Container readiness ordering is unpredictable, causing slow starts and races. Open Service Mesh apps must tolerate all traffic being dropped until Envoy is ready.
  4. Sidecar networking (e.g., a service mesh) pushes traffic through the kernel stack twice to reach the proxy, **adding latency** (Ch 9).
- **Security argument:** eBPF tools are *harder to sidestep*. An attacker's cryptominer won't have your sidecar injected, but eBPF network policy sees and can drop **all** traffic on the host (Ch 8).

## 6. Summary (book)
eBPF changes kernel behaviour for bespoke tools and policies. It observes any event across all applications, containerized or not, and can be deployed dynamically.

---

## Terms from earlier chapters
None, since this is the first chapter. It assumes general background, defined here briefly:
- **Packet / Ethernet:** a unit of network data. Ethernet is the link-layer framing; the 1993 example reads the EtherType at byte offset 12 to check for IP.
- **Container / Kubernetes pod / node:** a container is an isolated process group sharing the host kernel. A pod is one or more containers scheduled together. A node is the (virtual) machine they run on.
- **Service mesh / Envoy:** a layer that handles service-to-service traffic, commonly through a proxy such as Envoy.

## Errata / inconsistencies in the book
- ⚠️ The text says "using `cat` to echo the word hello" (p. 6), but the command shown is `echo`, not `cat`.
- ⚠️ The chapter says maps are covered in Ch 2 and helpers in Ch 2/Ch 6. Ch 2 itself then defers helper details to Ch 5. It's a cross-reference inconsistency only.

## Research relevance
- **Topic 1, [[1 - Formal Verification of the Verifier and JIT]]:**
  - The chapter's safety pitch rests entirely on the verifier ("won't allow data to be compromised"). That claim is the *trust assumption* the verifier-soundness literature tests.
  - Treat it as the *vendor claim*, not a result.
- **Topic 4, [[4 - Adversarial Robustness of eBPF-Based Monitoring]]:**
  - The "harder for bad actors to sidestep" argument compares eBPF against *sidecars*, not against an attacker who targets the eBPF tooling itself.
  - Node-wide visibility also makes the eBPF agent a single high-value target.
- **History anchor for [[0 - Seminal Papers - Foundational Reading]]:** McCanne & Jacobson (1993) and the XDP paper (CoNEXT '18) are the two peer-reviewed citations in this chapter.
  - ⓘ seccomp-bpf still uses *classic* BPF, not eBPF.
