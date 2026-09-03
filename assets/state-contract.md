# State Contract — atlas owns `state.roadmap`

Detailed reference for the orchestrated hook. **The critical path is already inline in `SKILL.md` §State integration (orquestado)** — read this only for the schema rationale and the edge cases.

Applies to `.jr-orchestrator-state.json` with `version == 2`.

---

## 1. Schema slice

atlas is the **sole writer** of the `roadmap` object. It never touches `step`, `owner`, `kb`, `skills`, or `agents`, and it preserves every other top-level key byte-for-byte, including keys it does not recognize.

```json
{
  "version": 2,
  "roadmap": {
    "created_by": "atlas",
    "source": "CHANGES.md",
    "changes": ["C-01", "C-02", "C-03"],
    "preserved_checkboxes": 2,
    "eliminated_with_progress": ["ingredients"]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `created_by` | string | Always `"atlas"` — identifies the writer. |
| `source` | string | Path of the generated index — always `"CHANGES.md"`. |
| `changes` | string[] | `C-NN` IDs generated, in order. Lets downstream phases know the roadmap without re-parsing. |
| `preserved_checkboxes` | number | How many completed changes were carried over from a prior `CHANGES.md` (0 on first run). |
| `eliminated_with_progress` | string[] | **Kebab-case names** of changes that were completed in the old file and no longer exist in the new one. `[]` if none. Names, not IDs: IDs are reassigned on every regeneration, so an ID here would point at a different change. |

**Thin index rationale**: per-change scope, governance, and dependencies live only in `CHANGES.md` (single source of truth). No duplication, no drift.

**Why `preserved_checkboxes` and `eliminated_with_progress` are mandatory**: they are the only machine-readable signal that a regeneration touched existing progress. Without them the orchestrator reads a clean roadmap and cannot tell that work was dropped.

---

## 2. Write algorithm — edge cases

The happy path is in `SKILL.md`. What it does not spell out:

**Re-read before write.** The state file has a second writer. atlas reads `state.kb` at the *start* of the run and writes `state.roadmap` at the *end* — the orchestrator may have advanced `step` in between. Re-read the file immediately before writing, and apply the `roadmap` change to that fresh copy, never to the version read at the start.

**Preserve unknown keys.** "Write the file back" means: parse, replace the `roadmap` value, re-serialize. Any top-level key not listed in §1 — present or future — must survive unchanged. Do not normalize, reorder, or drop fields you did not model. If `state.kb.files` is a long array, it is preserved verbatim; never summarize it back into the file.

**Never create.** The orchestrator (jr-orchestrator) is the sole creator. An absent state file is a no-op, not an error, and not an invitation to write one. This is what keeps atlas usable as a public standalone skill: forcing a state file on every invocation would pollute non-orchestrated repos.

**Never advance `step`.** The orchestrator owns step advancement. atlas signals completion by writing `state.roadmap`; the orchestrator reads it to decide when to advance.

**Foreign `created_by`.** If `roadmap.created_by` exists and is not `"atlas"`, another tool owns this slice. Overwrite it and note the takeover in the closing block — do not fail, but do not do it silently either.

---

## 3. Orchestrated input rules

### 3.1 Reading KB files

- **Primary**: `state.kb.files` is the authoritative list of KB paths.
- **Disk existence check (mandatory)**: verify each file exists before reading it. On a miss, emit `⚠ nodo ausente: {path} — continuando sin él` and continue with the rest. Never silently skip.
- **Fallback**: if `state.kb.files` is empty or absent, fall back to the standalone read path (glob `knowledge-base/`).

### 3.2 Pre-checks when orchestrated

All three pre-checks from `SKILL.md` §Pre-checks obligatorios apply. Only the resolution changes:

| Pre-check | Orchestrated behavior |
|-----------|----------------------|
| 1 — KB accesible | Satisfied by a non-empty `state.kb.files`. Skip the `knowledge-base/` directory check. |
| 2 — `04` y `08` presentes | **Runs.** Evaluate against the basenames in `state.kb.files`. This is the only content gate; skipping it lets a run proceed with no data model and fabricate the entire dependency graph. |
| 3 — `openspec/` | Runs unchanged, unless the user opted out. |

### 3.3 Using `state.kb.discovery`

Hint only, for naming and grouping — never a source of truth. A wrong discovery field degrades naming quality; it must not change scope correctness.

- `needs_infra: false` → the project already has infrastructure; `C-01` is the first domain change instead of `foundation-setup`.
- `system_type` → seeds FASE names and the agent role labels of the plan table.
- `domain` → domain-specific changes.
- `scale` → RBAC or multi-tenancy changes.

### 3.4 Standalone invariant

With no `.jr-orchestrator-state.json`, everything else is byte-identical: same pre-checks, same KB reads, same `CHANGES.md` format, same user output. The hook in §2 and the `state.kb` reads in §3.1–3.3 never fire, and no state file is created.
