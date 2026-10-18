# Optimization backlog

The performance and power work that 0SFS owns, ranked by what it should save and
what it should cost. Most of what makes flying faster lands in FOSS Earth: the
render loop, tiles, terrain, and backend-specific work. That list, with the
ranking method, is FOSS Earth's
[Optimization backlog](https://github.com/foss-earth/docs/blob/main/docs/optimization-backlog.md).
IDs from there (P1, T1, C1 and so on) are used here unchanged.

The [WebAssembly implementation brief](https://github.com/foss-earth/docs/blob/main/docs/proposals/implementation/webassembly-agent.md)
connects these items to gates W0–W12, with the repository ownership and proof.

Written 16 October 2026 against the working trees of that day. **No gain below was
measured.** They are estimates to order the work. Measure with `flightPerf=1`,
the frame budget, and the power harness (FOSS Earth M1).

## The 15 W goal

The target is a flight on an M5 MacBook Air in Low Power Mode that stays near
15 W. It is not a requirement, and nothing should be made worse to reach it. It
is a useful check that the cheap work has been done.

Where we start:

- On 10 September the pilot saw about 60 fps at 15 W aloft and 40 fps at 24 W
  near the ground, on an M5 ([Near-ground CPU performance](near-ground-performance.md)).
  The finite-ray fix came after that. The pilot then saw no slowdown near the
  ground, but power was not measured again.
- 0SFS sets no renderer values. It flies at every device pixel, so 4 pixels per
  CSS pixel on the Air's screen, with 4× MSAA and no frame cap. On a 120 Hz
  external display it draws 120 frames a second.
- Physics runs at 120 Hz whatever the frame rate.

What should reach it, cheapest first. Each step is a guess until the harness
measures it:

1. **Fewer pixels.** One device pixel per CSS pixel is a quarter of the fragment
   work, 1.5 is about half (P1). The HUD and panels are DOM and stay sharp.
2. **Fewer frames.** A 30 or 40 fps cap on battery roughly halves the render
   work per second (P4). Physics stays at 120 Hz.
3. **Less near the ground.** The collision BVH and ground cache (C1, F4) remove
   most of the collision work left after the finite-ray fix. The 10 September
   investigation tied that work to the jump from 15 W to 24 W.
4. **Less toward the horizon.** A cockpit view is mostly horizon; horizon detail
   (T1) and fog culling (T3) cut the triangles and downloads there.

Steps 1 and 2 need no new code: the settings exist. F1 below packages them.

## Ranking

| Rank | ID | Item | Gain (estimate) | Effort | Tier |
| ---: | --- | --- | --- | :-: | :-: |
| 1 | F1 | A battery flight preset: 1 to 1.5 device pixels per CSS pixel, 30 or 40 fps, MSAA off | Power −40 to −60% while flying | S | 1 |
| 2 | F2 | HUD writes only what changed | Style recalculation every frame | S | 1 |
| 3 | F3 | Attitude indicator at a lower, fixed rate | One canvas draw and composite per frame | S | 1 |
| 4 | E1 | Stop the exhaust ray march once the gas is opaque | Plume fragment work −20 to −50% where dense | S | 1 |
| 5 | F6 | Check the shipped JSBSim WebAssembly build flags | Physics CPU −10 to −30% if not optimized | S | 1 |
| 6 | F4 | Ground cache over the FOSS Earth collision BVH (C1) | Near-ground physics CPU −95% | M | 1 |
| 7 | E2 | Exhaust samples from the plume's size on screen | Plume work for distant and small plumes | S | 2 |
| 8 | E3 | Exhaust at half resolution with a depth-aware upsample | Plume fragment work −75% | M | 2 |
| 9 | E4 | Exhaust depth from the main pass, not a second depth pass (B9) | One geometry pass while exhaust shows | M | 2 |
| 10 | F7 | Load aircraft-only data when that aircraft is chosen | Startup bytes and parse | S | 2 |
| 11 | F9 | Aircraft level of detail; hide what the view cannot see | Draws and triangles | M | 3 |
| 12 | F5 | Flight model in a worker, after F4 | The physics loop's share of the main thread, not yet measured | L | 4 |

## Catalogue

### Settings and presets

| ID | Item | Status | Owner | Backend | Gain (estimate) | Effort | Tier |
| --- | --- | --- | --- | --- | --- | :-: | :-: |
| F1 | A "Battery flight" preset beside the six in `src/flight/presets`, which today set only controls and cameras: `renderer.resolutionScale` for 1 to 1.5 device pixels per CSS pixel, `renderer.frameRateCap` 30 or 40, `renderer.antialias` off, and the Save battery map values. Caps that divide 120 Hz give each frame the same number of physics steps (CP3). Offer it on battery where the browser says so (P6). | Open | fly | All | Power −40 to −60% while flying | S | 1 |
| F8 | A flight default for the frame cap on displays above 60 Hz: 60, unless the pilot asks for more | Open | fly | All | Power on 120 Hz displays −30 to −50% | S | 1 |

### HUD and instruments

| ID | Item | Status | Owner | Backend | Gain (estimate) | Effort | Tier |
| --- | --- | --- | --- | --- | --- | :-: | :-: |
| F2 | `flightHud.ts` writes three `textContent`s, five input values and three output values every frame, changed or not, and calls `throttle.update`. Each write invalidates style. Keep the last string per element and write only on change. (CP15) | Open | fly | All | Style and layout every frame | S | 1 |
| F3 | The attitude indicator draws on its own canvas every frame. Draw it at a fixed rate (30 Hz) or when its picture moves by a pixel. MSFS calls this the glass cockpit refresh rate. | Open | fly | All | One canvas draw and composite per frame | S | 1 |
| F10 | Join the attitude indicator to the globe's pass on WebGPU, as one more draw into the frame, instead of a second canvas the compositor layers | Open | fly, renderer | WebGPU | One composited layer less | M | 3 |

### Exhaust

The plume is a ray march through the gas volume: 32 steps per pixel by default,
up to 128 (`osfs.exhaust.sampleCount`), out to 2 km. Its GPU cost has not been
measured; the setting says so.

| ID | Item | Status | Owner | Backend | Gain (estimate) | Effort | Tier |
| --- | --- | --- | --- | --- | --- | :-: | :-: |
| E1 | End the march when transmittance falls below about 1/255: nothing further can show. The output changes by under one 8-bit step, so validate against the retained posed-ray comparison. | Open | exhaust | All | Plume fragment work −20 to −50% where the gas is dense | S | 1 |
| E2 | Choose the step count from the plume's size on screen, up to the setting | Open | exhaust | All | Small and distant plumes | S | 2 |
| E3 | Draw the plume into a half-resolution target and upsample with depth, as games draw volumetrics | Open | exhaust | WebGL2, WebGPU | Plume fragment work −75% | M | 2 |
| E4 | Read the main pass's depth instead of drawing every overlapping opaque mesh again into a float depth target (`engineExhaustDepth.ts`) (B9) | Open | exhaust, renderer | WebGL2, WebGPU | One geometry pass while exhaust shows | M | 2 |

### Physics and ground

| ID | Item | Status | Owner | Backend | Gain (estimate) | Effort | Tier |
| --- | --- | --- | --- | --- | --- | :-: | :-: |
| F4 | Once a frame, collect the triangles within reach of the aircraft (its size plus two steps of travel) from the FOSS Earth BVH (C1), and answer every gear and body ray of that frame from them (CP13) | Open | ground-contact, fly | All | Near-ground physics CPU −95% | M | 1 |
| F5 | JSBSim in a worker, after F4's ground cache. With shared memory (FOSS Earth C7, [WebAssembly builds, step 9](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md#9-jsbsim-in-a-worker)), the worker reads versioned ground data directly and paces with `Atomics.wait`; the main thread reads bounded, consistent state snapshots. Without isolation JSBSim stays on the main thread. (CP14) | Open | fly, jsbsim build | All | The physics loop's share of the main thread (`flightPerf=1` reports it) | L | 4 |
| F6 | Benchmark the shipped JSBSim build (2.4 MB) against WASM exceptions, then link-time optimization, then optional SIMD. The root already enables JavaScript exceptions across the engine; preserve its catches and error recovery. Require identical trajectories and record step time, size and startup ([WebAssembly builds, step 2](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md#2-jsbsims-exceptions-and-flags)). | Open | fly, research | All | Physics CPU −10 to −30%, estimate to test | S | 1 |

### Models and loading

| ID | Item | Status | Owner | Backend | Gain (estimate) | Effort | Tier |
| --- | --- | --- | --- | --- | --- | :-: | :-: |
| F7 | Load aircraft-only data with the aircraft: the F135 exhaust table's 878 kB JSON manifest is in every flight's build | Open | fly, F135 | All | Startup bytes and parse | S | 2 |
| F9 | Distance level of detail for aircraft, and hide the cockpit from outside and the airframe's unseen parts from inside (CP16) | Open | aircraft | All | Draws and triangles; matters with traffic | M | 3 |

### Looked at and left out

- **Fewer physics steps.** 120 Hz is JSBSim's fidelity, not waste.
- **Moving the five rays to the GPU.** The solver needs the answer before it
  steps, and WebGPU readback is asynchronous
  ([Collision compute comparison](https://github.com/0SFS/ground-contact/blob/main/docs/collision-compute-comparison.md)).

## Competitor techniques: status

What became of the 17 items in [Competitor performance techniques](competitor-performance.md):

| CP | Technique | Status | Now |
| ---: | --- | --- | --- |
| 1 | One pixel per CSS pixel | Setting exists, not the default | P1, F1 |
| 2 | Resolution scale setting | Done: `renderer.resolutionScale` | — |
| 3 | Frame-rate cap | Setting exists: `renderer.frameRateCap`; flight sets none | P4, F1, F8 |
| 4 | Horizon detail | Open | T1 |
| 5 | Centre of the screen first | Open | T4 |
| 6 | Requests while turning | Open | T5 |
| 7 | Skip levels of detail | Done: `optimizedLoadStrategy` is on | — |
| 8 | Cache bounded by bytes | Open; the byte setting exists but the adapter counts 1 per tile | T2 |
| 9 | Work off the main thread | Partial: decode in workers, mesh building not | R3 |
| 10 | Heights on the GPU | Open | R5 |
| 11 | Skirts | Open | R4 |
| 12 | Detail hysteresis | Done: the terrain selector refines and coarsens at different thresholds | — |
| 13 | Spatial index for ground | Open; prototype only | C1, F4 |
| 14 | Flight model off the main thread | Open | F5 |
| 15 | HUD text on change | Open | F2 |
| 16 | Distance level of detail | Open | F9 |
| 17 | GPU-compressed textures | Open | B3, X6 |
