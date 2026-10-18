<p align="center"><img src="assets/icon/icon.png" alt="Virtua Fighter 2 Recomp" width="160"></p>

<h1 align="center">Virtua Fighter 2 Recomp</h1>

<p align="center"><b>Virtua Fighter 2 (SEGA, arcade, Model 2, 1994) as a native PC game: the game's own i960 program,
recompiled to native code, with render scales up to 6x, widescreen and smooth motion at any refresh rate.</b></p>

<p align="center">
  <a href="https://github.com/mr-finster/vf2-recomp/releases/latest"><img alt="Latest release" src="https://img.shields.io/github/v/release/mr-finster/vf2-recomp?label=release"></a>
  <a href="LICENSE"><img alt="Licence: MIT" src="https://img.shields.io/badge/licence-MIT-blue"></a>
  <img alt="Platforms" src="https://img.shields.io/badge/platforms-macOS%20%7C%20Windows%20%7C%20Linux%20%2F%20Steam%20Deck-informational">
  <img alt="Android" src="https://img.shields.io/badge/Android-arm64%20%2F%20Vulkan-informational">
</p>

<p align="center">
  <a href="https://vf2-recomp.pages.dev/">Play</a> ·
  <a href="https://github.com/mr-finster/vf2-recomp/releases/latest">Download</a> ·
  <a href="https://youtu.be/mpaUiJwlYu8">Trailer</a> ·
  <a href="#status">Status</a> ·
  <a href="#controls">Controls</a> ·
  <a href="https://github.com/mr-finster/vf2-recomp/issues">Report a bug</a>
</p>

<p align="center">
  <a href="https://vf2-recomp.pages.dev/img/montage.jpg"><img src="https://vf2-recomp.pages.dev/img/montage.jpg" alt="Virtua Fighter 2 Recomp at 3x: fights on five stages and the widescreen player select" width="800"></a>
</p>

<p align="center">
  <a href="https://youtu.be/mpaUiJwlYu8"><img src="https://img.youtube.com/vi/mpaUiJwlYu8/maxresdefault.jpg" alt="Watch the trailer" width="560"></a>
  <br><sub>Trailer (click to watch on YouTube)</sub>
</p>

> [!IMPORTANT]
> No game files are included: no ROMs and nothing made from them. You need your own dump of the game as MAME's
> `vf2` set (`vf2.zip`, Version 2.1). It is read on your computer, and the game's program is recompiled there each
> time it starts. An unofficial fan project, not made, approved or supported by SEGA, and not affiliated with it.

> [!NOTE]
> **This is a fork** of [mr-finster/vf2-recomp](https://github.com/mr-finster/vf2-recomp) that adds an
> **Android (arm64) port**, built and measured on a Xiaomi 14T Pro (MediaTek **Dimensity 9300+**, Arm
> **Immortalis-G720 MC12**). The game is upstream's; what this fork adds for a phone, the settings it
> gained, and what the phone's measurements say are under [Android](#android-this-fork), with the GPU
> notes in [docs/ARM-IMMORTALIS-G720.md](docs/ARM-IMMORTALIS-G720.md). Upstream is the place for the
> desktop releases and for bugs that are not about Android.

## About

Virtua Fighter 2 Recomp runs the game's own program, for the board's Intel i960 CPU, recompiled to native code
(ARM64 or x86-64) from your ROM each time it starts, and checked instruction for instruction, cycle for cycle
against MAME's i960 interpreter. Only the i960 program is recompiled: the rest of the Model 2A board (the TGP
geometry processor, the geometrizer and polygon renderer, the System 24 tile layers, and the sound board's 68000
and SCSP) runs on code from MAME, inside an SDL 3 host. At 1x you get the arcade's frames and sound as MAME makes
them (pixel for pixel on Apple silicon; on x86-64 a few texels a frame round differently, the renderer's
multiply-adds not fused); above it, render scales up to 6x, widescreen and ultrawide, and smooth motion at your
screen's refresh rate.

## Status

| Area | State |
| --- | --- |
| The i960 program, recompiled to native code | Working |
| TGP, geometrizer and renderer, tile layers, sound board | Working: they run on MAME's code |
| Render scale up to 6x, widescreen and ultrawide, smooth motion | Working |
| Menu, keyboard and gamepads, two players, the arcade's settings | Working |
| macOS, Windows, Linux / Steam Deck | Working |
| Android (arm64, Vulkan through SDL 3) | Working on the Xiaomi 14T Pro (Dimensity 9300+ / Immortalis-G720): menu, touch pad, Back, immersive landscape, sound, the GPU's 3D exact against the CPU's. The GPU's own 3D runs under the game's frame rate there, so 2x at full speed is the CPU 3D's — see [Android](#android-this-fork) |
| The 3D drawn on the GPU (Display > 3D renderer: Auto, GPU or CPU) | Working on macOS (Metal), Windows (Direct3D 12 or Vulkan), Linux / Steam Deck (Vulkan) and Android (Vulkan, see [Android](#android-this-fork)): the same picture as the CPU's, checked when the game starts; Auto uses the GPU unless it's clearly slower, and any difference or error falls back to the CPU |
| Stereo reverb (the sound board's DSP) | Working: wide stereo, as on the cabinet (fixed in v0.1.2) |
| Portable mode, your own ROM folder | Working |
| Secret characters (Menu > Cheats): Dural, MUE and KOJAC | Working |
| Online play | Not yet |

## Secret characters

Menu > Cheats > **Secret character** puts a fighter the arcade never let you pick in Akira's spot on the player
select, for either player: **Dural**, the final boss, or **MUE** and **KOJAC**, two unused fighters still in the
game's code. Changing it restarts the game.

**How it works.** VF2's fighters are numbered in its code: Akira 0 to Shun 8, Dural 9, Lion 10, and, after him,
MUE 11 and KOJAC 12, which the arcade's select never offers. The player select turns a slot into a fighter from a
table (Akira's slot is fighter 0); the reads that set the fighter now take Akira's slot from a copy of that table
with the secret character in it, while the reads that draw the name and profile keep the original. So the game
loads the secret character exactly as it loads anyone else: their own model, moves, voice and health bar name,
for either player, in any mode.

<p align="center">
  <img src="https://github.com/user-attachments/assets/8ce2e641-575f-49da-bbfd-f3351824860f" alt="The secret characters in action at 2x: Dural, MUE and KOJAC on five stages, and Dural's portrait on the player select" width="800">
</p>
<p align="center">
  <img src="https://github.com/user-attachments/assets/a78b7e81-5c26-47ff-8c63-8485b6c6b77b" alt="The cast's select portraits, and Dural, MUE and KOJAC below" width="800">
</p>

| Character | Select screen | Health bar | Notes |
| --- | --- | --- | --- |
| Dural | Her own portrait (the game's own); Akira's name and profile | DURAL | Complete: the final boss's model and moves, the chrome body, a darker one on player 2's side |
| MUE | Lion's portrait; Akira's name and profile | AKIRA | Lion's body in a black vest, with Akira's moves and voice but Lion's stance and walk |
| KOJAC | Lion's portrait; Akira's name and profile | JACKY | Lion's body in a white vest, with Akira's moves and voice but Lion's stance and walk |

They take Akira's place, so Akira can't be picked while one is on; the computer's own fighters, Dural at the end of
arcade mode included, are unchanged.

### Who they are

- **Dural**: the final boss of Virtua Fighter and Virtua Fighter 2, a silver cyborg who fights with a mix of the
  cast's moves. The arcade never let you choose her outright; the select's table even has an eleventh, hidden slot
  for her, and the game carries a select portrait for her too.
- **MUE and KOJAC**: two fighters numbered after Lion, named MUE and KOJAC in the game's fighter lists (KOJACKY in
  its test menu). [The Cutting Room Floor](https://tcrf.net/Virtua_Fighter_2_(Arcade)) found them in the game's
  debug mode: Lion's first costume with a different vest (black for MUE, white for KOJAC), Akira's moves and voice,
  and Lion's idle and walking animations. They may have been debug test characters, or placeholders for fighters
  that were planned and never made.

## Play

From the [latest release](https://github.com/mr-finster/vf2-recomp/releases/latest), or the
[site](https://vf2-recomp.pages.dev/):

| System | Download |
| --- | --- |
| macOS 11+ (Apple silicon and Intel) | `VirtuaFighter2Recomp-mac.zip` |
| Windows 10 or 11 (64-bit) | `VirtuaFighter2Recomp-windows.zip` |
| Linux / Steam Deck (x86-64) | `VirtuaFighter2Recomp-deck.tar.gz` |

**Mac and Windows**

1. Unzip it (Mac: move **Virtua Fighter 2 Recomp** to Applications; Windows: the VirtuaFighter2 folder anywhere, not inside Program Files).
2. Open it (Windows: `VirtuaFighter2.exe`).
3. Choose your `vf2.zip` when it asks. A copy is kept, so you only do this once.

**Steam Deck**

1. Copy the download to the Deck (e.g. Downloads) and unpack it there: in Desktop Mode, right-click it in the file
   manager > Extract > Extract archive here. (Unpacking on Windows, or copying the folder through a USB stick, can
   lose the files' permissions, and then nothing launches.)
2. Put your `vf2.zip` in the `VirtuaFighter2/roms` folder.
3. Desktop Mode: Steam > Games > Add a Non-Steam Game to My Library > Browse, set the file type filter to "All
   files", pick the `vf2` file in the VirtuaFighter2 folder, Add Selected Programs. In its Properties, set Start In
   to the VirtuaFighter2 folder (e.g. `/home/deck/Downloads/VirtuaFighter2/`), and under Compatibility leave "Force
   the use of a specific Steam Play compatibility tool" off: it's a native Linux game, and Proton stops it launching.
4. Back in Game Mode, launch it from your Library with the default Gamepad layout.

The Windows download also runs on the Deck under Proton (add `VirtuaFighter2.exe` instead, with Proton forced on),
but the native Linux build above is the one to use.

On another x86-64 Linux PC: unpack it anywhere, put `vf2.zip` in its `roms` folder and run `VirtuaFighter2.sh`.

**Portable mode:** put a file named `portable.txt` next to the program (Windows: beside `VirtuaFighter2.exe`; Mac:
beside the app, in the same folder) and the game keeps everything in that folder: settings, saves, screenshots,
`log.txt` and the copy of your `vf2.zip` (in its `roms` folder), with nothing in your user data folder (`%APPDATA%`,
`~/Library/Application Support`). The Linux / Steam Deck build keeps everything in its folder already.

**Your own ROM folder** (e.g. Batocera's `/userdata/roms/ports/vf2`, or any folder an update won't touch): start the
game with `--roms <folder>` (or `--roms=<folder>`), set the environment variable `VF2_ROMS`, or add
`roms_folder = <folder>` to `settings.ini`; the command line wins, then `VF2_ROMS`, then `settings.ini`. `~` is your home
folder, and a relative path in `settings.ini` is from its folder. The game looks there first for `vf2.zip` and
only reads it (no copy is kept anywhere else); `log.txt` names the folder it used. If the zip isn't there, the log
says where it looked and the game carries on as without the option; a zip you then choose with the file picker is
copied into your ROM folder, so the next start finds it there. Examples: Windows `VirtuaFighter2.exe --roms "D:\ROMs\vf2"`;
Mac `open "Virtua Fighter 2 Recomp.app" --args --roms ~/ROMs/vf2`; Linux / Batocera
`./VirtuaFighter2.sh --roms /userdata/roms/ports/vf2`.

> [!NOTE]
> The Mac and Windows builds aren't signed. On macOS, open it once, close the message, then System Settings >
> Privacy & Security > Open Anyway. On Windows, if SmartScreen warns, choose More info > Run anyway.

## Android (this fork)

The same game on an arm64 phone: the i960 program recompiled to native code as the game starts, MAME's
Model 2A board around it, and SDL 3 in front — Vulkan through its GPU renderer, with the CPU's 3D as the
fallback. Built and measured on a **Xiaomi 14T Pro** (MediaTek **Dimensity 9300+**, Arm
**Immortalis-G720 MC12**, Android 16); nothing in it is specific to that phone. There are no Android
packages to download: the APK is built from source, below. The player-facing version of the same steps is
[android/README.md](android/README.md), and the port's plan with every phone run's numbers is
[vf2.md](vf2.md).

### What this fork adds

| Piece | What it is |
| --- | --- |
| The Android port | `android/`: a Gradle 8.12 + AGP 8.7.3 project driving the NDK's CMake (NDK 30.0.16248370, SDK CMake 3.22.1, `arm64-v8a`, minSdk 26, targetSdk 34), with SDL 3 and FreeType from `third_party/` |
| The host on Android | SDL's entry point and lifecycle (the sound stops in the background), the APK's `assets/` copied out once into the app's own folder, the `content://` ROM picker, immersive fullscreen and a landscape window, on-screen controls, Android's Back as the menu key, AAudio's *game* role, the launcher icon and git-based versioning |
| Mali / Vulkan | the GPU device is created with `SDL_CreateGPUDeviceWithProperties` and clip distance, depth clamping, indirect-first-instance and anisotropy **off** — the same insurance SDL's own GPU renderer takes, without which SDL "excludes every Mali GPU" ([SDL #12652](https://github.com/libsdl-org/SDL/issues/12652); SDL's renderer does it, our fallback device path did not) |
| Settings a phone needs | `Display > Vsync` (`0` asks for mailbox first — what lifted 2x here from 33 to 38-48 pictures/s), `Display > GPU frames in flight`, `Display > Performance hints` (Android 13 and up: the platform's own way of keeping the frame's threads and the GPU at their clocks), `Display > On-screen controls` |
| What `log.txt` says | the present mode the window really gave (`mailbox` / `immediate` / `vsync`) and the Vulkan driver's name, version and info; the cores' and the **GPU's** clocks every 10 s (`the GPU at: 0.39 of 1.30 GHz`, read from `/sys/class/devfreq`); a `pace:` line every 10 s with how many pictures a second and where a picture's time went |
| ROM-free checks | `make gpu3dbench` (the GPU's 3D against the CPU's, on the bench scene), `make touchcheck` (the on-screen pad, with SDL's own finger events), `make audiocheck` (the host's audio path and a test tone) — all three run on a PC, with no ROM |

The phone's GPU needs no tricks of its own beyond that insurance: what this generation of Arm GPUs wants,
what the numbers ruled out, and the sources are in [docs/ARM-IMMORTALIS-G720.md](docs/ARM-IMMORTALIS-G720.md).

### What the phone's measurements say

- **1x on the GPU**: 51-55 pictures/s against the game's own 57.5, `a picture 9.3-10.5 ms on the GPU`.
- **2x on the GPU with smooth motion off**: ~51-56. **3x**: 41-47. **4x**: ~32. **5x/6x**: 21/19, where a
  15-18 ms texture upload appears.
- **2x at the game's full rate is the CPU 3D's** (57-62 pictures/s with smooth motion on) on this phone,
  not the GPU's — the opposite of what the GPU renderer is for.
- The picture rate hardly moves with the render scale (1x ≈ 3x), so the **~19 ms a picture is our own
  loop's overhead, not fill**. On a CSF Mali like this one a compute pass and the tiler's binning cannot
  overlap, and our GPU 3D is compute — a sourced explanation, not a fix.
- `frames in flight` 1 ≈ 2 ≈ 3, so the swapchain was not the limiter. The two levers that measurably help
  are **vsync off** and **smooth motion off**.
- The governor still walks 2x down to 1x while 53-55 of 57.5 pictures are already on show: it costs
  sharpness and buys nothing. Known, and not yet fixed.
- **Stable 60 (the game's 57.5) at 2x-4x on the GPU is not reachable on this phone.** What is left is
  host-side: the per-picture overhead, the governor's stopping rule, and the Auto 3D verdict.

### What it needs

| | |
| --- | --- |
| Phone | arm64, Android 8.0 (API 26) or newer. A Vulkan driver is expected but not required — the 3D can run on the CPU, and the game falls back to it by itself. No permission is asked for at all |
| Build machine | Linux, macOS or Windows with a **JDK 17** or newer (the module compiles for Java 17) |
| Android SDK | `platform-tools`, `platforms;android-34`, `build-tools;34.0.0`, **CMake 3.22.1** and **NDK 30.0.16248370** — the last two are named in `android/app/build.gradle` |
| The repository | SDL 3 and FreeType are vendored in `third_party/` and are built by the same command: nothing to fetch, and no submodules |
| ROM | your own `vf2.zip` (MAME's `vf2` set, Version 2.1) |

### Build it, step by step

1. Install a JDK 17 and the Android SDK's command-line tools, then the pieces the build names (the
   `sdkmanager` is in `cmdline-tools/latest/bin`):

   ```
   sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0" "cmake;3.22.1" "ndk;30.0.16248370"
   ```

2. Point the build at the SDK, either with the environment:

   ```
   export ANDROID_HOME=~/Android/Sdk        # or ANDROID_SDK_ROOT
   ```

   or with a file `android/local.properties` holding `sdk.dir=/home/you/Android/Sdk`.

3. Build the debug APK. The first build also compiles SDL 3 and FreeType and takes a couple of minutes;
   later ones are seconds.

   ```
   cd android
   ./gradlew :app:assembleDebug
   ```

4. The APK is `android/app/build/outputs/apk/debug/app-debug.apk`, signed with the debug key (it installs
   by sideloading, not from the Play Store). Its `versionName` is the git tag — `v0.1.9` — and its
   `versionCode` is that name's major.minor.patch (`109`), so each tagged build is an upgrade.

### Run it, step by step

1. With the phone attached (Developer options > USB debugging on):

   ```
   adb install -r android/app/build/outputs/apk/debug/app-debug.apk
   ```

   or copy the APK over and open it on the phone.

2. Start the game. It asks for your `vf2.zip` with Android's own file picker, checks it and keeps a copy,
   so you do this once. Or drop the zip in and it is played where it lies:

   ```
   adb push vf2.zip /sdcard/Android/data/com.vf2recomp.vf2/files/roms/
   ```

3. Play. The touch pad is the d-pad at the bottom left and guard/punch/kick, Start and Menu at the bottom
   right; **Back** opens the game's menu (or goes a page back inside it); a gamepad in Xbox layout, or a
   keyboard, works as on the desktop (arrows + <kbd>Z</kbd> <kbd>X</kbd> <kbd>C</kbd> for player 1,
   <kbd>I</kbd> <kbd>J</kbd> <kbd>K</kbd> <kbd>L</kbd> for player 2, <kbd>Esc</kbd> for the menu), and the
   menu's Controls page rebinds all of it. The game starts immersive and in landscape, the picture follows
   the panel's refresh rate, and everything it keeps (`settings.ini`, the saves, `log.txt`) is in the app's
   own folder.

4. When something is off, the game's own log says what it did:

   ```
   adb shell run-as com.vf2recomp.vf2 cat files/log.txt > log.txt
   ```

   Its `render state`, `pace:` and `performance hints` lines are the ones to read; on a phone that feels
   slow, `Display > Performance hints` is the one setting worth trying both ways, and `Display > GPU
   frames in flight` and `Display > Vsync` the other two.

## Controls

Change them in the menu (<kbd>Esc</kbd>, or View / Select on a pad).

| Key | Does |
| --- | --- |
| <kbd>↑</kbd> <kbd>↓</kbd> <kbd>←</kbd> <kbd>→</kbd> | Player 1: move |
| <kbd>Z</kbd> <kbd>X</kbd> <kbd>C</kbd> | Player 1: guard, punch, kick |
| <kbd>1</kbd> | Player 1: start |
| <kbd>I</kbd> <kbd>J</kbd> <kbd>K</kbd> <kbd>L</kbd> | Player 2: move |
| <kbd>B</kbd> <kbd>N</kbd> <kbd>M</kbd> | Player 2: guard, punch, kick |
| <kbd>2</kbd> | Player 2: start |
| <kbd>Esc</kbd> | Menu: display (render scale, picture shape, antialiasing, 3D renderer...), sound, game, arcade settings, controls, cheats, quit |
| <kbd>P</kbd> | Pause |
| <kbd>F3</kbd> | Reset |
| <kbd>F5</kbd> / <kbd>F6</kbd> | Render scale / picture shape |
| <kbd>F9</kbd> | Frame timings |
| <kbd>F11</kbd> | Fullscreen |
| <kbd>F12</kbd> | Screenshot |

Gamepad (Xbox layout): d-pad or stick to move, X guard, A punch, B kick, Y guard + punch (throws), RB punch + kick,
LB guard + kick, RT all three, Start to start. View (tap), L3 + R3 or the touchpad click opens the menu (Menu can be
moved in Controls); View + LB / RB change the render scale / picture shape. In Arcade mode, set Coin in Controls (View
works) for a coin button.

## How it works

The recompiler (`src/rec`) finds the i960 program's code with an address map (`src/rec/vf2_map.cpp`: where the
functions start and where the indirect jumps lead, no bytes from the ROM), made from coverage runs, and turns it
into native code with SLJIT as the game starts. Around it, code from MAME runs the rest of the Model 2A board: the
TGP, the geometrizer and polygon renderer, the tile chip and the sound board's 68000 and SCSP. The SDL 3 host
composites the picture, scales and shapes it, and fills in the frames between the arcade's for smooth motion.

The 3D can also be drawn on the GPU (`src/gpu`): Metal on a Mac, and SDL_GPU compute (Direct3D 12 or Vulkan) on
Windows and Linux, where SDL's GPU renderer then draws the whole window. The GPU doesn't use the usual depth buffer:
the polygons are drawn in the game's own order, the order standing in for depth, by a compute rasteriser that
follows the CPU renderer's arithmetic step for step (integer division, no fused multiply-adds, float-to-integer
conversion as on x86), so its picture is the CPU's pixel for pixel (`gpu3dcmp` compares the two). When the game
starts it draws a test scene both ways and compares them, then checks its first pictures of the game itself; any
difference, error or missing driver and it uses the CPU for that run. Auto (the default) then picks the GPU unless
it's clearly slower than the CPU on that machine.

## Repository layout

| Path | Contents |
| --- | --- |
| `src/core` | the Fujitsu MB86233 (TGP) core, from MAME, and the shim MAME's CPU cores compile against |
| `src/rec` | the i960 recompiler: code discovery, native code (SLJIT) at start-up, the C++ build's, the runtime |
| `src/machine` | Model 2A: memory map, interrupts and timers, the TGP glue, geometrizer and renderer, the tile chip, the sound board, smooth motion |
| `src/host` | the SDL 3 host: window, input, pacing, scene sync, the GPU compositing (`d3d11_map.cpp`: Windows' texture uploads), the RmlUi menu |
| `src/gpu` | the GPU renderer for the 3D: Metal (`gpu3d_metal.mm`), SDL_GPU (`gpu3d_sdlgpu.cpp`), its HLSL compute shaders (`gpu3d.hlsl`) and their compiled SPIR-V / DXIL (`gpu3d_shaders.h`) |
| `tools/gpu/shaders.sh`, `tools/gpu3dcmp.cpp` | compiles `gpu3d.hlsl` into `gpu3d_shaders.h` with DXC; compares the GPU's 3D with the CPU's |
| `assets/` | the app icon, the menu (`ui/`) and its font, the player select's painted sides |
| `android/` | the Android port: the Gradle + CMake project (manifest, Java side, icons) and `android/README.md` for a player |
| `third_party/` | MAME's i960 and 68000 cores and more, SLJIT, RmlUi, stb_image, and (for the phone) SDL 3 and FreeType |
| `docs/ARM-IMMORTALIS-G720.md` | what the phone's GPU wants and what it does not: the notes behind the Android settings and the new log lines |
| `vf2.md` | the port's plan and log: the modules, each phone run's numbers, what is done and what is still open |
| `analiza.md` | the analysis of two third-party suggestion notes: what was taken, what was left out, and why |
| `tools/recomp` | the ahead-of-time recompiler (i960 to C++), for development |
| `tools/*.cpp` | the headless runner (`vf2run`), disassembler, ROM set check, lockstep checker (`vf2lock`), and the ROM-free checks (`gpu3dbench`, `touchcheck`, `audiocheck`) |
| `tools/*.py`, `tools/scripts` | comparisons with MAME's pictures, sound and traces; input scripts |
| `tools/mac`, `tools/win`, `tools/deck`, `tools/release` | the release packages and their licence notices |

## Run it yourself

### What you need

- A C++20 compiler (clang or g++), GNU make and **zlib**.
- **SDL 3** and **FreeType**, found through `pkg-config` (macOS: `brew install sdl3 freetype`; Arch:
  `sdl3` and `freetype2`). They can also be pointed at by hand through `SDL_CFLAGS`/`SDL_LIBS` and
  `FT_CFLAGS`/`FT_LIBS`.
- For the phone instead of the desktop: a JDK 17 and the Android SDK and NDK — see
  [Android](#android-this-fork) above.

### Build and run (desktop)

```
make                    # build/bin/vf2, and the headless tools
build/bin/vf2 path/to/vf2.zip
```

The three checks that need no ROM run here as well: `make gpu3dbench` (the GPU's 3D against the CPU's, on
the bench scene), `make touchcheck` (the on-screen pad, with SDL's own finger events) and `make
audiocheck` (the host's audio path and a test tone).

The release packages: `tools/mac/package.sh` (Mac), `tools/win/build.sh` (Windows, cross-compiled with MinGW-w64),
`tools/deck/build.sh` (Linux / Steam Deck: x86-64, in Valve's Steam Runtime "sniper" SDK container).
`tools/release/licenses.sh` gathers the third-party notices they carry.

`build/bin/vf2run` runs the board headless (pictures, RAM dumps, raw sound, an input script such as
`tools/scripts/start_game.txt`); `tools/compare.py` and `tools/sound_compare.py` compare its output with MAME's.
The address map is made from coverage runs (`vf2run --cov`, then `make map`). `make rec` also recompiles the
program to C++ (into `build/`, from your `roms/vf2.zip`: never committed) for `build/bin/vf2lock`, which runs the
interpreter and the recompiled code side by side and compares the CPU's whole state, the clocks, the RAMs and the
pictures as they go. `vf2 --interp` and `vf2run --interp` run the interpreter.

`tools/dump_textures.sh path/to/vf2.zip` saves the game's textures as PNGs (into `build/textures`): coloured as in
the game, the grey levels they're made of, and a list of where each one sits. It runs attract mode headless; for the
rest, play with `VF2_DUMP_TEXTURES=folder` set and each texture is saved the first time it's drawn.

## How it's made

Made with the help of AI coding tools. Every build is checked against the arcade: the recompiled i960 program runs
side by side with MAME's interpreter, and the pictures, RAM and sound are compared frame by frame with MAME's.

The whole story, with how the AI-assisted work was organised and checked: [docs/HOW-IT-WAS-MADE.md](docs/HOW-IT-WAS-MADE.md).

## Credits

Built on MAME's work on the Model 2: the i960 core and disassembler (Farfetch'd, R. Belmont), the 68000 core and
the TGP (Olivier Galibert), the Model 2 driver and video (R. Belmont, Olivier Galibert, ElSemi, Angelo Salese,
Matthew Daniels), the System 24 tile chip (Olivier Galibert), the SCSP and its DSP (ElSemi, R. Belmont), the
polygon scan converter (Ville Linde, Aaron Giles) and more (`third_party/mame/README.md` lists every file and its
authors), all BSD-3-Clause. Also RmlUi (MIT), stb_image (Sean Barrett, public domain / MIT), SDL 3 (zlib),
FreeType (FreeType License), zlib, and the Barlow Condensed font (SIL Open Font License).

Thanks to everyone who tested and reported: [duartepedro03](https://github.com/duartepedro03) and
[MagitekArmor](https://github.com/MagitekArmor) (performance and the GPU renderer on Windows),
[alex2005jp](https://github.com/alex2005jp) (older graphics cards), [darethejackal](https://github.com/darethejackal)
(Steam Deck), [vmicka72](https://github.com/vmicka72), [svera](https://github.com/svera) and
[terryarcade](https://github.com/terryarcade).

Virtua Fighter 2 is SEGA's. This project is not affiliated with or endorsed by SEGA.

## Licence

MIT for this project's own code: see [LICENSE](LICENSE). Code from MAME keeps its BSD-3-Clause licence and notices;
the rest of `third_party/` keeps its own, and the Barlow Condensed font the SIL Open Font License. The art, the
player select's painted sides (`assets/backgrounds/select.png`) and the app icon (`assets/icon/`, drawn by
`tools/release/make_icon.py`), is by Mr Finster under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

## Support

If you enjoy it, you can [buy me a coffee](https://buymeacoffee.com/mrfinster).
