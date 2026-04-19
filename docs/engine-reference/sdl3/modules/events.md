# SDL3 Events & Main Loop — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

This module covers the two ways to structure an SDL3 application: the
traditional manual main loop and the newer callbacks API. The callbacks API
is **recommended** unless you have a specific reason to roll your own.

## What Changed from SDL2

| Area | SDL2 | SDL3 |
|------|------|------|
| Entry | Manual `int main(int argc, char **argv)` with custom main loop | **Callbacks API**: `SDL_AppInit` / `SDL_AppIterate` / `SDL_AppEvent` / `SDL_AppQuit` |
| Event enum prefix | `SDL_QUIT`, `SDL_KEYDOWN`, ... | **`SDL_EVENT_QUIT`, `SDL_EVENT_KEY_DOWN`, ...** |
| Event filtering | `SDL_SetEventFilter` | Unchanged, plus `SDL_AddEventWatch` for observer-style hooks |
| User events | `SDL_USEREVENT` | `SDL_EVENT_USER` (through `SDL_EVENT_LAST`) |
| Event enable/disable | `SDL_EventState` | `SDL_SetEventEnabled` / `SDL_EventEnabled` |

## Callbacks API (Recommended)

Opt in by defining `SDL_MAIN_USE_CALLBACKS` before including `SDL_main.h`.
Then implement four functions:

```cpp
#define SDL_MAIN_USE_CALLBACKS 1
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

struct AppState {
    SDL_Window *window = nullptr;
    SDL_Renderer *renderer = nullptr;
    Uint64 last_tick_ns = 0;
    // ... game state ...
};

SDL_AppResult SDL_AppInit(void **appstate, int argc, char **argv) {
    if (!SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_GAMEPAD)) {
        SDL_Log("SDL_Init: %s", SDL_GetError());
        return SDL_APP_FAILURE;
    }
    auto *app = new AppState();
    if (!SDL_CreateWindowAndRenderer("My Game", 1280, 720, 0,
                                     &app->window, &app->renderer)) {
        delete app;
        return SDL_APP_FAILURE;
    }
    SDL_SetRenderVSync(app->renderer, 1);
    app->last_tick_ns = SDL_GetTicksNS();
    *appstate = app;
    return SDL_APP_CONTINUE;
}

SDL_AppResult SDL_AppIterate(void *appstate) {
    auto *app = static_cast<AppState*>(appstate);

    Uint64 now_ns = SDL_GetTicksNS();
    double dt = static_cast<double>(now_ns - app->last_tick_ns) * 1e-9;
    app->last_tick_ns = now_ns;

    // update(dt); render(app->renderer);

    SDL_RenderPresent(app->renderer);
    return SDL_APP_CONTINUE;
}

SDL_AppResult SDL_AppEvent(void *appstate, SDL_Event *event) {
    auto *app = static_cast<AppState*>(appstate);
    switch (event->type) {
        case SDL_EVENT_QUIT: return SDL_APP_SUCCESS;
        case SDL_EVENT_KEY_DOWN:
            if (event->key.key == SDLK_ESCAPE) return SDL_APP_SUCCESS;
            break;
    }
    return SDL_APP_CONTINUE;
}

void SDL_AppQuit(void *appstate, SDL_AppResult result) {
    auto *app = static_cast<AppState*>(appstate);
    if (app) {
        if (app->renderer) SDL_DestroyRenderer(app->renderer);
        if (app->window)   SDL_DestroyWindow(app->window);
        delete app;
    }
    // SDL_Quit is called for you
}
```

### Return values

- `SDL_APP_CONTINUE` — keep running
- `SDL_APP_SUCCESS` — exit cleanly (returns 0 from the process)
- `SDL_APP_FAILURE` — exit with error (returns non-zero; SDL logs)

### Why callbacks?

- **Portable** — works unchanged on Emscripten (no manual
  `emscripten_set_main_loop`), iOS (UIKit owns the main loop), and Android
- **Single place for each responsibility** — init/iterate/event/quit
  naturally separate
- **Simpler cleanup** — SDL calls `SDL_Quit` after your `SDL_AppQuit`, even
  on crashes that unwind through `SDL_APP_FAILURE`

### When to prefer manual main loop

- You're integrating SDL into a framework that already owns the event pump
- You want explicit control over thread scheduling (rare)
- You're porting an SDL2 codebase and don't want to restructure

## Manual Main Loop

```cpp
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

int main(int argc, char **argv) {
    if (!SDL_Init(SDL_INIT_VIDEO)) { /* ... */ return 1; }

    SDL_Window *window = SDL_CreateWindow("My Game", 1280, 720, 0);
    SDL_Renderer *renderer = SDL_CreateRenderer(window, nullptr);

    bool running = true;
    while (running) {
        SDL_Event event;
        while (SDL_PollEvent(&event)) {
            if (event.type == SDL_EVENT_QUIT) running = false;
            if (event.type == SDL_EVENT_KEY_DOWN && event.key.key == SDLK_ESCAPE)
                running = false;
        }
        // update(); render();
        SDL_RenderPresent(renderer);
    }

    SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window);
    SDL_Quit();
    return 0;
}
```

## Event Types Inventory

Common event categories (all prefixed `SDL_EVENT_`):

| Category | Events |
|----------|--------|
| Quit / app | `QUIT`, `TERMINATING`, `LOW_MEMORY`, `WILL_ENTER_BACKGROUND`, `DID_ENTER_FOREGROUND` |
| Window | `WINDOW_SHOWN`, `WINDOW_RESIZED`, `WINDOW_PIXEL_SIZE_CHANGED`, `WINDOW_CLOSE_REQUESTED`, `WINDOW_FOCUS_GAINED`, `WINDOW_FOCUS_LOST` |
| Keyboard | `KEY_DOWN`, `KEY_UP`, `TEXT_EDITING`, `TEXT_INPUT` |
| Mouse | `MOUSE_MOTION`, `MOUSE_BUTTON_DOWN`, `MOUSE_BUTTON_UP`, `MOUSE_WHEEL` |
| Gamepad | `GAMEPAD_ADDED`, `GAMEPAD_REMOVED`, `GAMEPAD_BUTTON_DOWN`, `GAMEPAD_BUTTON_UP`, `GAMEPAD_AXIS_MOTION`, `GAMEPAD_SENSOR_UPDATE` |
| Touch | `FINGER_DOWN`, `FINGER_UP`, `FINGER_MOTION` |
| Pen | `PEN_DOWN`, `PEN_UP`, `PEN_MOTION`, `PEN_BUTTON_DOWN`, `PEN_BUTTON_UP` |
| Drop | `DROP_FILE`, `DROP_TEXT`, `DROP_BEGIN`, `DROP_COMPLETE` |
| Audio | `AUDIO_DEVICE_ADDED`, `AUDIO_DEVICE_REMOVED`, `AUDIO_DEVICE_FORMAT_CHANGED` |
| Display | `DISPLAY_ADDED`, `DISPLAY_REMOVED`, `DISPLAY_ORIENTATION` |

## User Events

```cpp
// Register a user event range (optional, for avoiding collisions with libs)
Uint32 my_event_type = SDL_RegisterEvents(1);

SDL_Event ev;
SDL_zero(ev);
ev.type = my_event_type;
ev.user.code = 42;
ev.user.data1 = some_payload;
SDL_PushEvent(&ev);
```

Useful for posting events from worker threads to the main thread.

## Event Filters and Watchers

```cpp
// Filter — can drop events before they reach the queue
SDL_SetEventFilter([](void *userdata, SDL_Event *event) -> bool {
    return event->type != SDL_EVENT_WINDOW_EXPOSED;  // drop these
}, nullptr);

// Watch — observes events, cannot modify them
SDL_AddEventWatch([](void *userdata, SDL_Event *event) -> bool {
    // observe (e.g. for telemetry)
    return true;  // return value is ignored for watchers
}, nullptr);
```

## Main-Thread Rules

- `SDL_PollEvent`, `SDL_WaitEvent`, `SDL_PumpEvents` must be called on the
  thread that initialized the video subsystem (usually the main thread)
- On macOS and iOS, most SDL calls must happen on the main thread
- Post user events from worker threads via `SDL_PushEvent` (thread-safe)

## Common Mistakes

- Using `SDL_QUIT` / `SDL_KEYDOWN` / etc. without the `SDL_EVENT_` prefix —
  these SDL2-style identifiers don't exist in SDL3
- Calling `SDL_PollEvent` from a non-main thread
- Forgetting to consume ALL pending events each frame (partial drain → input
  lag)
- Returning `SDL_APP_CONTINUE` after an unrecoverable error instead of
  `SDL_APP_FAILURE`
- Allocating or computing in the event handler — prefer queueing intent and
  processing in `SDL_AppIterate` for testability
- Handling `SDL_EVENT_WINDOW_CLOSE_REQUESTED` and `SDL_EVENT_QUIT` differently
  when you want the same close behavior
