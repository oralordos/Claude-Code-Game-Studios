# SDL3 — Breaking Changes

Last verified: 2026-04-18

Changes focused on the **SDL2 → SDL3 transition** (the dominant source of
confusion for LLM-generated code) and selected post-GA notes.

## SDL2 → SDL3 (Jan 2025 — HIGH RISK)

### Initialization and main loop

| Area | SDL2 | SDL3 |
|------|------|------|
| Init | `SDL_Init(SDL_INIT_VIDEO \| SDL_INIT_AUDIO)` | `SDL_Init(SDL_INIT_VIDEO \| SDL_INIT_AUDIO)` — same shape, but subsystem flags were renumbered; do NOT hardcode numeric flag values |
| Main entry | Custom `SDL_main` shim with manual event loop | **Callbacks API (recommended)**: implement `SDL_AppInit`, `SDL_AppIterate`, `SDL_AppEvent`, `SDL_AppQuit`. Works on platforms where the app does not own its main loop (emscripten, iOS). |
| Quit | `SDL_Quit()` (manual) | `SDL_Quit()` — but with callbacks, the runtime calls it for you after `SDL_AppQuit` |
| Error return | Mostly `int` (`0` success, `< 0` error) | Mostly `bool` (`true` success, `false` error) for new APIs. Old-style `int` survives on some legacy functions. ALWAYS check the exact signature. |
| Error text | `SDL_GetError()` | `SDL_GetError()` — unchanged |

### Renames (the rule, not the exception)

SDL3 enforces consistent `SDL_VerbNoun()` naming and strict enums. Most SDL2
symbols have a new name. Examples (non-exhaustive — assume every symbol may
have been renamed and verify):

| SDL2 | SDL3 |
|------|------|
| `SDL_RWops` | `SDL_IOStream` |
| `SDL_RWFromFile` | `SDL_IOFromFile` |
| `SDL_RWread` / `SDL_RWwrite` | `SDL_ReadIO` / `SDL_WriteIO` |
| `SDL_GameController*` | `SDL_Gamepad*` (controller → gamepad) |
| `SDL_JoystickGetGUID` | `SDL_GetJoystickGUID` (verb-first) |
| `SDL_CreateWindow(title, x, y, w, h, flags)` | `SDL_CreateWindow(title, w, h, flags)` — position is set separately via `SDL_SetWindowPosition` |
| `SDL_GL_GetDrawableSize` | `SDL_GetWindowSizeInPixels` |
| `SDL_GetTicks` (Uint32) | `SDL_GetTicks` (returns Uint64 — ms since init) |
| `SDL_Delay` | `SDL_Delay` — unchanged, but prefer `SDL_DelayNS` for sub-ms precision |
| `SDL_CreateRGBSurface` | `SDL_CreateSurface` (pixel format argument, not masks) |
| `SDL_FreeSurface` | `SDL_DestroySurface` (Free* → Destroy* convention) |
| `SDL_LoadWAV` | `SDL_LoadWAV` — still present, but now returns bool and takes an `SDL_AudioSpec*` out-param |
| `SDL_OpenAudioDevice` | `SDL_OpenAudioDevice` — signature rewritten for the streams model |
| `SDL_QueueAudio` / `SDL_DequeueAudio` | Removed — use `SDL_AudioStream` with `SDL_PutAudioStreamData` / `SDL_GetAudioStreamData` |
| `SDL_Texture` access | Now intermediated through `SDL_Renderer` or the new GPU API; direct raw-pixel lock/unlock survives but naming changed |
| `SDL_Event` types | `SDL_KEYDOWN` → `SDL_EVENT_KEY_DOWN`, `SDL_QUIT` → `SDL_EVENT_QUIT`, etc. All events gained the `SDL_EVENT_` prefix. |

**Rule of thumb**: if you can recall an SDL2 symbol from memory, do NOT paste
it into SDL3 code without checking the migration guide.

### Event system

| Change | Detail |
|--------|--------|
| Event type prefix | `SDL_KEYDOWN` → `SDL_EVENT_KEY_DOWN`. Every event type enum gained the `SDL_EVENT_` prefix. |
| `SDL_Keysym` | Now `SDL_Keysym` is folded into `SDL_KeyboardEvent` directly (no nested struct) |
| Touch / pen | Dedicated pen events (`SDL_EVENT_PEN_DOWN`, etc.) separate from touch |
| Text input | `SDL_StartTextInput` / `SDL_StopTextInput` now per-window |
| Drop | `SDL_DROPFILE` → `SDL_EVENT_DROP_FILE` with per-item enumeration |

### Rendering (SDL_Renderer)

The 2D `SDL_Renderer` API survives and is still the path for simple 2D games.
Major changes:

- `SDL_CreateRenderer(window, index, flags)` → `SDL_CreateRenderer(window, driver_name)`
  — driver selected by **name string** ("vulkan", "opengl", "metal", "direct3d12",
  or `NULL` for default), not index/flags
- Coordinates are now `float` (was `int`) — `SDL_FRect`, `SDL_FPoint` are the
  default; integer variants exist but are legacy
- `SDL_RenderCopy` / `SDL_RenderCopyEx` → `SDL_RenderTexture` / `SDL_RenderTextureRotated`
- `SDL_RenderDrawRect` / `SDL_RenderFillRect` → `SDL_RenderRect` / `SDL_RenderFillRect`

### New: SDL_gpu (modern GPU API)

Entirely new in SDL3 — there is no SDL2 equivalent. Targets Vulkan, Direct3D 12,
and Metal from a single API. See `modules/gpu.md`.

- Create device: `SDL_CreateGPUDevice(format_flags, debug_mode, driver_name)`
- Shaders: SPIR-V for Vulkan, DXIL for D3D12, MSL for Metal — use
  `SDL_shadercross` to compile one HLSL source to all three
- Pattern: acquire command buffer → begin render pass → bind pipeline → draw → end → submit

### New: Audio streams

The old SDL2 callback / queue model is gone. SDL3 uses `SDL_AudioStream`:

- Open a device with a default logical device ID
- Create a stream with source & device format
- `SDL_PutAudioStreamData` pushes PCM in; SDL resamples & mixes automatically
- Optional callback via `SDL_SetAudioStreamGetCallback` for pull-model generation

### New: SDL_Storage

Entirely new in SDL3 — there is no SDL2 equivalent. Opaque handle
representing a logical place to read/write data, opened via
`SDL_OpenTitleStorage` (bundled game assets), `SDL_OpenUserStorage`
(saves / settings), `SDL_OpenFileStorage` (explicit filesystem path), or
`SDL_OpenStorage` (custom interface).

Storage may not be ready immediately after open — poll `SDL_StorageReady`
before the first read/write. This enables console save-slot selection,
cloud-save sync, and Android bundle extraction to happen transparently.

See `modules/storage.md`.

### New: SDL_AsyncIO

Entirely new in SDL3 — async file I/O with a queue-based completion
model. `SDL_AsyncIOFromFile` / `SDL_ReadAsyncIO` / `SDL_WriteAsyncIO`
submit work; `SDL_GetAsyncIOResult` drains outcomes from an
`SDL_AsyncIOQueue`. Use for streaming large assets without stalling the
main thread.

See `modules/storage.md`.

### New: Filesystem helpers

SDL3 adds several standalone filesystem helpers that had no SDL2
equivalent:

| Function | Purpose |
|----------|---------|
| `SDL_GetUserFolder(SDL_Folder)` | Cross-platform access to OS-standard user folders (Documents, Desktop, Pictures, Screenshots, SavedGames, etc.) — returns `NULL` on platforms without the concept |
| `SDL_GetCurrentDirectory()` | Current working dir (useful for command-line tools) |
| `SDL_EnumerateDirectory(path, cb, ud)` | Callback-per-entry enumeration |
| `SDL_GlobDirectory(path, pattern, flags, &count)` | Glob-style match (`*.png`, `level_*`) |
| `SDL_GetPathInfo(path, &info)` | stat-like metadata: type, size, create/modify/access times |
| `SDL_CreateDirectory(path)` | Creates intermediate dirs |
| `SDL_RenamePath`, `SDL_CopyFile`, `SDL_RemovePath` | Sync file/dir ops |

`SDL_PathInfo.create_time` / `modify_time` / `access_time` are SDL
nanosecond time (`Uint64`), NOT POSIX `time_t`. Convert with
`SDL_NSToTimeSpec` if you need wall-clock time.

### New: Properties

`SDL_PropertiesID` is a typed key/value container. Replaces several
ad-hoc per-object dictionaries in SDL2. Many APIs now accept a properties
bag for forward-compatible configuration:

```c
SDL_PropertiesID props = SDL_CreateProperties();
SDL_SetStringProperty(props, SDL_PROP_WINDOW_CREATE_TITLE_STRING, "My Game");
SDL_SetNumberProperty(props, SDL_PROP_WINDOW_CREATE_WIDTH_NUMBER, 1280);
SDL_Window *window = SDL_CreateWindowWithProperties(props);
SDL_DestroyProperties(props);
```

### Removed or replaced

| Removed | Replacement |
|---------|-------------|
| `SDL_bool` typedef | C99 `bool` (include `<stdbool.h>` or use C++ `bool`) |
| `SDL_HAPTIC_*` subsystem (in its old form) | Rumble / force-feedback now on gamepad API directly |
| `SDL_Joystick` legacy per-axis indices for modern controllers | Use `SDL_Gamepad` typed button/axis names |
| Software renderer as a hidden fallback | Explicit: pass `"software"` driver name |

### Subsystem flag changes

- `SDL_INIT_TIMER` — removed (timer is always available)
- `SDL_INIT_NOPARACHUTE` — removed (no-op in SDL3)
- `SDL_INIT_EVERYTHING` — replaced by passing all the flags you want; the
  omnibus macro is discouraged

## Post-3.2.0 Changes (MEDIUM RISK — assume incremental)

SDL follows semver. Within the 3.2.x line, expect:

- New function additions (non-breaking)
- Platform backend improvements (new Wayland / Android / iOS fixes)
- Bug fixes to SDL_gpu drivers (Vulkan barrier tracking, D3D12 frame pacing, etc.)
- No removals or renames of stable symbols

When you see a change in release notes that looks like a rename, check whether
it's on a function marked `_experimental` — those may break without a minor
bump.

## Satellite Library SDL2 → SDL3 Changes

| Library | Notable Change |
|---------|---------------|
| SDL_image | `IMG_Init` / `IMG_Quit` removed — decoders initialize lazily; `IMG_LoadTexture(renderer, path)` signature unchanged |
| SDL_ttf | `TTF_Init` / `TTF_Quit` still required; `TTF_RenderText_Solid` now takes a length parameter |
| SDL_mixer | Rewritten on top of SDL3 audio streams; per-channel API survives |
| SDL_net | Low-level sockets preserved; blocking semantics clarified |

## How to Use This File

- Before suggesting an SDL function name from memory, grep this file for
  the SDL2 version. If it appears in the rename table, substitute the SDL3
  name.
- When fixing a compile error about a missing symbol, check the migration
  guide URL in `VERSION.md` — it is the authoritative list.
- When an event type doesn't compile, add the `SDL_EVENT_` prefix first
  before investigating further.
