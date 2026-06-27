# Source: https://docs.pjthikari.dev/docs/midnight-purple-sa

MTA:SA ForkActive

# Midnight Purple: San Andreas

An independent fork of Multi Theft Auto: San Andreas that extends the platform with a second audio engine (FMOD) running alongside the native BASS stack. The goal is to enable advanced 3D audio, environmental effects, and engine sound simulation in Lua scripts — without breaking any existing script that already uses MTA's native audio functions.

**BASS remains untouched.**

All existing functions (\`playSound\`, \`playSound3D\`, \`setSoundVolume\`, etc.) continue to work exactly as before. FMOD is an additive, parallel system exposed through its own set of \`fmod\*\` Lua globals.

`5b7f91166`Upstream base:synced 2026-05-16. This fork is not affiliated with, endorsed by, or supported by the MTA team.

## What's New

### FMOD Audio Engine

A new \`CFMODManager\` (\`Client/core/\`) wraps the FMOD Core API and is updated every frame from \`DoPostFramePulse()\`, keeping the 3D listener position and orientation in sync with the game camera.

- FMOD initialised in right-handed coordinate mode to match GTA:SA's world axes (Z up, Y north).
- All FMOD sounds route through a master `ChannelGroup`. Three sub-groups sit under it — **SFX** (default), **Ambient**, and **Music** — each with an independent volume scalar.
- Automatic channel cleanup every frame — finished channels are purged from the internal map without manual book-keeping from scripts.
- Sounds start paused, 3D attributes are applied, then unpaused — preventing a single-frame positional pop on spawn.
- **Per-resource tracking** — every sound ID is registered against the `CLuaMain` that created it. When a resource stops, all its FMOD sounds are automatically freed.

## New Lua API (\`fmod\*\` globals)

All functions are available client-side. Sound and channel IDs are plain integers managed by \`CFMODManager\`.

#### Sound lifetime

| Function | Returns | Description |
| --- | --- | --- |
| `fmodCreateSound(path, [is3D=false], [loop=false])` | `soundId / false` | Load a sound asset from a resource path. |
| `fmodCreateStream(path, [is3D=false], [loop=false])` | `soundId / false` | Stream a long audio file from disk (music-quality, lower RAM). |
| `fmodCreateSoundFromMemory(data, [is3D=false], [loop=false])` | `soundId / false` | Load from a raw audio data string (e.g. TEA-decrypted .wavlocked contents). FMOD copies the buffer internally. |
| `fmodFreeSound(soundId)` | `bool` | Release the sound and free its memory. |

#### Playback

| Function | Returns | Description |
| --- | --- | --- |
| `fmodPlaySound(soundId, [x, y, z, [minDist=1, maxDist=100]])` | `channelId / false` | Spawn a playing instance of a sound. Omit position for a 2D (non-spatialized) channel. |
| `fmodStopSound(channelId)` | `bool` | Stop and remove the channel. |
| `fmodPauseSound(channelId)` | `bool` | Pause a channel without removing it. |
| `fmodResumeSound(channelId)` | `bool` | Resume a paused channel. |
| `fmodIsSoundPaused(channelId)` | `bool` | Returns true if the channel is paused. |
| `fmodIsSoundPlaying(channelId)` | `bool` | Returns true if the channel is still active and not paused. |

#### Per-channel control

| Function | Returns | Description |
| --- | --- | --- |
| `fmodSetSoundVolume(channelId, volume)` | `bool` | Volume 0.0 – 1.0. |
| `fmodGetSoundVolume(channelId)` | `number` | Read the current volume. |
| `fmodSetSoundPitch(channelId, pitch)` | `bool` | 1.0 = normal, 2.0 = one octave up. |
| `fmodGetSoundPitch(channelId)` | `number` | Read the current pitch multiplier. |
| `fmodSetSoundPosition(channelId, x, y, z)` | `bool` | Update world position for a 3D channel. |
| `fmodGetSoundPosition(channelId)` | `x, y, z` | Read the current world position. Returns false on a 2D channel. |
| `fmodSetSoundVelocity(channelId, vx, vy, vz)` | `bool` | Set world-space velocity for Doppler pitch shift. |
| `fmodSetSoundLooped(channelId, loop)` | `bool` | Change loop mode on a live channel without restarting it. |
| `fmodGetSoundLooped(channelId)` | `bool` | Read the current loop state. |

#### Environmental reverb

| Function | Returns | Description |
| --- | --- | --- |
| `fmodSetReverbPreset(preset)` | `bool` | Set global reverb. Presets: "off", "generic", "room", "bathroom", "cave", "city", "alley", "forest", "mountains", "plain", "sewerpipe", "underwater", "stonecorridor", "hallway", "stoneroom", "auditorium", "concerthall", "arena", "hangar", "parkinglot", "paddedcell", "livingroom", "quarry". |
| `fmodSetReverbWetLevel(wetDB)` | `bool` | Override the output level (−80 to +20 dB) without changing the preset character. |
| `fmodSetChannelReverb(channelId, wetDB)` | `bool` | Connect a channel to the reverb bus. Channels are dry by default; call this to opt in. |

#### DSP effects

| Function | Returns | Description |
| --- | --- | --- |
| `fmodSetChannelEcho(channelId, delayMS, feedbackPct, wetDB)` | `bool` | Attach or update an echo/delay effect. |
| `fmodRemoveChannelEcho(channelId)` | `bool` | Remove the echo DSP. |
| `fmodSetChannelLowPass(channelId, cutoffHz, [resonance=1.0])` | `bool` | Attenuate frequencies above cutoffHz. Useful for distance muffle or interior filtering. |
| `fmodRemoveChannelLowPass(channelId)` | `bool` | Remove the low-pass filter. |
| `fmodSetChannelHighPass(channelId, cutoffHz, [resonance=1.0])` | `bool` | Attenuate frequencies below cutoffHz. |
| `fmodRemoveChannelHighPass(channelId)` | `bool` | Remove the high-pass filter. |
| `fmodSetChannelFlanger(channelId, mix, depth, rate)` | `bool` | Cyclic phase-modulation effect. mix 0–100%, depth 0.01–1.0, rate 0–20 Hz. |
| `fmodRemoveChannelFlanger(channelId)` | `bool` | Remove the flanger DSP. |
| `fmodSetChannelChorus(channelId, mix, depth, rate)` | `bool` | Pitch-varied doubling effect. mix 0–100%, depth 0–100%, rate 0–20 Hz. |
| `fmodRemoveChannelChorus(channelId)` | `bool` | Remove the chorus DSP. |
| `fmodSetChannelDistortion(channelId, level)` | `bool` | Hard-clipping saturation. level 0.0 (clean) – 1.0 (full clip). |
| `fmodRemoveChannelDistortion(channelId)` | `bool` | Remove the distortion DSP. |
| `fmodGetChannelEffects(channelId)` | `string / false` | Returns a comma-separated string of active DSP names (e.g. "Echo,LowPass"), or false on error. |
| `fmodSetChannelOcclusion(channelId, directOcclusion [, reverbOcclusion])` | `bool` | Apply geometry-aware muffling. directOcclusion 0.0 (none) – 1.0 (fully blocked). FMOD applies a frequency-dependent LPF. |

#### Volume categories

| Function | Returns | Description |
| --- | --- | --- |
| `fmodSetChannelCategory(channelId, category)` | `bool` | Move a live channel to a category. Accepts "sfx", "ambient", "music" or integers 0, 1, 2. Default is "sfx". |
| `fmodSetCategoryVolume(category, volume)` | `bool` | Scale all channels in a category (0.0 – 1.0). |
| `fmodGetCategoryVolume(category)` | `number` | Read the current category volume scalar. |

#### Master volume

| Function | Returns | Description |
| --- | --- | --- |
| `fmodSetMasterVolume(volume)` | `bool` | Scale all FMOD sounds together (0.0 – 1.0, default 0.5). |
| `fmodGetMasterVolume()` | `number` | Read the current FMOD master volume. |

#### Diagnostics

| Function | Returns | Description |
| --- | --- | --- |
| `fmodGetVersion()` | `string` | FMOD Core library version (e.g. "2.02.21"). |
| `fmodGetLastError()` | `errorCode, errorString` | Read the last FMOD result code and its human-readable description. |

#### Named float parameters

| Function | Returns | Description |
| --- | --- | --- |
| `fmodSetParameter(name, value)` | `bool` | Store a named float for use by the ambience scripting layer. |
| `fmodGetParameter(name [, default=0])` | `number` | Read back a stored parameter. |

## Roadmap

- ✓Doppler effect — fmodSetSoundVelocity feeds velocity to set3DAttributes
- ✓Pause / resume — fmodPauseSound / fmodResumeSound / fmodIsSoundPaused
- ✓Streaming audio — fmodCreateStream uses FMOD\_CREATESTREAM
- ✓Loop state toggle — change loop mode on a live channel without restarting
- ✓Read-back functions — GetSoundVolume, GetSoundPitch, GetSoundPosition, GetSoundLooped
- ✓Additional DSP types — low-pass, high-pass, flanger, chorus, distortion
- ✓DSP chain query — fmodGetChannelEffects returns active DSP names
- ✓Per-resource sound tracking — sounds auto-freed when resource stops
- ✓Volume categories — SFX, Ambient, Music sub-groups
- ✓Server-triggered playback — fmod\_sync resource with client cache
- ✓Occlusion / obstruction — fmodSetChannelOcclusion with native set3DOcclusion
- ✓Engine sound layer — hikari\_Engine fully migrated to FMOD
- ✓Per-vehicle sound bank — engine\_table.lua with fmodCreateSoundFromMemory
- ✓Rev limiter / gear shift events — one-shot BOV and backfire/ALS via FMOD
- ✓Exhaust 3D positioning — bind to exhaust bone with fallback chain
- ✓Tunnel echo & reverb — raycast-based cover detection with hysteresis
- ✓Air-absorption LPF — distance-proportional low-pass for vehicles beyond 20 m
- ✓Rev limiter distortion — high-RPM layer gets brief fmodSetChannelDistortion
- ✓Occlusion per vehicle — throttled LOS raycast (200 ms) per vehicle
- ✓FMOD error propagation — all failures return false, errorCode, errorString
- ✓fmodGetVersion() — returns the FMOD Core library version string
- ✓Version label — in-game label reads Midnight Purple 1.7

## Build Instructions

### Windows

Windows Prerequisites

- Visual Studio 2022 — Desktop development with C++ + C++ MFC for latest v145 build tools
- Microsoft DirectX SDK June 2010
- Git for Windows (optional)

Additional requirement for this fork

- FMOD Core SDK (x86) → `vendor\fmod\` (`vendor\fmod\inc\` and `vendor\fmod\lib\x86\`)

Steps

1. Run `win-create-projects.bat` to generate the Visual Studio solution.
2. Open `Build\MTASA.sln`.
3. Compile (target: **Win32 / Release**).
4. Run `win-install-data.bat` to copy runtime data.

```
win-build.bat [Debug|Release] [Win32|x64]
```

### GNU/Linux (server only)

```
./linux-build.sh [--arch=x86|x64|arm|arm64] [--config=debug|release] [--cores=<n>]
./linux-install-data.sh   # optional
```

## License

Unless otherwise specified, all source code hosted on this repository is licensed under the GPLv3 license.

Grand Theft Auto and all related trademarks are © Rockstar North 1997–2026. 
Multi Theft Auto is © the Multi Theft Auto team. This fork is an independent project.