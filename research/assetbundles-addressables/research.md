# AssetBundles / Addressables (Unity) vs PCKPacker / Resource Packs (Godot)

## Official sources
- Unity AssetBundles intro: https://docs.unity3d.com/cn/2022.3/Manual/AssetBundlesIntro.html
- Godot PCKPacker class: https://docs.godotengine.org/zh-tw/4.x/classes/class_pckpacker.html
- Godot exporting packs/patches/mods: https://docs.godotengine.org/en/stable/tutorials/export/exporting_pcks.html
- Godot ProjectSettings.load_resource_pack / ResourceLoader: https://docs.godotengine.org/en/stable/classes/class_resourceloader.html

## Unity: AssetBundles & Addressables
- **AssetBundles**: archive files packaging non-code assets (textures, audio, prefabs, ScriptableObject data) that can be loaded/downloaded at runtime, independent of the base build.
  - Purposes: DLC, reducing initial install size, platform-specific optimized assets, reducing runtime memory pressure.
  - Bundles can depend on each other (e.g. a material bundle referencing a texture bundle).
  - Compression options: LZMA (whole-bundle, smaller but slower to decompress) vs LZ4 (chunked, faster).
  - Can contain serialized objects (incl. ScriptableObjects), but the C# class *definitions* must already be compiled into the app — bundles carry data, not code.
- **Addressables**: a higher-level package built on top of AssetBundles, adding:
  - Addressable keys/labels/groups instead of manual bundle-path management.
  - A catalog + remote catalog updates (ship new content without a full app rebuild).
  - Reference-counted async load/unload (`Addressables.LoadAssetAsync` / `Release`).
  - Automatic dependency & bundle-layout management.

## Godot: PCKPacker & Resource Packs
- Godot's analog to an "asset bundle" is a `.pck` file — a packed resource archive.
- `PCKPacker` builds a `.pck` at runtime/export time:
  - `pck_start(path, alignment, key, encrypt_directory)` — start a new pck
  - `add_file(target_path, source_path, encrypt)` — add a file
  - `add_file_from_buffer(target_path, data, encrypt)` — add from raw bytes
  - `add_file_removal(target_path)` — mark a file for removal (used for patches)
  - `flush(verbose)` — finalize and write; also auto-flushes on destruction
- Loading at runtime: `ProjectSettings.load_resource_pack("res://mod.pck")`, then load assets normally via `load()` / `ResourceLoader`.
- **Caveat**: if a file inside the pack shares a `res://` path with an existing file, it silently replaces it. Intentional for patches/mods, but a footgun for DLC namespacing if not planned for.
- No built-in catalog, versioning, labels/groups, or reference-counted async unloading — this is a low-level primitive, not an asset-management framework.
- Godot `Resource` objects are `RefCounted`, so unloading is largely automatic once nothing references a resource — there's no direct equivalent of `AssetBundle.Unload()` / `Resources.UnloadUnusedAssets()` to call manually in most cases.

## Answering the specific questions

**1. Is there an official Godot package equivalent to Addressables?**
No. Godot ships only the low-level primitive (`PCKPacker` + `ProjectSettings.load_resource_pack()` + `ResourceLoader`). No official package provides Addressables-style catalogs, labels/groups, remote content versioning, or reference-counted load/unload.

**2. Is it possible to customize an AssetBundle-like system in Godot?**
Yes — expected, not exotic. Typical shape:
- Export separate `.pck` files per content group (≈ "bundle").
- Download packs via `HTTPRequest` into `user://`.
- Call `ProjectSettings.load_resource_pack()` once downloaded.
- Maintain your own catalog (JSON or a custom `Resource`) listing available packs, versions, and asset paths — this replaces Addressables' catalog/labels.
- Since resources are ref-counted, "unloading" mostly falls out of dropping references — but if hot-swapping updated content mid-run matters, you need your own invalidation logic, since Godot won't auto-detect that a loaded resource has been superseded by a newer pack.

**3. Official vs custom — which is better?**
There's no true official Addressables-equivalent to pick *instead of* custom work, so the real choice is "thin custom layer on the official `PCKPacker` primitive" (recommended) vs. "a from-scratch bundle system that ignores `.pck` entirely" (no good reason to do this — don't reinvent packing/loading). Scope the custom layer to what's actually used: if Addressables here is mainly used for organizational lazy-loading rather than live-updatable DLC, the Godot version can be much simpler than a full Addressables clone.

## Follow-ups / things to verify later
- Confirm which Addressables features this specific project actually relies on (remote catalog updates? memory-budget/group loading? or just simple lazy loading?) — this determines how much custom tooling is really needed.
- Check Godot's `.pck` encryption support (key-based, per docs) if the project encrypts its AssetBundles.
- Check for community addons covering catalog/versioning before committing to fully custom code — evaluate trust/maintenance risk before adopting one.
