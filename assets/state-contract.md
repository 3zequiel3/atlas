# State Contract — atlas owns `state.roadmap`

Rationale and edge cases for the orchestrated hook. **The full algorithm is inline in `SKILL.md` §State integration (orquestado)** — this file explains *why*, not *what*. You never need it to run atlas correctly.

Applies to `.jr-orchestrator-state.json` with `version == 2`.

---

## 1. Schema slice

atlas is the sole writer of the `roadmap` object. It never touches `step`, `owner`, `kb`, `skills`, or `agents`, and it leaves every other top-level key with its value intact — including keys it does not recognize.

```json
{
  "version": 2,
  "roadmap": {
    "created_by": "atlas",
    "source": "CHANGES.md",
    "changes": [
      { "id": "C-01", "name": "foundation-setup" },
      { "id": "C-02", "name": "core-models" }
    ],
    "preserved_checkboxes": 2,
    "eliminated_with_progress": ["ingredients"]
  }
}
```

**Why `changes` carries both `id` and `name`.** The id is positional: it is reassigned on every regeneration, so a consumer that persists `"C-05"` across two runs may be pointing at a different change. The name is the stable identity. Shipping both lets a consumer render the roadmap in order *and* track a change across regenerations. Same reasoning applies to `eliminated_with_progress`, which carries names only — an id there would name something that still exists.

**Why `preserved_checkboxes` and `eliminated_with_progress` are mandatory.** They are the only machine-readable signal that a regeneration touched existing progress. Without them the orchestrator reads a clean roadmap and cannot tell that work was dropped.

**Thin index rationale.** Per-change scope, governance, and dependencies live only in `CHANGES.md`. No duplication, no drift.

---

## 2. Write semantics

The mechanism is parse → replace the `roadmap` value → serialize → write. That is a JSON round-trip, so byte-level identity of the file is **not** preserved: indentation, key order and number formatting may normalize. What must be preserved is **semantic**: every other key keeps its value, no key is dropped, no array is truncated or summarized. If `state.kb.files` holds 40 paths, all 40 come back.

**Re-read before write.** The state file has a second writer. atlas reads `state.kb` at the start of a run and writes `state.roadmap` at the end — the orchestrator may have advanced `step` in between. Re-read immediately before writing and apply the change to that fresh copy, never to the version read at the start.

**Never create.** The orchestrator (jr-orchestrator) is the sole creator. An absent state file is a no-op, not an error. This is what keeps atlas usable as a public standalone skill: forcing a state file on every invocation would pollute non-orchestrated repos.

**Never advance `step`.** The orchestrator owns step advancement. atlas signals completion by writing `state.roadmap`; the orchestrator reads it to decide when to advance.

**The hook is not gated on the KB source.** It fires on `version == 2` alone, even when `state.kb.files` was empty and the KB was read from disk instead. Gating it on the orchestrated *input* path would leave the orchestrator waiting forever for a signal atlas decided not to send.

---

## 3. Orchestrated input

The rules are in `SKILL.md` §Input — qué leer de la KB (Orquestado) and §Pre-checks obligatorios. Two points worth the extra words:

**Pre-check 2 runs in orchestrated mode.** It is the only content gate. Skipping it lets a run whose `state.kb.files` contains, say, only a vision document proceed to emit a full dependency graph, critical path and migration ordering derived from no data model at all — fabrication presented as canonical.

**`state.kb.discovery` is a hint, never a source of truth.** A wrong `system_type` degrades naming quality; it must never change scope correctness. The files are the truth.

---

## 4. Standalone invariant

With no `.jr-orchestrator-state.json`, everything else is identical: same pre-checks, same KB reads, same `CHANGES.md` format, same user output. The hook never fires, and no state file is created.
