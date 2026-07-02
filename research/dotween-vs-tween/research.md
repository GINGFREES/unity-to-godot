# DOTween (Unity) vs Tween (Godot)

## Official sources
- DOTween documentation: https://dotween.demigiant.com/documentation.php
- Godot Tween class: https://docs.godotengine.org/zh-tw/4.x/classes/class_tween.html

## Unity: DOTween
- **Purpose**: a third-party (free + paid "Pro" tier) tween engine for animating values/objects with an intuitive, chainable API.
- **Core concepts**: individual **Tweeners** animate a single value; **Sequences** group multiple tweens/callbacks into one timeline, with nesting supported.
- **Creation styles**: shortcut methods on Unity types (`transform.DOMoveX()`), or generic getter/setter-lambda tweens for arbitrary values.
- **Chaining conventions**: `DO*` creates a tween, `Set*` configures it, `On*` attaches a callback — e.g. `transform.DOMoveX(4, 1).SetLoops(2, LoopType.Yoyo)`.
- **Value types**: floats, doubles, ints, vectors, quaternions, colors, strings, plus custom types via plugins.
- **Loops**: Restart, **Yoyo** (plays backward then forward each cycle), and Incremental (value keeps increasing per cycle).
- **Ease**: a single unified `Ease` enum (e.g. `OutQuad`, `InOutQuint`), or custom `AnimationCurve`/function.
- **Callbacks**: `OnStart`, `OnComplete`, `OnKill`, `OnPlay`, `OnPause`, `OnRewind` attached directly to a tween/sequence.
- **Lifecycle safety**: optional "Safe Mode" auto-handles tweens whose target object gets destroyed.
- **DOTween Pro** (paid): adds extras like spiral tweens and 2D Toolkit integration.

## Godot: Tween
- **Purpose**: a built-in, lightweight, script-driven animation system for interpolating values over time — no third-party asset needed. Positioned as a more performant alternative to `AnimationPlayer` for simple/dynamic animations (e.g. values not known in advance).
- **Creation**: `some_node.create_tween()`, then chain `tween_property()` (animate an object property), `tween_method()` (call a function with interpolated values each step), `tween_callback()` (fire an arbitrary function as a sequence step), `tween_interval()` (a timed delay step).
- **Sequencing**: steps run **sequentially by default**; `.parallel()` runs a step alongside the previous one instead; `.chain()` explicitly returns to sequential after a parallel group.
- **Ease/Transition**: two separate enums combined — `TransitionType` (LINEAR, SINE, ELASTIC, BOUNCE, QUAD, …) and `EaseType` (IN, OUT, IN_OUT, OUT_IN) — rather than DOTween's single unified `Ease` enum.
- **Loops**: `set_loops(count)` (no args = infinite).
- **Control**: `play()`/`pause()`, `stop()` (reset without removing tweeners), `kill()` (abort + invalidate).
- **Critical caveats (from docs)**:
  - Tweens are **not designed to be reused** — create a new `Tween` per animation.
  - Tweens **start immediately** on creation — only create one when the animation should actually begin.
  - Avoid two tweens animating the same property simultaneously — the most recently created one wins.
  - Bind a tween to a node (lifecycle management) to avoid it outliving what it's animating.

## Comparison / Mapping
| DOTween | Godot Tween | Notes |
|---|---|---|
| `transform.DOMoveX(4, 1)` | `create_tween().tween_property(node, "position:x", 4, 1)` | Godot uses its native property-path reflection instead of per-type shortcut methods |
| `Sequence().Append().Join().AppendCallback()` | default sequential chain + `.parallel()` + `tween_callback()` | Godot models "sequence" as the default behavior of one Tween, not a separate object type |
| `Ease` enum (single) | `TransitionType` + `EaseType` (pair) | Needs a lookup table per easing curve during migration |
| `SetLoops(n, LoopType.Yoyo)` | `set_loops(n)` | Godot's native ping-pong (Yoyo) support is unconfirmed — see follow-ups |
| `OnStart/OnComplete/OnKill/OnPlay/OnPause/OnRewind` | `finished` / `loop_finished` / `step_finished` signals + `tween_callback()` steps | Different paradigm: DOTween attaches lifecycle callbacks; Godot treats some as sequence steps, others as signals |
| Safe Mode (auto-handle destroyed target) | `bind_node()` | Same goal, different mechanism |
| DOTween Pro extras (spiral, punch/shake, 2D Toolkit) | *(no built-in equivalent)* | Would need manual `tween_method()`-driven implementations |

## Answering the likely questions

**1. Is there a built-in Godot equivalent to DOTween?**
Yes — `Tween` is native to the engine (no asset/plugin required), and covers the same core scope: property tweening, sequencing, ease/transition curves, looping, and callbacks.

**2. Is DOTween Pro's extra functionality covered?**
Not directly. Spiral tweens, punch/shake shortcuts, and 2D Toolkit integration are DOTween Pro-specific; if the project uses them, they need hand-rolled Godot implementations (typically via `tween_method()` driving a custom interpolation function).

**3. What API differences actually matter for migration?**
- Ease unification: build a lookup table mapping each DOTween `Ease` value used in the project to a Godot `TransitionType`+`EaseType` pair.
- Loop/Yoyo: confirm whether `set_loops()` ping-pongs natively or only repeats forward (see follow-ups) — may need to manually author a forward step + a reversed step per cycle.
- Lifecycle callbacks: DOTween's six granular callbacks don't map 1:1 to Godot's three signals; some (`OnComplete` mid-sequence) are better expressed as a `tween_callback()` step rather than a Tween-level signal.
- Object-destruction safety: swap DOTween's Safe Mode reliance for explicit `bind_node()` calls.

**4. General migration shape**
Most DOTween calls translate conceptually 1:1 — a shortcut/generic tween becomes `create_tween().tween_property(...)`, and a DOTween `Sequence` becomes one Godot `Tween`'s default sequential chain with `.parallel()` inserted wherever DOTween used `.Join()`.

## Follow-ups / things to verify later
- Confirm whether Godot's `set_loops()` supports Yoyo/ping-pong loops natively, or only forward repeats — not established from the docs excerpt fetched; verify before assuming parity with `LoopType.Yoyo`.
- Inventory which DOTween value types/plugins the project actually tweens (standard floats/vectors/colors vs. custom plugin types) — Godot's property-based tweening covers Godot-native types directly; custom types may need `tween_method()`.
- Confirm whether DOTween Pro features (spiral tweens, punch/shake, 2D Toolkit) are actually used anywhere in the project.
- Check for any per-frame/high-frequency tween creation patterns in the Unity code — Godot explicitly discourages reusing/recreating tweens carelessly, so a pattern that worked fine in DOTween might need rethinking for Godot's "new Tween per animation" model.
