# SDL3 Rendering (SDL_Renderer) — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

`SDL_Renderer` is SDL's 2D GPU-accelerated render API. For 3D or custom
shaders, use `SDL_gpu` instead — see `gpu.md`.

## What Changed from SDL2

| Area | SDL2 | SDL3 |
|------|------|------|
| Create renderer | `SDL_CreateRenderer(window, index, flags)` | `SDL_CreateRenderer(window, name)` — name is driver string ("vulkan", "metal", "direct3d12", "opengl", "software", or NULL for default) |
| Create both | `SDL_CreateWindowAndRenderer(w, h, flags, &win, &ren)` | `SDL_CreateWindowAndRenderer(title, w, h, flags, &win, &ren)` — now takes a title |
| Coordinates | Integer rects (`SDL_Rect`) by default | **Float rects (`SDL_FRect`) are the default** — integer variants still exist but prefer float |
| `SDL_RenderCopy` | Blit texture | **Renamed to `SDL_RenderTexture`** |
| `SDL_RenderCopyEx` | Blit with rotation/flip | **Renamed to `SDL_RenderTextureRotated`** |
| `SDL_RenderDrawLine` / `DrawPoint` | Primitives | **Renamed to `SDL_RenderLine` / `SDL_RenderPoint`** (dropped "Draw") |
| `SDL_RenderDrawRect` / `FillRect` | Primitives | **Renamed to `SDL_RenderRect` / `SDL_RenderFillRect`** |
| Vsync | Flag at create | **Set explicitly via `SDL_SetRenderVSync(renderer, mode)`** |
| Scale mode on textures | `SDL_SetTextureScaleMode` | Same, but enum values renamed (`SDL_SCALEMODE_LINEAR`, `SDL_SCALEMODE_NEAREST`) |

## Typical Setup

```cpp
SDL_Window *window = nullptr;
SDL_Renderer *renderer = nullptr;
if (!SDL_CreateWindowAndRenderer("My Game", 1280, 720, SDL_WINDOW_RESIZABLE,
                                 &window, &renderer)) {
    SDL_Log("window/renderer: %s", SDL_GetError());
    return false;
}
SDL_SetRenderVSync(renderer, 1);

// Optional: logical presentation (resolution-independent rendering)
SDL_SetRenderLogicalPresentation(
    renderer, 640, 360,
    SDL_LOGICAL_PRESENTATION_LETTERBOX);
```

## Frame Loop

```cpp
SDL_SetRenderDrawColor(renderer, 10, 10, 20, 255);
SDL_RenderClear(renderer);

// Draw textured sprite at float coords
SDL_FRect dst { 100.0f, 50.0f, 32.0f, 32.0f };
SDL_RenderTexture(renderer, sprite_tex, /*src=*/nullptr, &dst);

// Draw rotated
SDL_RenderTextureRotated(
    renderer, sprite_tex, nullptr, &dst,
    /*angle_degrees=*/45.0, /*center=*/nullptr,
    SDL_FLIP_NONE);

// Primitives (float versions default)
SDL_SetRenderDrawColor(renderer, 255, 0, 0, 255);
SDL_RenderLine(renderer, 0.0f, 0.0f, 100.0f, 100.0f);

SDL_RenderPresent(renderer);
```

## Textures

```cpp
// Decode via SDL_image
SDL_Texture *tex = IMG_LoadTexture(renderer, "assets/player.png");
if (!tex) {
    SDL_Log("load texture: %s", SDL_GetError());
    return false;
}
SDL_SetTextureScaleMode(tex, SDL_SCALEMODE_NEAREST);  // pixel-art
// ...
SDL_DestroyTexture(tex);
```

For pixel-perfect 2D: pair `SDL_SCALEMODE_NEAREST` with a logical
presentation that's an integer scale of your native art size.

## Render Targets

```cpp
SDL_Texture *target = SDL_CreateTexture(
    renderer, SDL_PIXELFORMAT_RGBA8888,
    SDL_TEXTUREACCESS_TARGET, 320, 180);

SDL_SetRenderTarget(renderer, target);
// draw scene at 320x180
SDL_SetRenderTarget(renderer, nullptr);  // back to window

// Now upscale 'target' to fill the window
SDL_RenderTexture(renderer, target, nullptr, nullptr);
```

## Blend Modes

```cpp
SDL_SetRenderDrawBlendMode(renderer, SDL_BLENDMODE_BLEND);     // premul/std alpha
SDL_SetTextureBlendMode(tex, SDL_BLENDMODE_ADD);               // additive for VFX
// Custom blend modes are available via SDL_ComposeCustomBlendMode
```

## Common Mistakes

- Using integer `SDL_Rect` where `SDL_FRect` is expected — watch the signatures
- Forgetting `SDL_SetRenderVSync` and assuming vsync is on
- Leaking textures — every `SDL_CreateTexture` / `IMG_LoadTexture` needs a
  matching `SDL_DestroyTexture` (RAII wrapper recommended)
- Calling `SDL_RenderPresent` without `SDL_RenderClear` — shows uninitialized
  buffer contents on some backends
- Trying to share a renderer across threads (single-thread only)
- Mixing `SDL_Renderer` calls with `SDL_gpu` on the same window

## When to Graduate to SDL_gpu

Stay on `SDL_Renderer` if the game is:
- 2D with no custom shaders
- A tool / editor / debug UI host

Move to `SDL_gpu` when you need:
- Any 3D rendering
- Custom vertex/fragment shaders (post-processing, lighting, stylized effects)
- Compute shaders
- Fine-grained frame pacing control
