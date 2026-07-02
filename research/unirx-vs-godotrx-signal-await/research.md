# UniRx (Unity) vs GodotRx / native Signal + await (Godot)

## Official sources
- UniRx (Unity, archived): https://github.com/neuecc/unirx
- GodotRx (Godot 4, GDScript port of RxPY): https://github.com/Neroware/GodotRx
- Godot Signal built-in type: https://docs.godotengine.org/zh-tw/4.x/classes/class_signal.html
- Two `share.google/aimode/...` links provided were **not fetchable** — they redirect to an authenticated Google AI Mode session, not a public page. Not incorporated into this research; if there's specific content there worth capturing, it needs to be pasted in manually.

## Unity: UniRx
- **Purpose**: Reactive Extensions (Rx) for Unity — addresses concrete Coroutine limitations: "Coroutines can't return any values" and "can't handle exceptions," which leads to tightly-coupled code.
- **Core concepts**: `Observable` (event streams), `Subject` (both a source and a subscriber, for manual publishing), LINQ-style operators (`Where`, `Select`, `Merge`, time-based operators).
- **Unity integration**:
  - `MainThreadScheduler` — keeps operations on the main thread, respects `Time.timeScale`.
  - Coroutine bridge — `FromCoroutine()` / `ToYieldInstruction()` convert between `IEnumerator` and `Observable`.
  - `ObservableTriggers` — expose MonoBehaviour lifecycle, UI events, and physics callbacks as observables.
  - "MicroCoroutine" — a custom scheduler technique claimed to be ~10x faster than standard Unity coroutine iteration.
- **Memory management**: subscriptions must be manually disposed or bound via `.AddTo(gameObject)` to avoid leaks.
- **Status**: **archived February 2024** — the author explicitly recommends migrating to its successor, **Cysharp/R3**, instead of continuing with UniRx.

## Godot: GodotRx
- **Purpose**: brings ReactiveX concepts to Godot 4/GDScript — a fairly direct port of **RxPY** (Python's Rx implementation), chosen because GDScript and Python are syntactically similar.
- **Core API**: `Observable` as the primary construct, chainable `map()`/`filter()`, plus `ReactiveProperty` (single-value observable that emits on change), `ReactiveCollection` (multi-value variant), `oftype()`, `from_coroutine()`. Accessed via a `GDRx` singleton with factory methods like `just()`, `of()`, `concat_streams()`.
- **Architecture**: explicitly **not** built on top of Godot's native `Signal` system — it's an independent reactive framework that separately integrates with Godot's scheduler/coroutine mechanisms.
- **Maturity**: v1.0.2 (July 2025), 141 stars/8 forks — moderate but young. README itself warns: **"this library has not yet been fully battle tested in action."** Documentation leans on RxPY reference docs rather than Godot-specific guidance. No formal generics (GDScript limitation), handled via runtime type checks instead.

## Godot: native Signal
- **Purpose**: a built-in `Variant` type — Godot's native event system. Connected `Callable`s "listen and react to events, without directly referencing one another."
- **Declaration/emission**: `signal my_signal(args)` in GDScript (or `[Signal]` attribute in C#), triggered via `emit()`.
- **Connection**: `some_signal.connect(callable)`, with `Callable.bind()` for partial application; `disconnect()`, `is_connected()`, `get_connections()` round out the API.
- **`await` interaction**: `await some_object.some_signal` is native GDScript syntax — pauses the current function until that signal fires. This is the mechanism the earlier [coroutines-async-unitask-vs-gdscript-await](../coroutines-async-unitask-vs-gdscript-await/research.md) entry already covers in depth.
- **Caveat**: a connection is silently lost if the connected object is freed — same category of footgun already flagged for `await` in general.

## Comparison / Mapping
| Unity (UniRx) | Godot | Notes |
|---|---|---|
| `Observable`/`Subject` + LINQ operators | native `Signal` + `await`, or GodotRx `Observable` | Godot has no single "the" Rx equivalent — it's a choice between native primitives and a young third-party port |
| `FromCoroutine()`/`ToYieldInstruction()` bridge | not needed — `await` already unifies coroutine-like and value-returning async code | The bridging problem UniRx solves barely exists in GDScript |
| `ObservableTriggers` (lifecycle/UI/physics as observables) | connect directly to the relevant native `Signal` (e.g. `Node.tree_exited`, `Control` input signals) | No wrapping layer needed — these are already signals in Godot |
| `.AddTo(gameObject)` (auto-dispose on destroy) | manual guard (e.g. `is_instance_valid()`), no automatic equivalent | Same gap already noted for coroutines/await |
| MicroCoroutine (perf optimization over default coroutines) | native `Signal`/`await` is already engine-level, no GDScript-level scheduler needed | Godot doesn't have the "coroutines are slow, need a custom scheduler" problem UniRx was solving |

## The proposed conclusion — verdict
**Proposal**: for performance, use native `await` + `Signal` directly, and only build small UniRx-style **factory functions** (e.g. throttle/debounce/merge helpers) on top — rather than adopting GodotRx wholesale.

**Verdict: directionally correct, with one scope caveat.**

**Why it holds:**
1. GodotRx is *not* built on native `Signal` — it's a parallel event system implemented in GDScript. Every native signal elsewhere in a Godot project would need bridging into GodotRx's `Observable` world, adding overhead and a second mental model to maintain.
2. UniRx's core motivation — coroutines can't return values or propagate exceptions — is largely already solved by GDScript's native `await`, which does return values and composes with ordinary control flow. Much of the *reason* to reach for a full Rx layer doesn't transfer.
3. UniRx itself is archived, with its own author pointing users elsewhere (R3). Re-implementing UniRx's exact surface in Godot risks chasing a design pattern Unity's own ecosystem has moved past.
4. GodotRx self-reports as not battle-tested and lacking generics — a real production-risk flag for a foundational dependency.

**The caveat:**
This is the right call *if* the Unity project's UniRx usage is a bounded set of recognizable patterns (throttle, debounce, merge, combineLatest, a `Subject`-like manual-publish wrapper). If instead the codebase has deep, heavily-chained Rx pipelines (many operators composed together, custom schedulers), hand-building each pattern as an ad hoc factory function risks becoming its own unproven, undocumented mini-framework — at that scale, a small hand-rolled `Observable`/`Subject` class built *on top of* native `Signal` (not GodotRx) might be worth it instead of one-off helpers, while still avoiding GodotRx's separate event system.

**Advice:**
- Default to native `Signal` + `await` directly wherever a UniRx call site is simple (single subscription, no operator chaining).
- For the recurring operator patterns actually used in the project (not all of Rx — just the ones that show up), write small, named factory functions wrapping native signals (e.g. a `debounce(signal, seconds)` helper returning something awaitable) rather than adopting GodotRx.
- Don't adopt GodotRx as a foundational dependency given its "not battle tested" self-assessment and separateness from native `Signal`, unless the inventory below reveals genuinely deep Rx usage that justifies it.
- Note GDScript has no C#-style `try`/`except` exceptions — if the project's UniRx usage leans on `OnError` propagation semantics, that needs an explicit alternative (error-code/return-value pattern), not a direct port.

## Follow-ups / things to verify later
- Inventory the actual UniRx operators/patterns used in the project (which of `Where`/`Select`/`Merge`/`Throttle`/`CombineLatest`/etc., and how deeply chained) — determines whether "a few factory functions" is really sufficient or whether a small custom Observable class is warranted.
- Check whether any code relies on UniRx's exception propagation (`OnError`) specifically — GDScript has no direct exception-based equivalent.
- Revisit the two blocked `share.google/aimode` links if their content is important — they need to be pasted in directly since they aren't fetchable as URLs.
