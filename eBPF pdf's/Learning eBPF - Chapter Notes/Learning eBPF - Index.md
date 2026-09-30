---
tags: [ebpf, reading-notes, learning-ebpf, book]
related: "[[eBPF (Extended Berkeley Packet Filter)]]"
status: active
---
# Learning eBPF — Chapter Notes (Index)

← Back to [[eBPF (Extended Berkeley Packet Filter)]]

Chapter-by-chapter study notes on **Liz Rice, *Learning eBPF: Programming the Linux Kernel for Enhanced Observability, Networking, and Security* (O'Reilly, 2023)**. Local copy: [[Learning-eBPF - Full book.pdf]]. Example code referenced throughout lives in the book's companion repo `github.com/lizrice/learning-ebpf` (URL as printed in the book; not fetched).

These notes are *descriptive* reading notes, like [[eBPF Fundamentals]]. Evaluation of the claims belongs in [[eBPF Research - Index]]; each chapter note ends with pointers into that research.

## Chapters

| # | Note | Book pages | One-line gist |
|---|---|---|---|
| 1 | [[Ch1 - What Is eBPF and Why Is It Important]] | 1–13 | History (cBPF → eBPF), kernel vs. user space, why eBPF beats kernel patches/modules/sidecars |
| 2 | [[Ch2 - eBPF Hello World]] | 15–36 | First programs with BCC: kprobes, trace pipe, maps, perf/ring buffers, function calls, tail calls |
| 3 | [[Ch3 - Anatomy of an eBPF Program]] | 37–57 | The VM (registers, instructions), C → bytecode → xlated → JIT, bpftool, global variables as maps, BPF-to-BPF calls |
| 4 | [[Ch4 - The bpf() System Call]] | 59–78 | The syscall layer under every loader: BTF/map/program loading, refcounts, pinning, links, perf vs. ring buffer internals |

## How each note is laid out
1. **Summary sections**, following the chapter's own structure.
2. **Terms from earlier chapters**: concepts the chapter relies on, with where they were introduced and a short definition.
3. **Errata / inconsistencies in the book**: flagged with ⚠️ so they aren't cited by mistake.
4. **Research relevance**: ties to the four research themes.

## Markers
- ⚠️: an error or inconsistency in the book itself (same caveat convention as the research notes).
- ⓘ: background from outside the chapter, not stated by the book. **Verify before citing.**
