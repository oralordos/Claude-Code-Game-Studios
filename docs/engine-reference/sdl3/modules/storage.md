# SDL3 Storage & Filesystem — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

SDL3 introduces a three-layer model for file and persistence access:

1. **`SDL_IOStream`** — low-level byte stream abstraction (renamed from SDL2's `SDL_RWops`). Read/write primitives used by everything above. Covered in `assets.md`.
2. **`SDL_Filesystem` path functions** — platform-portable access to paths (`SDL_GetBasePath`, `SDL_GetPrefPath`, `SDL_GetUserFolder`, `SDL_GetCurrentDirectory`) plus directory / file operations (`SDL_EnumerateDirectory`, `SDL_GetPathInfo`, `SDL_CreateDirectory`, `SDL_CopyFile`, etc.).
3. **`SDL_Storage`** — **entirely new in SDL3.** Opaque handle representing a logical data store. Abstracts platform differences (console bundles, cloud saves, user data) behind one API. Has an async readiness handshake (`SDL_StorageReady`).

There is also **`SDL_AsyncIO`** — new in SDL3 — for non-blocking file I/O independent of the storage layer.

This module is the single largest source of SDL2→SDL3 mismatch risk after
audio and `SDL_gpu`. The training data is dominated by SDL2 `SDL_RWops` +
manual `fopen`-style code, and `SDL_Storage` has no SDL2 equivalent at all.

## SDL_Storage — Pick Your Open Function

`SDL_Storage` is opened via one of four factory functions. Choose based on
what the data represents, not where it happens to live.

| Function | Use for | Platform behavior |
|----------|---------|-------------------|
| `SDL_OpenTitleStorage(override, props)` | **Read-only game data** bundled with the game (baked levels, shaders, fonts, textures) | On desktop: relative to executable or `override` path. On console / mobile: platform handles bundle/APK/IPA access automatically — you do not touch the native APIs. |
| `SDL_OpenUserStorage(org, app, props)` | **Read-write user data** (saves, settings, screenshots the player takes) | On desktop: resolves under `SDL_GetPrefPath`. On console: goes through the platform save system (cloud sync, save slots). |
| `SDL_OpenFileStorage(path)` | Explicit filesystem path — editors, dev tools, moddable data outside the game's user folder | Just a filesystem wrapper. No platform translation. |
| `SDL_OpenStorage(iface, userdata)` | Custom storage implementation — remote assets, archive formats, virtual filesystems | You implement the vtable (`SDL_StorageInterface`). |

### Readiness Handshake

Storage is **not guaranteed ready immediately** after open. On console,
opening user storage may prompt the user to pick a save slot. On mobile,
the OS may be fetching bundled content. Always wait:

```cpp
SDL_Storage *storage = SDL_OpenUserStorage("MyStudio", "MyGame", 0);
if (!storage) { /* error */ return; }

while (!SDL_StorageReady(storage)) {
    // Pump events / draw loading UI / yield
    SDL_Delay(10);
}

// Now safe to read/write
```

For the callbacks API, prefer polling `SDL_StorageReady` inside
`SDL_AppIterate` and deferring the "show save menu" UI flow until it's true.
Do not `SDL_Delay` in a blocking loop on the main thread — that freezes
rendering.

### File Operations

```cpp
// Write a save file
const char *data = R"({"level": 3, "score": 1200})";
size_t len = SDL_strlen(data);
if (!SDL_WriteStorageFile(storage, "save1.json", data, len)) {
    SDL_Log("save: %s", SDL_GetError());
}

// Read a save file (check size first, allocate, then read)
Uint64 size = 0;
if (!SDL_GetStorageFileSize(storage, "save1.json", &size)) {
    // missing or error
    return;
}
std::vector<char> buf(size);
if (!SDL_ReadStorageFile(storage, "save1.json", buf.data(), size)) {
    SDL_Log("load: %s", SDL_GetError());
    return;
}
```

### Directory / Path Operations on Storage

```cpp
SDL_EnumerateStorageDirectory(storage, "saves",
    [](void *userdata, const char *dirname, const char *fname) -> SDL_EnumerationResult {
        SDL_Log("%s / %s", dirname, fname);
        return SDL_ENUM_CONTINUE;
    }, nullptr);

SDL_PathInfo info{};
if (SDL_GetStoragePathInfo(storage, "save1.json", &info)) {
    if (info.type == SDL_PATHTYPE_FILE) {
        SDL_Log("save size: %llu", (unsigned long long)info.size);
    }
}

SDL_CreateStorageDirectory(storage, "screenshots");
SDL_RenameStoragePath(storage, "save1.json", "save1.bak");
SDL_CopyStorageFile(storage, "save1.json", "save1.backup.json");
SDL_RemoveStoragePath(storage, "save_tmp.json");
```

### Clean Up

```cpp
SDL_CloseStorage(storage);   // returns bool; flushes platform queues
```

On console, `SDL_CloseStorage` may block briefly while the platform commits
writes. Always call it before quitting.

## Path Helpers (No Storage Handle Needed)

### SDL_GetBasePath
```cpp
char *base = SDL_GetBasePath();
// Executable directory; good for looking up bundled assets on desktop
// Never write here (may be read-only on macOS .app bundles)
SDL_free(base);
```

### SDL_GetPrefPath
```cpp
char *pref = SDL_GetPrefPath("MyStudio", "MyGame");
// Writable per-user data dir:
//   macOS: ~/Library/Application Support/MyStudio/MyGame/
//   Windows: %APPDATA%\MyStudio\MyGame\
//   Linux: $XDG_DATA_HOME/MyStudio/MyGame/  or ~/.local/share/MyStudio/MyGame/
SDL_free(pref);
```

For anything going to cloud save or console save slots, prefer
`SDL_OpenUserStorage` over `SDL_GetPrefPath` — it does the right thing on
platforms where `SDL_GetPrefPath` doesn't exist in a meaningful form.

### SDL_GetUserFolder (SDL3 NEW)

Cross-platform access to OS-standard user folders:

```cpp
const char *docs = SDL_GetUserFolder(SDL_FOLDER_DOCUMENTS);
if (docs) {
    // e.g. macOS "/Users/alice/Documents/", Windows "C:\\Users\\Alice\\Documents\\"
}
```

Available folders: `HOME`, `DESKTOP`, `DOCUMENTS`, `DOWNLOADS`, `MUSIC`,
`PICTURES`, `PUBLICSHARE`, `SAVEDGAMES`, `SCREENSHOTS`, `TEMPLATES`, `VIDEOS`.

Returns `NULL` if the platform has no concept of that folder (common on
console / embedded). Always null-check.

**Use this for:** "Open containing folder" buttons, screenshot destinations,
"Export to Downloads", replay-sharing flows. Do NOT use it for saves — those
go through `SDL_OpenUserStorage` or `SDL_GetPrefPath`.

### SDL_GetCurrentDirectory (SDL3 NEW)

```cpp
char *cwd = SDL_GetCurrentDirectory();
// Almost never what a game wants — use BasePath/PrefPath/UserFolder instead.
// Useful only for command-line tools / dev utilities.
SDL_free(cwd);
```

## Directory Enumeration Without Storage

For arbitrary filesystem paths (dev tools, editors, mod loaders):

```cpp
SDL_EnumerateDirectory("mods/",
    [](void *userdata, const char *dirname, const char *fname) -> SDL_EnumerationResult {
        auto *mods = static_cast<std::vector<std::string>*>(userdata);
        mods->emplace_back(std::string(dirname) + fname);
        return SDL_ENUM_CONTINUE;
    }, &mod_list);
```

Return values from the callback:
- `SDL_ENUM_CONTINUE` — keep going
- `SDL_ENUM_SUCCESS` — stop early, report success
- `SDL_ENUM_FAILURE` — stop, report failure

### SDL_GlobDirectory

Pattern-match filenames (`*.png`, `level_*.json`, etc.):

```cpp
int count = 0;
char **matches = SDL_GlobDirectory("levels/", "level_*.json", 0, &count);
for (int i = 0; i < count; ++i) {
    // matches[i] is a filename relative to "levels/"
}
SDL_free(matches);   // single free for the whole list
```

Pattern syntax is glob-style (`*`, `?`, `[abc]`). Flags include
`SDL_GLOB_CASEINSENSITIVE`.

## Path Metadata (stat-like)

```cpp
SDL_PathInfo info{};
if (SDL_GetPathInfo("assets/player.png", &info)) {
    switch (info.type) {
        case SDL_PATHTYPE_NONE:      /* doesn't exist */ break;
        case SDL_PATHTYPE_FILE:      /* regular file */ break;
        case SDL_PATHTYPE_DIRECTORY: /* directory */ break;
        case SDL_PATHTYPE_OTHER:     /* symlink, device, pipe, etc. */ break;
    }
    // info.size, info.create_time, info.modify_time, info.access_time
    // Times are Uint64 nanoseconds since SDL epoch — not POSIX time_t
}
```

## File & Directory Operations (Non-Storage)

```cpp
SDL_CreateDirectory("output/cache");   // creates intermediate dirs
SDL_RenamePath("tmp.dat", "final.dat");
SDL_CopyFile("template.json", "user.json");
SDL_RemovePath("stale.log");           // file or empty directory
```

Return `bool`. Errors go to `SDL_GetError()`. There is intentionally no
recursive-delete helper — loop `SDL_EnumerateDirectory` + `SDL_RemovePath`
yourself, or leave cleanup to the OS temp-file mechanism.

## SDL_AsyncIO — Non-Blocking File I/O (SDL3 NEW)

Blocking reads on the main thread cause frame hitches. Use `SDL_AsyncIO` for
anything larger than a few KB.

### Basic Pattern

```cpp
// Create a queue to collect completion events
SDL_AsyncIOQueue *queue = SDL_CreateAsyncIOQueue();

// Open a file for async reads
SDL_AsyncIO *aio = SDL_AsyncIOFromFile("big_texture.png", "r");

// Kick off a read; SDL fills the buffer in the background
std::vector<Uint8> buf(expected_size);
SDL_ReadAsyncIO(aio, buf.data(), /*offset=*/0, buf.size(),
                queue, /*userdata=*/nullptr);

// In a later frame, poll the queue:
SDL_AsyncIOOutcome outcome{};
while (SDL_GetAsyncIOResult(queue, &outcome)) {
    if (outcome.result == SDL_ASYNCIO_COMPLETE) {
        // buf is now filled with outcome.bytes_transferred bytes
        UploadTexture(buf);
    } else if (outcome.result == SDL_ASYNCIO_FAILURE) {
        SDL_Log("async read failed: %s", SDL_GetError());
    }
}

// Cleanup
SDL_CloseAsyncIO(aio, /*flush=*/true, queue, nullptr);
SDL_DestroyAsyncIOQueue(queue);
```

### Use For

- Streaming music / ambience while playing
- Loading the next level's assets in the background during gameplay
- Decompressing save-game snapshots without a hitch
- Any read > ~64 KB that happens during active play

### Do NOT Use For

- Tiny config files at startup (`SDL_LoadFile` is simpler)
- Write paths where you need a durable ack before continuing (use sync
  storage writes + a background thread if you need the write to be
  non-blocking, since `SDL_AsyncIO` completion means "submitted", not
  "durable on disk")

### Write Flush

After `SDL_WriteAsyncIO`, data may still be in kernel buffers when the
outcome reports complete. If you need durability guarantees, issue a
close with `flush=true` and wait for the close outcome before telling
the player "saved."

## Decision Guide

**I want to save the player's progress** → `SDL_OpenUserStorage` +
`SDL_WriteStorageFile`. On console this goes through the platform save
system; on desktop it hits `SDL_GetPrefPath` transparently.

**I want to load bundled level data that shipped with the game** →
`SDL_OpenTitleStorage` + `SDL_ReadStorageFile`. On desktop the override
path or executable relative dir; on console the platform bundle.

**I want to read a mod that lives in an arbitrary folder** →
`SDL_OpenFileStorage(path)` for a storage handle, or `SDL_IOFromFile` for
simple cases.

**I want the player's Documents folder for a "Save As" dialog** →
`SDL_GetUserFolder(SDL_FOLDER_DOCUMENTS)`.

**I want to list every file in a folder** → `SDL_EnumerateDirectory`
(always) or `SDL_GlobDirectory` (when you know a pattern).

**I want to stat a file** → `SDL_GetPathInfo`.

**I want to copy / move / delete a file** → `SDL_CopyFile` / `SDL_RenamePath`
/ `SDL_RemovePath`.

**I want to load a 100 MB asset without stalling** → `SDL_AsyncIO`.

**I want to load an asset inside a ZIP pack** → SDL does not decompress
archives itself. Pair `SDL_IOStream` with PhysicsFS, or bake a custom pack
format.

## Common Mistakes

- **Not calling `SDL_StorageReady`** after `SDL_OpenUserStorage` — reads
  and writes will fail silently or stall before the platform is ready.
- **Using `SDL_GetPrefPath` for console saves** — console platforms have
  no meaningful pref path; you must go through `SDL_Storage`.
- **Hardcoding `"./assets/..."`** — breaks on macOS `.app` bundles,
  Android, Emscripten, and console. Always anchor to `SDL_GetBasePath()`,
  `SDL_OpenTitleStorage`, or `SDL_IOFromFile` with a relative path SDL
  resolves.
- **Using `SDL_GetUserFolder` for save data** — on many consoles it
  returns `NULL` because the concept doesn't exist. Saves go through
  `SDL_OpenUserStorage`.
- **Treating `SDL_PathInfo.create_time` as `time_t`** — it's SDL
  nanosecond time. Use `SDL_NSToTimeSpec` or the conversion helpers.
- **Reading 100 MB on the main thread** via `SDL_LoadFile` — use
  `SDL_AsyncIO` or a worker thread.
- **Forgetting `SDL_free`** on `SDL_GetBasePath` / `SDL_GetPrefPath` /
  `SDL_GetCurrentDirectory` results — they allocate, you free.
- **Calling `SDL_CloseStorage` mid-write** without waiting for async
  completion — data may be lost on console.
- **Ignoring `SDL_WriteStorageFile` return value** — on a full save slot
  or broken cloud sync, writes fail and you must surface it to the player.
- **Using `SDL_IOStream` callbacks for save files that need to work on
  console** — go through `SDL_Storage`; it knows about save slots, cloud
  sync, and platform certification requirements.

## RAII Wrappers (C++)

See `sdl3-cpp-specialist` for the general pattern. Quick reference:

```cpp
struct StorageDeleter { void operator()(SDL_Storage *s) const noexcept { if (s) SDL_CloseStorage(s); } };
struct IOStreamDeleter { void operator()(SDL_IOStream *s) const noexcept { if (s) SDL_CloseIO(s); } };
struct AsyncIOQueueDeleter { void operator()(SDL_AsyncIOQueue *q) const noexcept { if (q) SDL_DestroyAsyncIOQueue(q); } };
// SDL_AsyncIO close is two-stage (flush + queue notification) — wrap in a helper type, not a raw unique_ptr

using StoragePtr       = std::unique_ptr<SDL_Storage,       StorageDeleter>;
using IOStreamPtr      = std::unique_ptr<SDL_IOStream,      IOStreamDeleter>;
using AsyncIOQueuePtr  = std::unique_ptr<SDL_AsyncIOQueue,  AsyncIOQueueDeleter>;
```

## See Also

- `assets.md` — `SDL_IOStream` plumbing, satellite-library asset loading
  (SDL_image, SDL_ttf, SDL_mixer), format recommendations
- `events.md` — main-thread rules when integrating async IO with the event loop
- `PLUGINS.md` — PhysicsFS for archive / zip-pack asset bundling
