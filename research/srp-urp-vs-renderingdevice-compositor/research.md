# SRP / URP (Unity) vs RenderingDevice / Compositor (Godot)

## Official sources
- Unity SRP Core package: https://docs.unity3d.com/Packages/com.unity.render-pipelines.core@17.6/manual/index.html
- Unity Scriptable Render Pipeline overview: https://docs.unity3d.com/2022.3/Documentation/Manual/ScriptableRenderPipeline.html
- Unity URP introduction: https://docs.unity3d.com/6000.6/Documentation/Manual/urp/urp-introduction.html
- Godot RenderingDevice: https://docs.godotengine.org/zh-tw/4.x/classes/class_renderingdevice.html
- Godot Compositor: https://docs.godotengine.org/zh-tw/4.x/classes/class_compositor.html

## Unity: SRP / URP
- **Scriptable Render Pipeline (SRP)**: lets developers control the entire per-frame rendering process from C#. Core building blocks: `ScriptableRenderContext` (schedules/executes rendering commands), the SRP Batcher (a performance optimization), and user-written script-based control of the render loop itself.
- **SRP Core package**: shared boilerplate/utility code (platform-specific graphics API handling, common rendering utilities) that both URP and HDRP are built on top of — used either to build a fully custom SRP or to extend/modify an existing one.
- **URP**: Unity's own *prebuilt* SRP implementation. Artist-friendly, tuned to run well across a broad hardware range (mobile through high-end PC/console). Most Unity content that "uses SRP" in practice means "uses URP" (or HDRP) rather than a fully bespoke pipeline — a fully custom SRP-from-scratch is comparatively rare.
- Extending URP without writing a whole new SRP is normally done via **Renderer Features** (`ScriptableRendererFeature`) — plugged into URP's existing pipeline to add custom passes/post-processing.

## Godot: RenderingDevice / Compositor
- **RenderingDevice**: a genuinely low-level graphics abstraction sitting *beneath* Godot's high-level `RenderingServer`, targeting modern APIs (Vulkan, D3D12, Metal, WebGPU) directly. Supports compute shaders and custom GPU work "not exposed by RenderingServer and high-level nodes."
  - A global instance exists for screen rendering (`RenderingServer.get_rendering_device()`); secondary/local instances can be created for off-thread GPU work (`RenderingServer.create_local_rendering_device()`).
  - Explicitly requires intermediate Vulkan-level knowledge; the docs recommend learning Vulkan fundamentals first.
  - Not available in headless mode or when using the Compatibility (GLES3-class) rendering backend.
- **Compositor / CompositorEffect**: a `Resource` that stores an array of `CompositorEffect` objects, applied during a `Viewport`'s rendering. This is the mechanism for inserting custom render/post-processing passes into Godot's *existing* built-in renderer.
  - Explicitly marked **experimental** — "more customization of the rendering pipeline will be added in the future."
- Godot's actual renderer backends (Forward+, Mobile, Compatibility) are chosen at the project-settings level; they are not a swappable/scriptable pipeline *object* the way a Unity `RenderPipelineAsset` is.

## Comparison / Mapping
| Unity concept | Closest Godot concept | Fit |
|---|---|---|
| Full custom SRP (swap the whole pipeline via a scripted `RenderPipelineAsset`) | *(no direct equivalent)* | Godot has no "swappable scripted pipeline object" layer between RenderingServer and RenderingDevice |
| URP (prebuilt SRP, artist-friendly, cross-platform tuned) | Built-in Forward+ / Mobile renderer (project-setting choice) | Conceptually similar role, different mechanism (setting, not a scriptable asset) |
| URP Renderer Feature (`ScriptableRendererFeature`) — add passes/post-processing to URP | `Compositor` + `CompositorEffect` | Closest real match, but experimental/still growing |
| Writing raw GPU/compute work beneath the pipeline | `RenderingDevice` | Much lower-level than SRP's `ScriptableRenderContext` — bypasses scene graph/culling entirely |

**Key architectural gap**: Unity's SRP model is a *mid-level* scripting surface — you still get scene culling, batching, and a structured per-frame callback, just with full control over how draws happen. Godot has no equivalent mid-level surface: `RenderingServer`/built-in renderers are not meant to be replaced wholesale, `Compositor`/`CompositorEffect` only *hooks into* the existing renderer at defined points, and `RenderingDevice` drops all the way to raw Vulkan-style graphics programming with no scene-graph integration.

## Answering the likely questions

**1. Is there an official Godot equivalent to a fully custom SRP?**
No. There's no swappable, scriptable "pipeline object" layer. The renderer backend (Forward+/Mobile/Compatibility) is a fixed choice per project, not something you replace with your own C#/GDScript-driven pipeline the way `RenderPipelineAsset` works in Unity.

**2. Is there an equivalent to URP specifically?**
Godot's built-in Forward+ (desktop-targeted) and Mobile renderers play a similar *role* (a maintained, pre-tuned rendering path per hardware class), but they're selected via project settings, not implemented as a pluggable asset.

**3. Is there an equivalent to a URP Renderer Feature (custom pass/post-processing)?**
Yes, functionally — `Compositor` + `CompositorEffect`. This is the closest match for teams whose "custom SRP" work was really just URP + a few Renderer Features. Caveat: still experimental, so verify it covers the specific passes/effects the project needs before committing to it.

**4. What if the project has a genuinely custom SRP (not just URP + features)?**
There's no equivalent surface to migrate that onto directly. Options are: (a) drop to `RenderingDevice` and hand-write the custom rendering logic at near-Vulkan level (large undertaking, no scene-graph help), or (b) reassess whether the custom pipeline logic can be re-expressed as `CompositorEffect` passes on top of Godot's built-in renderer instead of a full pipeline replacement.

## Follow-ups / things to verify later
- Confirm whether this project's Unity rendering is: (a) stock URP with no/minimal Renderer Features, (b) URP + custom Renderer Features, or (c) a fully custom SRP. This determines whether the Godot target is "just use Forward+/Mobile," "Forward+/Mobile + Compositor effects," or "a much larger RenderingDevice-based custom renderer effort."
- Check current Godot version's `Compositor`/`CompositorEffect` feature completeness against the specific passes/effects the project relies on (shadows, custom lighting models, specific post-processing stacks) — it's explicitly still evolving.
- If HDRP (not URP) is in use anywhere, note that HDRP targets high-end-only rendering (ray tracing, etc.) with even less Godot parity than URP — flag separately if relevant.
