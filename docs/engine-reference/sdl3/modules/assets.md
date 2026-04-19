# SDL3 Assets & IOStream — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

This module covers **loading and decoding** — `SDL_IOStream` plumbing plus
the satellite libraries that turn bytes into usable assets (images, fonts,
audio, models).

For **where the bytes live** (save paths, `SDL_Storage`, `SDL_AsyncIO`,
directory enumeration, user folders), see `storage.md`.

## What Changed from SDL2

| SDL2 | SDL3 |
|------|------|
| `SDL_RWops` | **`SDL_IOStream`** — renamed, same concept |
| `SDL_RWFromFile(path, "rb")` | `SDL_IOFromFile(path, "rb")` |
| `SDL_RWread(ctx, buf, size, n)` | `SDL_ReadIO(ctx, buf, total_bytes)` — no size/count split |
| `SDL_RWwrite(ctx, buf, size, n)` | `SDL_WriteIO(ctx, buf, total_bytes)` |
| `SDL_RWseek(ctx, offset, whence)` | `SDL_SeekIO(ctx, offset, whence)` |
| `SDL_RWtell(ctx)` | `SDL_TellIO(ctx)` |
| `SDL_RWclose(ctx)` | `SDL_CloseIO(ctx)` |
| `IMG_Init` / `IMG_Quit` | **Removed in SDL_image 3** — decoders init lazily |

## Reading a File

```cpp
SDL_IOStream *io = SDL_IOFromFile("assets/level.bin", "rb");
if (!io) {
    SDL_Log("open: %s", SDL_GetError());
    return false;
}
Sint64 len = SDL_GetIOSize(io);   // -1 on error / unknown
std::vector<std::uint8_t> buf(static_cast<size_t>(len));
size_t got = SDL_ReadIO(io, buf.data(), buf.size());
SDL_CloseIO(io);
```

### Load whole file in one call
```cpp
size_t size = 0;
void *data = SDL_LoadFile("assets/level.bin", &size);
// data is an SDL-allocated buffer; free with SDL_free
SDL_free(data);
```

`SDL_LoadFile` is a convenience for small, startup-time reads. For anything
large or runtime-critical, use `SDL_AsyncIO` (see `storage.md`) so you
don't hitch the frame.

## Writing a File

```cpp
SDL_IOStream *io = SDL_IOFromFile("save.dat", "wb");
SDL_WriteIO(io, bytes.data(), bytes.size());
SDL_CloseIO(io);
```

For save files that need to work on console, go through `SDL_Storage` —
see `storage.md`. Raw `SDL_IOFromFile` only works cleanly on desktop.

For atomicity, write to a temp file and rename — use `SDL_RenamePath`
(see `storage.md`).

## IOStream from Memory

Parse an in-memory blob as if it were a file:

```cpp
std::vector<Uint8> blob = LoadFromNetwork();
SDL_IOStream *io = SDL_IOFromConstMem(blob.data(), blob.size());
// Pass to SDL_image, SDL_ttf, SDL_mixer, or anything that takes SDL_IOStream
SDL_Surface *surf = IMG_Load_IO(io, /*closeio=*/true);  // closes io on completion
```

Useful for:
- Decoding assets pulled from a ZIP / pack file
- Feeding network-delivered content to a decoder
- Unit testing decoders without touching the filesystem

### Dynamic memory stream

```cpp
SDL_IOStream *io = SDL_IOFromDynamicMem();
// Write into it; SDL grows the buffer
SDL_WriteIO(io, bytes.data(), bytes.size());
// Retrieve the final buffer via the SDL_PROP_IOSTREAM_DYNAMIC_MEMORY_POINTER property
// when you're ready to publish / serialize
SDL_CloseIO(io);
```

## Typed Reads / Writes

SDL provides endian-aware fixed-width helpers so you can skip manual byte
juggling:

```cpp
Uint32 magic = 0;
if (!SDL_ReadU32LE(io, &magic)) { /* error */ }

Uint16 version = 0;
SDL_ReadU16BE(io, &version);

SDL_WriteU32LE(io, 0x12345678);
```

Suffix is the on-disk endianness (`LE` / `BE`). Data is converted to / from
native endianness automatically.

## Images — SDL_image 3.x

```cpp
#include <SDL3_image/SDL_image.h>

// Direct to texture (easiest path)
SDL_Texture *tex = IMG_LoadTexture(renderer, "assets/player.png");
if (!tex) {
    SDL_Log("load: %s", SDL_GetError());
    return nullptr;
}

// Load to surface (for CPU-side pixel access)
SDL_Surface *surf = IMG_Load("assets/icon.png");
// ... use surf ...
SDL_DestroySurface(surf);
```

SDL_image 3.x notes:
- **No `IMG_Init` / `IMG_Quit`** — decoders initialize lazily per format
- `IMG_LoadTexture_IO(renderer, io, closeio)` for in-memory or packed
  assets
- Supports PNG, JPG, WebP, AVIF, GIF, BMP, TGA, and more depending on
  build options

## Fonts — SDL_ttf 3.x

```cpp
#include <SDL3_ttf/SDL_ttf.h>

if (!TTF_Init()) { /* error */ }
TTF_Font *font = TTF_OpenFont("assets/fonts/Inter.ttf", /*ptsize=*/16);
SDL_Surface *text = TTF_RenderText_Blended(font, "Hello", 0, { 255, 255, 255, 255 });
SDL_Texture *tex = SDL_CreateTextureFromSurface(renderer, text);
SDL_DestroySurface(text);
// ...
TTF_CloseFont(font);
TTF_Quit();
```

SDL_ttf 3.x notes:
- Text-render functions now take an explicit UTF-8 **length** parameter
  (or `0` for `strlen`)
- Still requires `TTF_Init` / `TTF_Quit` (unlike SDL_image)
- For UI: cache rendered text textures; re-render only when the string
  or style changes

## Audio — SDL_mixer 3.x and Alternatives

For loading decoded audio, see `audio.md` and `PLUGINS.md`.

- **SDL_mixer** — channels + music, rewritten on top of SDL3 audio streams
- **`SDL_LoadWAV`** (core SDL) — only for WAV files
- **stb_vorbis / dr_flac / dr_mp3** — drop-in decoders if you're avoiding
  SDL_mixer
- **miniaudio / FMOD / Wwise** — full middleware replaces SDL audio
  entirely

## Data Formats — Recommendations

| Data | Recommendation |
|------|----------------|
| Tilemap levels | Tiled (TMX / JSON) via `nlohmann/json` or `pugixml` |
| Hierarchical scene data | JSON / TOML — parse once, bake to binary for release |
| Binary baked assets | Hand-rolled binary with version header, or FlatBuffers |
| 3D models | glTF 2.0 via `cgltf` (C, single-header) or `tinygltf` (C++) |
| Audio | WAV (`SDL_LoadWAV`), Ogg Vorbis / FLAC via SDL_mixer or dr_libs / stb_vorbis |
| Images | PNG / JPG / WebP via SDL_image, or `stb_image` if you want zero deps |
| Shaders | HLSL compiled to SPIR-V/DXIL/MSL via SDL_shadercross (see `gpu.md`) |

## Build-Time vs Runtime Processing

Production pipelines typically:

1. Author content in authoring formats (TMX, glTF, WAV, HLSL, PSD)
2. A **build step** converts to runtime formats (baked tilemap, packed
   glTF, compressed textures, compiled shaders)
3. Ship the runtime formats in a pack/zip
4. Load at runtime with minimal parsing

For a new project, starting with loose files + runtime parsing is fine.
Add a build step when load times or binary size start to matter.

## Archives

SDL does not decompress ZIP / 7z / custom archives itself. Options:

- **PhysicsFS** — pairs with `SDL_IOStream` for archive-aware VFS (ZIP,
  7z, custom). See `PLUGINS.md`.
- **Custom pack format** — a flat concatenation with an index header is
  often enough for a shipping game.
- **SDL_Storage custom interface** — implement `SDL_StorageInterface`
  backed by your archive reader (see `storage.md`).

## Common Mistakes

- Using SDL2 `SDL_RWops` symbols — they don't exist in SDL3
- `SDL_RWread(ctx, buf, size, n)` — `SDL_ReadIO` takes **total bytes**, not
  a size/count pair
- Using `free()` on `SDL_LoadFile` / `SDL_LoadWAV` results — SDL owns the
  memory, use `SDL_free`
- Forgetting `SDL_CloseIO` — file handle leaks accumulate fast
- Treating `SDL_ReadIO`'s return value as "error if < requested" — check
  `SDL_GetIOStatus` for the reason (EOF is not an error)
- Reading files directly on the main thread during gameplay — large assets
  should load via `SDL_AsyncIO` (see `storage.md`)
- Shipping a debug build's `assets/` path assumptions to Android / console
  — prefer `SDL_OpenTitleStorage` (see `storage.md`)
- Calling `IMG_Init` / `IMG_Quit` from SDL_image 2 tutorials — removed
  in SDL_image 3

## See Also

- `storage.md` — paths, `SDL_Storage`, directory enumeration, async I/O
- `audio.md` — WAV loading, `SDL_AudioStream`, SDL_mixer integration
- `gpu.md` — shader loading via SDL_shadercross
- `PLUGINS.md` — SDL_image, SDL_ttf, SDL_mixer, PhysicsFS, cgltf,
  stb_image and friends
