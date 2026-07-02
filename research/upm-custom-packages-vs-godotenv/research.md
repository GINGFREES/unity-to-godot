# UPM Custom Packages (Unity) vs GodotEnv (Godot)

## Official sources
- Unity Custom Packages (UPM): https://docs.unity3d.com/6000.4/Documentation/Manual/CustomPackages.html
- GodotEnv (Chickensoft): https://github.com/chickensoft-games/godotenv

## Unity: Custom Packages (UPM)
- **Purpose**: extend Editor/project functionality with reusable, shareable units — used internally across projects/teams or distributed publicly.
- **Creation**: Package Manager's `(+) > Create package` generates a starter folder + manifest.
- **Manifest (`package.json`)**: name, version, description, dependencies — expanded manually as the package matures.
- **Code organization**: `.asmdef` assembly definition files isolate a package's C# code into its own compiled assembly.
- **Package sources**: embedded (inside the project's `Packages/` folder), local (path reference), or git (referenced by git URL in the project's manifest).
- **Versioning/distribution**: semantic versioning; changelog, license/legal files, docs, tests, and samples are recommended/expected for shareable packages.
- **Referenced from**: the consuming project's own `manifest.json` dependency list (registry name, git URL, or file path).

## Godot: GodotEnv
- **Purpose**: a third-party CLI (MIT-licensed, Chickensoft) solving two separate problems UPM bundles together for Unity — Godot *engine version* management and *addon dependency* management — since Godot itself doesn't ship either.
- **Version management**: install/use/uninstall specific Godot versions (3.x and 4.x, .NET or standard builds) across Windows/macOS/Linux; maintains a symlink to the active version and sets a `GODOT` env var.
- **Addon management**: dependencies declared in `addons.json`/`addons.jsonc` at the project root. Each entry specifies:
  - addon name (must match its install folder name)
  - source URL (git repo, local path, symlink, or zip)
  - source type (remote / local / symlink / zip)
  - git checkout ref (branch or tag)
  - optional subfolder path within the source
- **Transitive dependencies**: an addon can itself ship an `addons.json` declaring its own dependencies; GodotEnv resolves these flatly.
- **Safety**: detects local modifications to an installed addon before reinstalling (won't silently clobber local edits).
- **Core commands**: `godotenv godot install/use/uninstall [version]`, `godotenv addons init`, `godotenv addons install`, `godotenv config list/set`.

## Comparison / Mapping
| Unity (UPM) | Godot (GodotEnv) |
|---|---|
| `package.json` manifest per package | `addons.json` entry per addon |
| Registry / git URL / local path source | git repo / local path / symlink / zip source (no registry concept) |
| `.asmdef` compiled assembly isolation | No per-addon compiled-assembly isolation — Godot C# is typically one project-wide assembly |
| Built into Editor (Package Manager window, Unity's own registry, scoped registries) | Third-party CLI, not built into Godot's editor |
| Nested package dependencies resolved by UPM | Nested `addons.json` resolved flatly by GodotEnv |

**Key gap**: UPM is official tooling with an in-Editor UI and a real package registry (including scoped/private registries). Godot has no official equivalent — its built-in Asset Library is an in-editor plugin browser, not a manifest-driven, version-pinned, transitively-resolved dependency system. GodotEnv (community) is the closest functional analog, but it's git/local/zip-based only — there's no private-registry concept to migrate a private UPM scoped registry onto.

## Answering the implied questions

**1. Is there an official Godot equivalent to UPM?**
No. Godot's built-in Asset Library covers manual, in-editor plugin discovery/install but not reproducible dependency management (no git-ref pinning, no transitive resolution, no CLI/CI scripting). GodotEnv is the established community substitute and is the closer match to what UPM does.

**2. Could the project just use Godot's built-in Asset Library instead of GodotEnv?**
Only for simple, manually-managed cases. It lacks reproducible pinning, transitive resolution, local-modification detection, and CI-friendly scripted installs — all things a UPM-based custom-package workflow likely already relies on. GodotEnv (or a hand-rolled `addons.json`-equivalent) is the more faithful replacement.

**3. Migration shape**
Each existing UPM custom package becomes a Godot addon folder:
- If it's Editor-only tooling → needs a `plugin.cfg` and the `_enter_tree()`/`_exit_tree()` plugin lifecycle.
- If it's a runtime-only library → just a folder of scripts/resources, no `plugin.cfg` needed, wired in via autoload/preload.
Each addon then gets declared as an `addons.json` entry pointing at wherever its source now lives (its own git repo, or a local path during an in-progress transition).

## Follow-ups / things to verify later
- Are this project's custom UPM packages Editor-only tooling, runtime libraries, or a mix? Determines whether Godot-side addons need `plugin.cfg` at all.
- Are packages currently hosted on a private/scoped UPM registry? If so, need a git-hosting plan per package, since GodotEnv has no private-registry concept — only git/local/zip sources.
- Do any C# custom packages rely on `.asmdef`-driven assembly isolation for build-time separation or access control? Godot's typically-single project-wide C# assembly may require rethinking that boundary.
- Confirm current GodotEnv version/maturity and any known issues (e.g., Windows symlink requirements) before adopting it as the team's standard tool.
