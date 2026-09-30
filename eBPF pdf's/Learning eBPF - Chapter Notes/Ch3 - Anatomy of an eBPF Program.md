---
tags: [ebpf, reading-notes, learning-ebpf, bytecode, jit, bpftool, xdp]
parent: "[[Learning eBPF - Index]]"
status: done
---
# Ch 3: Anatomy of an eBPF Program

← Back to [[Learning eBPF - Index]]

*Book pp. 37–57; code in `chapter3/`.* This chapter drops BCC for a program written **entirely in C** using **libbpf** (explained in Ch 5), so you can see what BCC hid. It follows one program through the whole lifecycle.

**Pipeline:** C (or Rust) source → Clang/LLVM → **eBPF bytecode** (ELF `.o`) → loaded into the kernel → **verifier** → **JIT-compiled** (or interpreted) → native machine code → **attached** to an event → runs each time the event fires.

- A program is a sequence of **bytecode instructions**. You could hand-write it like assembly, but almost all eBPF is written in C. Rust is increasingly used because its compiler can target eBPF.
- Conceptually, the bytecode runs in an **eBPF virtual machine** inside the kernel.

---

## 1. The eBPF virtual machine
- **Interpreter → JIT.**
  - Early eBPF **interpreted** the bytecode on every run.
  - It has mostly been replaced by **JIT** compilation, done once at load time. The reasons are **performance** and **avoiding Spectre-related vulnerabilities in the interpreter**.
  - JIT needs `CONFIG_BPF_JIT` and can be toggled at runtime with the sysctl `net.core.bpf_jit_enable`. Most architectures support it.
- The instruction set and register model were **designed to map closely onto real CPUs**, which keeps compiling or interpreting simple.

### Registers
| Register | Role |
|---|---|
| **R0** | Return value: from the program on exit, and from function/helper calls |
| **R1** | Holds the **context** argument at program start; also the 1st function argument |
| **R1–R5** | Function-call arguments (only as many as needed) |
| R6–R9 | General purpose |
| **R10** | **Stack frame pointer: read-only** |

- They are implemented in software and listed as `BPF_REG_0` to `BPF_REG_10` in `include/uapi/linux/bpf.h`.

### Instructions: `struct bpf_insn`
```c
struct bpf_insn {
    __u8  code;       /* opcode */
    __u8  dst_reg:4;  /* dest register */
    __u8  src_reg:4;  /* source register */
    __s16 off;        /* signed offset */
    __s32 imm;        /* signed immediate constant */
};
```
- An instruction is normally **8 bytes (64 bits)**: an opcode, up to two registers, an optional offset and/or immediate value.
- **Wide instruction encoding (16 bytes)** is used when loading a 64-bit value into a register. It takes up two offset slots.
- Footnote: some opcodes are modified by other fields. For example, the **atomic instructions (kernel 5.12)** put ADD/AND/OR/XOR in `imm`.
- **Opcode categories:**
  - **load** into a register (from an immediate, memory, or another register)
  - **store** a register to memory
  - **arithmetic**
  - **conditional jump**
- The kernel stores a loaded program as an array of `bpf_insn`. That array is **what the verifier checks** (Ch 6).
- References: the IO Visor "Unofficial eBPF spec", the Cilium *BPF and XDP Reference Guide*, and the kernel instruction-set docs. The eBPF Foundation is working on an OS-independent standard.

## 2. The example: XDP "Hello World" (`hello.bpf.c`)
```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
int counter = 0;                        // global variable
SEC("xdp")                              // ELF section → XDP program type
int hello(void *ctx) {                  // program name = function name
    bpf_printk("Hello World %d", counter);
    counter++;
    return XDP_PASS;                    // verdict: process packet normally
}
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```
- **Naming convention:** eBPF source files end in `.bpf.c` so they can be told apart from user-space C.
- **`SEC()`** puts code or data into a named **ELF section**. The section name tells the loader the program type (Ch 5).
- **The license is effectively mandatory:**
  - Some helpers are **"GPL only"**, and the **verifier rejects** a program whose license doesn't allow the helpers it uses.
  - Some program types must be GPL-compatible regardless, **BPF LSM** among them (Ch 9).
- **`bpf_printk` (libbpf) vs. `bpf_trace_printk` (BCC):** both wrap the kernel helper `bpf_trace_printk()` (Andrii Nakryiko's blog).
- **XDP (eXpress Data Path):**
  - It fires when a packet arrives **inbound** on a physical or virtual interface.
  - The program can inspect or modify the packet and return a **verdict**: pass, drop, or redirect.
  - Some NICs can **offload** XDP to run *on the card*, before the packet reaches the CPU. Uses include DDoS protection, firewalls and load balancers (Ch 8).

## 3. Compiling and inspecting the object file
```make
clang -target bpf -I/usr/include/$(shell uname -m)-linux-gnu -g -O2 -c hello.bpf.c -o hello.bpf.o
```
- **`-g`** is optional here. It adds debug info **and BTF**, which is needed for **CO-RE** (Ch 5) and for pretty-printing maps.
- `file hello.bpf.o` reports: *ELF 64-bit LSB relocatable, eBPF, …, with debug_info*.
- `llvm-objdump -S hello.bpf.o` disassembles `section xdp`, showing `<hello>` interleaved with the source:
  - `bpf_printk` → 5 instructions, `counter++` → 3, `return XDP_PASS` → 2.
  - **Offsets count instructions**, not bytes. The jumps 0→2 and 3→5 happen because `r6 = 0 ll` and `r1 = 0 ll` are **wide 16-byte loads**.
  - **Worked decoding:** `b7 02 00 00 0f 00 00 00` is opcode `0xb7` (meaning `dst = imm`), destination register 2, immediate 0x0f = 15, so **`r2 = 15`**.
  - `r0 = 2` sets the return value: **`XDP_PASS` = 2**. `95` = `exit`, and `85` = `call`.
  - The book notes the tools use "BPF" and "eBPF" interchangeably with no rhyme or reason.

## 4. Loading and inspecting with `bpftool`
- **Load (root needed):** `bpftool prog load hello.bpf.o /sys/fs/bpf/hello`. This loads the program and **pins** it.
  - Pinning is optional in general but **mandatory for bpftool** (why: Ch 4, *BPF Program and Map References*).
  - Success prints nothing. Check with `ls /sys/fs/bpf`.
- **List:** `bpftool prog list`.
- **Details:** `bpftool prog show id 540 --pretty` gives these fields:

| Field | Meaning |
|---|---|
| `id` | Assigned at load; **changes each load**; unique |
| `type` | `xdp`: which events it can attach to (Ch 7) |
| `name` | Function name |
| `tag` | **SHA hash of the instructions**; stable across reloads |
| `gpl_compatible` | License result |
| `loaded_at`, `uid` | When it was loaded, and by whom (0 = root) |
| `bytes_xlated` | Size of the **translated** bytecode (after the verifier, possibly kernel-modified) |
| `jited`, `bytes_jited` | JIT-compiled?, and native code size |
| `bytes_memlock` | 4096 B reserved that won't be paged out |
| `map_ids` | **165, 166**, even though the source declares no map (see §6) |
| `btf_id` | BTF blob (only present with `-g`) |

- **Identifying a program:** bpftool accepts an **id, name, tag or pinned path**. **Only the id and pinned path are guaranteed unique.** Names repeat, and multiple instances can share a tag.

### Translated bytecode: `bpftool prog dump xlated name hello`
- It's almost identical to the objdump output, but with references resolved: `r6 = map[id:165][0]+0`, `call bpf_trace_printk#-78032`.
- **xlated = post-verifier and possibly rewritten by the kernel** (the reasons come later in the book).

### JIT output: `bpftool prog dump jited name hello`
- The book shows **ARM64** assembly (`stp`/`ldp` prologue and epilogue, `movk`, `blr`, `ret`). The native symbol is `bpf_prog_<tag>_<name>`.
- "Error: No libbfd support" means you need to build bpftool from source (github.com/libbpf/bpftool).

## 5. Attach, observe, detach, unload
- **Loaded ≠ running.** The program must be **attached**, and the program type must match the event type (Ch 7).
- **Attach:** `bpftool net attach xdp id 540 dev eth0`. You can identify the program by id, a unique name, or tag.
  - bpftool can't attach every type. It recently gained auto-attach for k(ret)probes, u(ret)probes and tracepoints.
- **Check:**
  - `bpftool net list` shows the network hook categories **xdp**, **tc**, **flow_dissector** (Ch 7).
  - `ip link` shows `prog/xdp id 540 … jited` on eth0 (lo = loopback, eth0 = outside world).
- **Output:** `cat /sys/kernel/debug/tracing/trace_pipe`, or `bpftool prog tracelog`.
  - Lines read `<idle>-0 … Hello World 4531`: **no process context**. The packet has only just landed in memory, and no user process is associated with it yet. Contrast the Ch 2 syscall kprobe, which showed the command and PID.
- **Detach:** `bpftool net detach xdp dev eth0`. The program **stays loaded**.
- **Unload:** there's no inverse of `prog load`. Instead, **`rm /sys/fs/bpf/hello`** deletes the pin, and the program disappears.

## 6. Global variables are maps
- Global variable support was **added in 2019**. Before that, programmers wrote maps by hand.
- The compiler and loader put globals and static data into maps. `bpftool map list` shows:
  - **`hello.bss`**: an array map, key 4 B, value 4 B, 1 entry, holding `counter`. `.bss` stands for "block started by symbol", the usual section for globals.
  - **`hello.rodata`**: an array map, value 15 B, **`frozen`**, holding read-only data: `hello.____fmt = "Hello World %d"`.
- `bpftool map dump name hello.bss` gives `"counter": 11127` **only with BTF**. Without `-g` you see raw bytes: `19 01 00 00`, which is little-endian for **281**.
- This explains the "invisible" `map_ids`.

## 7. BPF-to-BPF calls (`hello-func.bpf.c`)
```c
static __attribute((noinline)) int get_opcode(struct bpf_raw_tracepoint_args *ctx) {
    return ctx->args[1];
}
SEC("raw_tp")
int hello(struct bpf_raw_tracepoint_args *ctx) {
    int opcode = get_opcode(ctx);
    bpf_printk("Syscall: %d", opcode);
    return 0;
}
```
- `noinline` forces a real call so it's visible. Normally you'd let the compiler decide.
- In the xlated dump:
  - `0: (85) call pc+7#bpf_prog_…_get_opcode` is **0x85 = call** with a **relative** jump of +7, landing at offset 8. `get_opcode` itself is at offsets 8–9: `r0 = *(u64 *)(r1 +8)`, then `exit`.
  - The result returns in **R0**, and `r3 = r0` passes it to printk.
- Each call **pushes the caller's state onto the stack**. The **stack is 512 bytes**, so **calls can't nest deeply**.
- Further reading: Jakub Sitnicki (Cloudflare), "Assembly within! BPF tail calls on x86 and ARM".

## 8. Summary (book)
- You saw C → bytecode → machine code, and bpftool for inspecting programs and maps and attaching to XDP.
- Different program types fire on different events: **XDP** on packet arrival, **kprobe/tracepoint** on reaching points in kernel code (more in Ch 7).
- Maps implement globals, and BPF-to-BPF calls exist. The next chapter looks at the syscall level.

## 9. Exercises (short version)
1. Attach and detach via `ip link set dev eth0 xdp obj hello.bpf.o sec xdp` / `xdp off`.
2. Inspect Ch 2 BCC programs with bpftool while they run. The output gains a `pids` field showing the owning process.
3. `hello-tail.py`: **each tail-call target is listed as a separate program**. Contrast BPF-to-BPF calls, which live inside one program.
4. **Warning:** returning **0 from XDP = `XDP_ABORTED`**, which **drops every packet** (and your SSH session). Try it in a container on a veth interface instead (`lizrice/lb-from-scratch`).

---

## Terms from earlier chapters
| Term | From | Short definition |
|---|---|---|
| BCC | Ch 2 | A Python framework that compiles and loads eBPF C at runtime, hiding what this chapter exposes |
| eBPF vs. BPF | Ch 1 | Interchangeable names |
| Verifier | Ch 1 | Load-time safety checker (detail in Ch 6) |
| JIT | Ch 1 (mentioned) | Compiles bytecode to native code once, at load time |
| BTF | Ch 1 (mentioned) | BPF Type Format: type metadata that enables portability and pretty-printing (Ch 5) |
| Map | Ch 2 | A kernel data structure shared by programs and user space |
| Helper function | Ch 2 | A kernel function callable from eBPF (e.g., `bpf_trace_printk`) |
| `trace_pipe` | Ch 2 | `/sys/kernel/debug/tracing/trace_pipe`, where printk-style output goes |
| kprobe | Ch 1–2 | A dynamic hook on a kernel function. Syscall kprobes carry process context |
| Raw tracepoint / `sys_enter` | Ch 2 | A stable kernel instrumentation point. `sys_enter` fires on every syscall, and `args[1]` is the syscall number |
| Tail call | Ch 2 | Jumps to another program through a program-array map, with no return |
| Context (`ctx`) | Ch 2 | The program's input argument; its type depends on the program type |
| 512-byte stack | Ch 2 | The eBPF stack limit |
| Root / `CAP_BPF` | Ch 2 | Privileges needed to load programs (the reason bpftool needs sudo) |

## Errata / inconsistencies in the book
- ⚠️ The JIT section says the program is "**108 bytes**" long, but the `bpftool` output it describes shows `bytes_jited: 148`. The 108 comes from a later reload (p. 53).
- ⚠️ The `ip link` output shows tag `9d0e949f…`, but the program was earlier listed as `d35b94b4…`. The screenshots come from different sessions (the p. 53 reload also shows `9d0e…`).
- ⚠️ The book says the `.rodata` value holds "**12 bytes**", but `"Hello World %d\0"` is **15 bytes**, which matches `value 15B`.
- ⚠️ The objdump source comment line `bpf_printk("Hello World %d", counter");` has a stray quote. This is a typo in the book.

## Research relevance
- **Topic 1, [[1 - Formal Verification of the Verifier and JIT]]:**
  - There are **three distinct artifacts**: compiled bytecode, **xlated** bytecode (post-verifier *and rewritten by the kernel*), and **JITed** native code.
  - Every hand-off is a trust boundary. The verifier reasons about one form, the kernel rewrites it, and the JIT translates it, which is exactly the gap JIT-correctness work targets.
  - The license/GPL-helper check also shows the verifier enforces **policy**, not only memory safety.
- **Topic 2, [[2 - Hardware-Assisted Isolation for eBPF]]:**
  - The book explicitly ties retiring the interpreter to **Spectre**.
  - **XDP NIC offload** moves eBPF execution onto separate hardware, which is its own trust boundary.
- **Topic 3, [[3 - Secure eBPF Program Supply Chain]]:**
  - The **tag** is a hash of the instructions, **not a signature**, and it isn't unique.
  - ⓘ The kernel computes it as a truncated SHA-1 with map references blanked. It identifies, but doesn't authenticate.
- **Topic 4, [[4 - Adversarial Robustness of eBPF-Based Monitoring]]:**
  - XDP has **no process attribution** (`<idle>-0`).
  - Pinned programs **outlive their loader**, and anyone with BPF privileges can **detach or unload** them. That's a tampering surface for eBPF security agents.
  - The **512-byte stack** limits how complex in-kernel detection logic can be.
