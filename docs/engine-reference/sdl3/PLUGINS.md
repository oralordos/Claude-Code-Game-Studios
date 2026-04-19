# SDL3 — Optional Libraries & Common Third-Party Dependencies

**Last verified:** 2026-04-18

SDL3 is a framework, not an engine. Most projects will pull in a handful of
additional libraries to cover subsystems SDL does not ship. This document
indexes the common ones.

---

## How to Use This Guide

**✅ Part of the SDL family** — Maintained by libsdl-org, tracks core SDL versioning
**🟡 Third-party, widely adopted** — Solid community, stable APIs
**⚠️ Experimental / niche** — Use with caution, verify still maintained
**📦 Header-only** — No build integration beyond adding to include path

---

## Official SDL Satellite Libraries

### ✅ SDL_image 3.x
- **Purpose:** Decode image files (PNG, JPG, WebP, AVIF, GIF, BMP, etc.) into `SDL_Surface` or `SDL_Texture`
- **When to use:** Any game loading image assets other than raw BMP
- **Status:** Production-Ready
- **Package:** vcpkg `sdl3-image`, apt `libsdl3-image-dev`, or CMake `FetchContent`
- **API:** `IMG_Load(path)` → `SDL_Surface*`, `IMG_LoadTexture(renderer, path)` → `SDL_Texture*`
- **Official:** https://github.com/libsdl-org/SDL_image

---

### ✅ SDL_ttf 3.x
- **Purpose:** TrueType / OpenType font rendering via FreeType
- **When to use:** Any UI / HUD text rendering that isn't bitmap fonts
- **Status:** Production-Ready
- **Package:** vcpkg `sdl3-ttf`, apt `libsdl3-ttf-dev`, or CMake `FetchContent`
- **API:** `TTF_OpenFont`, `TTF_RenderText_Blended`, `TTF_RenderText_Solid`, `TTF_RenderText_Shaded`
- **Official:** https://github.com/libsdl-org/SDL_ttf

---

### ✅ SDL_mixer 3.x
- **Purpose:** High-level audio mixing — channels, music, decoded formats (OGG, MP3, FLAC, MOD)
- **When to use:** Simple game audio where `SDL_AudioStream` is too bare-bones
- **Status:** Production-Ready (rewritten on top of SDL3 audio streams)
- **Package:** vcpkg `sdl3-mixer`, apt `libsdl3-mixer-dev`, or CMake `FetchContent`
- **Note:** For advanced needs (adaptive music, runtime DSP), consider FMOD, Wwise, or miniaudio instead
- **Official:** https://github.com/libsdl-org/SDL_mixer

---

### ✅ SDL_net 3.x
- **Purpose:** Low-level TCP/UDP sockets, DNS resolution
- **When to use:** Simple networking — HTTP polling, lobby queries, turn-based games
- **Status:** Production-Ready
- **⚠️ Not suitable for:** Real-time multiplayer game state sync (no built-in reliability layer over UDP, no prediction/rollback)
- **Alternatives:** ENet, yojimbo, GameNetworkingSockets, or a hand-rolled UDP protocol
- **Official:** https://github.com/libsdl-org/SDL_net

---

### ✅ SDL_shadercross
- **Purpose:** Cross-compile one HLSL shader to SPIR-V (Vulkan), DXIL (D3D12), and MSL (Metal) for SDL_gpu
- **When to use:** Any project using SDL_gpu — unless you enjoy maintaining three shader sources
- **Status:** Production-Ready (tracks SDL3 releases)
- **CLI tool:** `shadercross input.hlsl -s VERTEX -o out.spv --target SPIRV`
- **CMake integration:** Add a custom command per shader; depend on generated artifacts
- **Official:** https://github.com/libsdl-org/SDL_shadercross

---

## Commonly Paired Third-Party Libraries

### 🟡 Dear ImGui
- **Purpose:** Immediate-mode debug/tool UI
- **When to use:** Dev tools, debug overlays, level editors, anything not shipped to players
- **⚠️ Not for:** Player-facing UI in shipped games — looks programmer-art by default
- **Backend:** `imgui_impl_sdl3.cpp` + `imgui_impl_sdlgpu3.cpp` (or `imgui_impl_sdlrenderer3.cpp`)
- **License:** MIT
- **Official:** https://github.com/ocornut/imgui

---

### 🟡 Nuklear
- **Purpose:** Single-header immediate-mode UI (C)
- **When to use:** Very lightweight UI needs, C codebases
- **License:** MIT / public domain
- **Official:** https://github.com/Immediate-Mode-UI/Nuklear

---

### 🟡 RmlUi
- **Purpose:** HTML/CSS-inspired retained-mode UI — suitable for shipped games
- **When to use:** Player-facing UI when you don't want to hand-author everything
- **License:** MIT
- **Official:** https://github.com/mikke89/RmlUi

---

### 🟡 EnTT
- **Purpose:** Fast, header-only Entity Component System for C++
- **When to use:** Any SDL3 game with more than a handful of entities; DOD architecture
- **License:** MIT
- **Official:** https://github.com/skypjack/entt

---

### 🟡 glm
- **Purpose:** Header-only GLSL-style vector/matrix math (`vec3`, `mat4`, `quat`, etc.)
- **When to use:** Any 3D math; anywhere you need vectors beyond `SDL_FPoint`
- **License:** MIT
- **Official:** https://github.com/g-truc/glm

---

### 🟡 Box2D v3
- **Purpose:** Fast 2D rigid body physics
- **When to use:** 2D games with rigid body physics (platformers, top-down, puzzle physics)
- **License:** MIT
- **Note:** Box2D v3 is a rewrite of the classic v2 — API is different; check the migration guide
- **Official:** https://github.com/erincatto/box2d

---

### 🟡 Jolt Physics
- **Purpose:** Production-grade 3D physics (used in Horizon Forbidden West)
- **When to use:** 3D games with complex physics
- **License:** MIT
- **Official:** https://github.com/jrouwe/JoltPhysics

---

### 🟡 miniaudio
- **Purpose:** Single-header audio library with decoding, spatialization, mixing
- **When to use:** When SDL_mixer is too limited but you don't want FMOD/Wwise complexity
- **License:** MIT-0 / public domain
- **Official:** https://github.com/mackron/miniaudio

---

### 🟡 FMOD Studio / Wwise
- **Purpose:** Industry-standard audio middleware with designer tools
- **When to use:** Shipped commercial games with complex adaptive audio
- **License:** Commercial (free under revenue thresholds)
- **Note:** Replaces SDL audio entirely — these libraries own the audio thread

---

### 🟡 cgltf / tinygltf
- **Purpose:** glTF 2.0 model loading
- **When to use:** 3D games loading authored assets
- **License:** MIT
- **Official:**
  - https://github.com/jkuhlmann/cgltf (C, single header)
  - https://github.com/syoyo/tinygltf (C++)

---

### 🟡 stb_* (image, truetype, vorbis, ...)
- **Purpose:** Sean Barrett's single-header utility libraries
- **When to use:** When you want a lighter dep than SDL_image / SDL_ttf, or for specific formats SDL's libs don't cover
- **License:** MIT / public domain
- **Official:** https://github.com/nothings/stb

---

### 🟡 fmt / spdlog
- **Purpose:** `fmt` — modern type-safe formatting (C++20 `<format>` predecessor); `spdlog` — fast header-only logging
- **When to use:** Any C++ codebase wanting better than `printf` / `SDL_Log` — though SDL_Log is fine
- **License:** MIT
- **Official:** https://github.com/fmtlib/fmt, https://github.com/gabime/spdlog

---

### 🟡 PhysicsFS
- **Purpose:** Archive-aware virtual filesystem (ZIP, 7z, custom formats) on top of file IO
- **When to use:** Shipping asset bundles as ZIP / 7z instead of loose files; mod loaders that mount multiple archive search paths
- **Overlaps with SDL_Storage:** SDL3's `SDL_Storage` covers portable save/title data and directory enumeration, but it does NOT decompress archives. If you only need "write saves" + "read bundled loose files", `SDL_Storage` alone is enough. If you need to mount a `.zip` as a read-only tree, PhysicsFS is still the answer.
- **Alternative integration:** Implement a custom `SDL_StorageInterface` backed by PhysicsFS so the rest of the codebase only sees `SDL_Storage`. Keeps the archive format swappable later.
- **License:** zlib
- **Official:** https://icculus.org/physfs/

---

### 🟡 Catch2 / doctest
- **Purpose:** C++ unit testing frameworks
- **When to use:** `/test-setup` will scaffold doctest by default for SDL3 projects (lightest, compile-fast)
- **License:** Boost / MIT
- **Official:** https://github.com/catchorg/Catch2, https://github.com/doctest/doctest

---

## Quick Decision Guide

**I need to load PNG/JPG** → **SDL_image**
**I need to render text** → **SDL_ttf** (or stb_truetype for zero deps)
**I need simple music + SFX** → **SDL_mixer**
**I need adaptive/interactive music** → **FMOD Studio** or **Wwise**
**I need debug/tool UI** → **Dear ImGui**
**I need player-facing UI** → **RmlUi**, or author sprites + SDL_ttf
**I need 2D physics** → **Box2D v3**
**I need 3D physics** → **Jolt Physics**
**I need ECS** → **EnTT**
**I need 3D math** → **glm**
**I need to load 3D models** → **cgltf / tinygltf**
**I need shipping multiplayer** → **ENet** or **GameNetworkingSockets** (NOT SDL_net)
**I need a shader pipeline** → **SDL_shadercross**
**I need save games that work on console** → `SDL_OpenUserStorage` (built-in, see `modules/storage.md`) — no plugin needed
**I need bundled game data that works on Android / console** → `SDL_OpenTitleStorage` (built-in) — no plugin needed
**I need to ship assets inside a .zip** → **PhysicsFS**, optionally fronted by a custom `SDL_StorageInterface`
**I need async file reads for streaming** → `SDL_AsyncIO` (built-in, see `modules/storage.md`) — no plugin needed

---

## On-Demand WebSearch Strategy

For libraries NOT listed above, when a user asks:

1. **WebSearch** for latest documentation and SDL3 compatibility
2. Verify:
   - Compatible with SDL3 (not stuck on SDL2)
   - Still maintained (commit activity in the last year)
   - License is project-appropriate
3. Optionally cache findings in `docs/engine-reference/sdl3/plugins/[name].md` for future reference.

---

**Last Updated:** 2026-04-18
**SDL Version:** SDL 3.2 stable series
**LLM Knowledge Cutoff:** January 2026
