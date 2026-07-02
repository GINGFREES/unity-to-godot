# Audio (Unity) vs Audio (Godot)

## Official sources
- Unity Audio Overview: https://docs.unity3d.com/6000.4/Documentation/Manual/AudioOverview.html
- Godot Audio streams: https://docs.godotengine.org/zh-tw/4.x/tutorials/audio/audio_streams.html
- Godot Sync with audio: https://docs.godotengine.org/zh-tw/4.x/tutorials/audio/sync_with_audio.html
- `share.google/aimode/5eh0WFrMHH7B3IkNp` — **not fetchable** (same authenticated-session redirect as prior aimode links in this research set). Not incorporated; paste content directly if it matters.

## Unity: Audio system
- **AudioSource / AudioListener**: sound is emitted by `AudioSource` components on objects and heard by an `AudioListener` (typically on the Camera) — this source/listener split is what drives spatial audio simulation.
- **AudioClip**: the imported audio data container. Supports mono, stereo, and multichannel; accepted formats include AIFF, WAV, MP3, Ogg, and tracker module formats (.xm, .mod, .it, .s3m).
- **AudioMixer**: a mixing graph for combining multiple sources, applying effects, and mastering.
- **Spatial/3D audio**: distance-based attenuation and positional simulation between `AudioSource` and `AudioListener`; **Doppler Effect** support based on relative velocity between them.
- **Echo/reverb caveat**: Unity **cannot calculate echo/reverb from scene geometry automatically** — this must be implemented manually via Audio Filters or Reverb Zones.
- **Microphone class**: runtime audio recording from an available microphone, via script.

## Godot: Audio system
- **Player nodes split by spatial dimensionality** (unlike Unity's single `AudioSource` + a spatial-blend setting):
  - `AudioStreamPlayer` — non-positional, outputs to any audio bus.
  - `AudioStreamPlayer2D` — 2D positional; stereo panning based on proximity to screen edges; an `Area2D` can reroute its output to a specific bus.
  - `AudioStreamPlayer3D` — 3D positional; supports stereo/5.1/7.1 surround depending on configuration; an `Area3D` can similarly control routing.
- **AudioStream resources**: audio files load as `AudioStream` (and format-specific subclasses) and are assigned to a player node — the same data/player split Unity has via `AudioClip`+`AudioSource`.
- **AudioStreamRandomizer**: a built-in resource that randomly picks between multiple streams per playback, optionally varying pitch/volume — a native convenience Unity doesn't bake in (typically hand-rolled in Unity by randomly indexing an `AudioClip[]`).
- **Reverb via buses**: `AudioStreamPlayer3D` can send dry and reverb signals to separate buses when entering an `Area3D`, enabling per-room acoustic effects — Godot's answer to Unity's Reverb Zones, implemented through the bus system + `Area3D` rather than a dedicated zone component.
- **Doppler Effect**: supported the same way conceptually — tracks relative velocity between `AudioStreamPlayer3D` and the `Camera`.
- **Audio buses**: Godot's mixing graph (the Audio panel), with per-bus effects — the structural equivalent of Unity's `AudioMixer` groups.
- **Web platform caveat**: reverb buses and the Doppler effect are **not supported on Web export** when using the default Sample playback mode — they only work in Stream mode.
- **Syncing gameplay with audio** (relevant for rhythm games / tight audio-visual sync): Godot documents two explicit approaches:
  1. **System clock method** (recommended for short tracks): use `Time.get_ticks_usec()`, compensating with `AudioServer.get_time_to_next_mix()` and `AudioServer.get_output_latency()`. Drifts gradually over long playback.
  2. **Hardware clock method** (for long/continuous playback): combine `AudioStreamPlayer.get_playback_position()` with `AudioServer.get_time_since_last_mix()`, further refined by subtracting `AudioServer.get_output_latency()`. Can show slight jitter from multi-threading — validate that values don't decrease frame-to-frame and discard anomalies.
  - Latency sources called out: audio buffer size (adjustable in project settings, trades latency for CPU load/glitch risk), block-based mixing, graphics API delay (2-3 frames), and TV/display processing pipelines.

## Comparison / Mapping
| Unity | Godot | Notes |
|---|---|---|
| `AudioListener` (on Camera) | Implicit: the active `Camera2D`/`Camera3D` acts as the listener; dedicated `AudioListener2D`/`AudioListener3D` nodes may exist for overriding this | Not confirmed from the docs fetched this pass — see follow-ups |
| `AudioSource` (+ spatial blend setting) | `AudioStreamPlayer` / `AudioStreamPlayer2D` / `AudioStreamPlayer3D` | Godot splits by node type instead of one component with a spatial-blend slider |
| `AudioClip` | `AudioStream` (`AudioStreamOggVorbis`, `AudioStreamMP3`, `AudioStreamWAV`, etc.) | Same data/player split concept |
| `AudioMixer` (groups, effects, mastering) | Audio buses (Audio panel, per-bus effects) | Structurally equivalent signal-routing graph |
| Manual random-clip-selection scripting | `AudioStreamRandomizer` (built-in) | Godot bakes this in natively |
| Reverb Zones (Audio Filter/Reverb Zone component) | `Area3D` + `AudioStreamPlayer3D` reverb bus routing | Different mechanism, similar effect |
| Doppler (`AudioSource` + relative velocity) | `AudioStreamPlayer3D` + `Camera` relative velocity | Same concept |
| `Microphone` class (runtime recording) | *(not confirmed in this pass)* | Godot likely has an equivalent audio-input mechanism — verify separately |
| `AudioSettings.dspTime` / `AudioSource.PlayScheduled()` (general Unity knowledge — sample-accurate scheduling; **not confirmed from the fetched overview page**) | `AudioServer` latency-compensation APIs (`get_time_to_next_mix()`, `get_output_latency()`, `get_time_since_last_mix()`) + `AudioStreamPlayer.get_playback_position()` | Godot's docs give explicit, documented guidance for tight sync; Unity's equivalent needs confirming against its dedicated scripting docs, not just the overview page |

## Answering the likely questions

**1. What's the direct migration mapping for `AudioSource`/`AudioListener`?**
`AudioSource` → whichever `AudioStreamPlayer` variant matches the sound's spatial need (plain/2D/3D). The `AudioListener` side needs verification — Godot's default behavior (current Camera as implicit listener) may already cover most cases without an explicit listener node; confirm if the project needs an overridable listener position independent of the camera.

**2. How does Unity's `AudioMixer` map to Godot?**
Godot's bus system (Audio panel) is the structural equivalent — buses with per-bus effects, similar to `AudioMixer` groups. Confirm which specific `AudioMixer` features are used (snapshots, exposed/ducking parameters, sidechain compression) since 1:1 feature parity isn't confirmed here.

**3. How should tight audio-visual sync (e.g. rhythm-game mechanics) be handled?**
Use Godot's documented sync methods: the system-clock method for short tracks, the hardware-clock method for longer/continuous playback — both explicitly documented with latency-compensation formulas. If the Unity side currently uses `AudioSettings.dspTime`/`PlayScheduled()`-style sample-accurate scheduling, that needs its own confirmation pass against Unity's dedicated audio scripting docs (not covered by the overview page fetched here).

**4. Any platform-specific gotchas?**
If the project targets Web/HTML5 export, note that Godot's reverb-bus routing and Doppler effect don't work in the default Sample playback mode on Web — only in Stream mode.

## Follow-ups / things to verify later
- Confirm whether Godot has `AudioListener2D`/`AudioListener3D` nodes (or an equivalent) for overriding the implicit camera-as-listener behavior — not established from the docs fetched this pass.
- Confirm Godot's runtime microphone/audio-input recording API (likely `AudioEffectRecord` or similar) and compare against Unity's `Microphone` class — not covered in this pass.
- Inventory which `AudioMixer` features the project actually uses (snapshots, ducking, sidechain effects) — determines how completely Godot's bus system covers the need.
- Confirm whether the project needs rhythm-game-level audio-visual sync; if so, check Unity's actual current implementation (`dspTime`/`PlayScheduled` or something else) against Godot's two documented sync methods.
- Confirm whether the project targets Web/HTML5 export, given the reverb/Doppler Sample-vs-Stream mode limitation on that platform.
