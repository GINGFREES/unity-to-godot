# DOTS/ECS (Unity) vs Third-Party ECS Options (Godot) — and the GDScript vs C# question

## Official / reference sources
- Unity Entities package (DOTS/ECS): https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/index.html
- godot-ecs (pure GDScript ECS addon): https://github.com/godothub/godot-ecs
- flecs (native C ECS core library): https://github.com/SanderMertens/flecs
- Flecs-Godot bridges (found via search while researching this entry, not in the original link list but directly relevant — see below): Stagehand (https://github.com/VerdantInteractive/Stagehand), Glecs (https://github.com/GsLogiMaker/godot-glecs), godot-turbo (https://github.com/callmefloof/godot-turbo)
- `share.google/aimode/rpXAJaI9FymThFmGQ` — **not fetchable** (same authenticated-session redirect as prior aimode links in this research set). Not incorporated; paste content directly if it matters.

## Unity: DOTS/ECS
- **DOTS** (Data-Oriented Technology Stack) is the umbrella for three cooperating pieces: the **Entities** package (the ECS architecture itself), the **C# Job System**, and the **Burst Compiler**. They're designed to compound, not to be used independently.
- **Core concepts**: **Entity** (a lightweight ID), **Component** (`IComponentData` — plain data, no logic), **System** (`SystemBase`, managed, or `ISystem`, an unmanaged Burst-compatible struct — the logic, operating on entities matching a query), **World** (the container managing entities and systems).
- **Archetype/chunk memory layout**: entities sharing the same component set belong to the same **Archetype**; memory is allocated in **64KB chunks**, each holding many same-archetype entities with their components stored linearly/contiguously — a cache-friendly, Structure-of-Arrays-like layout, deliberately different from how individual `GameObject`/`MonoBehaviour` data is scattered in memory.
- **C# Job System**: exposes Unity's internal C++ job system to C#, letting systems' per-entity work (`IJobEntity`/`IJobChunk`) be scheduled safely across worker threads — safety here means Unity statically tracks read/write access per job so conflicting jobs can't race, and independent jobs run in parallel automatically.
- **Burst Compiler**: an LLVM-based compiler that translates a constrained, "Burst-compatible" subset of C# (no managed references, no GC allocation) directly to optimized native machine code — this is what gets DOTS code to near-native speed without leaving C# syntax.
- **Net effect**: ECS organizes data for bulk, cache-friendly processing; Jobs parallelize that processing across cores; Burst compiles the hot per-entity code to native speed. Requires Unity 2022.3.0f1+. Unity's own guidance treats ECS as coexisting with ordinary GameObjects/MonoBehaviours for the subsystems that actually need this scale, not as a wholesale replacement.

## Godot: no official ECS — three third-party strategies
Godot ships no built-in ECS; the default architecture is Nodes + scripts (GDScript or C#). The three projects given represent three structurally different strategies for getting ECS-like architecture (and, in one case, ECS-like performance) in Godot:

### 1. godot-ecs — pure GDScript
- **Language**: 100% GDScript, zero external dependencies, no GDExtension compilation ("plug-and-play, easy debugging" per the README).
- **Architecture**: Entities hold Components (plain data); Systems come in two modes — `ECSSystem` (single-threaded, stateful "Direct Mode," for regular game logic) and `ECSParallel` (multi-threaded "Scheduled Mode," with automatic dependency analysis to decide what can run concurrently).
- **Godot integration**: runs in a separate `ECSWorld` managed by an `ECSRunner`, deliberately decoupled from the Node/SceneTree hierarchy.
- **Maturity**: actively maintained — 328 stars, 25 releases, latest v3.1.0 (Jan 2026), MIT license.
- **The catch**: because it's pure GDScript, its systems still execute through the GDScript interpreter. It delivers ECS's *organizational* benefits (clean data/logic separation, some multi-threading via `ECSParallel`) but **not** DOTS-style native-code throughput — there's no Burst-equivalent compilation step anywhere in this stack.

### 2. flecs — the native performance core
- **Language**: C (a C99 API, zero dependencies) with a modern C++17 API layer on top.
- **Architecture**: standard ECS (entities/components/systems/queries) with **cache-friendly archetype/SoA storage**, a **lockless multi-core scheduler**, and a stated capability of processing **millions of entities per frame**. Builds in under 5 seconds; 8.5k GitHub stars; used in shipped commercial games; 13,000+ CI tests across major compilers.
- **Godot integration**: **none, natively** — flecs is engine-agnostic. It has community bindings for C#, Rust, Zig, Lua, and Clojure, but reaching it from Godot requires a native bridge.

### 3. Flecs-Godot bridges (surfaced during this research, not in the original link list)
- **Stagehand** (VerdantInteractive) — a GDExtension bringing flecs into Godot; a `FlecsWorld` node integrates the ECS world into Godot's scene hierarchy/lifecycle. Performance-critical code is written in C++, while Godot's scripting languages remain usable elsewhere.
- **Glecs** (GsLogiMaker/godot-glecs) — GDExtension-based Godot bindings for flecs.
- **godot-turbo** (callmefloof) — an ECS **engine module** (not a GDExtension) powered by flecs, targeting Godot 4.6+. Explicitly advertises a **"Script Bridge"** to execute GDScript/C# methods on ECS entities, plus renderer-bridge nodes (`InstancedRenderer3D`, `MultiMeshRenderer2D`/`3D`, `ComputeRenderer`) connecting ECS-driven data straight into Godot's rendering/instancing systems (a nice tie-in to the earlier [Shaders & GPU Instancing](../shaders-gpu-instancing/research.md) entry).
- **Common shape across all three**: the ECS simulation core runs in native C/C++ (flecs itself); GDScript/C# is used only as a **thin scripting/glue layer** on top. Hot-path logic lives in C++; GDScript handles orchestration, gameplay-facing hooks, and configuration.

## The GDScript vs C# question — this is the actual crux
This mirrors a pattern already established elsewhere in this repo (see [DOTween/Tween](../dotween-vs-tween/research.md) and [LINQ/Array](../linq-lazy-evaluation-vs-godot-array/research.md)): GDScript, being interpreted, has structural throughput limits that don't disappear just by adopting an ECS-shaped API.

- Unity's DOTS gets its performance specifically by **leaving C#'s normal execution model** for the hot path — Burst compiles a restricted subset to native code, Jobs parallelize it. **Neither exists for GDScript.** There is no "Burst for GDScript."
- Godot's **C#** support is architecturally closer to what DOTS needs (compiled, JIT'd), but Godot doesn't ship an official ECS framework, a Burst-equivalent compiler tied to it, or a bulk-entity job system for C# either — you'd still be adopting something third-party even on the C# side.
- **godot-ecs (pure GDScript)** gives the *organizational* win without a throughput win — GDScript is still GDScript underneath. Right fit if the motivation is code structure, not raw entity-count scale.
- **flecs + a Godot bridge** gives DOTS-comparable raw performance — but the hot logic has to be **C++**, not GDScript. None of the three bridges surveyed let you write the performance-critical simulation code in GDScript and still get flecs-level throughput; GDScript's role in all of them is the glue layer only.

**There is no option here that provides Unity-DOTS-equivalent raw performance with the simulation logic itself written in GDScript.** That combination isn't in the ecosystem surveyed — for the same structural reason GDScript has no Burst compiler: an interpreted language doesn't reach millions-of-entities-per-frame throughput regardless of how the data around it is organized.

## Conclusion & advice
Given the stated priority ("the main point is use GDScript"), the real decision is **what ECS is actually being adopted *for*** in this project — that determines which path (or "skip ECS entirely") is right:

1. **If the goal is architectural** (decoupling logic from the Node tree, cleaner systems/data separation, moderate multi-threading via a dependency-scheduled parallel mode) — **`godot-ecs` is the right fit and is GDScript-native**, matching the stated preference directly. This is very likely sufficient unless the Unity side was specifically leaning on DOTS for raw entity-count throughput.
2. **If the goal is throughput parity with what DOTS was actually providing** (large-scale simulation, crowds, particle-as-entity at DOTS-like scale) — that requires **flecs via a native bridge (Stagehand/Glecs/godot-turbo), with hot-path logic in C++**, not GDScript. "Use GDScript" can then only apply to the outer glue/gameplay layer — this needs to be an explicit, accepted trade-off decided up front, not discovered mid-migration.
3. **A hybrid is realistic and arguably the best default**: use a flecs-based bridge only for the specific subsystems that genuinely need bulk entity throughput, and keep everything else as ordinary Godot Nodes/GDScript — this mirrors Unity's own guidance that ECS and GameObjects are meant to coexist, not replace each other wholesale.
4. **Don't default to adopting any of these.** All three are third-party, with different maturity profiles (`godot-ecs`: single-addon GDScript project, 328 stars; the flecs bridges: newer/smaller GDExtension or module projects, with unconfirmed production track record *in Godot specifically*, even though flecs itself is production-proven elsewhere). Confirm the project's actual DOTS usage first (see follow-ups) — a project using DOTS lightly may not need any ECS framework in Godot at all.

## Follow-ups / things to verify later
- Confirm what the project's actual Unity DOTS/ECS usage looks like: which systems use it, roughly how many entities/frame, and whether it's there for raw throughput or just architectural separation — this is the single biggest factor in which Godot approach (if any) applies.
- If raw throughput matters, evaluate the three flecs-Godot bridges directly (Stagehand, Glecs, godot-turbo) — they surfaced during this research rather than being in the original request, and their maturity/support/API stability weren't deeply verified in this pass.
- Confirm whether the team/build pipeline can tolerate C++ in the hot path at all (skillset, GDExtension build complexity) — this determines whether options 2/3 above are realistic regardless of performance need.
- If `godot-ecs` (pure GDScript) is chosen, benchmark its `ECSParallel` mode against the project's actual entity counts early — "some multi-threading" in an interpreted language is not the same performance class as Unity's Jobs+Burst, and this should be validated, not assumed.
- Check whether the Unity project's Burst-compiled systems rely on SIMD/vectorized math (common in DOTS code) — if so, that's an additional reason plain GDScript can't be a drop-in replacement for that code, reinforcing the native-bridge requirement for throughput-critical subsystems.
