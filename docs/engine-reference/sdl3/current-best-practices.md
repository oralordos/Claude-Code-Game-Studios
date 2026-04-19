# SDL3 — Current Best Practices

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

Idioms that are **new in SDL3** or frequently mis-applied by LLMs familiar with
SDL2. These supplement the agent's baseline knowledge — they do not replace
reading the official wiki.

## Application Lifecycle — Prefer Callbacks

Use the `SDL_main_callbacks` model unless you have a specific reason not to.

```cpp
// main.cpp — portable entry point
#define SDL_MAIN_USE_CALLBACKS 1
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>
#include "game/app.hpp"

SDL_AppResult SDL_AppInit(void **appstate, int argc, char **argv) {
    auto *app = new game::App();
    if (!app->Init(argc, argv)) {
        delete app;
        return SDL_APP_FAILURE;
    }
    *appstate = app;
    return SDL_APP_CONTINUE;
}

SDL_AppResult SDL_AppIterate(void *appstate) {
    return static_cast<game::App*>(appstate)->Iterate();
}

SDL_AppResult SDL_AppEvent(void *appstate, SDL_Event *event) {
    return static_cast<game::App*>(appstate)->OnEvent(*event);
}

void SDL_AppQuit(void *appstate, SDL_AppResult result) {
    delete static_cast<game::App*>(appstate);
}
```

Why: this model works unchanged on Emscripten, iOS, Android, and desktop. With
a manual `while(running)` loop, Emscripten requires `emscripten_set_main_loop`
and iOS forbids the pattern entirely.

## Init What You Use

```cpp
// Explicit, readable
if (!SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_GAMEPAD)) {
    SDL_Log("SDL_Init: %s", SDL_GetError());
    return 1;
}
```

Do not call `SDL_Init(SDL_INIT_EVERYTHING)` — it is discouraged in SDL3.

## Error Handling Convention

New SDL3 APIs return `bool`. Check the return, then use `SDL_GetError()` for
the reason string.

```cpp
if (!SDL_CreateWindowAndRenderer("My Game", 1280, 720, 0, &window, &renderer)) {
    SDL_Log("window/renderer: %s", SDL_GetError());
    return false;
}
```

For older APIs that still return `int`, `0` is success and `-1` is failure.
Check the exact signature of each function you call.

## Resource Ownership — RAII Wrappers

SDL types are C structs with matching `SDL_Create*` / `SDL_Destroy*` pairs.
Wrap every one in a `std::unique_ptr` with a custom deleter or a dedicated
RAII type.

```cpp
struct SDLWindowDeleter { void operator()(SDL_Window *w) const { SDL_DestroyWindow(w); } };
struct SDLRendererDeleter { void operator()(SDL_Renderer *r) const { SDL_DestroyRenderer(r); } };
struct SDLTextureDeleter { void operator()(SDL_Texture *t) const { SDL_DestroyTexture(t); } };

using WindowPtr   = std::unique_ptr<SDL_Window,   SDLWindowDeleter>;
using RendererPtr = std::unique_ptr<SDL_Renderer, SDLRendererDeleter>;
using TexturePtr  = std::unique_ptr<SDL_Texture,  SDLTextureDeleter>;
```

Keep the SDL C API at the boundary; the rest of the codebase traffics in
typed wrappers.

## Event Loop

Read every pending event each iteration; do not assume one per frame.

```cpp
SDL_Event event;
while (SDL_PollEvent(&event)) {
    switch (event.type) {
        case SDL_EVENT_QUIT:          return SDL_APP_SUCCESS;
        case SDL_EVENT_KEY_DOWN:      input_.OnKey(event.key, true);  break;
        case SDL_EVENT_KEY_UP:        input_.OnKey(event.key, false); break;
        case SDL_EVENT_GAMEPAD_ADDED: controllers_.OnAdded(event.gdevice.which); break;
        // ...
    }
}
```

With the callbacks API, prefer handling each event in `SDL_AppEvent` rather
than calling `SDL_PollEvent` yourself.

## Timing

- `SDL_GetTicks()` returns `Uint64` milliseconds since init
- `SDL_GetTicksNS()` returns nanoseconds — use for frame timing
- `SDL_GetPerformanceCounter()` / `SDL_GetPerformanceFrequency()` still exist
  for high-resolution interval timing

```cpp
const Uint64 frame_start_ns = SDL_GetTicksNS();
// ... update / render ...
const Uint64 frame_end_ns = SDL_GetTicksNS();
const double dt_seconds = static_cast<double>(frame_end_ns - frame_start_ns) * 1e-9;
```

## Rendering — Pick One Path

SDL3 gives you two rendering surfaces. Pick one per project.

| Use | When |
|-----|------|
| `SDL_Renderer` (2D) | Pixel-art, 2D games, tools, UI only — simplest API, GPU-accelerated behind the scenes |
| `SDL_gpu` | 3D, advanced 2D (custom shaders, post-processing), anything needing explicit pipelines | 

Do not try to mix both on the same window — `SDL_gpu` owns the swapchain.

For the 2D renderer, prefer float coordinates:

```cpp
SDL_FRect dst { x, y, w, h };
SDL_RenderTexture(renderer, tex.get(), nullptr, &dst);
```

For vsync, set it explicitly:

```cpp
SDL_SetRenderVSync(renderer, 1);  // 0 = off, 1 = on, -1 = adaptive
```

## Audio — Streams, Not Queues

Open a default logical device and push PCM through an `SDL_AudioStream`.

```cpp
SDL_AudioSpec spec { SDL_AUDIO_F32, 2, 48000 };
SDL_AudioStream *stream = SDL_OpenAudioDeviceStream(
    SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK, &spec, /*callback=*/nullptr, /*userdata=*/nullptr);
SDL_ResumeAudioStreamDevice(stream);

// Each time you generate samples:
SDL_PutAudioStreamData(stream, pcm_buffer.data(), pcm_buffer.size_bytes());
```

The stream handles resampling, format conversion, and mixing to the device.
Never ship a hand-rolled resampler without benchmarking against SDL's.

## SDL_gpu — Shader Strategy

Compile one HLSL source to all backends via `SDL_shadercross`:

```
shaders/
├── lit_textured.vertex.hlsl      # source of truth
├── lit_textured.fragment.hlsl
└── built/                         # .gitignore this
    ├── lit_textured.vertex.spv
    ├── lit_textured.vertex.dxil
    ├── lit_textured.vertex.msl
    └── ...
```

At runtime, inspect `SDL_GetGPUShaderFormats(device)` and load the matching
artifact. Keep the build integrated with your CMake / build system so shaders
are compiled alongside code.

## Properties

Prefer the properties bag for APIs that accept one — it's forward-compatible:

```cpp
SDL_PropertiesID props = SDL_CreateProperties();
SDL_SetStringProperty(props, SDL_PROP_WINDOW_CREATE_TITLE_STRING, "My Game");
SDL_SetNumberProperty(props, SDL_PROP_WINDOW_CREATE_WIDTH_NUMBER, 1920);
SDL_SetNumberProperty(props, SDL_PROP_WINDOW_CREATE_HEIGHT_NUMBER, 1080);
SDL_SetBooleanProperty(props, SDL_PROP_WINDOW_CREATE_HIGH_PIXEL_DENSITY_BOOLEAN, true);
SDL_SetBooleanProperty(props, SDL_PROP_WINDOW_CREATE_RESIZABLE_BOOLEAN, true);
SDL_Window *window = SDL_CreateWindowWithProperties(props);
SDL_DestroyProperties(props);
```

## Logging

Prefer SDL's categorized logger over `printf`:

```cpp
SDL_LogInfo(SDL_LOG_CATEGORY_APPLICATION, "loaded %zu assets", count);
SDL_LogError(SDL_LOG_CATEGORY_RENDER, "shader compile failed: %s", err);
SDL_SetLogPriority(SDL_LOG_CATEGORY_RENDER, SDL_LOG_PRIORITY_DEBUG);
```

Integrates with platform log viewers (Xcode console, Android logcat,
Emscripten stdout).

## Gamepad Over Joystick

For anything marketed as a controller (Xbox, DualSense, Switch Pro, etc.), use
the **Gamepad** API, not raw Joystick. It gives you named buttons and axes
and handles mapping.

```cpp
case SDL_EVENT_GAMEPAD_ADDED: {
    SDL_Gamepad *gp = SDL_OpenGamepad(event.gdevice.which);
    // store gp, associate with player slot
    break;
}
case SDL_EVENT_GAMEPAD_BUTTON_DOWN: {
    if (event.gbutton.button == SDL_GAMEPAD_BUTTON_SOUTH) {
        input_.Jump();
    }
    break;
}
```

Use raw Joystick only for flight sticks, wheels, or other non-standard devices.

## Storage and Assets

- **Saves go through `SDL_Storage`, not `SDL_IOFromFile`** — use
  `SDL_OpenUserStorage(org, app)` for player data. It maps to
  `SDL_GetPrefPath` on desktop AND to platform save systems on console.
- **Bundled assets go through `SDL_OpenTitleStorage`** — on console and
  Android, raw filesystem paths don't reach bundled content. Title
  storage wraps the native bundle/APK access.
- **Always wait for `SDL_StorageReady`** before reading/writing — storage
  can be async on console (save slot prompt), Android (bundle extract),
  or cloud-save platforms.
- **Use `SDL_AsyncIO` for anything > ~64 KB** loaded during active play —
  `SDL_LoadFile` / sync `SDL_IOFromFile` stall the main thread.
- **`SDL_GetUserFolder(SDL_FOLDER_DOCUMENTS)` is for player-facing export**
  (screenshots, replays, "Save As" dialogs) — NOT for save games. Can
  return `NULL`; always null-check.
- Use `SDL_IOFromFile` and friends for portable file access when you don't
  need the full Storage abstraction (dev tools, editors, mod loaders)
- For organized asset bundles, prefer PhysicsFS (`physfs`) on top of
  `SDL_IOStream`, or a custom pack format, or a custom
  `SDL_StorageInterface` implementation
- Cache texture decode results — decoding PNG every frame is a common perf
  bug
- Treat `SDL_PathInfo.modify_time` as `Uint64` nanoseconds, not `time_t`

See `modules/storage.md` for the full storage and filesystem reference,
and `modules/assets.md` for loading / decoding formats.

## Threading

SDL3 threads, mutexes, condition variables, and semaphores are portable
primitives. Use them for platform-agnostic code. Inside a C++20 codebase,
prefer `std::thread` / `std::mutex` / `std::condition_variable` for general
concurrency and reserve SDL's primitives for interop with SDL callbacks
(audio thread, etc.).

## Do NOT

- Do not treat `SDL_GetError()` as thread-local guaranteed — it is, but
  assuming another thread's error text is yours will confuse debugging
- Do not call `SDL_DestroyWindow` on a window that owns an `SDL_Renderer`
  without destroying the renderer first
- Do not share `SDL_Renderer` between threads — it is single-thread
- Do not block the audio callback thread
- Do not mix SDL2 and SDL3 in one build — they are separate libraries
  with conflicting symbol sets; pick one
