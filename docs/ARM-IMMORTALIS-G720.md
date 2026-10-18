# Arm Immortalis-G720 MC12 (Dimensity 9300+) — Vulkan notes for this project

Written 2026-10-18, after the fifth phone run. Answers one question: **does this GPU need code or
tricks of its own, beyond what we already do?** Short answer: **yes, but only small ones — four
lines of insurance and two log lines.** The renderer path already carries the important
workaround, and the ceiling we measured is host-side, not driver-side.

Nothing here changes `third_party/sdl3` (SDL's own `AGENTS.md` asks for no AI-generated
contributions, and this tree is verbatim anyway). Every change proposed below is in our `src/`.

## What the GPU is

- Arm **5th-generation** (Valhall) architecture, 12 shader cores = **MC12**, in the MediaTek
  **Dimensity 9300/9300+** (Xiaomi 14T Pro). Tile-based (TBDR). New in this generation:
  **Deferred Vertex Shading (DVS)**, **FP16 transcendental instructions**, Variable Rate Shading,
  hardware ray tracing.
- Command processing is the **Command Stream Frontend (CSF)** — a firmware-driven processor split
  into **transfer / compute / geometry / fragment** streams (Valhall 3rd gen = Mali-G710 onward,
  so this GPU too).
- Our phone's log: `SDL_GPU: vulkan, Mali-G720-Immortalis MC12`, pipelines ready in 294 ms,
  the GPU's 3D exact against the CPU's.

## What we measured (fifth run, 2026-10-18)

- 1x + GPU, smooth off: 51-55 pictures/s, `a picture 9.3-10.5 ms on the GPU`, frames 57.5.
- 2x + GPU, smooth off: ~51-56; 3x: 41-47; 4x: ~32; 5x/6x: 21/19 (`upload 15-18 ms`).
- The picture rate barely moves with the scale (1x ≈ 3x) → **per-picture overhead**, not fill.
- `frames in flight` 1 ≈ 2 ≈ 3 → the swapchain was not the limiter.
- The loop is 90-96% busy throughout.

## Verified facts about our stack (our vendored SDL 3.4.18)

1. **SDL's GPU renderer already opts out of the four features that broke Android.** When it
   creates the device it sets clip distance, depth clamping, indirect-draw-first-instance and
   anisotropy to **false** (`third_party/sdl3/src/render/gpu/SDL_render_gpu.c:1742-1754`, "these
   properties allow using the renderer on more Android devices"). They default to **true**
   otherwise (`SDL_gpu_vulkan.c:12712`). This is exactly the fix for SDL issue **#12652**, where
   the reporter's own query "excludes every Mali GPU" because `shaderClipDistance` is only on
   ~65% of Android devices (SDL #10814 notes Mali's poor user-clip-plane support; #10964 added
   the requirement). **It is why our GPU renderer starts on this phone at all.**
   Our GPU 3D shares that device whenever the renderer is `gpu` (`src/host/main.cpp:1519`), so it
   inherits the opt-out for free.
2. **The one place that does not opt out is ours.** `gpu3d_create_sdlgpu()` falls back to its own
   `SDL_CreateGPUDevice(...)` when handed no device (`src/gpu/gpu3d_sdlgpu.cpp:540`) — that call
   asks for clip distance and depth clamp as **true**, so on a Mali/Android-like driver it can
   fail outright. Used today by `tools/gpu3dbench`, and it is the path that would run if the
   renderer were ever not `gpu` (the GLES2 fallback). The fix is to create it with
   `SDL_CreateGPUDeviceWithProperties` and the same four properties false, exactly as the renderer
   does. We use none of those four features.
3. **`vsync 0` means MAILBOX, and only then IMMEDIATE, and only then VSYNC**
   (`SDL_render_gpu.c:1614-1625`). Mali exposes FIFO/MAILBOX/IMMEDIATE, so on a phone
   `vsync 0` = mailbox when the window allows it — which is what lifted 2x from 33 to 38-48
   pictures/s. **We never log which mode actually won**: `SDL_WindowSupportsGPUPresentMode(dev,
   win, SDL_GPU_PRESENTMODE_MAILBOX/IMMEDIATE)` answers it in one call.
4. **The driver version is printable.** `SDL_GetGPUDeviceProperties(device)` carries
   `SDL_PROP_GPU_DEVICE_DRIVER_NAME_STRING` / `_DRIVER_VERSION_STRING` / `_DRIVER_INFO_STRING`
   (`SDL_gpu.h:2592-2595`, filled at `SDL_gpu_vulkan.c:12872`). Arm's own guidance is
   driver-release-specific (e.g. Mali Timeline debug-marker annotation needs drivers newer than
   **R51**), and Android GPU drivers change with OTAs, so a log line is worth having. (Our log
   prints the *device name* already, `main.cpp:1339`, but not the driver.)

## Architecture facts that explain our numbers

- **TBDR: bandwidth is the currency.** `loadOp`/`storeOp` decide what round-trips to memory;
  depth/stencil and MSAA colour want `DONT_CARE`; **Transaction Elimination** skips writes when a
  tile's hash is unchanged — Arm notes it pays off for **mostly static 2D content (UIs)**, not for
  a 3D game. Our menu's 3528x1536 painted background is exactly the static-UI case; the game is
  not. ([Khronos TBR guide](https://docs.vulkan.org/guide/latest/tile_based_rendering_best_practices.html),
  [Arm, Transaction elimination](https://developer.arm.com/documentation/101897/0304/Fragment-shading/Transaction-elimination))
- **The fragment (main) phase overlaps with binning and with compute — but binning and compute
  cannot overlap each other** (Arm, *Vulkan synchronization on Mali GPUs*, Feb 2026, Valhall 3rd
  gen and later). Our per-picture work is **a compute pass — the 3D rasteriser is compute shaders
  writing storage images/textures** (`src/gpu/gpu3d_sdlgpu.cpp:200-216`, `:108`, `:160`) **plus the
  renderer's render passes** (2D composition, blit, present). Those two cannot be overlapped by
  the tiler, which is a sourced candidate for the ~19 ms a picture, and for why the **CPU's** 3D
  (which issues no GPU compute) leaves the render passes free to overlap and can show more
  pictures/s in the loop.
- **Compute image stores invalidate Transaction Elimination metadata, and sampling after a GPU
  write forces a texture-cache invalidation** (same blog). A compute→sample hand-off once per
  picture therefore costs more on Mali than the same pixels written by a fragment pass.
- **Transfers are compute or fragment jobs**: buffer copies run as compute dispatches, and image
  copies are often fragment jobs. That is what the `upload 15-18 ms` at 5x/6x is.
- **Hidden Surface Removal** wants a consistent depth compare, depth writes enabled, and no
  `discard` / `gl_FragDepth`. We have **no depth attachment at all** — the recompiled game orders
  its own polygons, and our compute rasteriser has no `discard` (the only one in the tree is in
  the Metal backend, `src/gpu/gpu3d_metal.mm`). Nothing to fix; do not add one.
- **FP16 is genuinely cheaper here, and we cannot use it** (5th generation's FP16 transcendental
  instructions). The GPU 3D's compute shader interpolates spans and mixes colours in fp32 because it
  has to: the SDL_GPU backend is `gpu3d_quality::exact` only (`set_quality()` is a no-op,
  `src/gpu/gpu3d_sdlgpu.cpp:120-121`) and the picture is checked against the CPU's, so fp16's
  different rounding would be caught at the first check and the 3D sent back to the CPU. **This note
  used to ask for a half-precision experiment; that was wrong — `precise` (44 uses in
  `gpu3d.hlsl`) is the whole point of the exact path, and `analiza.md` says so too.**
- **Barriers are SDL's business** — this is a reason not to hand-roll Vulkan for the host: Mali
  wants specific pipeline stages, no general `VkMemoryBarrier`, and no `UNDEFINED`-layout
  transitions (they invalidate TE hashes). SDL_GPU emits them.
- **VRS and 2x MSAA modules** exist on this GPU but buy nothing for pixel-exact 1990s output;
  the "antialiasing" in this project is re-rendering the 3D at a higher scale, not MSAA.

## Worth doing, in order (all small)

1. **Done (2026-10-18) — log the present mode and the driver.** `apply_display_settings()` in
   `src/host/main.cpp` now prints one line whenever the renderer is `gpu`: which of
   mailbox/immediate/vsync the window accepts, what the current `vsync` setting really resolves to
   (`vsync off` → mailbox, else immediate, else vsync — the same order as SDL's `ChoosePresentMode`),
   and the driver's name, version and info. Seen on the desktop:
   `the GPU's present mode: mailbox (vsync off; mailbox, immediate no); the driver: radv 26.2.4 (Mesa 26.2.4-arch3.1)`.
   It settles "did vsync off really pick mailbox?" and "is this an old driver?".
2. **Done (2026-10-18) — `SDL_CreateGPUDeviceWithProperties`** in `gpu3d_create_sdlgpu`'s
   own-device path (`src/gpu/gpu3d_sdlgpu.cpp`): the four
   `SDL_PROP_GPU_DEVICE_CREATE_FEATURE_{CLIP_DISTANCE,DEPTH_CLAMPING,
   INDIRECT_DRAW_FIRST_INSTANCE,ANISOTROPY}_BOOLEAN` are set false, and `DEBUGMODE` keeps
   `SDL_CreateGPUDevice`'s old meaning (`GPU3D_VALIDATE`). `tools/gpu3dbench` still passes every
   scale on RADV, and `SDL_CreateGPUDevice` has disappeared from the phone's `libmain.so`.
3. **Fewer compute↔render switches per picture.** The 3D-on-GPU path already writes into the
   picture texture the renderer samples; the aim is one hand-off, not several. Measure first
   (the ~19 ms split is the next module's job), then act.
4. **Refuted (2026-10-18) — half precision in the GPU 3D compute shader.** Written off: the backend
   is exact-only and checked against the CPU's picture, so fp16's rounding would fail the check.
   See the correction above.
5. **Done (2026-10-18) — the two measurements `analiza.md` asked for.** The 10 s `pace:` block now
   also prints `the GPU at: 0.39 of 1.30 GHz` (`sysinfo::gpu_frequencies()`, `/sys/class/devfreq`),
   which is the only way to tell a throttled GPU from a busy one — a phone's PowerHAL is said to
   hold a program it doesn't take for a game at a quarter of the top clock; and `sysinfo::perf_hints_*`
   starts an Android 13+ (ADPF) performance hint session over the frame's threads with the frame's
   17.4 ms as its target, reporting each pass's real work back, which is the API meant to stop that
   kind of throttling (Display > Performance hints, `VF2_NO_PERF_HINTS=1` for an A/B run).
6. **At large scales, do not upload a CPU-composed picture** — the 5x/6x wall is the transfer
   (15-18 ms), and it is a compute/fragment job on this GPU by design.
7. **Do not chase**: MSAA, VRS, clip distance, depth clamp, user clip planes, AFBC control.
   None of it applies to what this game draws.

## What this does not fix

The GPU path's ceiling (54-56 pictures/s at 1x, ~51-56 at 2x with smooth motion off) and the
governor walking 2x → 1x. Those are host-side: per-picture overhead in our own loop and the
governor's rules. The notes above remove the ways this *driver* could be wasting time; they do not
raise a ceiling that our loop is setting.

## Sources

- Arm, *Vulkan synchronization on Mali GPUs: pipeline barriers, hardware behavior, and best
  practices* (Feb 2026) — CSF streams, compute/binning cannot overlap, TE and barriers:
  https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/vulkan-synchronization-how-pipeline-barriers-map-to-mali-gpu-hardware
- Arm, *Mali GPU Best Practices Developer Guide* (101897, latest):
  https://developer.arm.com/documentation/101897/latest/
- Khronos, *Tile Based Rendering (TBR) Best Practices*:
  https://docs.vulkan.org/guide/latest/tile_based_rendering_best_practices.html
- Arm, *Arm GPUs built on new 5th Gen architecture* (DVS, FP16):
  https://developer.arm.com/community/arm-community-blogs/b/announcements/posts/arm-gpus-built-on-new-fifth-gen-architecture
- Arm product page, *Immortalis-G720* (DVS, FP16 transcendentals, VRS):
  https://www.arm.com/products/silicon-ip-multimedia/immortalis-gpu/immortalis-g720
- SDL issue #12652 *GPU: Android spec unreasonable* (shaderClipDistance/depthClamp as mandatory,
  "excludes every Mali GPU"): https://github.com/libsdl-org/SDL/issues/12652
- SDL issue #10814 (user clip planes on Mali): https://github.com/libsdl-org/SDL/issues/10814
- Vulkanised 2023, *Getting started on mobile and best practices for Arm GPUs* (swapchain
  orientation, DPU-friendly presents — why the swapchain image should be in the panel's
  orientation, which ours is):
  https://vulkan.org/user/pages/09.events/vulkanised-2023/vulkanised_2023_getting_started_on_mobile_and_best_practices_for_arm_gpus.pdf
- Tools for a deeper look (a phone, not this machine): **Arm Performance Studio / Streamline**
  (Mali Timeline shows pipeline bubbles per CSF stream) and *Hardware CapsViewer for Vulkan* to
  read the driver version and supported extensions.
