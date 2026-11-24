# media.file — Play MP3s on a Muse Gadget

media.file is a Link command that plays an MP3 stored on Muse's VM through an ESP32 gadget. Give the MP3's location in `path`, and the gadget fetches the file and plays it. The command replies at the moment playback starts.

Try this: use the app or PC chat interface to ask, "Convert today's weather into an MP3 and send it back." Were you able to play the resulting file? If so, you can then send it to your gadget.

## Why I made it

I wanted to hear Muse's replies through the gadget's speaker when I talk to the gadget. Muse can make the reply audio itself, so no external TTS is needed. If Muse makes an MP3 and media.file points at it, the gadget can play it.

The stock SDK had a test feature for playing MP3s, so I used that processing and added a command that lets Muse ask, "play this MP3."

*Because it takes the form of a command addition, even if the SDK itself is updated, the impact of fixes stays within the added parts.*

## Usage examples

The agent can send audio at whatever timing it likes, so this also works for alarms, skill-completion notifications, and playing voice messages. For example, set the agent up like this:

When this skill finishes, make an audio that says "The work is finished" and send it to the gadget.

An example setup for returning talk replies as audio is in the separate file muse_setup.en.md.

It is specified as a path, but in practice the path is not something to worry about. Files stored inside Muse are not something the user needs to be aware of. And if the user places any MP3 they made where Muse can reach it (again, without having to think about it) and tells Muse to send it, Muse sends it to the gadget.

## Applying the patch

The target is Muse Gadget SDK upstream main ded4222 (2026-11-21, #490). Prepare a clean `ded4222` tree and run this at the SDK root.

```
git apply media_file.patch
```

Add CONFIG_HOMEHUB_MP3_COMMAND=y to the board's sdkconfig overlay, then build and flash. It is disabled by default.

## Changes

8 files in total. Only one existing line is rewritten, in muse_voice.c.

| File | Changes |
|---|---|
| main/muse_media.c / main/muse_media.h | New. Fetches the MP3 with GET /fs/raw using Range requests, and converts it to 16 kHz mono PCM based on the stock mp3_selftest(). The PCM and the resident worker's stack (32 KB) are placed in PSRAM. The worker's queue depth is 1. |
| components/muse/muse_voice.c / components/muse/muse_voice.h | Adds muse_voice_request_pcm() to receive PCM, and the playback handling in the voice task. Adds !s_media_play to the existing rest condition. |
| main/noise_control.cpp | Registers media.file right after display.draw_file. |
| main/app.c | Adds the media.file command branch and completion callback. |
| main/CMakeLists.txt | Adds muse_media.c and the dependency on minimp3. |
| main/Kconfig.projbuild | Adds HOMEHUB_MP3_COMMAND. Default is n. |

## Notes

- It does not sync with the text display during a talk turn. This is a command that plays an MP3 on its own.
- File size is up to 512 KB, and playback is up to about 60 seconds. The command timeout is 60 seconds. If the file size or playback time exceeds the limits, processing is cut off or decode_failed is returned.
- If the command is received during a talk, playback happens after that turn ends.
- Sending the next MP3 during playback returns decode_failed. Decoding pre-allocates the full 60 seconds of PCM space (1.92 MB), so during playback there is not enough free PSRAM.
- Enabling media.file in a configuration without audio (MUSE_ENABLED disabled) results in a link error at build time.
