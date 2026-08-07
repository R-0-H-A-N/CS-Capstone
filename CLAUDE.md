# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An Obsidian vault, not a software project — there is no build, lint, or test tooling here. It holds the research and planning notes for Rohan Kalia's CS capstone project on **eBPF security** (verifier/JIT trust, hardware-assisted isolation, program supply-chain integrity, and adversarial robustness of eBPF-based runtime monitoring). All content is Markdown with Obsidian-flavored wikilinks (`[[Note Name]]`) and YAML frontmatter. Version control (git) is used to track edits to these notes over time via the `obsidian-git` plugin; there is no application code to run.

## Structure

- **`eBPF (Extended Berkeley Packet Filter).md`** — the capstone project home note (problem statement, objectives, scope, architecture, timeline, deliverables, risks). Sections still contain unfilled template placeholders (e.g. `[Insert specific project title here]`) pending a chosen project direction — do not treat bracketed placeholder text as settled content.
- **`eBPF Fundamentals.md`** — a standalone, deliberately *descriptive* primer on eBPF itself (lifecycle, VM/ISA, verifier, maps, helpers/kfuncs, CO-RE, hook taxonomy, toolchain, glossary). It explains the mechanism and takes no position on open safety questions; the research notes are where evaluation happens.
- **`eBPF Research/`** — the literature review, organized as a hub-and-spoke:
  - `eBPF Research - Index.md` is the hub note (MOC). It explains the folder's organization, the quality-tag legend, and links every topic note.
  - `0 - Seminal Papers - Foundational Reading.md` is the citation *ancestry* (what the current literature cites), distinct from the topic notes' *current* literature.
  - `1 - Formal Verification of the Verifier and JIT.md`, `2 - Hardware-Assisted Isolation for eBPF.md`, `3 - Secure eBPF Program Supply Chain.md`, `4 - Adversarial Robustness of eBPF-Based Monitoring.md` — the four thematic source lists, each with numbered entries (`1.1`, `1.2`, ...) referenced elsewhere by that ID rather than re-quoted.
  - `Potential Research Topics.md` — candidate capstone angles synthesized from gaps/tensions across the four topic notes, each pointing back to its source section.
  - `Author Pages to Watch.md` — researcher pages to monitor for unindexed new work.
  - `Misc - Research Materials.md` — an unsorted intake dump (papers, talks, tools) for material not yet triaged into a themed note.

## Conventions to follow when editing notes

- **Frontmatter**: every note has `tags: [...]` and either `parent:` (topic notes, pointing to the Index) or `related:` (top-level notes, pointing to the project home). Match the pattern of sibling notes when adding a new one.
- **Navigation links**: topic notes open with `← Back to [[eBPF Research - Index]]` (or the equivalent parent) immediately under the title.
- **Quality tags**: sources are tagged `[PR]` (peer-reviewed), `[PRIM]` (primary kernel source — patch/LWN/LPC), `[TECH]` (tech report), or `[GREY]` (blog/talk/unreviewed code), per the legend in `eBPF Research - Index.md`. Preserve this tag on any source you move or re-cite.
- **Source IDs**: entries within a topic note are numbered `<section>.<n>` (e.g. `2.7`); these IDs are referenced by other notes, so don't renumber existing entries — append new ones at the next free number.
- **⚠️ caveat flags**: used inline to flag sourcing weaknesses (unconfirmed titles, secondhand citations, weak evaluations). Carry these forward rather than silently dropping them when a source is referenced elsewhere.
- **Wikilinks over duplication**: cross-reference other notes with `[[Note Name]]` (or `[[Note Name#Heading]]`) instead of restating their content.
- **Verified links**: when adding a source URL, note whether it was actually checked (the existing notes record fetch/verification dates, e.g. "fetched and returned HTTP 200 on 2026-07-29") rather than assuming a link is live.
- **Dated changelog**: `eBPF (Extended Berkeley Packet Filter).md` §10 ("Notes & Progress") accumulates a dated log of major changes to the vault — add an entry there for structurally significant additions (new research sections, a chosen project direction), not for minor edits.
