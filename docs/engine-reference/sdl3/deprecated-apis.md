# SDL3 — Deprecated / Forbidden Patterns

Last verified: 2026-04-18

If an agent suggests any API or pattern in the "Do NOT use" column, it MUST be
replaced with the "Use Instead" column. Most entries here are **SDL2 APIs that
must not be used in SDL3 code**.

## Symbol Renames (SDL2 → SDL3)

These do not compile in SDL3 — they will produce "undefined identifier" errors.
The migration guide lists all renames; this table highlights the ones most
likely to leak in from training data.

| Do NOT use (SDL2) | Use Instead (SDL3) | Reason |
|--------------------|--------------------|--------|
| `SDL_RWops` / `SDL_RWFromFile` | `SDL_IOStream` / `SDL_IOFromFile` | Renamed to IO |
| `SDL_GameController*` (any function) | `SDL_Gamepad*` | Terminology unified |
| `SDL_KEYDOWN`, `SDL_QUIT`, `SDL_MOUSEBUTTONDOWN`, ... | `SDL_EVENT_KEY_DOWN`, `SDL_EVENT_QUIT`, `SDL_EVENT_MOUSE_BUTTON_DOWN`, ... | All event enums gained `SDL_EVENT_` prefix |
| `SDL_FreeSurface` | `SDL_DestroySurface` | Free* → Destroy* |
| `SDL_FreeFormat` | `SDL_DestroyPixelFormat` | Free* → Destroy* |
| `SDL_CreateRGBSurface` / `SDL_CreateRGBSurfaceWithFormat` | `SDL_CreateSurface(w, h, format)` | Simplified |
| `SDL_RenderCopy` / `SDL_RenderCopyEx` | `SDL_RenderTexture` / `SDL_RenderTextureRotated` | Renamed |
| `SDL_RenderDrawLine` / `SDL_RenderDrawPoint` | `SDL_RenderLine` / `SDL_RenderPoint` | Dropped "Draw" |
| `SDL_Point` / `SDL_Rect` in render calls | `SDL_FPoint` / `SDL_FRect` | Float coords default |
| `SDL_QueueAudio` / `SDL_DequeueAudio` | `SDL_AudioStream` + `SDL_PutAudioStreamData` / `SDL_GetAudioStreamData` | Queue API removed |
| `SDL_OpenAudio` (top-level) | `SDL_OpenAudioDevice` + stream model | Global audio device gone |
| `SDL_bool` typedef | `bool` (C99) | SDL3 uses C99 bool |
| `SDL_CreateWindow(title, x, y, w, h, flags)` | `SDL_CreateWindow(title, w, h, flags)` + `SDL_SetWindowPosition` | Position is separate |
| `SDL_JoystickGetAxis` / `SDL_JoystickGetButton` (for gamepads) | `SDL_GetGamepadAxis` / `SDL_GetGamepadButton` | Prefer Gamepad API over raw Joystick for controllers |
| `SDL_Keysym` (nested inside event) | `SDL_KeyboardEvent` fields are inlined; use `event.key.key`, `event.key.scancode`, `event.key.mod` | Struct flattened |
| `SDL_RWread(ctx, buf, size, n)` | `SDL_ReadIO(ctx, buf, total_bytes)` | No size/count split |
| `SDL_GL_GetDrawableSize` | `SDL_GetWindowSizeInPixels` | Not GL-specific |

## Conventions (Not Just APIs)

| Deprecated Pattern | Use Instead | Why |
|--------------------|-------------|-----|
| Returning `int` (`0`/`< 0`) from SDL calls | Check `bool` return of new APIs | SDL3 new APIs return `bool`; many old-style `int` functions were rewritten |
| Manual main loop with `SDL_PollEvent` | `SDL_AppInit` / `SDL_AppIterate` / `SDL_AppEvent` / `SDL_AppQuit` callbacks | Required for Emscripten/iOS; recommended everywhere |
| `SDL_RenderPresent` with default vsync assumption | Explicitly set `SDL_SetRenderVSync(renderer, 1)` | Vsync is opt-in in SDL3 |
| Passing renderer driver by index | Pass by name string (`"vulkan"`, `"metal"`, `NULL` for default) | Indices were never stable |
| Using `SDL_INIT_EVERYTHING` | Init only the subsystems you use | Clarity, avoids pulling in unused backends |
| Treating `SDL_GetTicks` as 32-bit | It returns `Uint64` in SDL3 | Use `Uint64` / `uint64_t` for time math |
| Using `SDL_LockSurface` before software-render composition | Consider the Renderer or GPU API | Direct pixel manipulation is legacy-style |
| Rolling your own audio resampling | Let `SDL_AudioStream` do it | Free, correct, SIMD-optimized |

## SDL_main Gotchas

| Do NOT | Use Instead |
|--------|-------------|
| Define `int main(int argc, char **argv)` and forget to include `SDL3/SDL_main.h` on Windows | Always include `<SDL3/SDL_main.h>` in the file defining main OR use the callbacks API |
| Use `SDL_SetMainReady()` casually | Only when you control the platform entry point (rare — custom launchers) |
| Mix manual main loop with callbacks in the same translation unit | Pick one model per project |

## Threading / Mutex

| Deprecated | Use Instead |
|------------|-------------|
| `SDL_CreateMutex` returning NULL without error check | Always check the return; use `SDL_GetError()` on failure |
| `SDL_mutex` as a direct struct | Treat as opaque pointer; never dereference |
| `SDL_SemWait` / `SDL_SemPost` | `SDL_WaitSemaphore` / `SDL_SignalSemaphore` (renamed for consistency) |

## Shaders (SDL_gpu)

| Do NOT | Use Instead |
|--------|-------------|
| Write GLSL and expect it to load on D3D12 | Author in HLSL, compile via `SDL_shadercross` to SPIR-V / DXIL / MSL |
| Ship only SPIR-V | Compile per-backend; pick at runtime via `SDL_GetGPUShaderFormats(device)` |
| Hand-write MSL for Apple backends | Cross-compile from HLSL unless you need a platform-specific fast path |

## Networking

| Do NOT | Use Instead |
|--------|-------------|
| Use `SDL_net` for real-time multiplayer game state | Use ENet, yojimbo, or a custom UDP layer — SDL_net is low-level and blocking |
| Mix `SDL_net` with blocking reads on the main thread | Do networking on a worker thread; use SDL mutexes/semaphores for IPC |

## Storage / Filesystem

| Do NOT | Use Instead |
|--------|-------------|
| Hardcode `"./save.dat"` or any relative path for saves | `SDL_OpenUserStorage(org, app)` + `SDL_WriteStorageFile` — works on desktop AND console |
| Use `SDL_GetPrefPath` for console save games | `SDL_OpenUserStorage` — console platforms have no meaningful pref path; the storage API talks to the platform save system |
| Use `SDL_GetUserFolder(SDL_FOLDER_DOCUMENTS)` for save files | `SDL_OpenUserStorage` — user folders don't exist on console; Documents is for player-exported content, not game state |
| Start reading immediately after `SDL_OpenUserStorage` / `SDL_OpenTitleStorage` | Poll `SDL_StorageReady` first — storage can be async-initialized (console save menu, Android bundle extraction, cloud fetch) |
| Use `SDL_RWops` family (SDL2) | `SDL_IOStream` — all `SDL_RW*` functions renamed to `SDL_*IO` (see breaking-changes.md) |
| `SDL_RWread(ctx, buf, size, count)` | `SDL_ReadIO(ctx, buf, total_bytes)` — no size/count split |
| Block on large file reads on the main thread | `SDL_AsyncIO` + `SDL_AsyncIOQueue` — submit, continue rendering, drain outcomes later |
| Assume `SDL_CloseAsyncIO(aio, false, ...)` means data is durable | Pass `flush=true` and wait for the close outcome before telling the player "saved" |
| Roll your own recursive directory walker | `SDL_EnumerateDirectory` — cross-platform, callback-based. For pattern matching, `SDL_GlobDirectory` |
| Use `stat()` / `fstat()` directly | `SDL_GetPathInfo` — portable, returns `SDL_PathInfo` with type + nanosecond times |
| Convert `SDL_PathInfo.modify_time` with `localtime()` | It's SDL nanosecond time, not POSIX `time_t`. Use `SDL_NSToTimeSpec` / `SDL_TimeToDateTime` for wall clock |
| `free()` results of `SDL_GetBasePath` / `SDL_GetPrefPath` / `SDL_GetCurrentDirectory` | `SDL_free` — SDL owns the allocation |
| Call `IMG_Init` / `IMG_Quit` from SDL_image 2 habit | Removed in SDL_image 3 — decoders init lazily |

## When In Doubt

If the API name starts with `SDL_` and you cannot find it in this project's
reference docs, search the SDL3 wiki (`https://wiki.libsdl.org/SDL3/`)
**before** writing code. The wiki is the authoritative source; this file is
a shortlist of the patterns most likely to leak in from SDL2 tutorials.
