# Shaders & GPU Instancing (Unity) vs Shaders & MultiMesh (Godot)

## Official sources
- Unity shader introduction: https://docs.unity3d.com/6000.0/Documentation/Manual/shader-introduction.html
- Unity GPU instancing shader example: https://docs.unity3d.com/6000.0/Documentation/Manual/gpu-instancing-vertex-fragment-shader-example.html
- Unity intro to GPU instancing: https://docs.unity3d.com/Manual/GPUInstancing.html
- Godot Shader class: https://docs.godotengine.org/zh-tw/4.x/classes/class_shader.html
- Godot MultiMesh class: https://docs.godotengine.org/en/stable/classes/class_multimesh.html
- `share.google/aimode/3O5xK08gAxi0FDqKe` — **not fetchable** (same authenticated-session redirect issue as prior aimode links in this research set). Not incorporated; paste content directly if it matters.

## Unity: Shaders & GPU Instancing
- **What a shader is**: a GPU program. Unity has three categories: **Graphics Pipeline Shaders** (the common case — compute pixel colors as part of normal rendering), **Compute Shaders** (general GPU work outside the graphics pipeline), and **Ray Tracing Shaders**.
- **Authoring methods**: **ShaderLab** (Unity's own declarative language wrapping actual HLSL code — defines Properties, SubShader/Pass structure, Fallback), **Shader Graph** (node-based, code-free visual authoring), or hand-written HLSL directly.
- **Material relationship**: a `Shader` is a program container; a `Material` references a `Shader` and supplies the actual property values (textures, colors, floats) fed into it.
- **GPU instancing**: opt-in per shader via `#pragma multi_compile_instancing`. Per-instance properties are declared in an instancing buffer:
  ```hlsl
  UNITY_INSTANCING_BUFFER_START(Props)
      UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
  UNITY_INSTANCING_BUFFER_END(Props)
  ```
  The vertex input struct needs `UNITY_VERTEX_INPUT_INSTANCE_ID`; the vertex shader must call `UNITY_SETUP_INSTANCE_ID(v)` (mandatory — world matrices depend on it) and `UNITY_TRANSFER_INSTANCE_ID(v, o)` to pass the ID to the fragment stage; the fragment shader reads a value via `UNITY_ACCESS_INSTANCED_PROP(Props, _Color)`.
  - Minimal instanced shader (Built-in pipeline, ShaderLab/HLSL):
    ```hlsl
    Shader "Custom/InstancedColor"
    {
        Properties { _Color ("Color", Color) = (1,1,1,1) }
        SubShader { Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            #include "UnityCG.cginc"

            UNITY_INSTANCING_BUFFER_START(Props)
                UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
            UNITY_INSTANCING_BUFFER_END(Props)

            struct appdata { float4 vertex : POSITION; UNITY_VERTEX_INPUT_INSTANCE_ID };
            struct v2f { float4 pos : SV_POSITION; UNITY_VERTEX_INPUT_INSTANCE_ID };

            v2f vert(appdata v) {
                v2f o;
                UNITY_SETUP_INSTANCE_ID(v);
                UNITY_TRANSFER_INSTANCE_ID(v, o);
                o.pos = UnityObjectToClipPos(v.vertex);
                return o;
            }
            fixed4 frag(v2f i) : SV_Target {
                UNITY_SETUP_INSTANCE_ID(i);
                return UNITY_ACCESS_INSTANCED_PROP(Props, _Color);
            }
            ENDCG
        } }
    }
    ```
  - Once the shader supports it and "Enable GPU Instancing" is checked on the material, **ordinary GameObjects** sharing that mesh+material are automatically batched by Unity's renderer into fewer draw calls — no separate instancing-specific node/object type needed. Per-instance values (like `_Color` above) are typically set via a `MaterialPropertyBlock` per renderer.

## Godot: Shaders & MultiMesh
- **What a shader is**: a `Shader` resource written in Godot's own shading language (`.gdshader`, GLSL-like syntax, Godot-specific built-ins). `shader_type` declares its target: `spatial` (3D), `canvas_item` (2D), `particles`, `sky`, or `fog`.
- **Material relationship**: a `ShaderMaterial` references a `Shader` and supplies uniform values via `set_shader_parameter()`/`get_shader_parameter()` — uniform names in code must match exactly.
- **GPU instancing**: not a shader flag — it's a **distinct resource/node type**, `MultiMesh` (used through `MultiMeshInstance3D`/`MultiMeshInstance2D`). Set `instance_count`, then per instance: `set_instance_transform()`, `set_instance_color()` (needs `use_colors = true`), and `set_instance_custom_data()` (needs `use_custom_data = true`; data is packed into a `Color`, i.e. **4 floats per instance**). The shader reads this via the built-in `INSTANCE_CUSTOM` in the vertex stage.
  - Minimal instanced shader (`.gdshader`):
    ```glsl
    shader_type spatial;

    void vertex() {
        // INSTANCE_CUSTOM.rgb holds up to 3 custom floats set via set_instance_custom_data()
        COLOR = vec4(INSTANCE_CUSTOM.rgb, 1.0);
    }

    void fragment() {
        ALBEDO = COLOR.rgb;
    }
    ```
  - Setup (GDScript):
    ```gdscript
    var multimesh := MultiMesh.new()
    multimesh.mesh = my_mesh
    multimesh.use_custom_data = true
    multimesh.instance_count = 1000

    for i in range(1000):
        var t := Transform3D(Basis(), Vector3(i * 2.0, 0, 0))
        multimesh.set_instance_transform(i, t)
        multimesh.set_instance_custom_data(i, Color(randf(), randf(), randf(), 1.0))

    $MultiMeshInstance3D.multimesh = multimesh
    ```
  - **Caveats from the docs**: only 4 custom floats per instance (a real capacity constraint); once a MultiMesh batch's shared light budget is consumed, remaining instances get no further lighting; the whole batch is spatially indexed as one (distant instances may still render, hurting culling); blend shapes are ignored.

## Comparison / Mapping
| Concept | Unity | Godot |
|---|---|---|
| Shader language | HLSL via ShaderLab wrapper, or visual Shader Graph | Godot Shading Language (`.gdshader`, GLSL-like) |
| Pipeline targeting | Determined by active render pipeline (Built-in/URP/HDRP) + Pass structure | Explicit `shader_type`: spatial/canvas_item/particles/sky/fog |
| Material↔Shader link | `Material` → `Shader`, values via Inspector/`MaterialPropertyBlock` | `ShaderMaterial` → `Shader`, values via `set_shader_parameter()` |
| Instancing activation | Per-shader opt-in flag (`#pragma multi_compile_instancing`); ordinary GameObjects auto-batch | Explicit `MultiMesh` resource/node — a different construct entirely, not a flag on normal nodes |
| Per-instance data capacity | Typed instancing buffer, limited mainly by GPU constant-buffer size (comparatively generous) | Fixed: transform + color + **4 floats** of custom data (`INSTANCE_CUSTOM`) — a hard, small cap |
| Are instances full scene objects? | Yes — each is still an ordinary GameObject with components | **No** — MultiMesh instances are pure rendering data (indices into a buffer), not Nodes; no per-instance script/physics/collision |

## Why each engine designed it this way
- **Unity**: because Unity's renderer already treats every `Renderer` as an individual object flowing through a common pipeline, GPU instancing is layered in as a pipeline *optimization*: declare shader support, enable it on the material, and the engine transparently batches same-mesh/same-material GameObjects at render time. This keeps the scene-authoring model unchanged (still ordinary GameObjects) — the cost is that per-shader instancing setup is verbose (several macros), and batching can silently fail to trigger if a shader variant or material mismatch occurs.
- **Godot**: `MultiMesh` is a deliberate, separate, explicit resource — consistent with Godot's general "pay for what you use" pattern. Choosing instancing means opting into a genuinely different representation (data in a buffer, not Nodes), which trades away per-instance scene-graph conveniences (no individual signals/physics/scripts) for very predictable, explicit performance. This also connects to the earlier [SRP/URP vs RenderingDevice/Compositor](../srp-urp-vs-renderingdevice-compositor/research.md) finding: Godot doesn't support swappable custom render pipelines the way Unity's SRP does, so it doesn't need a ShaderLab-style indirection layer either — one shading language, tied directly to the engine's fixed renderer, is enough.

## Key migration risk
If the Unity project's GPU-instanced objects **also carry per-object gameplay logic** (colliders, scripts, individual behavior per GameObject), a direct port to `MultiMesh` will lose that — MultiMesh instances have no individual Node identity, so there's nothing to attach a collider/script to per instance. The right migration shape in that case is a **hybrid**: keep a lightweight external data model (e.g. an array of structs/Resources) driving gameplay logic, and separately push transforms/custom data into a `MultiMesh` purely for rendering, each frame or on change.

## Follow-ups / things to verify later
- Confirm whether the project's instanced GameObjects carry per-object gameplay components (colliders/scripts) — decides whether a hybrid data-model + MultiMesh approach is needed, per the migration risk above.
- Inventory how much per-instance custom data the project's instancing shaders actually use beyond transform/color — Godot's 4-float cap may require packing tricks or a separate per-instance data texture indexed by instance ID if more is needed.
- Confirm which Unity render pipeline (Built-in/URP/HDRP) the instanced shaders target — affects exact HLSL syntax/includes, and ties back to the [SRP/URP research entry](../srp-urp-vs-renderingdevice-compositor/research.md).
- Check whether Shader Graph is used in the Unity project — Godot's visual shader editor (`VisualShader`) is the closer authoring-tool analog and may warrant its own research entry if used heavily.
