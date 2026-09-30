---
tags: [ebpf, reading-notes, learning-ebpf, bcc, maps, tail-calls]
parent: "[[Learning eBPF - Index]]"
status: done
---
# Ch 2: eBPF's "Hello World"

← Back to [[Learning eBPF - Index]]

*Book pp. 15–36; code in `chapter2/`.* This chapter gives a first concrete feel for eBPF using the **BCC Python framework**, which is easy for learning but not recommended for distributed production apps (reasons in Ch 5). It builds up in four steps: trace output, then a hash map, then a perf buffer, then tail calls.

---

## 1. BCC "Hello World" (`hello.py`)
```python
from bcc import BPF
program = r"""
int hello(void *ctx) {
    bpf_trace_printk("Hello World!");
    return 0;
}
"""
b = BPF(text=program)                             # BCC compiles + loads the C
syscall = b.get_syscall_fnname("execve")          # arch-specific kernel fn name
b.attach_kprobe(event=syscall, fn_name="hello")   # attach to kprobe
b.trace_print()                                   # loop reading trace output
```
- There are **two halves**:
  - the **eBPF program** (C, runs in the kernel)
  - the **user-space loader** (Python), which compiles, loads, attaches, and reads the output
- **`bpf_trace_printk()`** is a **helper function**: a kernel-provided function eBPF can call to interact with the system. Helpers are one of the features that distinguish *extended* from *classic* BPF.
- **The event is `execve`**, the syscall that runs a new program. Its kernel function name varies by architecture, and `get_syscall_fnname` resolves it.
  - Shell built-ins such as `echo` don't call execve, so they produce no trace.
- **Attachment is via a kprobe.** Footnote: **fentry/fexit** (kernel 5.5+) is a faster alternative to kprobe/kretprobe, covered later in the book.
- **Output format:** `bash-5412 [001] .... 90432.904952: 0: bpf_trace_printk: Hello World`.
  - The command, PID, CPU and timestamp come from the kernel's **tracing infrastructure**, not from eBPF.

### Privileges
- Running as **root** is easiest. An "Operation not permitted" error usually means you're unprivileged.
- **`CAP_BPF`** (kernel **5.8**) allows some operations, such as creating certain map types.
  - Loading **tracing** programs needs **`CAP_PERFMON` + `CAP_BPF`**.
  - Loading **networking** programs needs **`CAP_NET_ADMIN` + `CAP_BPF`**.

### What the example reinforces from Ch 1
- **Dynamic:** no reboot or restart is needed. The program fires immediately for *pre-existing* processes.
- **Zero app changes:** any shell, script or executable that calls execve is seen.

### Trace pipe limitations
- `bpf_trace_printk()` always writes to **`/sys/kernel/debug/tracing/trace_pipe`** (reading it requires root).
- Its drawbacks:
  - The format is inflexible.
  - Output is **strings only**, so structured data is hard to pass.
  - There's **one pipe per machine**, shared by all programs, which gets confusing. Maps are the better way out.

## 2. BPF maps
- **Definition:** a data structure accessible from **both eBPF programs and user space**. Maps are a key eBPF-vs-cBPF difference. "BPF map" and "eBPF map" mean the same thing.
- **Typical uses:**
  1. User space writes **config** for a program to read.
  2. A program stores **state** for another program, or for its own future runs.
  3. A program writes **results/metrics** for user space to present.
- Map types are defined in `uapi/linux/bpf.h`. They are generally **key–value stores**:
  - **Arrays:** the key is always a **4-byte index**.
  - **Hash tables:** the key can be any type.
  - **Specialised types:** FIFO **queue**, FILO **stack**, **LRU**, **longest-prefix match**, **Bloom filter** (a probabilistic "is it present?" check).
  - **Object maps:**
    - **sockmap / devmap** hold sockets and devices for traffic redirection.
    - **Program array** holds eBPF programs, used for tail calls.
    - **Map-of-maps** holds maps.
  - **Per-CPU variants:** separate memory per core.
    - Non-per-CPU maps raise concurrency questions. **Spin-lock support** for some maps arrived in kernel **5.1** (Ch 5).

### Hash table example (`hello-map.py`): execve count per user ID
```c
BPF_HASH(counter_table);                       // BCC macro → hash map (default u64 key/value)
int hello(void *ctx) {
  u64 uid; u64 counter = 0; u64 *p;
  uid = bpf_get_current_uid_gid() & 0xFFFFFFFF; // UID = low 32 bits (GID = high 32)
  p = counter_table.lookup(&uid);               // pointer to value, or 0 if absent
  if (p != 0) { counter = *p; }
  counter++;
  counter_table.update(&uid, &counter);
  return 0;
}
```
- **`counter_table.lookup()` is not valid C.** BCC uses a **C-like dialect** that it rewrites into real C before compiling.
- On the Python side, `b["counter_table"].items()` is polled every 2 s.
- **`sudo ls` counts twice:** execve of `sudo` under UID 501, then execve of `ls` under UID 0.
- Hash vs. array: an array would have worked here because the key is an integer. Either way, **user space must poll**. The alternative is push-style buffers.

### Perf and ring buffer maps (`hello-buffer.py`)
- **Perf buffers** reuse the kernel's existing **perf subsystem**.
- **BPF ring buffers**, their successor, are preferred on kernel **5.8+** (Andrii Nakryiko's blog; BCC `BPF_RINGBUF_OUTPUT` appears in Ch 4).
- **How a ring buffer works:**
  - Memory is arranged in a logical ring with separate **write** and **read** pointers. Each variable-length record has a length header.
  - If the read pointer catches the write pointer, the buffer is empty.
  - If a write would overtake the read pointer, the **data is not written and a drop counter increments**. Reads report that drop counter.
  - The buffer size must be tuned for jitter between reads and writes.
```c
BPF_PERF_OUTPUT(output);
struct data_t { int pid; int uid; char command[16]; char message[12]; };
int hello(void *ctx) {
   struct data_t data = {};
   char message[12] = "Hello World";
   data.pid = bpf_get_current_pid_tgid() >> 32;
   data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
   bpf_get_current_comm(&data.command, sizeof(data.command));   // exe name
   bpf_probe_read_kernel(&data.message, sizeof(data.message), message);
   output.perf_submit(ctx, &data, sizeof(data));
   return 0;
}
```
- **User space:**
  - `b["output"].open_perf_buffer(print_event)` registers a callback.
  - `b.perf_buffer_poll()` loops.
  - `b["output"].event(data)` decodes the struct.
- Each program now gets its **own buffer** instead of the shared trace pipe.
- **Context helpers** (UID, PID, comm) collect *who and what triggered the event*, all inside the kernel with no synchronous switch to user space. This is what makes eBPF valuable for observability.
  - **Which helpers and context are available depends on the program type** (Ch 7).

## 3. Function calls
- **Early eBPF could call only helpers.** The workaround was `static __always_inline`, which pastes a copy of the function body into every caller (Fig. 2-5).
  - Compiler inlining is also why some **kernel** functions can't be kprobed (Ch 7).
- **BPF-to-BPF function calls ("BPF subprograms")** arrived in kernel **4.16 + LLVM 6.0**. **BCC doesn't support them** (they're shown in Ch 3).

## 4. Tail calls (`hello-tail.py`)
- **Definition (ebpf.io):** call and execute another eBPF program, **replacing the execution context**, like `execve()`. **Execution never returns to the caller.**
  - The general motivation is avoiding stack growth, which matters because the **eBPF stack is limited to 512 bytes**.
- **Signature:** `long bpf_tail_call(void *ctx, struct bpf_map *prog_array_map, u32 index)`
  - `ctx` passes the context on.
  - `prog_array_map` is a `BPF_MAP_TYPE_PROG_ARRAY` holding **program file descriptors**.
  - `index` selects the program.
  - It **doesn't return on success**. On failure (e.g., no entry at that index) the caller **continues**.
- **BCC sugar:** `prog_array_map.call(ctx, index)` is rewritten to `bpf_tail_call(...)`.
- **The example:**
  - The main program is attached to the **`sys_enter` raw tracepoint**, which fires on every syscall.
  - Its context is `struct bpf_raw_tracepoint_args`, where **`args[1]` = syscall opcode (number)**.
  - It tail-calls `syscall[opcode]`. If there's no entry, it prints `"Another syscall: %d"`.
  - The tail-call targets:
    - `hello_execve` prints "Executing a program".
    - `hello_timer` handles several timer opcodes (222 = create, 226 = delete).
    - `ignore_opcode` prints nothing, which silences noisy syscalls.
- **User space:**
  - `b.attach_raw_tracepoint(tp="sys_enter", fn_name="hello")`
  - `b.load_func(name, BPF.RAW_TRACEPOINT)` returns an **fd** per tail-call program.
  - `prog_array[ct.c_int(59)] = ct.c_int(exec_fn.fd)`, and so on.
- **Rules:**
  - Tail-call programs must have the **same program type** as the caller.
  - Each one is **a separate eBPF program**.
  - The map can be sparse, and **several indices can point to one program**.
- **Versions:**
  - Tail calls exist since kernel **4.2**.
  - Mixing them with BPF-to-BPF calls was only allowed from **5.10**. It needs JIT support: x86 first, ARM since 6.0.
- **Limits:** up to **33 chained tail calls** × **1M instructions** per program gives lots of room for complex in-kernel logic.
- Further reading: Paul Chaignon's blog on tail-call cost across kernel versions.

## 5. Exercises (short version)
1. Print different messages for odd and even PIDs.
2. Attach `hello-map` to multiple syscalls (`openat`, `write`), with several programs sharing one map.
3. Count **all syscalls per UID** via `sys_enter`.
4. Use BCC's `RAW_TRACEPOINT_PROBE(sys_enter)` macro to auto-attach.
5. Key the hash map by syscall number to get system-wide syscall counts.

---

## Terms from earlier chapters
| Term | From | Short definition |
|---|---|---|
| Kernel / user space | Ch 1 | The privileged layer that manages hardware, vs. the unprivileged layer where apps run |
| System call (syscall) | Ch 1 | The interface user space uses to ask the kernel to act; `execve` runs a program |
| kprobe | Ch 1 | A trap set on (almost) any kernel instruction or function. eBPF could attach to kprobes from 2015 |
| Map | Ch 1 (defined here) | A data structure shared by eBPF programs and user space |
| Helper function | Ch 1 (defined here) | A kernel function an eBPF program may call |
| Verifier | Ch 1 | Load-time checker that only admits safe programs |
| `strace` | Ch 1 | A tool that lists an application's syscalls |
| Dynamic loading | Ch 1 | Programs load/unload at runtime without a reboot and see pre-existing processes |
| eBPF vs. BPF | Ch 1 | The names are now interchangeable; the kernel says "BPF" |

## Errata / inconsistencies in the book
- ⚠️ **Tail-call function name mismatch.** The C defines `hello_execve()`, but the prose and the loader call `hello_exec` (`b.load_func("hello_exec", ...)`). As printed, one of the two must be wrong. The Ch 3 exercise output shows `hello_exec`.
- ⚠️ **PID/TGID footnote (p. 26)** says the *lower* 32 bits of `bpf_get_current_pid_tgid()` are the "thread group ID".
  - ⓘ In kernel terms it's the reverse. The **upper** 32 bits are the TGID, which user space calls the PID; that's why `>> 32` gives the process ID. The **lower** 32 bits are the kernel's per-thread ID.
- ⚠️ The chapter says helpers are covered "in Chapter 5", while Ch 1 pointed to Ch 6. Cross-reference only.
- ⓘ Syscall numbers are **architecture-specific**. 59 = execve and 222/226 = timer_create/delete are x86-64 numbers, so the example needs other numbers on ARM64.

## Research relevance
- **Topic 4, [[4 - Adversarial Robustness of eBPF-Based Monitoring]]:**
  - **Ring-buffer drops.** The book itself notes that writes are silently dropped when the buffer is full (only a counter increments). Flooding events to cause drops is an evasion avenue, and reporting event loss is a methodology point (see [[Running and Testing eBPF - Practical Approaches]]).
  - **Inlining creates blind spots.** Compiler-inlined kernel functions can't be kprobed.
  - ⓘ **Syscall-entry hooks read arguments before the kernel copies them.** That opens a time-of-check/time-of-use (TOCTOU) gap for security tools hooking `execve`/`openat` at entry.
  - **Program arrays** are map entries writable from user space. ⓘ Whoever can update the map can re-route a monitor's tail-call logic.
- **Topic 1, [[1 - Formal Verification of the Verifier and JIT]]:**
  - Tail calls mean **each program is verified separately**, while behaviour emerges from the chain (up to 33).
  - Tail calls combined with BPF-to-BPF calls **need JIT support**. That's a place where verifier and JIT correctness interact.
- **Privilege model:** the `CAP_BPF` / `CAP_PERFMON` / `CAP_NET_ADMIN` split (5.8) is the baseline for "who can load what", and it matters for the supply-chain threat model in [[3 - Secure eBPF Program Supply Chain]].
