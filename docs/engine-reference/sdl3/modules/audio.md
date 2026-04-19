# SDL3 Audio — Quick Reference

Last verified: 2026-04-18 | Engine: SDL 3.2 stable series

SDL3 replaced SDL2's callback / queue audio model with a **streams-based
model**. The callback-based API in SDL2 tutorials is gone.

## Concept Model

```
SDL_AudioDeviceID         — logical or physical playback/capture device
SDL_AudioSpec             — format (sample format, channels, frequency)
SDL_AudioStream           — source-format-to-device-format converter + buffer
```

Typical flow:
1. Open a **logical** device (usually `SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK`)
2. Create an `SDL_AudioStream` bound to that device, describing the format
   you want to *push* in (e.g. stereo F32 at 48 kHz)
3. Push PCM with `SDL_PutAudioStreamData`; SDL resamples/converts and mixes
   into the device
4. Resume the device-stream linkage to start playback

## What Changed from SDL2

| SDL2 | SDL3 |
|------|------|
| `SDL_QueueAudio` / `SDL_DequeueAudio` | **Removed** — use `SDL_PutAudioStreamData` / `SDL_GetAudioStreamData` |
| Global `SDL_OpenAudio` | **Removed** — use `SDL_OpenAudioDevice` / `SDL_OpenAudioDeviceStream` |
| Callback-based audio with a thread SDL owns | Opt-in via `SDL_SetAudioStreamGetCallback`; otherwise push-model |
| Converting samples via `SDL_AudioCVT` | **Removed** — streams do this for you |
| Mixing via `SDL_MixAudioFormat` | Still present; prefer streams for mixing |
| `SDL_AudioSpec` with `callback` / `userdata` fields | Fields removed; callbacks are wired through stream APIs |

## Simplest Playback Setup

```cpp
// Describe what we'll PUT into the stream
SDL_AudioSpec source_spec {};
source_spec.format = SDL_AUDIO_F32;
source_spec.channels = 2;
source_spec.freq = 48000;

// Open default playback device and get a stream back, bound to it.
// SDL will resample/convert from source_spec to whatever the device wants.
SDL_AudioStream *stream = SDL_OpenAudioDeviceStream(
    SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK,
    &source_spec,
    /*callback=*/nullptr,   // push model
    /*userdata=*/nullptr);
if (!stream) {
    SDL_Log("open audio stream: %s", SDL_GetError());
    return false;
}

// Device starts paused — resume when you want audio
SDL_ResumeAudioStreamDevice(stream);
```

## Push Samples Each Frame

```cpp
std::array<float, 2 * 1024> pcm;   // stereo, 1024 frames
fill_pcm(pcm);                     // your DSP / decode / mix

if (!SDL_PutAudioStreamData(stream, pcm.data(), pcm.size() * sizeof(float))) {
    SDL_Log("put audio: %s", SDL_GetError());
}

// Optional: check how much audio is already queued (to avoid runaway growth)
int queued_bytes = SDL_GetAudioStreamQueued(stream);
if (queued_bytes > AUDIO_MAX_QUEUED_BYTES) {
    // skip this push — we're ahead
}
```

## Pull-Model Callback

If you prefer SDL to ask for audio (like the SDL2 callback style):

```cpp
static void SDLCALL stream_callback(void *userdata, SDL_AudioStream *stream,
                                    int additional_amount, int total_amount) {
    // SDL wants `additional_amount` more bytes soon
    auto *synth = static_cast<Synth*>(userdata);
    auto buf = synth->Render(additional_amount / sizeof(float));
    SDL_PutAudioStreamData(stream, buf.data(), buf.size() * sizeof(float));
}

SDL_AudioStream *stream = SDL_OpenAudioDeviceStream(
    SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK, &source_spec,
    stream_callback, &my_synth);
SDL_ResumeAudioStreamDevice(stream);
```

The callback runs on an audio thread SDL manages — do not block, allocate,
or touch gameplay state directly. Use lock-free queues to pass data in.

## Loading a WAV

```cpp
SDL_AudioSpec wav_spec {};
Uint8 *wav_data = nullptr;
Uint32 wav_len = 0;
if (!SDL_LoadWAV("assets/jump.wav", &wav_spec, &wav_data, &wav_len)) {
    SDL_Log("load wav: %s", SDL_GetError());
    return;
}

// Push the whole file into the stream (SDL handles resampling if wav_spec
// doesn't match the stream's source_spec — actually, match them or create
// a dedicated stream per clip with the WAV's own format).
SDL_PutAudioStreamData(stream, wav_data, wav_len);

SDL_free(wav_data);  // NOT free/delete — SDL owns it
```

For music / long assets or non-WAV formats, use `SDL_mixer` or
a decoder library (dr_flac, stb_vorbis, miniaudio's decoders).

## Multiple Concurrent Sounds

Two approaches:

**A — One stream per sound** (simplest for small numbers of SFX):
- Create a stream per clip, push all data at once, resume
- SDL will mix all streams into the same device automatically

**B — Software mix ahead of time** (for many concurrent sounds):
- Maintain your own voice/channel mixer
- Output one final stereo buffer per frame
- Push to a single stream

For non-trivial needs: prefer `SDL_mixer` or FMOD/Wwise. The push-model
primitives are building blocks, not a full audio engine.

## Spatial Audio

SDL3 audio does NOT provide 3D positional audio out of the box. Options:
- Manual attenuation + panning — fine for simple cases
- `miniaudio` (single-header; has spatialization)
- FMOD Studio or Wwise (commercial middleware)

## Pause / Stop

```cpp
SDL_PauseAudioStreamDevice(stream);   // freezes output, stream data kept
SDL_ResumeAudioStreamDevice(stream);
SDL_FlushAudioStream(stream);         // drain any buffered data
SDL_ClearAudioStream(stream);         // drop buffered data
SDL_DestroyAudioStream(stream);       // close stream AND the bound logical device (if opened via SDL_OpenAudioDeviceStream)
```

## Common Mistakes

- Using `SDL_QueueAudio` from SDL2 tutorials — it does not exist in SDL3
- Forgetting to `SDL_ResumeAudioStreamDevice` — the default is paused
- Calling `free()` on WAV data instead of `SDL_free`
- Blocking in the pull callback — causes audio glitches
- Pushing uncapped amounts of audio per frame — grows memory unbounded
- Mixing raw buffer sizes with frame counts — always compute in bytes for
  `SDL_PutAudioStreamData`
- Assuming the device format matches what you pushed — the stream resamples
  silently; query `SDL_GetAudioStreamFormat` to confirm

## Quick Decision Guide

**One-shot SFX, simple game** → SDL_mixer on top of SDL_audio
**Music with fade in/out** → SDL_mixer
**Custom DSP / synth** → push-model `SDL_AudioStream`
**Adaptive music (stems, crossfades, transitions)** → FMOD Studio or Wwise
**3D positional audio** → miniaudio or FMOD Studio or Wwise
**VR / binaural audio** → third-party middleware
