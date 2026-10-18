<p align="center"><img src="assets/icon/icon.png" alt="Virtua Fighter 2 Recomp" width="160"></p>

<h1 align="center">Virtua Fighter 2 Recomp — Android port</h1>

<p align="center"><b>Virtua Fighter 2 (SEGA, arcade, Model 2, 1994) on an arm64 Android phone: the game's own i960
program recompiled to native code as the game starts, MAME's Model 2A board around it, and SDL 3 in front —
Vulkan through its GPU renderer, with the CPU's 3D as the fallback.</b></p>

<p align="center">
  <img alt="Android" src="https://img.shields.io/badge/Android-arm64%20%2F%20Vulkan-informational">
  <a href="LICENSE"><img alt="Licence: MIT" src="https://img.shields.io/badge/licence-MIT-blue"></a>
</p>

> [!IMPORTANT]
> No game files are included: no ROMs and nothing made from them. You need your own dump of the game as MAME's
> `vf2` set (`vf2.zip`, Version 2.1). It is read on the phone, and the game's program is recompiled there each
> time it starts. An unofficial fan project, not made, approved or supported by SEGA, and not affiliated with it.

This is a **fork** of [mr-finster/vf2-recomp](https://github.com/mr-finster/vf2-recomp) that adds an
**Android (arm64) port**, built and measured on a Xiaomi 14T Pro (MediaTek **Dimensity 9300+**, Arm
**Immortalis-G720 MC12**, Android 16); nothing in it is specific to that phone. The game, the i960
recompiler, MAME's Model 2A board, the renderers and the desktop builds are upstream's — this README covers
the phone only. For everything else, from the desktop downloads to the controls and the credits, see the
[upstream README](https://github.com/mr-finster/vf2-recomp#readme).

The port in more detail: [android/README.md](android/README.md) (the same instructions as below, for a
player: build, install, ROM, controls), [vf2.md](vf2.md) (the port's plan and every phone run's numbers,
module by module), [docs/ARM-IMMORTALIS-G720.md](docs/ARM-IMMORTALIS-G720.md) (what the phone's GPU wants
and what it does not, with sources), and [analiza.md](analiza.md) (an analysis of two third-party
suggestion notes: what was taken and what was left out).

## Changes from upstream

### What the phone adds

| Piece | What it is |
| --- | --- |
| The Android port | `android/`: a Gradle 8.12 + AGP 8.7.3 project driving the NDK's CMake (NDK 30.0.16248370, SDK CMake 3.22.1, `arm64-v8a`, minSdk 26, targetSdk 34), with SDL 3 and FreeType from `third_party/` |
| The host on Android | SDL's entry point and lifecycle (the sound stops in the background), the APK's `assets/` copied out once into the app's own folder, the `content://` ROM picker, immersive fullscreen and a landscape window, on-screen controls, Android's Back as the menu key, AAudio's *game* role, the launcher icon and git-based versioning |
| Mali / Vulkan | the GPU device is created with `SDL_CreateGPUDeviceWithProperties` and clip distance, depth clamping, indirect-first-instance and anisotropy **off** — the same insurance SDL's own GPU renderer takes, without which SDL "excludes every Mali GPU" ([SDL #12652](https://github.com/libsdl-org/SDL/issues/12652); SDL's renderer does it, our fallback device path did not) |
| Settings a phone needs | `Display > Vsync` (`0` asks for mailbox first — what lifted 2x here from 33 to 38-48 pictures/s), `Display > GPU frames in flight`, `Display > Performance hints` (Android 13 and up: the platform's own way of keeping the frame's threads and the GPU at their clocks), `Display > On-screen controls` |
| What `log.txt` says | the present mode the window really gave (`mailbox` / `immediate` / `vsync`) and the Vulkan driver's name, version and info; the cores' and the **GPU's** clocks every 10 s (`the GPU at: 0.39 of 1.30 GHz`, read from `/sys/class/devfreq`); a `pace:` line every 10 s with how many pictures a second and where a picture's time went |
| ROM-free checks | `make gpu3dbench` (the GPU's 3D against the CPU's, on the bench scene), `make touchcheck` (the on-screen pad, with SDL's own finger events), `make audiocheck` (the host's audio path and a test tone) — all three run on a PC, with no ROM |

### Files of upstream's that this fork touches

| File | Change |
| --- | --- |
| `src/host/main.cpp` (changed) | the SDL entry point on Android, the APK assets copy (`asset_join()` — its double separator was why the menu never loaded), background/foreground lifecycle, the `content://` ROM picker, the shared-storage ROM folder, immersive fullscreen by default, no `SDL_WINDOW_RESIZABLE` on Android, the on-screen pad, Back as the menu key, the AAudio stream role, `vsync` and `frames_in_flight`, the present mode and driver log line, the GPU clock line, and the performance-hint calls |
| `src/host/sysinfo.{h,cpp}` (changed) | `cpu_frequencies()` (Module 11) and `gpu_frequencies()` (the GPU's devfreq clock), plus `perf_hints_start/report/stop/available` — Android's performance hints, reached through `dlopen("libandroid.so")` because minSdk is 26 and the calls are Android 13's |
| `src/machine/parallel.h` (changed) | `row_pool::worker_tids()`: the thread ids a hint session is asked over (Android only) |
| `src/gpu/gpu3d_sdlgpu.cpp` (changed) | the own-device path (the tools, and the day the renderer isn't `gpu`) creates the device with `SDL_CreateGPUDeviceWithProperties` and the four Mali-hostile features off, as SDL's own renderer does |
| `tools/release/make_icon.py` (changed) | a new `--android <variant>` mode, writing the launcher icons and the adaptive icon's two layers; the desktop output is unchanged (byte-identical) |
| `Makefile` (changed) | the three ROM-free checks' targets |
| `android/` (new) | the whole Gradle + CMake project: `build.gradle`, `app/CMakeLists.txt` (mirrors the Makefile's sources; `-O2` for the Debug type, without which the game ran at ~8 fps), the manifest, SDL's Java side, the icons, and `android/README.md` |
| `src/host/touch_controls.h` (new) | the on-screen pad: a d-pad, guard/punch/kick, Start and Menu, drawn in the renderer's logical coordinates and read per finger |
| `tools/gpu3dbench.cpp`, `tools/touchcheck.cpp`, `tools/audiocheck.cpp` (new) | the three checks above |
| `third_party/sdl3`, `third_party/freetype` (vendored) | the phone build uses these copies (`pkg-config`'s SDL 3 and FreeType are still what the desktop build uses) |
| `docs/ARM-IMMORTALIS-G720.md`, `vf2.md`, `analiza.md` (new) | the GPU notes, the port's plan and log, and the analysis of the two suggestion notes |

### What deliberately does not differ

- **The desktop builds play as upstream's.** Every Android branch is behind `#ifdef __ANDROID__`, and the two
  new display settings fall back to SDL's own values off Android (`frames in flight 1`, performance hints
  off), the GPU clock line is simply absent where there is no `devfreq`, and nothing else changes.
- **`third_party/sdl3` is verbatim.** SDL's own `AGENTS.md` asks for no AI-generated contributions, so every
  change lives in our `src/` or in `android/`.
- **The recompiler, the address map, the machine and the renderers are untouched** — including the CPU/GPU 3D
  paths and their pixel-for-pixel check.

## What the phone's measurements say

- **1x on the GPU**: 51-55 pictures/s against the game's own 57.5, `a picture 9.3-10.5 ms on the GPU`.
- **2x on the GPU with smooth motion off**: ~51-56. **3x**: 41-47. **4x**: ~32. **5x/6x**: 21/19, where a
  15-18 ms texture upload appears.
- **2x at the game's full rate is the CPU 3D's** (57-62 pictures/s with smooth motion on) on this phone, not
  the GPU's — the opposite of what the GPU renderer is for.
- The picture rate hardly moves with the render scale (1x ≈ 3x), so the **~19 ms a picture is our own loop's
  overhead, not fill**. On a CSF Mali like this one a compute pass and the tiler's binning cannot overlap, and
  our GPU 3D is compute — a sourced explanation, not a fix.
- `frames in flight` 1 ≈ 2 ≈ 3, so the swapchain was not the limiter. The two levers that measurably help are
  **vsync off** and **smooth motion off**.
- The governor still walks 2x down to 1x while 53-55 of 57.5 pictures are already on show: it costs sharpness
  and buys nothing. Known, and not yet fixed.
- **Stable 60 (the game's 57.5) at 2x-4x on the GPU is not reachable on this phone.** What is left is
  host-side: the per-picture overhead, the governor's stopping rule, and the Auto 3D verdict.

Between them these are the reason the Display rows and the log lines above exist: everything measured here
came out of a phone run's `log.txt`, and the GPU note next to it carries its sources.

## Still owed

The phone has run it: `i960 native` (SLJIT's executable memory works), the GPU's 3D exact against the CPU's
on the Immortalis-G720, the menu, the touches, Back, the immersive landscape window and the sound all work,
and the board holds the game's 57.5 frames a second. What is left: the A/B run of `Display > Performance
hints` and of the new `the GPU at:` line; the governor's 2x → 1x step; the Auto 3D verdict, which still rests
on the start-up test scene; an on-screen pad for player 2; and the 2D renderer's `Surface.setFrameRate` idea
(forcing the surface's refresh rate) is parked until the phone's own rate proves to be the problem.

## What it needs

| | |
| --- | --- |
| Phone | arm64, Android 8.0 (API 26) or newer. A Vulkan driver is expected but not required — the 3D can run on the CPU, and the game falls back to it by itself. No permission is asked for at all |
| Build machine | Linux, macOS or Windows with a **JDK 17** or newer (the module compiles for Java 17) |
| Android SDK | `platform-tools`, `platforms;android-34`, `build-tools;34.0.0`, **CMake 3.22.1** and **NDK 30.0.16248370** — the last two are named in `android/app/build.gradle` |
| The repository | SDL 3 and FreeType are vendored in `third_party/` and are built by the same command: nothing to fetch, and no submodules |
| ROM | your own `vf2.zip` (MAME's `vf2` set, Version 2.1) |

## Build it, step by step

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

3. Build the debug APK. The first build also compiles SDL 3 and FreeType and takes a couple of minutes; later
   ones are seconds.

   ```
   cd android
   ./gradlew :app:assembleDebug
   ```

4. The APK is `android/app/build/outputs/apk/debug/app-debug.apk`, signed with the debug key (it installs by
   sideloading, not from the Play Store). Its `versionName` is the git tag — `v0.1.9` — and its `versionCode`
   is that name's major.minor.patch (`109`), so each tagged build is an upgrade.

## Run it, step by step

1. With the phone attached (Developer options > USB debugging on):

   ```
   adb install -r android/app/build/outputs/apk/debug/app-debug.apk
   ```

   or copy the APK over and open it on the phone.

2. Start the game. It asks for your `vf2.zip` with Android's own file picker, checks it and keeps a copy, so
   you do this once. Or drop the zip in and it is played where it lies:

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

   Its `render state`, `pace:` and `performance hints` lines are the ones to read; on a phone that feels slow,
   `Display > Performance hints` is the one setting worth trying both ways, and `Display > GPU frames in
   flight` and `Display > Vsync` the other two.

## Thanks

Hi everyone!

Huge thanks to [mr-finster](https://github.com/mr-finster/vf2-recomp) for the amazing work on the original
recompilation!

Based on that project, I managed to set up this fork and release my very first Android `.apk` build, with the
help of AI. It's still a work in progress — performance isn't fully optimal yet (especially on Mali GPUs), but
it's completely playable: you can navigate the menus, adjust settings and play matches without issues.

If you have any ideas, performance optimization tips, or want to contribute, feel free to join in, open an
[issue](https://github.com/soulafein83/vf2-recomp/issues) or submit a
[pull request](https://github.com/soulafein83/vf2-recomp/pulls). Any feedback or contributions are very
welcome!

## Licence

MIT, as upstream: see [LICENSE](LICENSE). Code from MAME keeps its BSD-3-Clause licence and notices, and the
rest of `third_party/` keeps its own — the full list and the credits are upstream's
([README](https://github.com/mr-finster/vf2-recomp#credits)).
