# SDL3 UI — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

**SDL3 does not ship a UI framework.** Unlike Godot (Control nodes), Unity
(UI Toolkit / UGUI), or Unreal (UMG / CommonUI), SDL3 gives you pixel
surfaces, input events, and font rendering — you assemble UI on top.

## Decision: Which UI Library?

| Library | Best For | License | Notes |
|---------|----------|---------|-------|
| **Dear ImGui** | Debug overlays, level editors, tool UI, prototypes | MIT | Easy, ugly by default, not for shipped player UI |
| **RmlUi** | Shipped player-facing UI | MIT | HTML/CSS-inspired, retained-mode, larger integration cost |
| **Nuklear** | Very lightweight dev UI in C | MIT / PD | Single-header, more primitive than ImGui |
| **Hand-rolled** | Games where UI is small or highly stylized | — | `SDL_ttf` + textured quads; most control, most code |
| **CEGUI** | Commercial games with complex UI flows | MIT | Mature but heavier; less momentum than RmlUi in 2025+ |

If the game has any meaningful HUD or menu system, **pick a library before
sprint one**. Rolling your own UI framework is a project of its own.

## Dear ImGui Integration (SDL3)

Dear ImGui ships first-class SDL3 backends:

```cpp
#include <imgui.h>
#include <imgui_impl_sdl3.h>
#include <imgui_impl_sdlrenderer3.h>   // or imgui_impl_sdlgpu3.h for SDL_gpu

// During init, after creating window + renderer:
IMGUI_CHECKVERSION();
ImGui::CreateContext();
ImGui_ImplSDL3_InitForSDLRenderer(window, renderer);
ImGui_ImplSDLRenderer3_Init(renderer);

// Each frame:
ImGui_ImplSDLRenderer3_NewFrame();
ImGui_ImplSDL3_NewFrame();
ImGui::NewFrame();

ImGui::Begin("Debug");
ImGui::Text("FPS: %.1f", 1.0f / dt);
ImGui::End();

ImGui::Render();
ImGui_ImplSDLRenderer3_RenderDrawData(ImGui::GetDrawData(), renderer);

// Forward events:
SDL_Event e;
while (SDL_PollEvent(&e)) {
    ImGui_ImplSDL3_ProcessEvent(&e);
    // ... then your own dispatch
}

// Cleanup:
ImGui_ImplSDLRenderer3_Shutdown();
ImGui_ImplSDL3_Shutdown();
ImGui::DestroyContext();
```

For SDL_gpu: use `imgui_impl_sdlgpu3.cpp` instead of the SDL_Renderer
backend. It records into your existing command buffer — no separate pass.

## Hand-Rolled UI Patterns

For small games where you want full control:

### Retained-mode (data-driven)
- Define UI elements as data (position, size, texture, callback)
- Build once, mutate when state changes
- Render each frame by walking the tree

```cpp
struct Button {
    SDL_FRect bounds;
    const char *label;
    std::function<void()> on_click;
    bool hovered = false;
    bool pressed = false;
};
```

### Immediate-mode (rebuild each frame)
```cpp
bool button(SDL_FRect r, const char *label) {
    bool hovered = SDL_PointInRectFloat(&mouse_pos, &r);
    bool clicked = hovered && mouse_down_this_frame;
    draw_rect(r, hovered ? hover_color : base_color);
    draw_text(r, label);
    return clicked;
}

// Each frame:
if (button({10, 10, 100, 30}, "Start")) start_game();
if (button({10, 50, 100, 30}, "Quit"))  quit = true;
```

Immediate-mode is conceptually closer to ImGui — easy to start, harder to
theme richly.

## Text Rendering — SDL_ttf

See `assets.md` for loading. For UI, render text to textures and cache by
`(font, string, color)`:

```cpp
struct TextKey { TTF_Font *font; std::string str; SDL_Color color; };
// operator== and hash ...

std::unordered_map<TextKey, SDL_Texture*> text_cache;

SDL_Texture *render_text(TTF_Font *f, std::string_view s, SDL_Color c) {
    TextKey k{f, std::string(s), c};
    if (auto it = text_cache.find(k); it != text_cache.end()) return it->second;
    SDL_Surface *surf = TTF_RenderText_Blended(f, s.data(), s.size(), c);
    SDL_Texture *tex  = SDL_CreateTextureFromSurface(renderer, surf);
    SDL_DestroySurface(surf);
    text_cache.emplace(std::move(k), tex);
    return tex;
}
```

Invalidate the cache when fonts change or when strings become stale. For
constantly-changing text (FPS counter, score), keep a small "hot" buffer
separate from the main cache.

## Input Focus and Navigation

Unlike engine UI frameworks, SDL does not give you automatic focus
management. If your game supports gamepad navigation, implement it:

- Maintain a **focus** variable (index into a navigable widget list)
- On `SDL_EVENT_GAMEPAD_BUTTON_DOWN` dpad events, step focus
- On `SDL_GAMEPAD_BUTTON_SOUTH`, fire the focused widget's action
- Draw a focus ring around the focused widget

A good navigation system spans ~100 lines — don't skip it; games without
gamepad-navigable menus feel broken on console/TV setups.

## Accessibility

Because you own the UI, you own the accessibility story:

- **Screen reader**: SDL3 does not integrate with AccessKit out of the
  box. If you need screen reader support, consider RmlUi (better story)
  or implement AccessKit manually
- **Text scaling**: expose a UI scale slider; re-render text textures when
  it changes
- **High contrast**: expose a color theme option
- **Remap**: see `input.md` — do not hardcode gamepad or keyboard bindings
- **Colorblind**: use shape + color, not color alone, for state

## Common Mistakes

- Rendering text by calling `TTF_RenderText_*` every frame (no cache)
- Hardcoding UI coordinates for 1920×1080 and breaking on other
  resolutions — use `SDL_SetRenderLogicalPresentation` or compute layouts
  from window size
- Mixing ImGui with shipped player-facing UI (fine for debug, jarring for
  players)
- Not forwarding events to ImGui's `ProcessEvent` — ImGui silently does
  nothing
- Hit-testing with integer rects against float mouse coords — use
  `SDL_PointInRectFloat`
- Ignoring gamepad navigation because the game "is PC only" — players
  frequently use gamepads on PC

## See Also

- `input.md` — keyboard/mouse/gamepad/touch/pen input
- `assets.md` — font loading via SDL_ttf
- `PLUGINS.md` — Dear ImGui, RmlUi, Nuklear integration details
