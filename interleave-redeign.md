# Interleaves — Redesign Design Document
*Captured June 2026 — updated during build*

---

## Object Model (as implemented, `interleaves_v2` DB)

```js
Score {
  id, name, createdAt, order,
  pages: [{ pageId, src, annotation }],  // annotation field unused (see Annotation Model below)
  hasPassages,  // informational: has ≥1 passage ever been saved from this score
  inPool        // behavioral: included in practice shuffle? user-toggleable
}

Passage {
  id, sourceId,          // → Score.id
  name, createdAt, order,
  tier, weight, sessions,
  compasSettings, measures,
  pages: [{ pageId, practiceAnnotation }]  // pageId matches a Score page
}
```

Two stores: `scores`, `passages`. `sessions` store deferred until history/SM-2 needed.

### Score
A source document. Practiceable directly (page-by-page) until it has passages.
- One PDF (imports as one Score, all pages intact) or one image (single-page Score)
- Owns the actual page images (`src`)
- `inPool`: controls shuffle inclusion, independent of `hasPassages` — user-toggleable via 🔀/⏸ badge on the sidebar header. Defaults to `true`, auto-flips to `false` the first time a passage is saved from it, but can be flipped back either direction any time.

### Passage
A practice excerpt derived from a Score. What gets shuffled and timed.
- References Score pages by `pageId` — never copies `src`
- Single-page (common) or multi-page (built via "+ page" during Save Passage)
- Has optional `compasSettings` (BPM, beats, accent pattern) and `measures` (enables Blitz mode)
- Frequency weighting via `tier` (−1/0/+1) and `weight` (×1–×3, time multiplier)

### Session *(future)*
Not yet built. Deferred until spaced repetition / history is needed.

---

## Annotation Model — current state and next iteration

**Built (current):** single layer, no mode toggle.
- Marks on a **Score** page are transient — used only to circle a passage before hitting "Save passage." Navigating away without saving discards them. Nothing persists to `Score.pages[].annotation`.
- Marks on a **Passage** page are permanent practice marks (`practiceAnnotation`), scoped to that one passage only.
- Rationale: the earlier two-layer/mode-toggle design (score notes vs. passage marks, switched by a button) caused mode confusion — too easy to draw in the wrong layer without realizing it. Stripped down to one layer to remove the failure mode entirely, at the cost of losing permanent score-level annotation (fingerings etc.) for now.

**Next iteration (not yet built):** scope-determined layers, no toggle needed.
- Annotations drawn **on a Score** (not yet turned into a passage) are permanent and cascade to every Passage derived from that Score page.
- Annotations drawn **on a Passage** are local to that passage only.
- No mode switch required — the layer is implied by *where you are* (Score view vs. Passage view), which are already distinct places in the UI. This reintroduces two layers but avoids the earlier confusion because the split follows object type, not a stateful toggle.
- Open question: editing a Score after passages exist would retroactively change what those passages display (e.g. updating a fingering). Likely desired, but needs a deliberate decision — possibly a re-sync action rather than automatic cascade, to avoid surprising the user mid-practice.

---

## Sidebar Structure (as implemented)

Two sections:
- **Scores** — collapsible headers, page-count + derived-passage-count shown, 🔀/⏸ pool toggle, page buttons (p.1, p.2…) for direct navigation, rename/delete
- **Passages** — flat shuffled list, tier badge, weight picker, rename/delete, drag-to-reorder

"Save passage" button floats bottom-right of canvas, visible only when a Score page is active.

---

## Workflow Phases

```
Import → (Score Assembly if needed) → Practice / Save Passage (interleaved, not sequential)
```

Practice Setup is **not a separate mandatory phase** — user can practice a Score immediately on import, and save passages opportunistically while practicing or browsing, in any order. This was a deliberate simplification from the original linear-phase model: phases are available, not required.

### 1. Import — *implemented*
- Single **Import** button opens a persistent window; user accumulates files/photos before one commit
- Camera and file-picker buttons both re-triggerable without closing the window
- Named capture session: prompts for piece name before first camera shot in a session, auto-labels subsequent pages `pg.2`, `pg.3`…
- PDFs import as one Score with all pages intact (not split into separate items)
- Naming deferred to Score rename in sidebar if skipped at import

### 2. Score Assembly — *not yet built*
Still needed for: multiple loose photos of one piece, or a PDF split across multiple files. Deferred — no current user need since single-PDF and single-photo cases (the common ones) are handled by Import directly.

### 3. Save Passage — *implemented*
- User opens a Score page, circles/boxes a passage, hits **"Save passage"**
- Panel: editable name (auto-suggested from Score name + count), "Continues on next page" checkbox → reveals +/− page controls for multi-page passages
- On save: Score's `hasPassages` set true, `inPool` auto-set false (first time only) — user can toggle back via sidebar badge
- Marks on the Score page are cleared after save (transient, not persisted — see Annotation Model)

### 4. Practice — *implemented*
- Pool = all Passages + all pages of Scores where `inPool !== false`
- Shuffle-per-loop or full-shuffle, tier-weighted (−1→1 copy, 0→2 copies, +1→4 copies)
- Timer is **per page**, not per Score (changed from earlier per-piece assumption)
- Rest periods on by default, configurable interval/duration

#### Blitz Mode *(planned, not built)*
- Each Passage served once
- Timer derived from musical data: `prep (2s) + count-in (1 measure) + play time (measures × beats ÷ BPM × 60)`
- Requires `compasSettings` and `measures` on the Passage

### 5. Review *(future)*
- Session history, SM-2 spaced repetition, repertoire structure view

---

## Compás Integration *(planned, not built)*
```js
passage.compasSettings = { bpm: 112, beats: 4, accentPattern: [true,false,false,false] }
```
- One-button capture from Compás's current state onto the active Passage
- Serve-time: Interleaves configures Compás automatically (embedded = direct call; standalone = `compas_apply` localStorage bridge, same pattern as shared `theme` key)
- Compás should live in its own swappable `compas.js` file so the metronome can be upgraded independently of Interleaves — architecture decision made, not yet executed (Compás is still inline in the current build)

---

## Key Decisions Log

| Decision | Choice | Rationale |
|---|---|---|
| File copies | One copy per piece, Score owns `src` | No duplicate confusion |
| Multi-page PDFs | Import as one Score, not split pages | Matches how pieces are actually structured |
| Score vs. Passage | Separate DB objects, separate sidebar sections | Scores = browse/read; Passages = shuffle/practice |
| Practice pool inclusion | `inPool` flag, user-toggleable (🔀/⏸) | Auto behavior alone was ambiguous; explicit override needed |
| Annotation layers | Collapsed to one layer (practice marks only) for now | Two-layer mode-toggle caused "wrong layer" errors |
| Next annotation iteration | Scope by object (Score=permanent+cascades, Passage=local), no toggle | Removes mode confusion without losing permanent score notes |
| Prep as mandatory phase | Rejected — practice/annotate interleaved freely | User should practice immediately, not gate on setup |
| Timer unit | Per page | Simpler than per-piece for now |
| Save Passage feedback | Toast + button always visible on Score pages | No score-view badge clutter |
| Passage verb | "Save passage" | Short, musical, self-explanatory |

---

## Open Questions

- **Scope-based annotation cascade**: does editing a Score's permanent layer retroactively update existing Passages automatically, or require an explicit re-sync? Needs a decision before building.
- **Score Assembly**: still needed for multi-photo pieces; not yet built, no blocking need yet.
- **Compás extraction**: move from inline to standalone `compas.js` — architecture agreed, not executed.
- **`Score.pages[].annotation` field**: currently dead (nothing writes to it under the single-layer model). Either repurpose for the cascade iteration above, or remove from schema.
- **Migration**: none needed — pre-release, clean-slate DB each schema change so far (`v1`→`v2`).

---

*Living document — reflects actual implementation state, not just plans. Update after each build session.*

