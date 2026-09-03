# State Contract — atlas owns `state.roadmap`

Reference: C-13a frozen contract (`.jr-orchestrator-state.json` v2, §4.2 I/O matrix).

---

## 1. Schema slice

atlas is the **sole writer** of the `roadmap` object inside `.jr-orchestrator-state.json`.
It NEVER touches `step`, `owner`, `kb`, `skills`, or `agents`.

```json
{
  "version": 2,
  "roadmap": {
    "created_by": "atlas",
    "source": "CHANGES.md",
    "changes": ["C-01", "C-02", "C-03"],
    "preserved_checkboxes": 2,
    "eliminated_with_progress": []
  }
}
```

Fields:

| Field | Type | Description |
|-------|------|-------------|
| `created_by` | string | Always `"atlas"` — identifies the writer. |
| `source` | string | Path of the generated index — always `"CHANGES.md"`. |
| `changes` | string[] | `C-NN` IDs generated, in order. Lets downstream phases know the roadmap without re-parsing. |
| `preserved_checkboxes` | number | How many `[x]` checkboxes were preserved from a prior CHANGES.md (0 if first run). |
| `eliminated_with_progress` | string[] | `C-NN` IDs that had `[x]` in the old file but were removed in the new one. Empty array if none. |

**Thin index rationale**: per-change scope, governance, and dependencies live only in `CHANGES.md` (single source of truth). No duplication, no drift.

---

## 2. Conditional write algorithm

Run this hook **after** `CHANGES.md` is fully written and the user output is ready, before returning.

```
1. Check project root for `.jr-orchestrator-state.json`
   ├─ ABSENT → no-op. Do NOT create the file. Done.
   ├─ PRESENT, but file is empty or JSON is malformed →
   │    surface one-line note: "⚠ State file could not be parsed — skipping state update."
   │    Continue. Done.
   ├─ PRESENT, version != 2 →
   │    surface one-line note: "⚠ State file is not v2 — skipping state update."
   │    Continue. Done.
   └─ PRESENT, version == 2 → update ONLY state.roadmap:
        • Set created_by: "atlas"
        • Set source: "CHANGES.md"
        • Set changes: list of C-NN IDs generated, in order
        • Set preserved_checkboxes: count of [x] restored (0 on first run)
        • Set eliminated_with_progress: list of C-NN IDs that had [x] and were removed
        Write the file back. Done.
```

**Why never create**: the orchestrator (jr-orchestrator) is the sole creator of the state file. atlas updating-only avoids two writers racing to create it.

**Why strictly conditional**: atlas is a public standalone skill. Forcing a state file on every invocation would pollute non-orchestrated repos.

**Why never touch `step`**: the orchestrator owns `step` advancement. atlas signals completion only by writing `state.roadmap`; the orchestrator reads it to decide when to advance.

---

## 3. Orchestrated input rules

When running orchestrated (`.jr-orchestrator-state.json` present and `version == 2`):

### 3.1 Reading KB files

- **Primary**: use `state.kb.files` as the authoritative list of `knowledge-base/*.md` paths.
- **Disk existence check (mandatory)**: before reading each file in `state.kb.files`, verify it exists on disk. If a file is absent, emit `⚠ archivo ausente: {path}` and continue with the files that do exist. Never silently skip or fail on a missing file.
- **Fallback**: if `state.kb.files` is empty or absent → fall back to standalone read path (glob `knowledge-base/` on disk and run all three standard pre-checks).

### 3.2 Using `state.kb.discovery`

- Read `state.kb.discovery` as **context** for naming/grouping changes (hint only):
  - `needs_infra: true` → expect a `foundation-setup` change (C-01).
  - `system_type` → seed FASE names (e.g. `web_app` → FASE 1: Infraestructura + Auth).
  - `domain` → infer domain-specific changes.
  - `scale` → infer RBAC or multi-tenancy changes.
- **Discovery is a hint only**: `state.kb.files` remain the source of truth. A wrong discovery field degrades naming quality but does NOT corrupt scope correctness.

### 3.3 Pre-checks when orchestrated

When `state.kb.files` is used as file list:
- **Skip** the `knowledge-base/` directory existence check (files are resolved from state).
- **Run** disk existence check on each listed file (§3.1).
- **Run** the `openspec/` check normally (unless user explicitly opts out — see SKILL.md §Pre-checks).

When falling back to disk glob:
- Run all three standard pre-checks unchanged (see SKILL.md §Pre-checks obligatorios).

### 3.4 Standalone invariant

With no `.jr-orchestrator-state.json`:
- Three standard pre-checks, KB reads, frozen `CHANGES.md` format, and user output are identical.
- The orchestrated hook (§2) and `state.kb` read (§3.1–3.2) never fire.
- No state file is created.
