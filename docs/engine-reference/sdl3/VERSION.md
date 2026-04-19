# SDL3 — Version Reference

| Field | Value |
|-------|-------|
| **Engine Version** | SDL 3.2 (stable series) |
| **First GA Release** | SDL 3.2.0 — January 2025 |
| **Project Pinned** | 2026-04-18 |
| **Last Docs Verified** | 2026-04-18 |
| **LLM Knowledge Cutoff** | January 2026 |
| **Risk Level** | HIGH — SDL3 GA is entirely post the LLM's deep SDL2 training data. Model knowledge of SDL3 internals, function renames, and new subsystems (SDL_gpu, SDL_main callbacks, audio streams) is partial and easy to confuse with SDL2. Always verify against this reference before suggesting APIs. |

## Framework, Not Engine

SDL3 is a **library**, not a full game engine. It provides windowing, input,
audio, 2D rendering, a cross-platform GPU API, and events. It does NOT provide:

- A scene graph or entity system (bring your own, e.g. EnTT, custom ECS)
- A physics engine (bring Box2D / Chaos / Jolt / Bullet)
- A UI framework (bring Dear ImGui / Nuklear / custom)
- An asset pipeline or editor (hand-rolled, or author externally)
- A networking abstraction beyond SDL_net (consider ENet / yojimbo for gameplay)

Teams using this template with SDL3 should expect to write more foundation code
than they would in Godot / Unity / Unreal, in exchange for full control over the
engine architecture.

## Knowledge Gap Warning

SDL 3.0.0 stable shipped in **January 2025**. The transition from SDL2 to SDL3
is one of the largest API breaks in SDL's history:

- ~1500+ function/symbol renames (consistent `SDL_VerbNoun` style)
- Error-return convention changed (`0`/`-1` in SDL2 → `bool` true/false in SDL3
  for most new APIs)
- New subsystems that have no SDL2 equivalent: **SDL_gpu**, **SDL_main
  callbacks**, **SDL_Properties**, new audio streams
- Old subsystems removed (e.g. render-to-texture without the new GPU API for
  advanced cases, older joystick APIs, SDL_RWops renamed to `SDL_IOStream`)

The LLM has seen large amounts of SDL2 tutorial code. Without this reference,
agents will confidently mix SDL2 and SDL3 symbols. Always cross-check.

## Post-Cutoff Version Timeline

| Version | Release | Risk Level | Key Theme |
|---------|---------|------------|-----------|
| 3.0.0 (preview) | 2024 | — | Preview builds — do not target |
| 3.2.0 | Jan 2025 | HIGH | First stable GA — SDL_gpu, main callbacks, audio streams, renamed APIs |
| 3.2.x | 2025–2026 | MEDIUM | Patch / minor releases on the 3.2 line — bug fixes, incremental additions, driver updates |

> **Note**: SDL's semver commitment is that the 3.2.x line is ABI-stable.
> Breaking API changes would bump the minor (to 3.3) or major (to 4). Verify
> the exact patch version in this project via `sdl3-config --version` or the
> vendored `SDL_version.h`.

## Satellite Libraries

SDL3 is distributed with several optional satellite libraries. Each has its
own version number, independent of core SDL.

| Library | Purpose | Tracked in |
|---------|---------|-----------|
| SDL_image 3.x | Image file decoding (PNG, JPG, WebP, etc.) | PLUGINS.md |
| SDL_ttf 3.x | TrueType font rendering | PLUGINS.md |
| SDL_mixer 3.x | High-level audio mixing (channels, music) | PLUGINS.md |
| SDL_net 3.x | Low-level TCP/UDP networking | PLUGINS.md |
| SDL_shadercross | HLSL/MSL/SPIR-V cross-compilation for SDL_gpu | PLUGINS.md |

These libraries were rewritten alongside SDL3 and are NOT ABI-compatible with
their SDL2 counterparts.

## Verified Sources

- Official docs: https://wiki.libsdl.org/SDL3/FrontPage
- Migration guide (SDL2 → SDL3): https://wiki.libsdl.org/SDL3/README/migration
- News / release announcements: https://libsdl.org/news.php
- SDL3 source: https://github.com/libsdl-org/SDL
- SDL_gpu intro: https://wiki.libsdl.org/SDL3/CategoryGPU
- SDL_shadercross: https://github.com/libsdl-org/SDL_shadercross

## When to Update This File

- When the project upgrades to a new SDL 3.x patch or minor release
- When a satellite library is upgraded (SDL_image, SDL_ttf, etc.)
- When the LLM knowledge cutoff advances (refresh the "LLM Knowledge Cutoff"
  row and re-assess Risk Level)
- After running `/setup-engine refresh` to pick up upstream changes
