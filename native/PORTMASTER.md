# PortMaster: Melee port, and the existing Dusklight port (research, 2026-09-15)

## What PortMaster expects
- A zip per port: `port.json` (version 4: title, porter, desc, inst, genres, `arch: ["aarch64"]`,
  `min_glibc`), `README.md` (thank-you to the upstream project, controls, build notes),
  `screenshot.jpg` (4:3, gameplay), `gameinfo.xml`, a `Port Name.sh` launcher and a lowercase
  `portname/` data dir with `libs.aarch64/` and `licenses/`. Launcher conventions: source
  `control.txt`, `get_controls`, `$GPTOKEYB` for pad-to-key mapping, `pm_platform_helper`,
  `pm_finish`, keep everything under `$GAMEDIR` (never touch OS libraries).
- Build environment: porters build on old userlands (Ubuntu 20.04/22.04 arm64 rootfs, or the
  `ghcr.io/monkeyx-net/portmaster-build-templates/portmaster-builder` images) so binaries run on the
  oldest CFW glibc; the Dusklight porters' Aurora fork has a commit "Support Bullseye's C++20 library
  subset" (Debian 11, glibc 2.31, GCC 10). Our Flip toolchain targets glibc 2.37 and bundles libstdc++;
  a PortMaster build must be rebuilt against a Bullseye/Focal sysroot.
- Display and input come from the CFW: SDL2 (KMSDRM, fbdev, or Wayland on ROCKNIX), the CFW's own
  Mali/Adreno/Panfrost driver, `gptokeyb`. Ports do not open DRM themselves.
- Submission: PR to `PortsMaster/PortMaster-New` with documented testing on AmberELEC, ArkOS, ROCKNIX,
  muOS at 640x480 and higher. Nintendo-derived reimplementations (Ship of Harkinian, 2Ship, sm64coopdx,
  zelda3, Dusklight) all live in the **Multiverse** repo `PortsMaster-MV/PortMaster-MV-New`, which the
  project does not curate; a Melee port would go there too.
- Miyoo Flip: PortMaster runs on the stock OS through `chrisj951/MiyooFlipPortMaster` (also in
  SpruceOS and Surwish). It ships the muOS prebuilt libs, swaps the Mali driver when launching a port,
  and installs ports under the stock `Ports` listing that `melee-native` already uses.

## The existing Dusklight port (Multiverse, merged 2026-07-29, porters bmdhacks and jckhng)
`ports/dusklight/`: `Dusklight.sh`, a 37 MB `dusklight.aarch64`, `libs.aarch64/libSDL3.so.0`
(3.6 MB), `runtime/TwilitRealm/Dusklight/config.json.default`, `res/`. Tested on ArkOS, ROCKNIX, muOS,
dArkOS at 480x320 to 1280x720; runs on RK3326 (R36S), H700 and up.

How it renders:
- `--backend opengles`: **bmdhacks' hand-rolled GLES 3.0 device** inside Aurora (`lib/gl/*`,
  `bmdhacks/aurora` main, commit "Dusklight GLES backend: carry the fork onto encounter/main f8573d3",
  117 files against upstream). It implements Aurora's internal `webgpu` interface directly, so **Dawn is
  not used at all**: passes, programs, program-binary and pipeline caches, and "native vertex fetch
  (CPU-expanded hardware attributes + content-hash geometry cache)". Upstream's FIFO/command-processor
  refactor, texture replacement streaming and external render targets are deliberately not carried.
- **SDL3-over-SDL2 shim** (`bmdhacks/SDL` branch `sdl2-backend`, shipped as `libSDL3.so.0`): the
  game links SDL3, the shim implements SDL3's video/audio/GPU on top of the CFW's SDL2, so the port
  gets the device's display path (KMSDRM/fbdev/Wayland) without knowing about it. Aurora's GLES device
  presents the EFB through EGLImages the shim provides (`lib/webgpu/sdl2shim_present.cpp`). The launcher
  pins `SDL_VIDEODRIVER=sdl2` (and the inner SDL2 to Wayland on ROCKNIX).
- Handheld defaults: `internalResolutionScale 0.5`, bloom/DoF off, frame interpolation off,
  `decoupleSimFromRender`, performance governors pinned for the session, shader caches wiped when the
  binary changes.
- jckhng first tried the minimal route (`jckhng/aurora` branch `portmaster-upstream-1.4.1-minimal`,
  34 files: keep Dawn's GL backend, add an SDL2-shim presenter and "PortMaster GX submission path"
  tweaks, June-July 2026) before the shipped port moved to bmdhacks' full GL device. That mirrors our
  own measurement: the PR set through Dawn GL is 19 FPS on the Flip, the escape hatch 58.

Independent confirmation of our vertex-path argument: their `RENDERING_NOTES.md` root-causes the
"porcupine" (exploded skinned vertices) on Asahi/Mesa and Adreno/turnip to a driver miscompile of the
lone storage byte-load that feeds the matrix-palette index, shows that no WGSL-level fix survives CSE,
and concludes that **native vertex attributes are the only robust fix and are mandatory on Mali**
(`GL_MAX_VERTEX_SHADER_STORAGE_BLOCKS = 0`). Upstream Aurora's `load_word` guard comment ("discourage
some Adreno drivers... vertex explosions in Dusklight") is aimed at the same bug class.

## How their stack compares with ours
| | Dusklight Multiverse port | melee-native Flip / `zalo/aurora-arm` |
| --- | --- | --- |
| Vertex path | CPU-expanded attributes + geometry cache (fork-only) | `cpuVertexDecode` + resident display lists + instanced points, opt-in, tested, PR-shaped |
| Renderer | full GLES 3.0 device replacing Dawn (117 files, not upstreamable as-is) | Dawn kept; nine upstream-shaped PRs + GLES direct submission through a Dawn interop extension, byte-identical toggles |
| Presentation | SDL3-over-SDL2 shim (works on every PortMaster CFW) | own DRM/GBM/EGL display code (Flip stock OS only) |
| Draw batching / uniform table | unknown (not described) | measured: draws halved, bit-exact |
| Measured | no public numbers found; runs on RK3326-class devices | frozen Onett 58-59 FPS on RK3566 |

The shim is the piece we lack for PortMaster and it is orthogonal to the renderer: it solves "how does
an SDL3 Aurora app get a window on a CFW", not per-draw cost. Whether their GL device is as fast as our
fast path is not knowable from the repos; the test is to run their `dusklight.aarch64` and a Dusklight
built on our Aurora on the same device and scene (needs our Dusklight Flip build first; see
`~/Desktop/aurora-bench/dusklight-flip.md`).

## Melee PortMaster port: plan
1. **Presentation:** adopt the SDL3-over-SDL2 shim instead of `native/platform/flip/display.cpp`.
   Aurora's `BackendBinding.cpp` needs a case for the shim's EGL surface (the Dawn
   `SurfaceSourceEGLNativeWindow` from our Dawn PR is exactly the surface type), and the GLES fast
   path's present worker needs to hand frames to the shim (or fall back to Dawn's swapchain present).
   Alternative with no shim: static SDL3 with KMSDRM, but CFWs differ (fbdev on RK3326, Wayland on
   ROCKNIX), which is why the shim exists.
2. **Toolchain:** rebuild against a Bullseye/Focal aarch64 sysroot (PortMaster docker image), drop the
   bundled Mali driver (the CFW provides one; the stock-Flip PortMaster layer swaps it), keep bundled
   libstdc++ only if the target glibc allows it, set `min_glibc`.
3. **Launcher:** `Melee.sh` following `Dusklight.sh` (governors, `gptokeyb`, log tee, cache wipe on
   binary change), disc image from `melee/assets/*.iso|*.ciso|*.rvz`, saves under `$GAMEDIR/runtime`.
   Our `MELEE_FLIP_*` env defaults become the port's config; one pipeline cache per renderer variant.
4. **Controls:** replace the Flip-specific mapping (D-pad/stick and A/B swap) with `gptokeyb`/SDL
   controller config from `get_controls`; keep the swap as an option.
5. **Devices:** RK3566 (Flip, RG353) is the proven target; RK3326 has half the CPU and a Mali-G31,
   where the fast path matters even more and 60 FPS is unlikely; RK3588 and Snapdragon devices have
   Vulkan, where the PR set alone applies.
6. **Submission:** Multiverse PR with the four-CFW testing checklist; screenshot of a match.

## Dusklight on PortMaster: contribute rather than fork
The port exists and is maintained (crash fix merged 2026-09-13). The useful contribution is on the
renderer: (a) offer the upstream-shaped PR set (vertex loaders, resident display lists, texture
identity, memo, async frames, texture arrays, pass fusion) to bmdhacks/jckhng as a replacement for the
fork's private vertex/geometry cache, since both sides now agree native vertex fetch is mandatory;
(b) measure their GL device against our Dawn-plus-direct-submission path on one device; (c) if ours is
faster or equal, propose the shim + our Aurora as the port's next base, which would also be the
vehicle for the Melee port. First contact: the Multiverse PR threads (#145, #151) and the RENDERING_NOTES
authors.
