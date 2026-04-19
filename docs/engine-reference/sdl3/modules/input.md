# SDL3 Input — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

Covers keyboard, mouse, gamepad, touch, and pen. For the event loop itself
(how events are delivered), see `events.md`.

## What Changed from SDL2

| Area | SDL2 | SDL3 |
|------|------|------|
| Terminology | `SDL_GameController*` | **`SDL_Gamepad*`** — all functions renamed |
| Event enums | `SDL_KEYDOWN`, `SDL_MOUSEMOTION`, etc. | **`SDL_EVENT_KEY_DOWN`, `SDL_EVENT_MOUSE_MOTION`, etc.** — all gained `SDL_EVENT_` prefix |
| Key event fields | Nested `SDL_Keysym` | **Flattened**: `event.key.key`, `event.key.scancode`, `event.key.mod`, `event.key.down` |
| Button names | `SDL_CONTROLLER_BUTTON_A` | **`SDL_GAMEPAD_BUTTON_SOUTH`** — position-based names (`SOUTH`, `EAST`, `WEST`, `NORTH`) are preferred over A/B/X/Y |
| Axis names | `SDL_CONTROLLER_AXIS_*` | `SDL_GAMEPAD_AXIS_*` |
| Mouse position | Integer | Float (`SDL_GetMouseState` returns floats) |
| Touch / pen | Merged | Separated — dedicated `SDL_EVENT_PEN_*` events |
| Text input | Global start/stop | Per-window: `SDL_StartTextInput(window)` / `SDL_StopTextInput(window)` |

## Keyboard

### Events
```cpp
case SDL_EVENT_KEY_DOWN:
    if (event.key.repeat) break;  // skip auto-repeat for single-shot actions
    switch (event.key.key) {      // 'key' = logical key (layout-aware)
        case SDLK_ESCAPE: quit_requested = true; break;
        case SDLK_SPACE:  input_.Jump();         break;
    }
    break;

case SDL_EVENT_KEY_UP:
    // similar
    break;
```

### Polling
```cpp
const bool *keystate = SDL_GetKeyboardState(nullptr);
if (keystate[SDL_SCANCODE_W]) {
    // scancode = physical key position, layout-independent
}
```

### Key vs Scancode
- **Key** (`SDL_Keycode`, `SDLK_*`): layout-aware. Use for text-style input
  ("press Q to quit").
- **Scancode** (`SDL_Scancode`, `SDL_SCANCODE_*`): physical position. Use
  for WASD movement and anything where the position matters more than the
  label.

## Mouse

```cpp
case SDL_EVENT_MOUSE_MOTION:
    // event.motion.x, .y are FLOAT window coords
    // .xrel / .yrel are relative (delta since last event)
    break;

case SDL_EVENT_MOUSE_BUTTON_DOWN:
    if (event.button.button == SDL_BUTTON_LEFT) { /* ... */ }
    break;

case SDL_EVENT_MOUSE_WHEEL:
    // event.wheel.y is scroll direction (float — supports smooth trackpads)
    break;
```

Polling:
```cpp
float x = 0, y = 0;
SDL_MouseButtonFlags buttons = SDL_GetMouseState(&x, &y);
if (buttons & SDL_BUTTON_MASK(SDL_BUTTON_LEFT)) { /* held */ }
```

For FPS-style relative mouse:
```cpp
SDL_SetWindowRelativeMouseMode(window, true);
// Now SDL_EVENT_MOUSE_MOTION.xrel/.yrel are continuous deltas; cursor is captured
```

## Gamepad

### Opening
```cpp
case SDL_EVENT_GAMEPAD_ADDED: {
    SDL_Gamepad *gp = SDL_OpenGamepad(event.gdevice.which);
    // Assign to a player slot; store gp for later use
    break;
}
case SDL_EVENT_GAMEPAD_REMOVED: {
    // event.gdevice.which is the instance id to remove
    break;
}
```

### Buttons
```cpp
case SDL_EVENT_GAMEPAD_BUTTON_DOWN:
    switch (event.gbutton.button) {
        case SDL_GAMEPAD_BUTTON_SOUTH:  input_.Confirm();  break;  // Xbox A, PS Cross, Switch B
        case SDL_GAMEPAD_BUTTON_EAST:   input_.Cancel();   break;  // Xbox B, PS Circle, Switch A
        case SDL_GAMEPAD_BUTTON_WEST:   input_.Menu();     break;
        case SDL_GAMEPAD_BUTTON_NORTH:  input_.Action();   break;
        case SDL_GAMEPAD_BUTTON_START:  input_.Pause();    break;
    }
    break;
```

**Prefer `SOUTH`/`EAST`/`WEST`/`NORTH` over `A`/`B`/`X`/`Y`** — the mapping
flips between Xbox/PS and Switch. Position-based names are unambiguous.

### Axes
```cpp
float lx = SDL_GetGamepadAxis(gp, SDL_GAMEPAD_AXIS_LEFTX)  / 32767.0f;
float ly = SDL_GetGamepadAxis(gp, SDL_GAMEPAD_AXIS_LEFTY)  / 32767.0f;
float lt = SDL_GetGamepadAxis(gp, SDL_GAMEPAD_AXIS_LEFT_TRIGGER) / 32767.0f;

// Apply deadzone
if (std::abs(lx) < 0.15f) lx = 0.0f;
```

### Rumble
```cpp
SDL_RumbleGamepad(gp, /*low_freq=*/0x7fff, /*high_freq=*/0x3fff, /*duration_ms=*/200);
```

### Type Detection
```cpp
SDL_GamepadType type = SDL_GetGamepadType(gp);
// SDL_GAMEPAD_TYPE_XBOX360 / _XBOXONE / _PS3 / _PS4 / _PS5 / _NINTENDO_SWITCH_PRO / ...
```

Use this to swap glyph art (Xbox A vs PS Cross vs Switch B) in tooltips.

### Sensor Data (Gyro / Accel)
```cpp
if (SDL_GamepadHasSensor(gp, SDL_SENSOR_GYRO)) {
    SDL_SetGamepadSensorEnabled(gp, SDL_SENSOR_GYRO, true);
    // Then handle SDL_EVENT_GAMEPAD_SENSOR_UPDATE
}
```

## Touch

```cpp
case SDL_EVENT_FINGER_DOWN:
case SDL_EVENT_FINGER_UP:
case SDL_EVENT_FINGER_MOTION:
    // event.tfinger.x, .y are normalized 0..1
    // event.tfinger.fingerId is the finger ID for multitouch
    break;
```

## Pen (SDL3 NEW — stylus/tablet)

```cpp
case SDL_EVENT_PEN_DOWN:
case SDL_EVENT_PEN_UP:
case SDL_EVENT_PEN_MOTION:
case SDL_EVENT_PEN_BUTTON_DOWN:
    // event.pmotion / event.pbutton fields
    // Pressure, tilt, and rotation available
    break;
```

Separate from touch — if you support both, don't handle them as the same
event type.

## Text Input

```cpp
SDL_StartTextInput(window);
// ...
case SDL_EVENT_TEXT_INPUT:
    // event.text.text is a null-terminated UTF-8 string
    ui_.InsertText(event.text.text);
    break;
case SDL_EVENT_TEXT_EDITING:
    // IME composition in progress
    break;
// ...
SDL_StopTextInput(window);
```

Use `SDL_StartTextInput` only when a text field has focus — otherwise the
on-screen keyboard appears on iOS / Android.

## Accessibility Considerations

- Always support **both** keyboard and gamepad for core gameplay
- Expose key/button remapping — use scancodes for physical layout, let
  the user rebind per-action
- Respect `SDL_ShowCursor` state from the platform
- Test with the **SDL_HINT_XINPUT_ENABLED** / gamepad hint system if
  targeting Windows with legacy XInput drivers

## Common Mistakes

- Using `SDL_CONTROLLER_*` (SDL2) names — all renamed to `SDL_GAMEPAD_*`
- Using `event.key.keysym.sym` — struct was flattened; use `event.key.key`
- Assuming `A`/`B` labels work cross-platform — use `SOUTH`/`EAST` and
  map to display glyphs based on `SDL_GetGamepadType`
- Polling with `SDL_GetKeyboardState` when the key state is already in
  `event.key` — the event loop version is usually right
- Not calling `SDL_StopTextInput` when a text field loses focus — keeps the
  on-screen keyboard up on mobile
- Ignoring `event.key.repeat` for menu actions — causes multi-triggers when
  the user holds the key
- Not applying deadzones on analog sticks (values drift around zero)
