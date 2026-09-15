# Miyoo Flip performance wins

Every optimization that took the native Melee port on the Miyoo Flip V2
(Rockchip RK3566, 4x Cortex-A55 at 1.99 GHz, Mali-G52 MP2 at 900 MHz, ~1 GiB
RAM, proprietary Mali GLES blob, no Vulkan) from about 3 FPS to the 60 Hz
boundary, with the measured impact of each. Numbers are quoted from the
iteration logs under `native/validation/2026-09-09-flip/` and
`native/validation/2026-09-10-flip/RENDERER_ITERATION.md` and from
`native/FLIP_HANDOFF.md`; the per-trial raw logs live on the development
machine and are not committed.

Target class: this hardware is representative of ARM, draw-call-constrained
GLES systems in general: low-cost handheld emulation devices, Raspberry Pi
class boards, and weaker phones. On these systems the CPU cost of a draw call
inside the GL driver (15-25 us on this blob) dwarfs the GPU cost of the
triangles it submits, and the per-pass cost (0.5-0.7 ms of driver time per
render pass regardless of contents) is similarly large. Most of the wins below
therefore reduce draws, passes, or per-draw CPU work rather than GPU work.

## Trajectory

| Stage | Frozen Onett frame time | Notes |
| --- | --- | --- |
| Stock g13 Mali driver, release v8 | 309.6 ms (~3-4 FPS), ~450 draws | 90% of the frame inside the driver; 124 ms of it texture-fetch barriers |
| First backend rewrite (g29 driver, CPU vertex decode, batching) | 106.5 ms (9.4 FPS) | Fountain of Dreams 158.7 ms |
| Direct GLES submission (v61) | 38.6 ms mean | Battlefield 24.5 ms |
| Pipelined presentation and dirty uploads (v70) | 29.9 ms | Moving Onett ~28 FPS, Battlefield ~40 FPS |
| Second renderer iteration (v79 to v143) | 17.1 mean / 16.7 median, p95 16.9 | Moving Onett, Battlefield, Pokemon Stadium, Hyrule all at 16.7 ms median, 52-59 FPS |
| Fountain of Dreams (v143) | 23.9 ms, p95 36 | ~37-40 FPS; 25k point sprites per frame |

Final Onett split (async frames, v140): game thread ~5 ms, GX translation
worker ~8.5 ms, render worker ~12-14 ms (the limiter, almost all inside the
Mali driver for ~271 draws and 3 passes), GPU ~11-12 ms.

Bit-exactness: through v139 every default change reproduced the reference
EFB capture byte for byte (Onett `da46d4a79a8f`, Fountain `cc2fea00f05f`).
From v140 the clamp-texture atlas moves ~100-170 pixels by one unit;
`MELEE_FLIP_TEXTURE_ATLAS=0` restores exactness.

## Platform and launcher

| Change | Where | Impact |
| --- | --- | --- |
| Online all four A55 cores while the game runs (stock Ports settings leave two offline), restored on exit | `native/platform/flip/launch.sh`, `MELEE_FLIP_ALL_CORES` | 288 -> 235 ms at the time (+23-34% FPS); render-worker run-queue wait 4.65 -> 0.72 s per 25 s window |
| Performance governors for CPU, GPU and memory controller, restored on every exit path | `launch.sh`, `MELEE_FLIP_PERFORMANCE` | DMC alone 37.3 -> 34.0 ms; CPU 1992 MHz, GPU 900 MHz, DMC 780 -> 1056 MHz |
| App-local Mali g29p1 GLES driver on `LD_LIBRARY_PATH` instead of the firmware's g13p0 | packaging, `launch.sh`, `MELEE_FLIP_DRIVER` | 251 -> 114 ms on matched Onett controls; makes the no-barrier vertex path legal |
| Shader cache seeded from trial runs | packaging | no compile stalls on first boot; no isolated frame number |

## First backend rewrite (Aurora patch, v9-v55)

| Change | Impact |
| --- | --- |
| CPU-decoded compact interleaved vertex inputs replacing integer-texture vertex pulling (which faulted on Mali) | Onett vertex upload 6.25 -> 1.49 MB/frame |
| 64 KiB uniform windows with 4 KiB records so adjacent draws share a bound range | 114 -> 94 ms together with the above |
| Removing the per-draw texture-fetch barrier on the g29 driver (images byte-identical) | the barriers were 124 of 279 ms of driver time |
| Adjacent compatible draw batching (record index in the vertex stream, flat varying) | Fountain ~2500 -> ~470 draws/frame; first release 106.5 ms / 9.4 FPS Onett |
| Upload all immutable geometry once before any pass instead of interleaved with passes | 106.7 -> 70.5 ms |
| CPU staging with used-range `WriteBuffer`, smaller texture staging, smaller pools | ~265 MiB less reserved memory |
| On-demand pipeline compilation instead of prewarming the historical cache | startup/memory; live pipelines 143 in a fixed scene |
| Idle GX texture wrappers expire after 60 frames instead of 600 | memory; no isolated number |

## Direct GLES submission and presentation (Aurora patch, Dawn patch, present worker, v56-v78)

| Change | Impact |
| --- | --- |
| Direct GLES submission of supported GX passes inside Dawn's EGL context (Dawn keeps resources, shaders, presentation) | Onett 49.6 -> 38.6 ms, Battlefield 30.0 -> 24.5 ms, bit-exact |
| GLES state and binding caches, GLES 3.1 separate vertex format/buffer binding | -1.4 ms |
| Resource-only packet encoding (`MELEE_FLIP_DIRECT_PACKET`) | 38.3 -> 37.4 ms |
| Threaded EGL swap and DRM/GBM presentation worker with fences | 37.3 -> 31.3 ms; presentation 10.6 ms overlaps 16.7 ms of submission |
| Changed-range uploads against shadow copies (`MELEE_FLIP_DIRTY_UPLOAD`) | 37.3 -> 35.9 ms alone; 30.1 ms combined with the two above |
| NEON conversion of common s16/u16 XY/XYZ attributes | no isolated number |
| Deep profiling timers compiled out by default | 32.1 -> 30.1 ms |

## Second renderer iteration (Aurora patch, v79-v143)

Baseline v79: 30.7 ms; game thread 6.5 ms then 22.4 ms blocked joining the
FIFO worker; FIFO worker 28.6 ms; render worker 25.1 ms; GPU 11.3 ms.

| Change | Layer | Impact |
| --- | --- | --- |
| Specialized vertex loaders: one decode plan per attribute configuration, monomorphic conversions, every output byte written once | `lib/gx/flip_vertex.hpp` | FIFO 28.6 -> 24.4 ms; frame 30.7 -> 27.2 |
| Resident display-list geometry: `GXCallDisplayList` decoded once into a GPU arena, later calls append indices; per-frame record stream keeps resident draws mergeable | `lib/gx/flip_resident.hpp`, `GXDispList.cpp` | FIFO -> 16.4 ms; uploads ~3.8 MB -> ~0.3 MB/frame; frame 25.7 |
| Stable texture identities from the image description (including mode0/mode1/TLUT) with sampled content verification | `GXTexture.cpp`, `gx/texture.cpp` | FIFO -> 13.6 ms; frame 22.6 |
| Persistently mapped uniform and index streams (`GL_EXT_buffer_storage`, fenced slots; needs `glDrawRangeElements` on Mali) | `gfx/frame.cpp` | render upload 4.1 -> 0.2 ms; frame 22.6 -> 19.3 |
| Direct-path trimming: redundant state filtering, texture-parameter memo, sampler state folded into textures, no per-draw immediates | `gfx/flip_gles.cpp` | ~2 ms render |
| Framebuffer-object cache keyed by attachment names | Dawn `CommandBufferGL.cpp` | ~2 ms; frame 19.3 -> 17.4 |
| Async frames: the game thread no longer joins the FIFO worker; frame markers travel through the GX stream; FIFO buffer compacted between frames | `aurora.cpp`, `gx/fifo.cpp` | frame = max of three workers; 56 FPS frozen Onett; compaction fixed a 600 MB / 44 s leak |
| Pipeline-state memo: XXH3 of raw pipeline-relevant GX state skips config/shader-info rebuilds for the ~460 identical dirty events per frame | `gx/command_processor.cpp` | FIFO 11.7 -> 9.9 ms |
| Resident invalidation index (pointer -> entry multimap instead of a scan over all entries x 26 array pointers) | `flip_resident.hpp` | was 43% of the FIFO worker in moving Onett; FIFO 20 -> 15 ms, 60-120 ms tail frames gone |
| Mapped vertex stream: never `WriteBuffer`/`glBufferSubData` into a buffer the GPU may still read (the blob waits a full GPU frame) | `gfx/frame.cpp`, `encoding.cpp` | moving Onett render 27 -> 17.7 ms; every measured scene at the 60 Hz boundary |
| Opt-in opaque draw sort by pipeline/texture/window | `flip_gles.cpp`, `MELEE_FLIP_SORT_OPAQUE` | -0.5-0.7 ms, program switches 147 -> 83; off by default (coplanar decal risk) |
| Swapchain texture pool instead of a new texture per frame | Dawn `SwapChainEGL.cpp`, `present_worker.cpp` | render worker 16.3 -> 15.3 ms |
| Resident arena misses uploaded through a staging buffer and `glCopyBufferSubData` | `flip_resident.hpp` | removes ~4 implicit GPU syncs per second; no isolated number |
| Specialized line/point expansion (Fountain: ~1850 GX_POINTS draws, 25k sprites in one display list) | `flip_vertex.hpp` | Fountain FIFO 46 -> 23 ms; 21 -> 37-40 FPS |
| Skip the empty ImGui overlay pass | `lib/imgui.cpp` | one pass fewer per frame |
| Shadow-pass fusion: the second 256x256 shadow map is recorded into the first pass with shifted viewport/scissor and two resolves, per-channel exactness rules, exact split fallback | `gfx/recording.cpp` | Onett passes 7 -> 5; -0.7 ms; moving p95 20 -> 17 ms |
| Two-target texture conversion pass for the fused shadows | `gfx/tex_copy_conv.cpp` | passes 5 -> 4 |
| Instanced point sprites: one record per point, divisor-1 attributes, shared 6-index quad | `command_processor.cpp`, `flip_gles.cpp` | Fountain FIFO 25 -> 21 ms (was writing 3 MB of quad corners per frame) |
| Texture arrays: GX textures become layers of shared 2D arrays grouped by size/mips/format/sampler; layer index in the uniform record | `gfx/texture.cpp`, `gx/shader.cpp` | Onett 360 -> 343 draws, Fountain 474 -> 445; +30-50 MB RAM |
| Scene rendered straight into the presented texture; present copy pass skipped | `gfx/recording.cpp`, Dawn swapchain hooks | passes 4 -> 3; render callback 13.8 -> 11.8 ms |
| Texture content re-verified once per 4 frames instead of per bind | `gx/texture.cpp` | FIFO -1 to -3 ms on Fountain |
| Clamp-texture atlas: 1024x1024 layers, shelf packer, one-texel gutter, `clamp(uv)*scale+offset` in the shader | `gfx/texture.cpp`, `gx/shader.cpp` | Onett moving 333 -> 271 draws, Fountain 431 -> 369; ~44 MB; costs +-1 on ~100-170 px |
| Fountain reflection pass (80x60 RGB565 second camera, ~150 draws) rendered every other frame | `recording.cpp`, `MELEE_FLIP_SCALED_COPY_INTERVAL` | reflection pass 210 -> 105 submits; -3-4.7 ms render worker on those frames |
| Half-resolution sprite pass: point runs render into a 320x240 target with copied depth and composite back | `gfx/flip_sprites.cpp` | Fountain main-pass GPU 19-24 -> 9-13 ms; net 26.6 -> 23.9 ms, p95 50 -> 36 after the atlas |

## Rules learned about the Mali blob

- Any per-draw uniform change (`glBindBufferRange`, `glUniform*`, a constant
  vertex attribute) costs ~15 us of CPU, as much as the draw itself. Only data
  embedded in the vertex stream is free. This killed the per-draw constant
  record experiment (render worker 15.3 -> 20.5 ms for -1 ms GPU).
- GL call count is not the cost. One VAO per layout cut vertex-buffer binds
  82 -> 21 per frame with zero change in the render worker. The driver's
  draw-time descriptor build reacts only to draws and passes.
- Never write into a buffer the GPU may still be reading; the blob blocks the
  CPU for about one GPU frame. Never read back from write-combined mapped
  memory on the producer thread (a 4-byte read per record cost ~20 ms/frame).
- Index data consumed from a persistently mapped element buffer draws stale
  geometry with `glDrawElements`; `glDrawRangeElements` with explicit ranges
  is correct.
- Dynamic uniform-record indexing in the TEV fragment shaders costs ~2.7 ms of
  the ~13 ms GPU frame; flat varyings cost the same, so only a uniform index
  would help.
- Each render pass costs 0.5-0.7 ms of driver time regardless of its draws;
  `glBlitFramebuffer` costs the same as a copy pass because the pass setup,
  not the draw, is the cost.
- The g29p1 blob exports no Vulkan entry points and no
  `GL_EXT_multi_draw_indirect`; it has `EXT_buffer_storage`,
  `ARM_shader_framebuffer_fetch` and base-vertex draws.

## Tried and rejected

| Experiment | Result |
| --- | --- |
| Removing texture-fetch barriers on the g13 driver, barrier masks | -4% but 501 new GPU faults and a wrong image; the marker sweep's mask failed in moving Onett |
| Dedicated thread pinning | 319 vs 288 ms, slower |
| Extra submission threads around the shared EGL context | Dawn serializes the context; rejected on design |
| GLSL uniform-window interposers on g13 | wrong images; timings invalid |
| Context-local GL state shadowing | no speed change |
| Native specialized GLSL generator | ~42 ms vs ~35 ms for replaying existing programs |
| Uber shader | 245-260 ms/frame, ~3000 texture binds, not exact |
| Texture-pair batching | ~4800-5700 texture binds/frame, render 17 -> 19 ms |
| Per-draw constant uniform records | render 15.3 -> 20.5 ms for -1 ms GPU |
| One VAO per attribute layout | neutral |
| TEV constants as flat varyings | GPU 10.6 vs 10.5 ms |
| Present blit instead of copy pass | same 0.8 ms, 20 pixels moved |
| Native fenced streams, decoded-vertex cache, Dawn mapped uploads | gains too small to enable |
| Dawn `disable_robustness` | no change |
| Pipeline prewarming | crashes at startup; must stay off |

## Chart

`native/validation/2026-09-14-flip/onett-frame-time.png` plots the frozen-Onett
frame time per optimization (log scale), with markers colored by where each
change lives: Aurora PR set, Dawn escape hatch / direct GLES, or platform.
Data: `onett-frame-time.csv` next to it; regenerate with
`native/tools/plot_onett_frame_time.py <csv> <png>` (needs matplotlib).

## Aurora-only estimate

What the upstream Aurora PR set (`integration/flip-prs`: vertex loaders,
resident display lists, stable texture identities, pipeline memo, async frames,
instanced points, texture arrays + atlas, pass fusion, ImGui skip) buys on the
Flip without the Dawn escape hatch, direct GLES, mapped GL streams, FBO cache,
swapchain pool, present worker, dirty uploads, scene-on-surface, or half-res
sprites. This is an estimate; the `miyoo-flip-aurora-prs` melee branch exists to
measure it.

- Fully captured: the GX translation side. The FIFO worker went 28.6 -> ~9 ms
  through loaders, resident lists, texture identities, memo and instanced
  points, all in the PR set, and async frames removes the 22 ms/frame join.
- Partly captured: draws 360 -> ~271 and passes 7 -> 4 (arrays, atlas, fusion,
  ImGui skip) shrink whatever the render path costs per draw and per pass.
- Not captured: the render worker still runs Dawn's GL backend. The last
  measurement of that path on this device (v5x, ~450 draws, 7 passes) was a
  49.6 ms frame, render-worker bound, before direct GLES took it to 38.6 and the
  Dawn-side fixes to ~15 ms.
- Estimate: game ~5 ms and FIFO ~9 ms no longer matter; the frame is the Dawn GL
  render worker at ~270 draws and 4 passes, roughly 28-35 ms, i.e. ~30 FPS on
  Onett versus 17 ms / 58 FPS with everything. In milliseconds that is about
  94% of the 309.6 -> 17.1 reduction, but only about half the final frame rate.
- On targets with a Vulkan driver (Raspberry Pi 4/5, most phones) Aurora's
  Vulkan backend has far lower per-draw CPU cost than the Mali GLES blob through
  Dawn GL, so the PR set alone should land much closer to the full result there;
  the escape hatch exists because this device has GLES only.

## Aurora-only: measured (2026-09-14, v151-aurora-prs on the device)

The estimate above was too optimistic. Same trial script, same device, same
session, frozen Onett:

| Build | Presented FPS | Frame | Game thread | Render worker |
| --- | --- | --- | --- | --- |
| v150 (full stack: direct GLES, Dawn hooks) | 58.5-59.1 | 17.1 ms | ~5 ms | ~13 ms |
| v151-aurora-prs, warm shader cache | 18.6-19.5 | ~50 ms | 4.6 ms, then 47.6 ms waiting in begin_frame | ~50 ms |
| v151-aurora-prs + MELEE_FLIP_ASYNC_PRESENT=1 | 21.2-21.4 | ~46 ms | 4.7 ms, 42 ms waiting | ~44 ms |

CPU samples of the Aurora-only render worker (12 s, 7,664 samples): **libmali
70%**, Dawn core + GL backend 19% (BindGroupTracker::Apply, ExecuteRenderPass,
VertexStateBufferBindingTracker::Apply, RefCounted, SyncScopeUsageTracker),
libc 8%, Aurora's own render code under 1%. With the direct path the same
worker was ~13 ms with libmali at 62%. So the Mali driver spends about four
times longer on the same ~270 draws and 4 passes when Dawn's GL backend drives
it: a `glBindBufferRange` per draw for the dynamic-offset uniform (on this blob
any per-draw uniform change costs as much as the draw), bind groups re-applied
per draw, FBO gen/attach/check/delete per pass, and no redundant-state
filtering. The GX translation side is fine: the FIFO worker is no longer on the
critical path. The synchronous DRM flip on the render worker costs ~4 ms.

What could still move upstream (WebGPU-level, generic): the first-iteration
uniform table (64 KiB windows of 4 KiB records, one bind per pass, record index
in the vertex stream) plus adjacent-draw batching, which removes the per-draw
UBO rebind and merges draws; on the Dawn side the FBO cache, swapchain texture
reuse and redundant-state filtering in the GL backend. The last stretch from
~25 ms to 13 ms came from bypassing Dawn's command execution entirely and has
no WebGPU-level equivalent.

## Dawn-side candidates: measured (2026-09-15)

The three Dawn changes proposed for encounter/dawn were bisected on the device
in the Aurora-only build (render work per frame net of the display wait):
EGL surface source only 47.1 ms; plus swapchain storage ring 46.8 ms (neutral);
plus framebuffer object cache 57.0 ms (regression: the render thread blocks
inside GL calls, most likely because render targets stay attached to live
framebuffers while Dawn's copy paths read them); all three 56.4 ms. The ~2 ms
FBO-cache win in this document came from the direct GLES path with its own copy
paths and does not transfer to Dawn's command execution on this driver. The
Flip's "Melee Native Dev" listing runs the surface-source + ring variant.

## Upstream candidates

Backend-independent and generic to any GX game on Aurora: specialized vertex
loaders and line/point expansion, pipeline-state memo, stable texture
identities with sampled verification, async frames with FIFO compaction,
instanced point sprites, texture arrays (and the opt-in atlas), resident
display lists (needs the `GXCallDisplayList` marker and an invalidation hook
from the game), skipping the empty ImGui pass, and the adjacent-pass fusion
pattern. Dawn-side: the EGL native-window surface source, the GL
framebuffer-object cache, and swapchain texture reuse. Not upstreamable: the
Direct-GL escape hatch and everything in `flip_gles.cpp` that depends on it,
the launcher's device specifics, and the Mali driver bundling.
