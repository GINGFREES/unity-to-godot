# Animation (Unity: Mecanim/Legacy) vs Animation (Godot: AnimationPlayer/AnimationTree) — plus Video & Movie Recording

This is a large chapter covering two related-but-distinct topics on each side:
1. **Character/property animation** — Unity's Mecanim + Legacy systems vs Godot's AnimationPlayer + AnimationTree.
2. **Video playback and gameplay/movie recording** — grouped under "Animation" in Godot's docs, and a materially different set of tools in Unity.

## Official sources

**Unity — animation:**
- Animation section overview: https://docs.unity3d.com/6000.4/Documentation/Manual/AnimationSection.html
- Mecanim overview: https://docs.unity3d.com/6000.4/Documentation/Manual/animation-mecanim.html
- Legacy overview: https://docs.unity3d.com/6000.4/Documentation/Manual/animation-legacy.html
- Animator Controller: https://docs.unity3d.com/6000.4/Documentation/Manual/animation-animator-controller.html
- Animation State Machines: https://docs.unity3d.com/6000.4/Documentation/Manual/AnimationStateMachines.html
- Blend Trees: https://docs.unity3d.com/6000.4/Documentation/Manual/class-BlendTree.html
- Animation Layers: https://docs.unity3d.com/6000.4/Documentation/Manual/AnimationLayers.html
- Avatar Creation and Setup: https://docs.unity3d.com/6000.4/Documentation/Manual/AvatarCreationandSetup.html
- Root Motion: https://docs.unity3d.com/6000.4/Documentation/Manual/RootMotion.html
- Animator component reference: https://docs.unity3d.com/6000.4/Documentation/Manual/class-Animator.html
- Animation Clips landing page: https://docs.unity3d.com/6000.4/Documentation/Manual/animation-clips-landing.html
- Legacy `Animation` component reference: https://docs.unity3d.com/6000.4/Documentation/Manual/class-Animation.html

**Unity — video & recording:**
- VideoPlayer: https://docs.unity3d.com/6000.4/Documentation/Manual/VideoPlayer.html
- Unity Recorder package: https://docs.unity3d.com/Manual/com.unity.recorder.html

**Godot — animation:**
- Animation docs index: https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/index.html
- Introduction to Animation (AnimationPlayer basics): https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/introduction.html
- Animation track types: https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/animation_track_types.html
- AnimationTree: https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/animation_tree.html
- 2D Skeletons: https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/2d_skeletons.html
- Cutout Animation (2D IK): https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/cutout_animation.html

**Godot — video & movies:**
- Playing videos: https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/playing_videos.html
- Creating movies (Movie Maker mode): https://docs.godotengine.org/zh-tw/4.x/tutorials/animation/creating_movies.html

**Not incorporated:** `share.google/aimode/...`-style links were not part of this request. No blocked sources this pass.

---

## Part A — Character/Property Animation

### Unity: Mecanim (the modern system)

- **AnimationClip**: the basic unit of motion (a single "Idle"/"Walk"/"Run" etc.), imported from DCC tools (FBX) or authored directly in Unity's Animation window. Supports **Curves** (drive arbitrary properties or even Animator parameters over time), **Animation Events** (call a script method at a specific point in playback), **masking** (restrict to specific bones), and **loop optimization** (align start/end pose for seamless loops).
- **Animator component**: attached to a GameObject; references a **Controller** and (for humanoids) an **Avatar**. Key fields: **Apply Root Motion**, **Update Mode** (Normal / Animate Physics / Unscaled Time), **Culling Mode**.
- **Animator Controller**: the asset containing state machines, blend trees, and parameters, edited visually in the Animator window.
  - **State machine**: states (each holding a Motion — a clip or a blend tree) + transitions between them. Special nodes: **Entry/Exit**, **Any State** (transition into any state regardless of current state — used for global interrupts like "Hit"/"Death"), and **sub-state machines** for nesting.
  - **Transitions**: blend duration + conditions; also expose **Exit Time**, **Transition Offset**, and **Interruption Source** (whether an in-progress transition can itself be interrupted).
  - **Parameters**: **Float, Int, Bool, Trigger**, set via `Animator.SetFloat/SetInteger/SetBool/SetTrigger`, read by transition conditions and blend trees.
  - **Blend Trees**: continuously blend multiple clips by parameter value (clips should share similar timing/normalized-time alignment). Types: **1D** (one float parameter, e.g. Walk→Run by Speed), **2D** (two parameters; Simple Directional / Freeform Directional / Freeform Cartesian sub-modes), and **Direct** (each motion's weight driven by its own parameter).
  - **Animation Layers**: multiple state machines run in parallel per Controller. Each has a **Weight**, a blend mode (**Override** or **Additive** — additive requires matching properties between the additive clip and base layers), an **Avatar Mask** (restrict to specific bones, e.g. upper-body-only), and can be a **Synced Layer** (reuse another layer's state-machine structure with different clips, with a Timing option for stretch-to-source-length vs blend-length).
- **Avatar system (Humanoid vs Generic)**: Humanoid avatars map a model's bones to a standardized skeleton, enabling **clip retargeting across different humanoid models** (a walk clip authored on one character can drive a differently-proportioned one). Requires bone mapping (manual, or via Human Template presets), muscle definitions (joint limits), and a roughly T-posed source model. Generic avatars are for non-humanoid rigs and are **not** retargetable across different skeletons.
- **Root Motion**: whether the character's actual position/rotation is driven by animation data. **Body Transform** (center of mass) vs **Root Transform** (Y-plane projection, computed per frame); per-clip baking controls (**Root Transform Rotation**, **Position (Y)**, **Position (XZ)**, each with "Bake Into Pose" + a "Based Upon" setting). When enabled, Unity calls `OnAnimatorMove()` on the object's script each frame so root-motion deltas can be intercepted/reconciled with physics or a NavMeshAgent before being applied — a common source of bugs (foot sliding, pops) if inconsistently configured.
- **Supporting tools**: **Animator Override Controller** (reuse a controller's structure with different clips), **Playables API** (lower-level runtime graph API for programmatic animation graphs beyond the Controller graph).

### Unity: Legacy animation system

- **Component**: `Animation` (distinct from `Animator`). Fields: default `Animation` clip, `Animations` array, `Play Automatically`, `Animate Physics`, `Culling Type`.
- **No state machine, no Controller asset, no parameters, no blend trees** — a flat list of clips controlled imperatively via script: `Animation.Play()`, `Animation.CrossFade()` (with `PlayMode.StopSameLayer`/`StopAll`), and `AnimationState` objects exposing manual blend weights, layers, wrap mode, and `AnimationBlendMode.Additive`.
- Clips used with Legacy must have their **Legacy flag** set on import — modern Mecanim clips aren't usable here unless flagged.
- **Status**: still supported, explicitly deprecated in spirit — Unity's own docs say "for new projects, use the Animator component." Retained for old projects, or for genuinely trivial cases (a single looping clip on a prop) where its lower overhead can still make sense.

| Aspect | Legacy (`Animation`) | Mecanim (`Animator`) |
|---|---|---|
| Control model | Imperative (script calls Play/CrossFade) | Declarative state machine + parameters |
| Blending | Manual weights / CrossFade calls | Built-in transitions, blend trees, layers |
| Retargeting | None | Avatar system (humanoid retargeting) |
| Root motion | Not natively supported | First-class |
| Clip requirement | Must be flagged "Legacy" | Standard AnimationClip |
| Editor tooling | None | Animator window (visual graph editor) |

### Godot: AnimationPlayer

- A `Node` (not spatial) acting as a data container for one or more `Animation` resources, with a timeline/keyframe editor. **Important caveat**: because it's a plain `Node`, 2D/3D children parented directly under it do **not** inherit its transform — don't parent scene content under an AnimationPlayer (a common habit coming from Unity, where the Animator sits directly on the GameObject it drives).
- **Animation Libraries**: individual `Animation` resources are registered into Animation Library resources for reuse/sharing across different AnimationPlayers.
- **Tracks**: a timeline row bound to a node path + property, auto-created on first keyframe. Per-track: **Update Mode** (Continuous / Discrete / Capture), **Interpolation** (Nearest / Linear / Cubic / Linear Angle / Cubic Angle — the Angle variants take the shortest angular path, important for rotations), **Loop Mode** (Clamp / Wrap).
- **Markers**: named points on the timeline for playing a sub-section via `play_section_with_markers()`.
- **RESET animation**: a special `_RESET` animation storing the scene's default pose; used both for scene-save consistency ("Reset On Save") and as the fallback value AnimationTree blending uses for properties an animation doesn't touch.
- **Blending**: simple, fixed-duration crossfade between clips — explicitly the motivating limitation for AnimationTree's existence.

**Track types** (from `animation_track_types.html`):

| Track type | Purpose | Notes |
|---|---|---|
| Property Track | Baseline — animates any exposed property | All specialized types are optimized variants of this |
| Position/Rotation/Scale 3D | Dedicated transform tracks | Support compression; meant for imported 3D model animation |
| Blend Shape Track | Animates `MeshInstance3D` morph targets | Also compressible |
| Method Call Track | Calls a method at a timestamp with args | Suppressed during editor preview (safety) — direct analog of Unity's Animation Events |
| Bezier Curve Track | Animates one float/property along a hand-edited curve | **Cannot be mixed with property tracks** in the same blending context — a hard constraint |
| Audio Playback Track | Places audio clips on the timeline | Supports blend-weight-aware volume |
| Animation Playback Track | Triggers other AnimationPlayers' animations | Orchestrates multi-node/cutscene timelines |

### Godot: AnimationTree

- Exists specifically to answer AnimationPlayer's "blending limited to fixed crossfade" gap. **Does not own animation data** — it references and drives an existing AnimationPlayer (author clips in the Player, wire blending/state logic in the Tree). Conceptually splits what Unity bundles into Animator (component) + Controller (asset) into two nodes.
- **Root node types**: `AnimationNodeAnimation` (single clip), `AnimationNodeBlendTree` (a graph of Blend2/Blend3/OneShot/TimeSeek/TimeScale/Transition nodes), `AnimationNodeBlendSpace1D`/`BlendSpace2D` (parametric interpolation over a line/plane of animation points), `AnimationNodeStateMachine` (named states + transitions; can nest blend trees/spaces/other state machines inside a state).
- **State machine transitions**: switch modes **Immediate** / **Sync** (fast-forwards new state to match old state's position) / **At End**; properties **Xfade Time**, **Xfade Curve**, **Reset**, **Priority** (used by pathfinding), **Advance Mode**. **Advance Condition/Expression** auto-fires a transition from a bool parameter or an arbitrary expression (e.g. `is_walking && !is_idle`). `travel()` pathfinds across the state graph:
  ```gdscript
  var state_machine = animation_tree["parameters/playback"]
  state_machine.travel("TargetState")
  ```
- **Blend tree nodes**: `Blend2`/`Blend3` (weighted mix of 2-3 inputs, with track filtering for layering — e.g. upper-body-only overlays, Godot's answer to Unity's Avatar Mask + Additive layers), `OneShot` (play once then revert), `TimeSeek`, `TimeScale`, `Transition` (lightweight named-input switcher). Example:
  ```gdscript
  animation_tree["parameters/OneShot/request"] = AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE
  ```
- **Blend spaces**: `BlendSpace2D` auto-triangulates (Delaunay) between animation points on a 2D plane and blends the nearest ones — the direct analog of Unity's 2D Blend Tree. **Sync modes** (`None`/`Independent`/`Cyclic Mutable`/`Cyclic Constant`) control whether non-dominant points still advance time; the cyclic modes require every blend point to be a plain `AnimationNodeAnimation` with finite, immutable length (can't nest sub-trees as points under those modes).
- **Blending correctness requirements**: every animated property needs a determinate rest value (Skeleton3D bones use **Bone Rest**; other properties default to 0 or a `RESET` animation's first keyframe). Untracked properties in a given animation are treated as sitting at rest during blending. **Humanoid rigs should be imported in a T-pose** so bone rests sit mid-range — directly parallel to Unity's own T-pose requirement for Avatar mapping. Bones needing >180° rotation across blended animations should use root motion instead, since interpolation can't disambiguate rotations past 180°.
- **Root Motion**: designate a bone as the **Root Motion Track**; Godot cancels that bone's visual transform during normal playback. Runtime API pulls deltas rather than Unity's push-callback model:
  ```gdscript
  animation_tree.get_root_motion_position()
  animation_tree.get_root_motion_position_accumulator()  # cumulative, for blended cases
  ```
  fed into `CharacterBody3D.move_and_slide()`. Conceptually parallel to Unity's `OnAnimatorMove()` pattern, but Godot's is an explicit per-frame pull rather than an engine-invoked callback.
- **Shared-resource caveat**: AnimationTree's internal graph nodes are resources — if the same AnimationTree resource is shared across multiple scene instances, mutating one instance's node config can leak to all of them.
- **Parameters**: exposed via property paths (`animation_tree["parameters/eye_blend/blend_amount"] = 1.0`), inspectable in the editor (except Advance Expressions, which live as expression text rather than simple params).

### Godot: 2D Skeletons & Cutout Animation (IK)

- **2D Skeletons**: `Skeleton2D`/`Bone2D` hierarchy deforming a `Polygon2D` mesh via weight painting. No IK on this page — 2D IK lives in Cutout Animation instead.
- **Cutout Animation**: sprite-based "puppet" rigging — a hierarchy of pivoted sprites, "Make Bones" to build a skeleton over sprite nodes, and **2D IK chains** for posing limbs by dragging an end-effector. **Important gap**: 2D IK chains are **editor-only, for keyframing** — there is no runtime IK solve in this system, unlike Unity's Animation Rigging package (a separate package, not confirmed whether the project uses it).

---

## Part B — Video Playback & Movie/Gameplay Recording

### Unity: VideoPlayer

- Component playing video onto a GameObject's texture, a Render Texture, a camera plane, or the target display, from a **VideoClip asset** (asset-pipeline managed) or a **URL** (`file://`/`http://`, bypasses the asset pipeline — you own reachability at runtime).
- `Play()` auto-prepares if needed (causing a delay); call `Prepare()` ahead of time + wait for `prepareCompleted` for instant playback. Supports panoramic/360 video and playback-sync APIs.
- Codec support is broad and generally platform/hardware-decoder dependent (typically MP4/H.264 family) — replaced the older, deprecated MovieTexture system.

### Unity: Screen/gameplay recording (Unity Recorder)

- **No built-in runtime recording API in core Unity.** The **Unity Recorder** package (`com.unity.recorder`, installed via Package Manager) is the standard tool: Movie Recorder (H.264 MP4, VP9 WebM, ProRes QuickTime), Image Sequence, Animation Clip, GIF, and Audio recorders.
- **Editor-only** — works during Play mode in the Editor, but **does not function in standalone builds**. Not usable for shipping in-game/player-facing recording.

### Godot: VideoStreamPlayer

- Node for video playback. **Format support is narrow and deliberate**: effectively only **Ogg Theora** video (+ optional Ogg Vorbis audio) in an Ogg container (`.ogv` preferred). **H.264/H.265 excluded for software-patent reasons** (a licensing decision, not a technical gap); **WebM support was removed in Godot 4.0** (maintenance burden); **AV1 avoided** due to lack of widespread GPU decode support. Theora decoding is **CPU-only**, capping practical resolution (~1080p30 desktop, ~720p30 or lower mobile/web as rough ceilings).
- No URL-streaming support — files must be local/imported. Audio downmixes to stereo max.
- **Migration implication**: existing MP4/H.264 video assets need transcoding to Ogg Theora, with an expected quality/performance ceiling drop versus Unity's typically hardware-decoded formats.

### Godot: Movie Maker mode

- A **non-real-time, deterministic frame-by-frame renderer** to a video file — explicitly **not** a general gameplay screen-recorder (the docs point to OBS Studio/SimpleScreenRecorder for that use case instead). Purpose: trailers, high-quality pre-rendered cutscenes, deterministic capture free of dropped-frame/audio-desync issues that plague real-time recording under load.
- Enabled via the editor's film-reel icon (must be re-enabled each session) or CLI: `godot --path /path/to/project --write-movie output.avi` (also works on exported builds, minus runtime toggling).
- Output formats: **OGV** (Theora+Vorbis, best size/quality, editor-builds only, no browser playback), **AVI** (MJPEG+PCM, fast encode, 4 GB cap), **PNG sequence + WAV** (lossless, for external compositing, supports transparency).
- Must quit cleanly (window close / `get_tree().quit()` / an AnimationPlayer's "Movie Quit On Finish") — force-quitting corrupts AVI/WAV headers. Exposes `OS.has_feature("movie")` so scripts can swap in higher-quality-but-too-slow-for-real-time settings only during capture. Supports supersampling since capture speed isn't tied to real-time constraints.

**Video/Movie comparison:**

| Unity | Godot | Notes |
|---|---|---|
| `VideoPlayer` (VideoClip or URL; broad, typically hardware-decoded codecs) | `VideoStreamPlayer` (Ogg Theora `.ogv` only, CPU-decoded) | Real transcoding cost + resolution ceiling drop when porting existing video assets |
| Unity Recorder package (editor-only, not in builds) | Movie Maker mode (editor + CLI, deterministic, not real-time) | **Neither is a built-in in-game/player-facing recorder** — this is parity between the engines, not a migration gap |

---

## Comparison / Mapping (Character Animation)

| Unity | Godot | Notes |
|---|---|---|
| `AnimationClip` | `Animation` resource (in an Animation Library) | Raw keyframe data |
| `Animator` component + Animator Controller asset | `AnimationPlayer` + `AnimationTree` | Unity bundles playback+logic on one component referencing one asset; Godot splits data (Player) from blending/state logic (Tree) across two nodes |
| Controller state machine | `AnimationNodeStateMachine` | Both: named states, transitions, conditions, "any state"-style global interrupts (Unity: Any State node; Godot: any state can have a transition wired to it) |
| Parameters (Float/Int/Bool/Trigger) + transition Conditions | AnimationTree `parameters/...` properties + Advance Condition/Expression | Godot's expression-based conditions are more flexible (arbitrary boolean expressions) than Unity's fixed parameter comparisons |
| Blend Tree (1D/2D/Direct) | `AnimationNodeBlendSpace1D`/`BlendSpace2D`, `AnimationNodeBlendTree` (`Blend2`/`Blend3`) | Comparable concepts, different graph shape |
| Animation Layers (Override/Additive + Avatar Mask) | `Blend2`/`Blend3` with track filtering, composed in the blend graph | Godot has no first-class "layer list" UI like Unity's Animator Layers panel — layering is graph composition instead |
| Avatar system (humanoid retargeting) | Godot 4.x retargeting/bone-map features | **Not confirmed in this research pass** — needs dedicated follow-up |
| Root Motion (`OnAnimatorMove` callback, per-clip bake settings) | Root Motion Track + `get_root_motion_*_accumulator()` → `CharacterBody3D.move_and_slide()` | Godot: explicit per-frame pull API vs Unity's engine-invoked callback — same concept, inverted control flow |
| Animation Events (clip-embedded method calls) | Method Call Track | Direct analog; Godot's is suppressed during editor preview |
| Legacy `Animation` component (imperative Play/CrossFade) | *(no separate "legacy" system)* | Godot's `AnimationPlayer` alone (no `AnimationTree`) already covers Legacy's simple/lightweight use case |
| Animation Rigging package (runtime IK, if used) | Cutout Animation 2D IK (**editor-only**, no runtime solve) | Real gap if the project needs runtime IK — confirm usage |

---

## Answering the likely questions

**1. What's the direct migration mapping for a typical Mecanim setup?**
`AnimationClip` → `Animation` resource; `Animator`+Controller → `AnimationPlayer`+`AnimationTree`; the Controller's state machine → `AnimationNodeStateMachine`; blend trees → `AnimationNodeBlendSpace1D/2D` or `Blend2/Blend3`; parameters → AnimationTree `parameters/...` paths.

**2. What about projects still using the Legacy `Animation` component?**
There's no separate "legacy" concept to preserve in Godot — a plain `AnimationPlayer` without an `AnimationTree` already covers simple, non-blended, script-driven playback, which is what Legacy was used for. Migration is more straightforward here than for Mecanim's richer feature set.

**3. Does Godot support humanoid retargeting like Unity's Avatar system?**
Not confirmed from this research pass — this needs a dedicated follow-up, since it's a significant feature if the project shares animation clips across multiple differently-proportioned humanoid characters.

**4. How does root motion carry over?**
Conceptually directly — both engines support animation-driven root movement — but the control flow inverts: Unity pushes deltas to your script via `OnAnimatorMove()` each frame; Godot requires you to pull deltas via `get_root_motion_*_accumulator()` and feed them into `CharacterBody3D.move_and_slide()` yourself.

**5. What has to change for video assets?**
Any existing MP4/H.264 (or WebM) video needs transcoding to Ogg Theora (`.ogv`) — Godot's `VideoStreamPlayer` doesn't support H.264/H.265 (patent reasons) or WebM (dropped in 4.0), and Theora decode is CPU-only, which caps practical resolution/framerate versus Unity's typically hardware-decoded playback.

**6. Is there an in-game recording feature on either side?**
No, on both. Unity Recorder is editor-only (not usable in builds); Godot's Movie Maker mode is a deterministic, non-real-time capture tool, not a real-time gameplay recorder either. If the project needs player-facing recording, that's a custom/third-party solution regardless of engine.

## Follow-ups / things to verify later
- Confirm which Unity animation system(s) the project actually uses — Mecanim only, Legacy only, or a mix — since that changes how much of the Legacy-specific migration guidance even applies.
- Research Godot 4.x's humanoid retargeting/bone-map capabilities specifically (not covered in this pass) if the project relies on Unity's Avatar-based clip retargeting across multiple character models.
- Confirm whether Animation Layers with Avatar Masks (e.g. upper-body-only overlays) are used, and verify Godot's `Blend2`/`Blend3` track filtering covers the same granularity.
- Confirm whether Unity's **Synced Layers** feature (reusing one layer's state-machine structure with different clips) is used — no confirmed Godot equivalent was found in this pass.
- Confirm whether runtime 2D IK is required anywhere — Godot's Cutout Animation IK is editor-only (keyframing aid), not a runtime solver, a real gap versus Unity's Animation Rigging package if that's relied on live.
- Confirm root motion usage specifically, including any `OnAnimatorMove`-based reconciliation with a NavMeshAgent or physics, since that logic needs to be restructured around Godot's pull-based accumulator API.
- Inventory existing video asset codecs/resolutions — determines Theora-transcoding scope and whether the CPU-decode resolution ceiling is actually acceptable for the project's use case.
- Confirm whether the project needs in-game/player-facing video or gameplay recording — since neither engine provides this out of the box, this would require the same kind of custom/third-party solution on either side (not a migration-specific gap).
- Confirm whether Unity's Playables API is used for programmatic animation graphs beyond what the Animator Controller offers — no Godot equivalent was identified in this pass; would need separate research if relied upon.
