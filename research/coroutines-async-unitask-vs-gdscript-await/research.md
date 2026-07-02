# Coroutines / Async-Await / UniTask (Unity) vs `await` (GDScript)

## Official sources
- Unity Coroutine: https://docs.unity3d.com/6000.2/Documentation/ScriptReference/Coroutine.html
- Unity async/await support (`Awaitable`): https://docs.unity3d.com/6000.0/Documentation/Manual/async-await-support.html
- UniTask (Cysharp, third-party): https://github.com/cysharp/unitask
- GDScript coroutines/await overview: https://gdscript.com/solutions/coroutines-and-yield/
- GDQuest glossary "await": https://school.gdquest.com/glossary/keyword_await — **fetch blocked (HTTP 403)**, not incorporated below; revisit with a different access method if more detail is needed.

## Unity: three coexisting async mechanisms
1. **Coroutines** (`IEnumerator` + `StartCoroutine`/`StopCoroutine`)
   - A function suspends via `yield return` and resumes later, integrated into Unity's frame loop — not a real thread.
   - `yield return null` → resume next frame; `yield return new WaitForSeconds(n)` → resume after a delay; other `YieldInstruction`s cover more cases; coroutines can nest (`yield return StartCoroutine(...)`).
   - Cannot return a value directly (no generic result) — workarounds use `ref`/callback patterns.
   - Automatically tied to the owning `MonoBehaviour`'s lifecycle (stops if the GameObject is disabled/destroyed).
2. **Native async/await** (`Awaitable`, introduced Unity 2023.1+, alongside standard .NET `Task`)
   - Real C# `async`/`await`; can return values, unlike coroutines.
   - `Awaitable` was added specifically because plain `Task` carries GC allocation overhead and doesn't align with Unity's single-main-thread execution model.
   - Care is needed about which Unity APIs are safe to call in a continuation depending on what thread it resumes on.
3. **UniTask** (Cysharp, third-party, free)
   - Solves `Task`'s allocation overhead with a **struct-based**, zero-allocation `UniTask<T>`.
   - Integrates with Unity's `PlayerLoop` for frame-based waits (`UniTask.Yield()`, `UniTask.DelayFrame()`) without spinning up real threads.
   - Standard `CancellationToken`-based cancellation.
   - Supports `AsyncEnumerable`/LINQ for async streams, and event/UI bindings.
   - **Caveat**: a `UniTask` instance can only be awaited once unless `.Preserve()` is called — a real footgun vs. `Task` (awaitable many times) and vs. Coroutines (no such restriction conceptually).
   - No threading by default — must explicitly call `UniTask.SwitchToThreadPool()` to leave the main thread.

## Godot: `await` (GDScript, Godot 4)
- GDScript has **one** unified async model, not three: any function containing `await` implicitly becomes a coroutine usable with `await` by its caller.
- `await` pauses execution until a **signal** fires (or an awaited call resolves) — e.g. `await get_tree().create_timer(1.0).timeout`, or awaiting a custom/node signal.
- Calling a coroutine-like function *without* `await` doesn't block — it "immediately returns a value that may be ignored" while the callee keeps running in the background (fire-and-forget).
- Custom signals emitted within the same frame should use `call_deferred` to avoid timing/ordering issues.
- Godot 3.x used `yield()` instead, returning a `GDScriptFunctionState` object with a manual `.resume()` method — a more manual/lower-level mechanism than Godot 4's `await`.

## Comparison / Mapping
| Unity | Godot | Notes |
|---|---|---|
| Coroutine (`IEnumerator`, `StartCoroutine`) | `await` on a signal (function implicitly becomes a coroutine) | Godot has no separate "coroutine" API surface — it's just `await` + signals, unified with ordinary functions |
| `yield return null` (wait one frame) | `await get_tree().process_frame` *(verify exact signal name/timing — see follow-ups)* | |
| `yield return new WaitForSeconds(n)` | `await get_tree().create_timer(n).timeout` | |
| `StopCoroutine()` | *(no direct equivalent)* | needs a manual cancellation pattern; resuming an `await` on a freed node is a common runtime error |
| Native `async`/`Awaitable` (Unity 2023.1+) | `await` (native GDScript keyword) | Both are "real" async/await and can return values, unlike Coroutines |
| UniTask (3rd-party, zero-alloc, `PlayerLoop`-integrated) | *(no third-party layer needed)* | GDScript's `await` is already a native, frame-integrated language feature — it's solving the same problem UniTask retrofits onto Unity |
| `CancellationToken` (Task/UniTask) | *(no built-in equivalent)* | needs a manual flag/guard pattern |
| `UniTask.SwitchToThreadPool()` | `Thread` / `WorkerThreadPool` | true multithreading is a separate, opt-in system in both engines, not part of the await/coroutine model itself |

## Answering the likely questions

**1. Does GDScript have an equivalent to Unity Coroutines?**
Not as a separate API — `await` + signals cover the same ground natively. Any function that awaits automatically behaves like what Unity calls a coroutine, with no distinct IEnumerator-style mechanism to learn.

**2. Does GDScript have something like C#'s native async/`Awaitable`?**
Yes, and it's the *only* async model in GDScript — there's no legacy alternative to also account for. Unity, by contrast, has three coexisting systems (Coroutines, native async/`Awaitable`, and UniTask), and each may need a different migration treatment depending on where it's used.

**3. Do we need a UniTask-equivalent library in Godot?**
No. `await` is already a native language feature — single-threaded by default and frame-integrated via signals — so it doesn't have the allocation/PlayerLoop-integration problems UniTask was built to solve for Unity's `Task`.

**4. What's the biggest migration risk?**
Loss of Unity's automatic coroutine-stop-on-destroy behavior. Unity coroutines stop automatically when their owning `MonoBehaviour` is disabled/destroyed; GDScript's `await` has no automatic equivalent, and resuming an `await` tied to a freed node is a known runtime error. Anything relying on that automatic stop needs an explicit guard (e.g. checking `is_instance_valid()` before/after an await) written into the Godot port.

## Follow-ups / things to verify later
- Inventory which of the three Unity async mechanisms (Coroutine / native async-Awaitable / UniTask) is used at each call site in the project — the right Godot-side translation differs per mechanism.
- Confirm the Unity project's version: `Awaitable` only exists from Unity 2023.1+; older projects may only have Coroutines + UniTask, with no native `Awaitable` to compare against.
- Verify Godot's exact per-frame await signal(s) (`process_frame` vs a physics-frame equivalent) against the current Godot version's docs before relying on them to replace `WaitForEndOfFrame`/`WaitForFixedUpdate` patterns — not independently confirmed in this pass.
- Check for coroutine code that fakes a "return value" via `ref` params or callbacks — GDScript's `await` supports real return values natively, so this pattern can likely be simplified rather than literally ported.
- Design the cancellation/destroy-guard pattern for GDScript before migrating any coroutine that currently depends on `StopCoroutine()` or implicit stop-on-destroy semantics.
- Retry fetching the GDQuest glossary page (blocked with HTTP 403 this pass) if more authoritative detail on `await`'s execution model is needed.
