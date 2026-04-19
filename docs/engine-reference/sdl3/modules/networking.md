# SDL3 Networking — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

SDL3's core library does **not** include networking. Networking lives in
the optional satellite library **SDL_net 3.x**. SDL_net is a thin sockets
wrapper suitable for simple cases — if you need serious multiplayer, look
at ENet, GameNetworkingSockets, or a dedicated netcode library.

## What SDL_net Provides

- TCP sockets (blocking & non-blocking)
- UDP sockets
- DNS resolution
- Simple address abstraction (`SDLNet_Address`, `SDLNet_Server`, etc.)

## What SDL_net Does NOT Provide

- Reliability layer over UDP (no ACKs, no retransmit, no ordering)
- Client-side prediction / server reconciliation
- Lobby matching, NAT traversal, relay
- Built-in serialization / schema tools

**If your game needs any of the above, do not use SDL_net for gameplay.**
Options:

| Need | Use |
|------|-----|
| Real-time game state sync | ENet, GameNetworkingSockets, yojimbo |
| Turn-based games with simple messaging | SDL_net is fine |
| HTTP / REST (leaderboards, patches) | cpr, libcurl, native platform HTTP APIs |
| NAT traversal / matchmaking | Platform services (Steam, EOS, PlayStation Network) |
| WebSockets (e.g. Emscripten) | Platform/Emscripten APIs, or a third-party lib |

## Minimal TCP Client (SDL_net 3)

```cpp
#include <SDL3_net/SDL_net.h>

if (!SDLNet_Init()) {
    SDL_Log("SDLNet_Init: %s", SDL_GetError());
    return false;
}

SDLNet_Address *addr = SDLNet_ResolveHostname("example.com");
// Wait synchronously (block) for resolution
while (SDLNet_GetAddressStatus(addr) == 0) {
    SDL_Delay(10);
}
if (SDLNet_GetAddressStatus(addr) < 0) {
    SDL_Log("resolve: %s", SDL_GetError());
    return false;
}

SDLNet_StreamSocket *sock = SDLNet_CreateClient(addr, /*port=*/8080);
SDLNet_UnrefAddress(addr);

// Connect (blocking)
while (SDLNet_GetConnectionStatus(sock) == 0) {
    SDL_Delay(10);
}
if (SDLNet_GetConnectionStatus(sock) < 0) {
    SDL_Log("connect: %s", SDL_GetError());
    return false;
}

// Send
const char *msg = "hello\n";
SDLNet_WriteToStreamSocket(sock, msg, SDL_strlen(msg));

// Read
char buf[1024];
int got = SDLNet_ReadFromStreamSocket(sock, buf, sizeof(buf));
if (got > 0) {
    SDL_Log("received %d bytes", got);
}

SDLNet_DestroyStreamSocket(sock);
SDLNet_Quit();
```

## Minimal UDP (SDL_net 3)

```cpp
SDLNet_DatagramSocket *sock = SDLNet_CreateDatagramSocket(nullptr, 9999);

// Non-blocking receive
SDLNet_Datagram *dgram = nullptr;
int got = SDLNet_ReceiveDatagram(sock, &dgram);
if (got > 0 && dgram) {
    // dgram->buf / dgram->buflen / dgram->addr / dgram->port
    SDLNet_DestroyDatagram(dgram);
}

// Send
SDLNet_SendDatagram(sock, remote_addr, remote_port, msg, msg_len);

SDLNet_DestroyDatagramSocket(sock);
```

## Threading Strategy

Network I/O **should not run on the main thread** for anything beyond a
dev console. Options:

1. **Dedicated network thread** — blocks on reads, posts events to the
   main thread via `SDL_PushEvent` or a lock-free queue
2. **Non-blocking sockets + poll loop** — net thread runs a tight loop,
   main thread reads decoded messages from an SPSC queue
3. **coroutines / async (C++20)** — if you're already in an async
   runtime, wrap the socket in an awaiter

## Serialization

SDL_net gives you raw bytes. You pick the serialization:

- **Hand-rolled** binary — cheapest, most fragile (version carefully)
- **FlatBuffers** — zero-copy, schema-evolvable
- **Protobuf** — widely understood, slightly heavier runtime
- **Cap'n Proto** — zero-copy, RPC-capable
- **MessagePack / CBOR** — small, schemaless, good for small events
- **Cereal / bitsery / zpp_bits** — C++-native binary serialization

Always version your protocol — stamp every packet with a protocol version
byte and reject mismatches.

## Security Notes

- **Never trust the network** — validate every field from every peer
- Client-authoritative state is an invitation to cheat — keep authority on
  the server
- For encryption, pair SDL_net with BoringSSL / mbedTLS, or use
  GameNetworkingSockets which includes Valve's authenticated-encryption
  layer
- For web/browser targets, only WebSockets / WebRTC work — SDL_net will
  not function in Emscripten

## Common Mistakes

- Using SDL_net for real-time gameplay without a reliability layer on top
- Blocking the main thread on `SDLNet_ReadFromStreamSocket`
- Forgetting to call `SDLNet_Init` before any other net function
- Leaking `SDLNet_Datagram` — every received datagram needs
  `SDLNet_DestroyDatagram`
- Assuming SDL_net is available on all platforms SDL3 supports — check
  before depending on it for mobile/console
- Mixing blocking and non-blocking I/O on the same socket
- Treating `SDLNet_GetConnectionStatus` as boolean — it's `1` connecting,
  `0` pending, `-1` failed

## Recommended Reading

- SDL_net docs: https://wiki.libsdl.org/SDL_net/FrontPage
- ENet: http://enet.bespin.org/
- GameNetworkingSockets: https://github.com/ValveSoftware/GameNetworkingSockets
- Glenn Fiedler's netcode articles: https://gafferongames.com/
