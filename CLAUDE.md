# Unity → Godot Migration Research

This repo is a technology-by-technology research and migration plan for moving
a Unity project to Godot. **No real Unity project source files are read or
available here** — technology usage is elicited entirely through conversation
with the project owner, then researched against official Unity/Godot
documentation. Any claim about "how this project actually uses X" that hasn't
been confirmed in conversation is left as an explicit open follow-up, not
assumed.

## How this repo is organized

```
unity-to-godot/
├── CLAUDE.md                          ← this file — master plan/instructions
└── research/
    ├── ROADMAP.md                     ← main-points index of every technology
    ├── ROADMAP.zh-tw.html             ← zh-tw translation of ROADMAP.md
    ├── _skill-template/SKILL.md       ← canonical copy of the format template
    └── <technology-slug>/             ← one folder per technology
        ├── research.md                ← Unity vs Godot deep-dive + official doc links
        ├── research.zh-tw.html        ← zh-tw translation of research.md
        ├── SKILL.md                   ← copy of the canonical format template
        └── MILESTONE.md               ← ONLY created once migration work on
                                          this technology actually starts (see below)
```

## Workflow (follow this order for new technologies)

1. **Research** — when a new Unity technology is raised, create
   `research/<slug>/research.md` covering: official doc links, Unity-side
   explanation, Godot-side explanation, a comparison/mapping table, direct
   answers to whatever the project owner actually asked, and a "Follow-ups"
   section for anything not yet confirmed about this specific project. Follow
   the structure of existing files in `research/` as the pattern.
2. **Index** — add a short entry (main points only, not the full writeup) to
   `research/ROADMAP.md`, linking to the new folder. Also append any new
   unresolved questions to `ROADMAP.md`'s "Open items across all technologies"
   section.
3. **Translate** — produce `research.zh-tw.html` for the new folder, and
   refresh `research/ROADMAP.zh-tw.html`, following `SKILL.md` Part 2's rules
   exactly (translation + HTML formatting rules).
4. **Do not create `MILESTONE.md` yet.** Milestones are deferred — a
   technology only gets a `MILESTONE.md` when the project owner says work on
   *actually migrating* that technology is starting, not during research.
   When that happens, follow `SKILL.md` Part 1's structure, informed by this
   file's overall roadmap and by that folder's `research.md`.
5. **Keep `SKILL.md` copies in sync.** `research/_skill-template/SKILL.md` is
   canonical. If the format needs to change, edit the canonical copy first,
   then propagate identically to every technology folder's `SKILL.md`. Don't
   let one folder's copy silently diverge.

## Current technologies (see `research/ROADMAP.md` for full detail)

1. AssetBundles/Addressables → PCKPacker/Resource Packs
2. UPM Custom Packages → GodotEnv
3. SRP/URP → RenderingDevice/Compositor
4. DOTween → Tween
5. Coroutines/Async-Await/UniTask → GDScript `await`
6. UniRx → native Signal/await (not GodotRx)
7. C# Iterators/LINQ Lazy Evaluation → Array
8. Shaders & GPU Instancing → Shaders & MultiMesh
9. Audio (Unity) → Audio (Godot)
10. Animation (Mecanim/Legacy) → Animation (AnimationPlayer/AnimationTree), plus Video/Movie Recording

All are in the **research-only** stage — none have a `MILESTONE.md` yet.
`ROADMAP.md`'s "Open items across all technologies" section tracks exactly
what's still unconfirmed per technology; resolve those before starting that
technology's milestone.

## Ground rules for working in this repo
- Never assume specifics about the actual Unity project's code — confirm
  through conversation, and record anything unconfirmed as a follow-up rather
  than guessing.
- Prefer official documentation (Unity docs, Godot docs, or the specific
  library's own docs/README) as sources; note in the research file when a
  source couldn't be fetched instead of silently omitting it.
- Every technology folder should stay self-contained: its own `research.md`,
  its own `SKILL.md` copy, its own translations — no cross-folder dependencies
  except explicit links (e.g. "see the SRP/URP entry").
