# Horizon Voice Chat

> **This is a maintained fork** of [Nirokx/horizon-voice-chat](https://github.com/Nirokx/horizon-voice-chat)
> (MIT, original author Nirokx). It builds on current Horizon Nightlies through the native hook points and
> works on Windows with the bundled `opus.dll`; these changes are also offered upstream
> ([Nirokx/horizon-voice-chat#2](https://github.com/Nirokx/horizon-voice-chat/pull/2)). Download the
> `.hnmod`, `opus.dll`, `rnnoise.dll` and their licences from [Releases](https://github.com/BnCatt/horizon-voice-chat/releases):
> put the `.hnmod` in `horizon-native-mods/` and the DLLs in `horizon-mods/.horizon-mod-data/voicechat/`,
> then build in the launcher.

A native Horizon mod (Minecraft 26.3, native hook API 409 and later; tested on Horizon Nightly
2026-11-05, `f319701`) that is a client for
the [Simple Voice Chat](https://github.com/henkelmax/simple-voice-chat) protocol. It uses the same
protocol as the Java mod, so on any server with the voice chat mod or plugin you can talk with players
who play on the regular Java launcher with Simple Voice Chat.

It looks like the Java mod: vanilla buttons, sliders and text fields (drawn from the game's own GUI
sprites, so a resource pack restyles them), the mod's own icons, an icon in the bottom left corner and a
speaker over the head of whoever is talking.

- **Voice Chat (V)**: four icon buttons (microphone on/muted, voice chat on/off, groups, settings) and
  **Adjust Volumes**, with the connection's status under them. Escape goes back one screen, V closes.
- **Settings**: voice volume (0-200 %), microphone amplification (0-500 %) and the voice activation
  threshold as sliders (the threshold with a live microphone meter behind it), push to talk or voice
  activation, **noise suppression** (rnnoise), the microphone and the speakers by name (or the
  system's default), hide icons, the group list on screen, group names above players, the protocol.
- **Groups**: the server's groups, a lock on those with a password and the member count (the members
  in a tooltip); **Join** (a password group asks for its password in the row), **Leave** for yours, and
  **Create Group...** (name, optional password, type). The wheel scrolls; long names are cut with "…".
- **Adjust Volumes**: a slider for each other player with voice chat, 0-400 %, kept per player.
- **Talking**: hold **Caps Lock** (push to talk), or switch to voice activation. Voices come from where
  the players stand: they get quieter with distance and pan left or right, as the Java mod's do.
- **Keys** (Options > Mods > Horizon Voice Chat): voice chat (V), push to talk (Caps Lock), whisper (not
  bound), mute the microphone (J), voice chat off/on (K).
- **On screen**: the status icon (bottom left: microphone, muted, off, disconnected, whispering), what
  doesn't work above it in red, your group's members (top left, a speaker by whoever is talking), each
  player's group above their head (`[Builders]`; yours in green) and a speaker above it while they talk.
  `/vc groups` lists every group with its members; `/vc join <group> [password]`, `/vc leave` and
  `/vc create <name> [password] [normal|open|isolated]` work too, and so does the server's `/voicechat`.
- **`/vc`**: `status`, `mute`/`unmute`, `off`/`on`, `mode ptt|voice`, `volume <0-200>`,
  `volume <player> <0-400>`, `gain <10-500>`, `threshold <dB>`, and the group commands above.
- **Options > Mods page**: the same settings as buttons that step through values.

Both Simple Voice Chat protocols are supported: **2.6.x** (compatibility version 20, the current
releases) and **2.7** (21, the pre-release). The mod asks with 2.6.x first and then 2.7, and remembers
which one each server answered. On a 2.7 server the first visit may show "Your voice chat client version
is not compatible" once, before the 2.7 request succeeds. Settings > Protocol fixes the version if you
prefer.

It needs **libopus**, and **rnnoise** for noise suppression (without it the button says N/A and the
microphone goes out as it is). It looks for both in its own data folder first
(`horizon-mods/.horizon-mod-data/voicechat/`: `opus.dll` and `rnnoise.dll` on Windows,
`libopus.0.dylib`/`librnnoise.0.dylib` on macOS, `libopus.so.0`/`librnnoise.so.0` on Linux), then the
system's. Windows has neither: build them with `tools/build_opus.py` and `tools/build_rnnoise.py`
(below) and put them in that folder.

## Not affiliated

This is an unofficial project. It is not made, endorsed or supported by Max Henkel or the Simple
Voice Chat project; please don't report its problems to them. "Simple Voice Chat" is named to say
which mod and plugin it works with.

## Where the protocol code comes from

This is **not** a clean-room implementation. The mod was written with an AI coding assistant
(Claude Code), which read [Simple Voice Chat's source](https://github.com/henkelmax/simple-voice-chat)
on GitHub (its packet classes, `AudioChannel`, `MicThread` and some config defaults) to implement the
protocol. No code is copied verbatim (this is Rust, Simple Voice Chat is Java), but the protocol and
audio code follow Simple Voice Chat's implementation, and comments name its classes. Simple Voice Chat
is "Copyright (c) Max Henkel - All Rights Reserved". If you are its author and want this changed or
taken down, open an issue and it will be.

## Build and install

Uses the Horizon launcher's own Rust and Horizon source (nothing else to install):

```sh
L=~/.local/share/horizon-launcher
export PATH=$L/rust/1.99.1-x86_64-unknown-linux-gnu/bin:$PATH CARGO_HOME=$L/cargo
(cd mixins/voicechat/engine && cargo test --offline && rm -f Cargo.lock)   # the engine's tests
python3 tools/package.py --install        # target/voicechat-1.2.0.hnmod (+ .hmod), the .hnmod copied into the launcher's native mods
python3 -m unittest tools/test_make_icons.py
```

`package.py` makes two files. `voicechat-1.2.0.hnmod` is the native mod alone, the file the launcher reads
from `horizon-native-mods/` today; it draws the icons from pixels compiled into the mod
(`engine/src/icons.rs`). `voicechat-1.2.0.hmod` is the same mod with `horizon-mod.toml` and `assets/`
(the icons as GUI textures, `voicechat:gui/icons/<name>`), the combined form for launchers that load it;
the mod then draws those textures, which a resource pack can replace. The icons come from
`tools/make_icons.py` (16×16 pixel grids, the mod's own drawings), which writes both the PNGs and
`icons.rs`.

Then build the game with its mods in the launcher. The mod declares `"hooks": 409`: the server's
payloads come through the `client_packet` hook point (API 408) and go out through `send_packet`
(API 409), so the launcher builds it on any Horizon with that API. Its one reach into the game's own
state is `mixins/voicechat/world.rs`, a module of `game::remote` that reads the joined address, the
other players and the camera (the hook points don't offer those to a client yet); a Nightly that
moves those fields breaks the build there and nowhere else.

On Windows, `opus.dll` from the official libopus source with the launcher's own MinGW:

```sh
# opus-x.y.z from https://downloads.xiph.org/releases/opus/ (check SHA256SUMS.txt)
python tools/build_opus.py <opus-x.y.z dir> "%APPDATA%/horizon-launcher/rust/<ver>-x86_64-pc-windows-gnu/mingw64/bin/gcc.exe" out
# rnnoise-0.2.tar.gz from https://github.com/xiph/rnnoise/releases/tag/v0.2 (the script checks its SHA-256)
python tools/build_rnnoise.py rnnoise-0.2.tar.gz "<the same gcc.exe>" out
# copy out/opus.dll, out/rnnoise.dll and out/COPYING.* into horizon-mods/.horizon-mod-data/voicechat/
```

`VOICECHAT_TEST_OPUS_DIR=out cargo test` runs the Opus and noise suppression tests against those DLLs.

For tests without a microphone, `HORIZON_VOICECHAT_TEST_TONE=1` sends a steady 440 Hz tone instead of
the microphone, and the log (`logs/latest.log`, lines `Voice Chat: …`) counts the packets sent and heard.

## Layout

- `mixins/voicechat/engine/`: the mod's own crate (built once and cached), with its unit tests
  - `proto.rs`: Simple Voice Chat's packets, both versions; `buf.rs`: `FriendlyByteBuf`
  - `crypt.rs`: AES-128-GCM, as `Secret` encrypts; `opus.rs`: libopus, loaded at run time
  - `net.rs`: the UDP connection (authenticate, check, keep alive) and the mic thread
  - `mix.rs`: jitter buffers, distance and panning, resampling; `audio.rs`: microphone and speakers (cpal)
  - `denoise.rs`: rnnoise, loaded at run time; `dylib.rs`: finding the C libraries
  - `screens/`: each screen's layout; `ui.rs`: the screen stack, clicks, drags and typing;
    `skin.rs`: vanilla's nine-slice widgets; `icons.rs`: the icons' pixels; `text.rs`: every label
  - `hud.rs`: the status icon and group list; `labels.rs`: group names and speakers above players;
    `lib.rs`: the session, keys, `/vc` and settings
- `mixins/voicechat/*.rs`: one small hook adapter per hook point
- `mixins/voicechat/horizon-mixin.toml`: where each hook goes
- `horizon-mod.toml`, `assets/`: the content half of the `.hmod` (the icons)
- `tools/package.py`: makes the `.hnmod` and `.hmod`; `tools/make_icons.py`: the icons;
  `tools/build_opus.py`, `tools/build_rnnoise.py`: the Windows DLLs

## Tested

- Engine unit tests: the packet formats of both versions, the AES-GCM test vector, Opus through the
  system libopus, the mixer, distance and panning, the session, the group display, and the menu
  (hotkeys and groups shown, joining a password group, typing a new group).
- The bundle applies to Horizon `c9d474d` with horizon-mixins 0.6.0, and the modded client
  type-checks (`cargo check -p horizon-client`).
- Live, against Paper 26.3 with the Simple Voice Chat 2.6.27 plugin: two clients driven by this engine
  authenticated and connected to voice chat; one heard the other's voice (a test tone through
  Opus and the server); creating and joining groups, the member lists and the labels above players all
  came from the server.
- 1.1.0 on Horizon `f319701` (and `main` at `63eccfc`): the bundle applies and the game builds; live
  against Paper **1.21.11** (through Horizon's older-servers adapter) with the Simple Voice Chat
  2.6.27 plugin, two Horizon clients connected to voice chat and one heard 1000+ sound packets of the
  other's test tone, on Windows with the built `opus.dll`.
- 1.2.0: the engine's 50 unit tests (each screen's layout at several GUI sizes, sliders dragged and
  stored, the group screens, device choice, the HUD, nine-slice sprites, rnnoise turning hiss down by
  over 70 dB while keeping a tone, Opus) pass on Windows with the built DLLs; the changed hook adapters
  type-check against stand-ins with `f319701`'s signatures (not a full game build). The screens have not been seen in the real game yet.
- Not tested yet: the screens in the real game in the real game, 2.7 servers live (no 2.7 plugin build is published), a real microphone and speakers
  in the game, and talking with the Java mod itself.

## License

MIT, see [LICENSE](LICENSE), for the parts that are this project's own work. Simple Voice Chat itself is
not covered by this license and remains its author's, including whatever of its design the protocol
code follows (see above).
