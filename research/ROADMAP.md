# Unity → Godot Research Roadmap

Index of main points across all technology research. Each technology has its own
folder under `research/` with `research.md` (deep-dive), `SKILL.md` (milestone
format template), and `MILESTONE.md` (actual phases/steps). This file only holds
the *main points* — read the folder for full detail. This roadmap is the source
used to build the top-level `CLAUDE.md`.

## How this repo is organized
- `research/<technology>/research.md` — Unity-side + Godot-side technical comparison, official doc links.
- `research/<technology>/SKILL.md` — shared template defining what sections a milestone doc needs.
- `research/<technology>/MILESTONE.md` — concrete phases/steps/risks for migrating that one technology.
- No real Unity project files are read directly — technology usage is elicited through conversation, since project files aren't available to Claude.

## Technologies

### 1. AssetBundles / Addressables → PCKPacker / Resource Packs
- Folder: [research/assetbundles-addressables/](assetbundles-addressables/)
- Main points:
  - No official Godot equivalent to Addressables; Godot only has the low-level `PCKPacker` + `ProjectSettings.load_resource_pack()` primitive.
  - Customizing a bundle-like system on top of `.pck` files is the expected approach — build a thin custom layer, don't reinvent packing.
  - Scope the custom layer (catalog/versioning) to what the project actually uses — full Addressables parity may be overkill.
  - Footgun: `.pck` files silently overwrite existing resources at the same `res://` path.
  - Status: research done; project-specific usage (remote catalogs? simple lazy loading?) still unconfirmed — see MILESTONE.md Open Risks.

### 2. UPM Custom Packages → GodotEnv
- Folder: [research/upm-custom-packages-vs-godotenv/](upm-custom-packages-vs-godotenv/)
- Main points:
  - No official Godot equivalent to UPM; Godot's built-in Asset Library is manual/in-editor only, no manifest-driven pinned/transitive dependency system.
  - GodotEnv (community, chickensoft-games) is the closest analog — handles both Godot version management and addon dependency management via `addons.json`.
  - GodotEnv sources are git/local/zip only — no private registry concept, so a private UPM scoped registry needs a git-hosting plan per package.
  - Migration shape: each UPM package → a Godot addon folder (with `plugin.cfg` if Editor tooling, plain scripts if runtime-only) + an `addons.json` entry.
  - `.asmdef` compiled-assembly isolation has no direct Godot equivalent (Godot C# is typically one project-wide assembly).
  - Status: research done; need to confirm whether project's packages are editor-only, runtime, or mixed, and whether a private registry is in use — see research.md follow-ups.

### 3. SRP / URP → RenderingDevice / Compositor
- Folder: [research/srp-urp-vs-renderingdevice-compositor/](srp-urp-vs-renderingdevice-compositor/)
- Main points:
  - No Godot equivalent to a fully swappable, scripted SRP pipeline object — Godot's renderer backend (Forward+/Mobile/Compatibility) is a fixed project setting, not a pluggable asset.
  - Closest match for "URP + Renderer Features" style customization is `Compositor` + `CompositorEffect` — but it's explicitly experimental/still growing.
  - `RenderingDevice` is the true low-level escape hatch (near-Vulkan), but bypasses the scene graph entirely and needs intermediate Vulkan knowledge — much lower-level than Unity's `ScriptableRenderContext`.
  - A genuinely custom (non-URP) SRP has no direct migration target — options are a large RenderingDevice-based rewrite, or re-expressing the logic as Compositor effects on Godot's built-in renderer.
  - Status: research done; need to confirm whether project uses stock URP, URP+custom Renderer Features, or a fully custom SRP/HDRP — this decides the migration difficulty tier.

### 4. DOTween → Tween
- Folder: [research/dotween-vs-tween/](dotween-vs-tween/)
- Main points:
  - Godot's `Tween` is built into the engine — no third-party asset needed, and it covers the same core scope (property tweening, sequencing, ease/transition, loops, callbacks).
  - DOTween's single `Ease` enum splits into Godot's `TransitionType` + `EaseType` pair — needs a lookup table during migration.
  - DOTween's `LoopType.Yoyo` (native ping-pong) parity with Godot's `set_loops()` is unconfirmed — flagged as a follow-up.
  - DOTween's 6 lifecycle callbacks (OnStart/OnComplete/OnKill/OnPlay/OnPause/OnRewind) don't map 1:1 to Godot's `finished`/`loop_finished`/`step_finished` signals — some are better expressed as a `tween_callback()` step instead.
  - DOTween Pro extras (spiral tweens, punch/shake, 2D Toolkit) have no built-in Godot equivalent — need manual `tween_method()` implementations if used.
  - Status: research done; need to confirm Yoyo-loop parity, DOTween Pro feature usage, and whether custom value-type plugins are involved.

### 5. Coroutines / Async-Await / UniTask → GDScript `await`
- Folder: [research/coroutines-async-unitask-vs-gdscript-await/](coroutines-async-unitask-vs-gdscript-await/)
- Main points:
  - Unity has three coexisting async mechanisms (Coroutines, native async/`Awaitable` since 2023.1+, and third-party UniTask); GDScript has just one — `await` + signals — covering all three roles.
  - No UniTask-equivalent library is needed in Godot — `await` is already native, zero-framework-overhead, and frame-integrated.
  - Biggest migration risk: Unity coroutines auto-stop when their MonoBehaviour is destroyed; GDScript's `await` has no automatic equivalent — resuming an await on a freed node is a common runtime error, needs an explicit guard pattern.
  - `StopCoroutine()` and `CancellationToken` both have no direct GDScript equivalent — need manual cancellation patterns.
  - Status: research done; need to inventory which of the 3 Unity mechanisms is used per call site, confirm Unity version (for `Awaitable` availability), and verify Godot's exact per-frame await signal names.

### 6. UniRx → native Signal/await (not GodotRx)
- Folder: [research/unirx-vs-godotrx-signal-await/](unirx-vs-godotrx-signal-await/)
- Main points:
  - GodotRx (a GDScript port of RxPY) is **not** built on Godot's native `Signal` system — it's a parallel event framework, which means bridging overhead against everything else in a Godot project that uses native signals.
  - UniRx's core motivation (coroutines can't return values/propagate exceptions) is largely already solved natively by GDScript's `await`, so most of the *reason* to adopt a full Rx layer doesn't transfer.
  - UniRx itself is archived (Feb 2024); its own author recommends its successor R3 instead — reinforces not cloning UniRx's exact surface.
  - GodotRx self-reports as not battle-tested and lacking generics — a real risk as a foundational dependency.
  - **Conclusion**: user's proposed approach (native `await`+`Signal`, with small custom factory functions only for the operators actually used, skipping GodotRx) is directionally correct — confirmed as the recommended approach, conditional on the project's UniRx usage being a bounded set of patterns rather than deep chained pipelines.
  - Status: research + conclusion done; need to inventory actual UniRx operator usage depth in the project, and confirm whether `OnError`/exception-propagation semantics are relied on (no direct GDScript equivalent).

### 7. C# Iterators / LINQ Lazy Evaluation → Array
- Folder: [research/linq-lazy-evaluation-vs-godot-array/](linq-lazy-evaluation-vs-godot-array/)
- Main points:
  - Godot has no native `Set` type — `Array.has()`/`find()` are O(n); the correct migration target for `HashSet<T>`/LINQ set-operators (`Distinct`/`Union`/`Intersect`/`Except`) is `Dictionary`-as-set (keys as members), not `Array`.
  - For general collection transforms, `Array.filter()`/`map()`/`reduce()` are readable stand-ins for `Where()`/`Select()`/`Aggregate()`, but are **eager** — no deferred execution, no chain-wide short-circuiting, no "re-enumerate reflects mutated source" semantics like LINQ.
  - Custom lazy generators (C# `yield return`) have no direct GDScript language-level equivalent — Godot 4's `await` is signal-based async, not a general lazy-sequence generator.
  - Prefer typed (`Array[T]`) or Packed arrays over untyped `Array` when porting `List<T>`/typed C# collections, for performance + type-safety parity.
  - Both LINQ chains and `Array` functional methods carry real per-call overhead vs. explicit loops — avoid both in hot/per-frame paths, symmetric advice on both sides.
  - Status: research + conclusion done; need to confirm which meaning of "set" applies to the project's actual code (ADT vs. general collections), and whether Godot's current version supports any custom-iterator protocol for lazy sequences.

### 8. Shaders & GPU Instancing → Shaders & MultiMesh
- Folder: [research/shaders-gpu-instancing/](shaders-gpu-instancing/)
- Main points:
  - Unity: GPU instancing is a per-shader opt-in flag (`#pragma multi_compile_instancing` + macros) — ordinary GameObjects auto-batch transparently at render time.
  - Godot: instancing is a distinct resource/node type (`MultiMesh`/`MultiMeshInstance3D`) — instances are pure rendering data (buffer indices), **not** Nodes; no per-instance script/physics/collision.
  - Godot's per-instance custom data is capped at 4 floats (`INSTANCE_CUSTOM`, Color-packed) — much smaller than Unity's typed instancing-buffer capacity.
  - Key migration risk: if instanced Unity GameObjects also carry per-object gameplay logic (colliders/scripts), a straight MultiMesh port loses that — needs a hybrid data-model + MultiMesh-for-rendering-only approach.
  - Godot's single unified shading language (no ShaderLab-style wrapper) ties back to entry #3 — no swappable custom pipeline concept means no need for a pipeline-abstraction layer around shaders either.
  - Status: research + sample code done; need to confirm whether instanced objects carry per-object gameplay logic, how much custom per-instance data is actually used, and which Unity render pipeline the instancing shaders target.

### 9. Audio (Unity) → Audio (Godot)
- Folder: [research/audio-unity-vs-godot/](audio-unity-vs-godot/)
- Main points:
  - Godot splits `AudioSource` by spatial dimensionality into distinct node types (`AudioStreamPlayer`/`AudioStreamPlayer2D`/`AudioStreamPlayer3D`) rather than one component with a spatial-blend setting.
  - `AudioMixer` groups map structurally to Godot's audio bus system; `AudioClip` maps to `AudioStream` (same data/player split Unity already has).
  - Godot bakes in `AudioStreamRandomizer` (random clip + pitch/volume variation) natively — Unity typically hand-rolls this.
  - Godot documents two explicit, latency-compensated audio-sync methods (system clock for short tracks, hardware clock for long/continuous) — useful reference for rhythm-game-style needs.
  - Web export caveat: Godot's reverb-bus routing and Doppler effect don't work in the default Sample playback mode on Web, only in Stream mode.
  - Status: research done; need to confirm Godot's AudioListener equivalent, its microphone/audio-input API, which AudioMixer features are actually used, and whether the project targets Web export.

### 10. Animation (Mecanim/Legacy) → Animation (AnimationPlayer/AnimationTree), plus Video/Movie Recording
- Folder: [research/animation-unity-vs-godot/](animation-unity-vs-godot/)
- Main points:
  - Unity bundles playback+logic into `Animator`+Controller asset; Godot splits it into `AnimationPlayer` (raw data) + `AnimationTree` (blending/state logic) across two nodes.
  - Core concepts map reasonably well: Controller state machines ↔ `AnimationNodeStateMachine`, Blend Trees ↔ `AnimationNodeBlendSpace1D/2D`/`Blend2`/`Blend3`, Animation Events ↔ Method Call Tracks, root motion exists on both sides but Unity pushes deltas via `OnAnimatorMove()` while Godot requires pulling via `get_root_motion_*_accumulator()`.
  - Legacy `Animation` component has no separate Godot equivalent needed — plain `AnimationPlayer` without a Tree already covers that simple/imperative use case.
  - Real gaps: Unity's humanoid Avatar retargeting has no confirmed Godot equivalent yet (needs dedicated follow-up research); Godot's 2D IK (Cutout Animation) is editor-only with no runtime solve, unlike Unity's Animation Rigging package.
  - Video: Godot's `VideoStreamPlayer` only supports Ogg Theora (`.ogv`) — no H.264/H.265 (patent reasons) or WebM (dropped in 4.0) — so existing MP4/H.264 assets need transcoding, with a lower CPU-decode resolution ceiling than Unity's typically hardware-decoded `VideoPlayer`.
  - Movie/recording: neither engine has a built-in in-game/player-facing recorder — Unity Recorder is editor-only, Godot's Movie Maker mode is deterministic/non-real-time — this is parity, not a migration gap.
  - Status: research done (large chapter, both engines' docs + several linked sub-pages covered); several real follow-ups remain, see below.

### 11. DOTS/ECS → Third-Party ECS Options (godot-ecs / flecs bridges)
- Folder: [research/dots-ecs-vs-godot-ecs-flecs/](dots-ecs-vs-godot-ecs-flecs/)
- Main points:
  - Godot has no official/built-in ECS; three third-party strategies exist: `godot-ecs` (pure GDScript, organizational benefits only), `flecs` (native C ECS core, no Godot integration by itself), and flecs-Godot bridges (Stagehand/Glecs/godot-turbo — GDExtension/module projects putting hot logic in C++ with GDScript as a glue layer only).
  - **No option provides Unity-DOTS-equivalent raw throughput with the simulation logic written in GDScript** — GDScript has no Burst-equivalent compiler, so that combination doesn't exist in the ecosystem surveyed.
  - Conclusion: decide what ECS is actually needed *for* first — if it's architectural (data/logic separation), `godot-ecs` fits and is GDScript-native; if it's raw throughput parity, a flecs bridge with C++ hot-path logic is required; a hybrid (flecs bridge only for the subsystems that need scale, ordinary Nodes/GDScript elsewhere) is the realistic default, mirroring Unity's own ECS+GameObject coexistence guidance.
  - Status: research + conclusion done; need to confirm actual scale/purpose of the project's DOTS usage, evaluate the 3 flecs-Godot bridges' maturity (surfaced via search, not deeply vetted), and confirm team tolerance for C++ in the build if throughput parity is actually needed.

<!-- Next technology entries go here, same format:
### N. <Unity tech> → <Godot tech>
- Folder: research/<slug>/
- Main points:
  - ...
  - Status: ...
-->

## Open items across all technologies
- AssetBundles/Addressables: unconfirmed whether project relies on remote catalog updates vs simple lazy loading (affects custom layer scope).
- UPM/GodotEnv: unconfirmed whether custom packages are editor-only, runtime, or mixed; unconfirmed whether a private/scoped UPM registry is in use.
- SRP/URP: unconfirmed whether project uses stock URP, URP+Renderer Features, or a fully custom SRP/HDRP — biggest swing factor in rendering migration effort.
- DOTween/Tween: unconfirmed whether Godot's `set_loops()` supports native Yoyo/ping-pong; unconfirmed DOTween Pro feature usage and custom value-type plugins.
- Coroutines/Async/UniTask vs await: unconfirmed which of Unity's 3 async mechanisms is used where; unconfirmed exact Godot per-frame await signal names; need a destroy-guard pattern before migrating any coroutine relying on stop-on-destroy.
- UniRx vs Signal/await: unconfirmed depth/breadth of UniRx operator usage in the project; unconfirmed reliance on `OnError` exception-propagation semantics.
- LINQ/Array: unconfirmed whether project's "set" usage means the Set ADT (→ Dictionary) or general collections (→ Array); unconfirmed Godot custom-iterator-protocol support for lazy sequences.
- Shaders/GPU Instancing: unconfirmed whether instanced objects carry per-object gameplay logic (colliders/scripts); unconfirmed per-instance custom-data volume vs Godot's 4-float cap; unconfirmed target Unity render pipeline.
- Audio: unconfirmed Godot AudioListener equivalent and microphone/audio-input API; unconfirmed which AudioMixer features are used; unconfirmed rhythm-game-level sync needs; unconfirmed Web export target.
- Animation: unconfirmed whether project uses Mecanim, Legacy, or both; unconfirmed Godot 4.x humanoid retargeting capability (needs dedicated follow-up research); unconfirmed usage of Animation Layers/Avatar Masks, Synced Layers, runtime 2D IK, Playables API, and root-motion/NavMeshAgent reconciliation logic; unconfirmed existing video asset codecs (Theora-transcoding scope) and whether in-game recording is actually needed.
- DOTS/ECS: unconfirmed actual scale/purpose of project's DOTS usage (architectural vs raw throughput); unconfirmed maturity of the 3 flecs-Godot bridges (Stagehand/Glecs/godot-turbo); unconfirmed team tolerance for C++ in the build pipeline if throughput parity is needed; unconfirmed SIMD/vectorized-math usage in Burst-compiled systems.
