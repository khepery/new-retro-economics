# Merge Conflict Report

**Generated:** 2026-03-10  
**Repository:** khepery/new-retro-economics  
**Main branch SHA:** `5d3d283` (Merge PR #7 — Add Google Gemini as AI provider)

---

## Summary

| Category | Count |
|----------|-------|
| Open PRs | 14 |
| Clean (no conflicts with main) | 4 |
| Conflicting with main | 10 |
| All conflicts in single file | `econo_AI_v23_UNIFIED.html` |

### Root Cause

All 10 conflicting PRs were branched from commit `89aad0d` (pre-Gemini). Since then, **8 PRs have been merged into main** (PRs #7, #11, #12, #13, #18, #19, #21, #22), all modifying the same monolithic file `econo_AI_v23_UNIFIED.html`. The merged Gemini provider code (PR #7) is the primary conflict source — it added a large `isGemini` branch in `requestLessonFromAI()` that overlaps with every PR touching AI provider logic.

---

## 1. PR-to-Main Conflict Status

### ✅ Clean — No conflicts with main

| PR | Title | Status | Branch |
|----|-------|--------|--------|
| #1 | Replace intro animation + MVP selection | Draft | `copilot/replace-intro-animation-and-add-mvp` |
| #2 | Implement AI Generator feature | Draft | `copilot/fix-ai-generator-button-functionality` |
| #14 | Scene 3: flip-card definitions | Draft | `copilot/implement-scene-3-audit-recommendations` |
| #15 | Scene 4: textarea auto-save | Draft | `copilot/implement-recommendations-audit-scene-4` |

These PRs can be merged into main without conflict resolution.

### ❌ Conflicting with main

| PR | Title | Status | Conflict Regions | Severity |
|----|-------|--------|-------------------|----------|
| #3 | Streamline AI system prompt | Open | 1 | Low |
| #4 | Lower AI temperature + JSON mode | Open | 1 | Low |
| #5 | Add Gemini provider + model selectors | Open | 11 | **High** |
| #6 | Exponential backoff for 429s | Open | 2 | Low |
| #8 | Add fal.ai, Together.ai, Replicate | Open | 1 | Low |
| #9 | Tabbed settings panel (Task 10) | Open | 5 | **Medium** |
| #10 | Visual style picker (Task 11) | Open | 3 | Medium |
| #16 | Scene 5: escalating countdown | Draft | 1 | Low |
| #17 | Settings panel: tabbed categories | Draft | 3 | Medium |
| #20 | Fix mobile responsiveness | Open | 3 | Medium |

---

## 2. Conflict Details per PR

### PR #3 — Streamline AI system prompt (1 conflict)

**Conflict at line ~12863:** The PR's change to the JSON mode conditional (`isGroq && useJsonMode` → `(isGroq || isOpenAI) && useJsonMode`) conflicts with the Gemini `isGemini` branch block that was added by PR #7 on main.

```
main (HEAD):    if (config.isGemini) { ... } else { requestBody = { ... }; if (config.isGroq && useJsonMode) { ... } }
PR #3:          if ((config.isGroq || config.isOpenAI) && useJsonMode) { requestBody.response_format = ... }
```

**Resolution:** Keep the Gemini branch from main, apply the `|| config.isOpenAI` change inside the `else` block.

---

### PR #4 — Lower AI temperature + JSON mode (1 conflict)

**Conflict at line ~12978:** Same pattern as PR #3 — the JSON mode conditional expansion conflicts with the Gemini branch.

```
main (HEAD):    if (config.isGemini) { ... } else { ... if (config.isGroq && useJsonMode) { ... } }
PR #4:          if ((config.isGroq || config.isOpenAI) && useJsonMode) { ... }
```

**Resolution:** Same as PR #3 — keep Gemini branch, apply conditional change inside `else`.

> **Note:** PR #3 and PR #4 make overlapping changes (both expand JSON mode to OpenAI). Only one should be merged; the other should be closed or rebased to avoid duplication.

---

### PR #5 — Add Gemini provider + model selectors (11 conflicts) ⚠️ HIGH

**This PR is largely superseded by PR #7** (already merged). Both add Google Gemini as a provider. PR #5 adds model selectors for OpenAI and DeepSeek on top.

**Conflicts span:**
- Line ~7086: HAL AI source selector radio buttons
- Lines ~10247–10285: Provider config functions (`getProviderConfig`, `describeProvider`)
- Lines ~11517–11576: AI source label updates, provider sync
- Line ~11817: Startup modal provider selection
- Lines ~13051–13194: `requestLessonFromAI` Gemini branch, `halAskAI` Gemini branch

**Resolution:** This PR needs a full rebase. The Gemini provider code should be dropped (already on main). Only the OpenAI/DeepSeek model selector additions should be preserved and rebased onto current main.

---

### PR #6 — Exponential backoff for 429s (2 conflicts)

**Conflict 1 at line ~12919:** Same Gemini branch pattern — the `requestBody` construction area.

**Conflict 2 at line ~15053:** The exponential backoff retry logic in `requestLessonFromAI`. The PR adds `onStatusUpdate` callback and backoff wait logic, but the function structure changed on main.

**Resolution:** Rebase onto current main; preserve the backoff logic within the updated function structure.

---

### PR #8 — Add fal.ai, Together.ai, Replicate (1 conflict)

**Conflict at line ~15508:** The image generation function area. The merged PRs on main restructured code around the image provider section.

**Resolution:** Simple rebase — the new provider functions are additive and the conflict is minor.

---

### PR #9 — Tabbed settings panel, Task 10 (5 conflicts) ⚠️ MEDIUM

**Conflicts span:**
- Line ~3604: CSS `<style>` section — settings panel width/height
- Lines ~6429–6773: Settings panel HTML structure (large block — 128 vs 216 lines)
- Lines ~7332–7379: Settings panel buttons/controls
- Lines ~7426–7534: AI generator modal HTML
- Line ~10121: `DOMContentLoaded` initialization

**Resolution:** Significant rebase needed. The settings panel HTML was restructured by merged PRs #18/#19/#21/#22. PR #9's tab system needs to be rebuilt on top of the current panel structure.

> **Note:** PR #9 and PR #17 both implement settings panel tabs (5-tab vs 4-tab). These are **mutually exclusive** — only one should be merged.

---

### PR #10 — Visual style picker, Task 11 (3 conflicts)

**Conflicts span:**
- Lines ~7208–7297: Style selector HTML (dropdown → thumbnail grid)
- Line ~13751: `selectImageStyle()` and `DOMContentLoaded` init

**Resolution:** Medium rebase. The image settings area was reorganized by merged PRs. The thumbnail grid HTML and JS need to be relocated within the new structure.

---

### PR #16 — Scene 5: escalating countdown (1 conflict)

**Conflict at line ~18422:** The `playBeep()` call site in `startThinking`. Minor context shift from merged PRs.

**Resolution:** Simple rebase — the new `playCountdownWarning()` function and call-site change are isolated.

---

### PR #17 — Settings panel: tabbed categories (3 conflicts) ⚠️ MEDIUM

**Conflicts span:**
- Line ~3604: CSS settings panel dimensions
- Lines ~6428–6677: Settings panel HTML restructuring (large block)
- Lines ~10008–10049: Tab switching JS and initialization

**Resolution:** Significant rebase. Same root cause as PR #9 — the settings panel was already restructured by merged PRs.

> **Note:** PR #17 and PR #9 are **mutually exclusive** (both restructure settings into tabs). Choose one.

---

### PR #20 — Fix mobile responsiveness (3 conflicts)

**Conflicts span:**
- Lines ~3114–3127: CSS media query additions for comic panels
- Line ~24576: HAL sidebar CSS media query

**Resolution:** Simple rebase — the CSS additions are isolated responsive rules that just need updated line positions.

---

## 3. Inter-PR Conflicts (PR vs PR)

PRs that conflict with each other if merged sequentially (beyond their conflicts with main):

| PR A | PR B | Conflict Regions | Reason |
|------|------|-----------------|--------|
| #1 | #3, #4, #5, #6, #8, #9, #10, #16, #17, #20 | 1–11 | PR #1 modifies many sections; triggers overlapping hunks |
| #2 | #3, #4, #5, #6, #8, #9, #10, #16, #17, #20 | 1–11 | Same as PR #1 pattern |
| #14 | #16, #17, #20 | 1–3 | Minor overlaps in adjacent code sections |
| #15 | #16, #17, #20 | 1–3 | Minor overlaps in adjacent code sections |
| #9 | #17 | **Mutually exclusive** | Both restructure the settings panel into tabs |
| #3 | #4 | **Overlapping changes** | Both expand JSON mode to OpenAI |

---

## 4. Recommended Merge Order

To minimize conflict resolution work, merge PRs in this order:

### Phase 1 — Clean merges (no conflicts)
1. **PR #14** — Scene 3 flip-card definitions
2. **PR #15** — Scene 4 textarea auto-save
3. **PR #1** — Intro animation + MVP (draft — mark ready first)
4. **PR #2** — AI Generator feature (draft — mark ready first)

### Phase 2 — Simple rebases (1–2 conflicts)
5. **PR #16** — Scene 5 escalating countdown (1 conflict)
6. **PR #8** — Add fal.ai, Together.ai, Replicate (1 conflict)
7. **PR #4** — Lower AI temperature + JSON mode (1 conflict) — *merge this OR PR #3, not both*
8. **PR #6** — Exponential backoff for 429s (2 conflicts)

### Phase 3 — Medium rebases (3 conflicts)
9. **PR #20** — Fix mobile responsiveness (3 conflicts)
10. **PR #10** — Visual style picker (3 conflicts)

### Phase 4 — Large rebases / decisions needed
11. **PR #9 OR PR #17** — Settings panel tabs (choose one; 5 or 3 conflicts)
12. **PR #5** — Gemini + model selectors (11 conflicts; largely superseded by merged PR #7)

### PRs to close or supersede
- **PR #3** — Close if PR #4 is merged (overlapping JSON mode changes)
- **PR #5** — Consider closing; Gemini already on main via PR #7. Extract only the model selector feature into a new PR.

---

## 5. Architectural Observation

All conflicts stem from a single monolithic HTML file (`econo_AI_v23_UNIFIED.html`, ~24,000+ lines) containing all CSS, HTML, and JavaScript. Any two PRs modifying different features will likely conflict because git cannot cleanly merge hunks in such a large single file.

**Recommendation:** Consider splitting the application into separate files (e.g., `styles.css`, `app.js`, `index.html`) to reduce future merge conflicts.
