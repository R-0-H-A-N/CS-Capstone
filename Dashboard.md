---
tags: [dashboard, moc]
related: "[[eBPF (Extended Berkeley Packet Filter)]]"
---

# Vault Dashboard

← Back to [[eBPF (Extended Berkeley Packet Filter)]]

Live view of the whole vault, powered by Dataview. Every section below is a query against the vault's frontmatter and file metadata — it updates automatically as notes change. The only upkeep required: set `status:` (`active` / `done` / `backlog`) and, for the numbered research notes, `source_count:` in a note's frontmatter when its state changes.

---

## Still open

Notes not yet marked `done`, and every unchecked task in the vault.

```dataview
TABLE status AS "Status"
FROM ""
WHERE status AND status != "done"
SORT status ASC, file.name ASC
```

```dataview
TASK
FROM ""
WHERE !completed
GROUP BY file.link
```

---

## Research corpus

Source counts pulled directly from each topic note's frontmatter.

```dataview
TABLE source_count AS "Sources", status AS "Status"
FROM "eBPF Research"
WHERE source_count
SORT file.name ASC
```

---

## Recently touched

```dataview
TABLE file.mtime AS "Last modified", status AS "Status"
FROM ""
WHERE file.name != "CLAUDE"
SORT file.mtime DESC
LIMIT 10
```

---

## Everything, by folder

```dataview
TABLE status AS "Status", tags AS "Tags"
FROM ""
WHERE file.name != "Dashboard" AND file.name != "CLAUDE"
SORT file.folder ASC, file.name ASC
```

---

## By tag

```dataview
TABLE rows.file.link AS "Notes"
FROM ""
WHERE tags
FLATTEN tags AS tag
GROUP BY tag
```

---

*Live Dataview dashboard — open in Obsidian to see rendered results.*
