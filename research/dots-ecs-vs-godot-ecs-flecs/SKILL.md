# SKILL: Documentation & Milestone Format Reference

This is the canonical version of `SKILL.md`. A copy of this exact file lives in
every technology folder under `research/` so each folder is self-contained and
doesn't require jumping elsewhere to know the required format. If this file
changes, propagate the change to every technology folder's copy.

This file is a **template/formatter**, not a plan or a translation itself —
it defines the shape that two kinds of deliverables must take:
1. The milestone document (`MILESTONE.md`) for this technology.
2. The zh-tw/HTML translation of this folder's documents.

Edit `MILESTONE.md` (once it exists) or the `.zh-tw.html` files when the actual
content changes — never encode content directly into this file.

## Part 1 — Milestone format (`MILESTONE.md`)

**Note:** `MILESTONE.md` is intentionally **not created yet** for most technologies.
Milestones are only started once work on that specific technology's migration
actually begins — not during the research-only phase. When it's time to start
one, follow this structure:

1. **Goal** — one sentence: what "done" means for moving this technology to Godot.
2. **Current Unity Usage** — how this project actually uses the technology today
   (link back to `research.md` for the general Unity-side explanation; this section
   is project-specific: which features of it are actually exercised).
3. **Target Godot Approach** — the chosen Godot-side strategy, decided from
   `research.md`'s options (official primitive, custom layer, or hybrid).
4. **Open Risks / Unknowns** — anything not yet verified (pull these from
   `research.md`'s "Follow-ups" section).
5. **Phases** — numbered phases, each with:
   - **Steps** — concrete, checkable actions (checkbox list)
   - **Exit criteria** — how to know the phase is done
   Standard phase skeleton (adapt as needed per technology):
   1. Research & confirm scope (what does this project actually use?)
   2. Proof of concept (smallest possible test in Godot)
   3. Design the Godot-side approach/custom layer if needed
   4. Migrate/implement for real content
   5. Validate (test parity with Unity behavior)
   6. Document the final approach for the team
6. **Acceptance Criteria** — how we know the whole technology migration is complete.
7. **References** — links back to `research.md` and official docs.

Notes on using this part of the template:
- Keep steps concrete enough to check off — avoid vague steps like "migrate assets."
- Every phase should be able to fail/stop without blocking other technologies —
  migrations should be as independent as possible across the folders in `research/`.
- When a `MILESTONE.md` phase reveals a new open question, add it back to
  `research.md`'s "Follow-ups" section rather than answering it inline.

## Part 2 — Translation format (zh-tw + HTML)

Every `research.md` in a technology folder (and, later, every `MILESTONE.md`)
must have a Traditional Chinese (zh-tw) translation delivered as a **standalone
HTML file**, so the material is easy to read without a markdown renderer.

**File naming & location:**
- `research.md` → `research.zh-tw.html`, in the same folder.
- `MILESTONE.md` (once it exists) → `MILESTONE.zh-tw.html`, in the same folder.
- The top-level `ROADMAP.md` → `ROADMAP.zh-tw.html`, in `research/`.

**Translation rules:**
- Translate headings and prose into Traditional Chinese (zh-tw), matching the
  terminology style used on `docs.godotengine.org/zh-tw` where a term overlaps
  with something already covered there.
- Do **not** translate: code blocks, class/method/property names, file paths,
  official doc URLs, or proper nouns (e.g. "Addressables", "PCKPacker",
  "UniRx", "GodotEnv"). These stay in their original form inside translated
  sentences.
- Preserve the source document's structure 1:1 (same headings, same tables,
  same ordering) — this is a translation, not a re-summary.

**HTML rules:**
- Self-contained: inline `<style>`, no external stylesheets/fonts/scripts.
- `<html lang="zh-Hant">`, `<meta charset="utf-8">`, and a `<title>` matching
  the source document's top-level heading.
- Render tables as real `<table>` elements and code blocks as `<pre><code>`
  with monospace styling — don't leave raw markdown syntax in the output.
- Keep it readable on a normal window width (a centered content column with a
  reasonable `max-width` is enough; no need for elaborate design).

## Notes on using this template
- This file only defines *format*. What actually goes in `MILESTONE.md` or the
  translations is separate work — don't skip writing the real content by only
  updating this file.
- If a technology's research and milestone workflow genuinely needs a different
  structure, propose the change here first (in the canonical copy) so every
  folder stays consistent, rather than diverging one folder's copy silently.
