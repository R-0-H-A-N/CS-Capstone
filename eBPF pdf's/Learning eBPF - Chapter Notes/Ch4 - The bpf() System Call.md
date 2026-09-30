---
tags: [ebpf, reading-notes, learning-ebpf, syscall, bpf-syscall, lifecycle]
parent: "[[Learning eBPF - Index]]"
status: done
---
# Ch 4: The bpf() System Call

← Back to [[Learning eBPF - Index]]

*Book pp. 59–78; code in `chapter4/`.* This chapter covers what happens at the **syscall level** when user space loads and manages eBPF. It traces a BCC program with `strace`. Libraries (BCC, libbpf, …) wrap these calls, but their abstractions **map almost directly** onto the commands shown here.

**Key distinction:**
- **User space** uses the `bpf()` syscall.
- **eBPF programs in the kernel never use syscalls**. They access maps through **helper functions**.

```c
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```
- `cmd` is *which* operation: load a program, create a map, update an element, … (the full list is in `linux/bpf.h`).
- `attr` holds that command's parameters, and `size` is the byte length of `attr`.

---

## 1. The example: `hello-buffer-config.py`
This is Ch 2's perf-buffer "Hello World" plus a **config hash map**, so the message can be set per UID.
```c
struct user_msg_t { char message[12]; };
BPF_HASH(config, u32, struct user_msg_t);   // key u32 (UID), value 12-byte msg; BCC default u64/u64 if unspecified
BPF_PERF_OUTPUT(output);
struct data_t { int pid; int uid; char command[16]; char message[12]; };
int hello(void *ctx) {
  ... // pid, uid, comm as in Ch 2
  p = config.lookup(&data.uid);
  if (p != 0) bpf_probe_read_kernel(&data.message, sizeof(data.message), p->message);
  else        bpf_probe_read_kernel(&data.message, sizeof(data.message), message);
  output.perf_submit(ctx, &data, sizeof(data));
  return 0;
}
```
- **Python side:** `b["config"][ct.c_int(0)] = ct.create_string_buffer(b"Hey root!")`, and the same for UID 501. `ctypes` makes the key/value types match the C definitions.
- **Output:**
  - `sudo ls` produces "Hi user 501!" (for sudo), then "Hey root!" (for ls).
  - `sudo -u daemon ls` falls back to "Hello World" because UID 1 has no entry.
- **Tracing it:** `strace -e bpf ./hello-buffer-config.py`

## 2. The syscall sequence

| Step | Syscall | Returns | Notes |
|---|---|---|---|
| 1 | `bpf(BPF_BTF_LOAD, {btf=...})` | fd **3** | Loads a BTF blob. BTF arrived upstream in **5.1** (back-ported by some distros), so older kernels won't show this call |
| 2 | `bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_PERF_EVENT_ARRAY, key_size=4, value_size=4, max_entries=4, map_name="output"})` | fd **4** | Why 4 entries is explained in §5 |
| 3 | `bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_HASH, key_size=4, value_size=12, max_entries=10240, map_name="config", btf_fd=3})` | fd **5** | 10,240 is BCC's default size. `btf_fd` describes the key/value layout, which is what lets bpftool pretty-print |
| 4 | `bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_KPROBE, insn_cnt=44, insns=0x…, license="GPL", prog_name="hello", expected_attach_type=BPF_CGROUP_INET_INGRESS, prog_btf_fd=3})` | fd **6** | **The verifier runs here.** Failure returns a **negative** value |
| 5 | `bpf(BPF_MAP_UPDATE_ELEM, {map_fd=5, key=…, value=…, flags=BPF_ANY})` ×2 | 0 | Writes the two config entries |

**File descriptors (fds).** An fd is the identifier returned when a file or file-like object is opened, used in later syscalls.
- **fds are per-process.** Two processes can have different fds for the same map, or the same fd number for different maps.

**`BPF_PROG_LOAD` fields worth knowing:**
- `prog_type` decides what the program can attach to (Ch 7).
- `insn_cnt` is the **number of bytecode instructions**. `insns` points to them in user memory.
- `license="GPL"` unlocks GPL-only helpers.
- `expected_attach_type` is **only used by some program types**. The `BPF_CGROUP_INET_INGRESS` seen here is meaningless for a kprobe: it's just enum value **0**, the first entry in `bpf_attach_type`.
- `prog_btf_fd` references the BTF blob, the same one the config map uses.

**Other map commands:**
- `BPF_ANY` means "create the key if it doesn't exist".
- `BPF_MAP_LOOKUP_ELEM` reads an element, and `BPF_MAP_DELETE_ELEM` deletes one.
- `BPF_MAP_GET_NEXT_KEY` iterates over keys.
- `bpftool map dump name config` formats values as `{"message": "Hey root!"}` **using the BTF** supplied at `BPF_MAP_CREATE`.

## 3. Object lifetime: reference counts
- **The kernel refcounts programs and maps** and frees them when the count reaches 0. That's why BCC's program and maps vanish when the script exits.
- **What holds a reference to a program:**
  1. **The loader's fd.** It's released when the process exits.
  2. **A pin** in the BPF filesystem (`/sys/fs/bpf`).
     - A pin is **in memory, not on disk**, so it **doesn't survive a reboot**.
     - This is why **bpftool must pin**: its fd disappears when the command exits.
  3. **Attachment to a hook.** Behaviour depends on the program type:
     - **Tracing** types (kprobes, tracepoints) are tied to a **user-space process**. Their reference goes away when that process exits.
     - **Network and cgroup** programs are **not tied to any process** and **stay attached after the loader exits**. Example: `ip link set dev eth0 xdp obj hello.bpf.o sec xdp` leaves the XDP program loaded with no pin at all.
  4. **A BPF link** (§4).
- **What holds a reference to a map:**
  - Each program that uses it.
  - Each user-space fd.
  - A pin: user space can open a pinned map by **path**.
- **`BPF_PROG_BIND_MAP`** covers maps a program never touches, such as metadata stored in a global variable. It binds the map to the program so the map isn't freed when the loader exits.
- Further reading: Alexei Starovoitov, "Lifetime of BPF Objects".

## 4. BPF links
- A link is an **abstraction layer between a program and the event it's attached to**.
- A link can itself be **pinned**, which adds a reference. The loader can then exit and the program stays attached.
- libbpf creates links automatically, which shows up as `bpf(BPF_LINK_CREATE)` (exercise 8).

## 5. Other syscalls involved (optional deep dive in the book)
Traced with `strace -e bpf,perf_event_open,ioctl,ppoll`.

### Attaching to a kprobe (no `bpf()` involved)
1. `perf_event_open({type=0x6, …}) = 7` creates a perf event fd for the kprobe.
   - Type **6** comes from a dynamic PMU: `cat /sys/bus/event_source/devices/kprobe/type` prints `6`.
2. `ioctl(7, PERF_EVENT_IOC_SET_BPF, 6)` attaches program fd 6 to the event fd 7.
3. `ioctl(7, PERF_EVENT_IOC_ENABLE, 0)` turns it on.

### Perf buffer setup: one buffer per CPU
Repeated for CPU `X` = 0..3:
```
perf_event_open({type=PERF_TYPE_SOFTWARE, config=PERF_COUNT_SW_BPF_OUTPUT, ...}, -1, X, -1, PERF_FLAG_FD_CLOEXEC) = Y
ioctl(Y, PERF_EVENT_IOC_ENABLE, 0)
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, ...})
```
- `pid = -1, cpu = X` means "all processes on CPU X".
- **4 repetitions = 4 CPU cores.** That's why `max_entries=4`, and why the type is called *PERF_EVENT_**ARRAY***: it's an array of per-core ring buffers.
- The fds Y are 8–11. User space then waits with `ppoll([fd 8, 9, 10, 11], …)`, which **blocks** until some core's buffer has data.
- **No `bpf()` calls read the data.** It arrives through the perf fds.
- Libraries handle the per-core details for you (Ch 10).

### Ring buffers (`hello-ring-buffer-config.py`)
- They're preferred on kernel **5.8+**. They perform better, and **preserve ordering even across CPU cores**, because there's **one buffer shared by all cores**.

| Perf buffer (BCC) | Ring buffer (BCC) |
|---|---|
| `BPF_PERF_OUTPUT(output);` | `BPF_RINGBUF_OUTPUT(output, 1);` |
| `output.perf_submit(ctx, &data, sizeof(data));` | `output.ringbuf_output(&data, sizeof(data), 0);` |
| `b["output"].open_perf_buffer(print_event)` | `b["output"].open_ring_buffer(print_event)` |
| `b.perf_buffer_poll()` | `b.ring_buffer_poll()` |

- The map is created with `bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_RINGBUF, key_size=0, value_size=0, max_entries=4096, map_name="output"}) = 4`. There are **no per-CPU `perf_event_open`/`ioctl`/`UPDATE_ELEM` calls**.
- **Program loading, the config map, and kprobe attachment are unchanged.**
- **ppoll vs. epoll:**
  - `ppoll` must be passed the whole fd set **on every call**.
  - `epoll` keeps the set in a **kernel object**:
    1. `epoll_create1(EPOLL_CLOEXEC) = 8`
    2. `epoll_ctl(8, EPOLL_CTL_ADD, 4, {EPOLLIN})`
    3. `epoll_pwait(8, …)` blocks until data arrives.
  - BCC uses ppoll for perf buffers and epoll for ring buffers.

## 6. Reading a map from user space (`strace -e bpf bpftool map dump name config`)
**Step 1: find the map.** For each map in the kernel:
- `BPF_MAP_GET_NEXT_ID {start_id}` returns the next map ID.
- `BPF_MAP_GET_FD_BY_ID {map_id}` returns an fd for it.
- `BPF_OBJ_GET_INFO_BY_FD` returns info including the name, which bpftool compares.
- The loop ends when `GET_NEXT_ID` returns `-1 ENOENT`.

**Step 2: read the elements.**
- `BPF_MAP_GET_NEXT_KEY {key=NULL}` returns the first key, written into `next_key`.
- `BPF_MAP_LOOKUP_ELEM {key}` writes the value into `value`, and bpftool prints the pair.
- It repeats until `GET_NEXT_KEY` returns `ENOENT`.

bpftool held this map as fd 3, which differs from the loader's fd. This again shows that fds are per-process.

## 7. Summary (book)
- `BPF_PROG_LOAD` and `BPF_MAP_CREATE` create objects. They're refcounted and freed at 0, and pins and links add references.
- `BPF_MAP_UPDATE_ELEM`, `LOOKUP_ELEM`, `DELETE_ELEM` and `GET_NEXT_KEY` manage map contents.
- **Attachment varies by program type:**
  - **kprobes** use `perf_event_open` + `ioctl`.
  - **cgroups** use `bpf(BPF_PROG_ATTACH)`.
  - **Raw tracepoints** use `bpf(BPF_RAW_TRACEPOINT_OPEN)`.
- Object discovery uses `BPF_MAP_GET_NEXT_ID`, `BPF_MAP_GET_FD_BY_ID` and `BPF_OBJ_GET_INFO_BY_FD`.

## 8. Exercises (short version)
1. Check that `insn_cnt` equals the instruction count in `bpftool prog dump xlated`.
2. Run two instances, so there are two `config` maps, and follow the fds in `strace` of `bpftool map dump`.
3. **Change `config` with `bpftool map update` while the program runs**, and confirm with `sudo -u <user>`.
4. `bpftool prog pin name hello /sys/fs/bpf/hi`, quit the loader, and check that the program is still loaded. Clean up with `rm`.
5. Convert to `RAW_TRACEPOINT_PROBE(sys_enter)`. You'll see `bpf(BPF_RAW_TRACEPOINT_OPEN, {name="sys_enter", prog_fd=6}) = 7`, meaning attachment is a single `bpf()` call.
6. Run `opensnoop` (from the libbpf-tools in BCC). `bpftool link list` shows `perf_event` links, whose prog IDs match `bpftool prog list`. That output also shows `run_time_ns` and `run_cnt`.
7. `bpftool link pin id <N> /sys/fs/bpf/mylink`. The link and program survive `opensnoop` exiting.
8. Strace the Ch 5 libbpf version to see `bpf(BPF_LINK_CREATE)`.

---

## Terms from earlier chapters
| Term | From | Short definition |
|---|---|---|
| Syscall / `strace` | Ch 1 | The user→kernel request interface, and the tool that lists an app's syscalls |
| BCC, `BPF_HASH`, `BPF_PERF_OUTPUT` | Ch 2 | Python framework and its C macros for defining maps |
| Helper functions (`bpf_get_current_uid_gid`, `bpf_probe_read_kernel`, `perf_submit`) | Ch 2 | Kernel functions eBPF calls; programs use these, not syscalls, to reach maps |
| Perf buffer / ring buffer | Ch 2 | Kernel→user streaming maps. The ring buffer (5.8+) is the successor |
| kprobe on `execve` | Ch 1–2 | The hook the example attaches to |
| Raw tracepoint `sys_enter` | Ch 2 | Fires on every syscall (used in exercise 5) |
| Verifier | Ch 1 | Runs during `BPF_PROG_LOAD` (Ch 6) |
| BTF | Ch 1, Ch 3 | Type metadata. It's why bpftool can pretty-print maps (Ch 5) |
| Pinning, `/sys/fs/bpf` | Ch 3 | Seen with `bpftool prog load`; explained here |
| Program ID vs. fd, `bpftool prog/map list` | Ch 3 | IDs are global and assigned at load; fds are per-process handles |
| xlated bytecode | Ch 3 | Post-verifier instructions, the basis for exercise 1 |
| XDP / `ip link … xdp` | Ch 3 | A network program that persists after its loader exits |

## Errata / inconsistencies in the book
- ⚠️ p. 64: "matches the length of the **msg_t** structure". The struct is named **`user_msg_t`**.
- ⚠️ p. 76: says `config` "is the same map that hello-buffer-config.py refers to with file descriptor **4**". Per the book's own Table 4-1, **config is fd 5** (fd 4 is `output`).
- ⚠️ Exercise 4 says "clean up the **link** by removing the pin", but what was pinned is a **program**, not a link.

## Research relevance
- **Topic 3, [[3 - Secure eBPF Program Supply Chain]]:**
  - **`BPF_PROG_LOAD` is the single choke point** where bytecode, license, BTF and program type enter the kernel. It's the natural place for signature enforcement.
  - ⓘ This is where the signed-BPF work cited in the vault (3.1, 3.8) hooks in. See that note rather than restating it here.
  - Note that **BTF is supplied by the loader**: it's user-controlled metadata the kernel trusts for layout.
- **Topic 4, [[4 - Adversarial Robustness of eBPF-Based Monitoring]]:**
  - **Discovery is trivial.** Any sufficiently privileged process can enumerate *every* map with `GET_NEXT_ID` → `GET_FD_BY_ID` → `OBJ_GET_INFO_BY_FD`, then read or write it.
  - Exercise 3 literally demonstrates **rewriting a running program's config map** from outside. That's a direct tampering primitive against eBPF security agents that keep policy in maps.
  - **Lifetime semantics:** network and cgroup programs, pins, and links persist without their loader. That's good for resilient agents, and equally useful to an attacker installing persistent eBPF.
  - **Buffer choice:** per-CPU perf buffers **lose cross-CPU ordering**, while the ring buffer preserves it. Both can drop data under load (Ch 2), which affects event-correlation accuracy in detection tools.
- **Topic 1, [[1 - Formal Verification of the Verifier and JIT]]:**
  - Verification happens inside `BPF_PROG_LOAD`, and failure is just a negative return.
  - Exercise 1 ties `insn_cnt` to the xlated dump, a practical way to observe what the verifier and kernel did to a program.
