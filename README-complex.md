# MuJoCo tabletop clearing — scope, parameters, and verification

## Phase 0 decisions

- Deliverable: `complex.html` is the second page, after the scripted fallback passes. It uses the official MuJoCo WebAssembly build and Three.js from CDNs; there is no build step, npm, or backend.
- Mode: deadline mode — correctness and an honest working version come before polish.
- Scene: a white tabletop, a fixed open cardboard box in the lower-left, and a light-blue/silver UR-style six-axis arm with a black two-claw gripper.
- Gripper: one wrist with two opposing claws, both connected to slide joints and driven symmetrically.
- Arm: one connected MJCF kinematic tree from fixed base through six hinge joints to wrist and both claws. Arm bodies never use free joints.
- Object order: red cube → grey bar → purple arch → rear stack (blue arch, red rectangular block, orange cylinder) → black pen → tape roll. The bottle is out of scope and must not reappear.
- Build order: start with only the red cube; add exactly one requested object/group at a time and re-run verification after each addition. The tape roll is last because round side-lying objects are harder.
- Loop: reach → lower → close → lift vertically → move over the box → release under gravity → return.
- Physics contract: props are independent free bodies; carried props move only from MuJoCo contact and friction. No attach/parent trick and no scripted prop transforms.
- Counter: an object counts only when its measured MuJoCo body position is inside the box region.
- Scope cut: only after the Phase 5 tuning order and Phase 6 strategy fail may a resistant shape be removed, and the report must say so. No success is faked.

## Planned controls

- Start: run the full sequence.
- Step: advance one task phase.
- Reset: restore MuJoCo state and the counter.
- Mouse drag: orbit the Three.js camera.

## Parameter ledger

Values are intentionally unset until the relevant phase is run and measured. Each tuning round changes one variable only.

| Round | Object(s) | Mass | Pad/object friction | Gripper inertia | Carry speed | Solver | Result |
|---|---|---:|---|---|---|---|---|
| Baseline | red cube | 0.020 kg | 1.2 / 0.9 | 0.035 kg per finger, armature 0.002 | Move 1.8 s; Close 1.4 s with 0.7 s hold | dt 0.0025 s, implicitfast, Newton, elliptic, impratio 10, iterations 20, ls_iterations 20 | Phase 4 full loop PASS |
| Add 1 | grey bar | 0.015 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | 2/2 PASS |
| Add 2 | purple arch | 0.025 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | 3/3 PASS |
| Add 3 | blue arch | 0.025 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | 4/4 PASS after moving the stack clear of the box wall |
| Add 4 | red rectangular block | 0.018 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | 5/5 PASS |
| Add 5a | orange cylinder | 0.012 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | FAIL: ejected/tunneled |
| Add 5b | orange cylinder | 0.012 kg | 1.2 / 1.1 | unchanged | Move 1.8 s, smoothstep | unchanged | 6/6 PASS before the final object count changed |
| Add 5c | orange cylinder | 0.012 kg | 1.2 / 1.2 | unchanged | Move 1.8 s, smoothstep | unchanged | one 8/8 PASS, next run failed at friction cliff |
| Final | orange cylinder | 0.012 kg | 1.2 / 1.4 | 0.035 kg per finger, armature 0.002 | Move 3.6 s, two-segment cosine easing | unchanged | two consecutive 8/8 PASS, no tunneling |
| Add 6 | black pen | 0.010 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | 7/7 PASS |
| Add 7 | tape roll | 0.020 kg | 1.2 / 0.9 | unchanged | Move 1.8 s, smoothstep | unchanged | final 8/8 PASS |
| Smoothing A | all objects | unchanged | unchanged | unchanged | default Move 2.8 s; 65% approach + 35% insertion, cosine easing | unchanged | 8/8 PASS |
| Smoothing B | all objects | unchanged | unchanged | unchanged | release 1.4 s: 0.392 s opening + 1.008 s dwell | unchanged | 8/8 PASS; late-settle drift <= 0.1 mm |
| Damping rejected | all objects | unchanged | unchanged | unchanged | unchanged | j0..j5 = 5, 5, 4, 3, 2, 1.8 | FAIL at purple arch; discarded |
| Damping rejected | all objects | unchanged | unchanged | unchanged | unchanged | j0..j5 = 4.5, 4.5, 3.5, 2.5, 1.75, 1.5 | FAIL at purple arch; discarded |
| Servo damping rejected | all objects | unchanged | unchanged | unchanged | unchanged | actuator dampratio 0.7 | 8/8, but Move endpoint speed regressed to 0.4204 rad/s; discarded |
| Minimum jerk | all objects | unchanged | unchanged | unchanged | default Move 2.8 s; orange 3.6 s | j0..j5 unchanged | 8/8; endpoint speed improved to 0.1220 rad/s |
| Split-stop rejected | all objects | unchanged | unchanged | unchanged | two minimum-jerk Move segments with a full stop at the waypoint | unchanged | 8/8 but visibly stop-start and clumsy; discarded |
| Smoothing final | all objects | unchanged | unchanged | fingers unchanged; armature unchanged | continuous cubic Bezier Move, 3.15 s incl. 0.347 s hold; orange 4.05 s incl. 0.446 s hold | j0..j5 = 4.2, 4.2, 3, 2, 1.5, 1.2 | two consecutive Reset-to-8/8 PASS; max endpoint speed 0.0499 rad/s |

## Phase status

- Phase 0: PASS — scope is recorded and was confirmed in the project prompt.
- Phase 1: PASS — `complex.html` loaded the official Google DeepMind `@mujoco/mujoco@3.17.0` single-threaded WebAssembly module from jsDelivr over a local static server. The visible status reported initialization, and no model or scripted fallback was present.
- Phase 2: PASS — the MJCF compiled; after one simulated second the 20 g cube rested at measured `z=0.775 m` on a table whose top is `z=0.720 m`. The fixed base, six hinge bodies, wrist, and both slide-jointed claws rendered as one connected neutral chain with no sinking or explosion.
- Phase 3: PASS — actuator-driven extended, folded, and neutral poses all kept the measured body chain connected within 4 mm; maximum settled joint-target error was 0.047 rad. Rendering synchronizes MuJoCo `xpos`/`xquat` to body groups under one Z-up-to-Y-up root, while geom meshes stay local children.
- Phase 4: PASS for the red-cube gate — Start completed reach → lower → close → vertical lift → move → gravity release → return. Step inspection showed the cube suspended after lift and still held over the box; measured MuJoCo position inside the box produced `Cleared 1 / 1 objects`. Reset restored the initial state.
- Phase 5: PASS — all props use realistic 10–25 g masses. Pad sliding friction is 1.2; object sliding friction is 0.9 except the orange cylinder at 1.4. Finger mass is 35 g each with 0.002 armature. Carry follows one continuous cubic Bezier joint path with a minimum-jerk time parameter: 3.15 simulated seconds including a 0.347 s final hold, except the orange cylinder at 4.05 s including a 0.446 s hold. Solver is the unchanged baseline shown in the ledger. Two final runs had no slip, ejection, tunneling, or detached arm bodies.
- Phase 6: PASS — named per-shape strategies are listed below and earlier working grasps were not changed while adding later objects.
- Phase 7: PASS — after replacing the split-stop Move with a continuous curve, the final eight-object configuration cleared twice consecutively with Reset between runs. Both runs ended at measured 8/8; all 16 Move and 16 Release records were present and browser error/warn log was empty. Maximum six-axis joint-speed norm at a Move endpoint fell from the original 0.1356 to 0.0499 rad/s. Maximum object displacement during the final 0.21 s of the box dwell was 11.4 mm in run 1 and rounded to 0 mm in run 2.
- Phase 8: PASS — the MuJoCo-style studio restyle changed only Three.js display code: background/fog, procedural floor texture, lights/shadows, visual materials, visual table legs, and camera. The MJCF block hash and the complete task/control/counter tail hash matched the pre-restyle version. The post-restyle full run ended 8/8 with an empty browser error/warn log, and the narrow-viewport screenshot kept the arm, bench, box, controls, and counter legible.
- Phase 9: NOT COMPLETE — no public repository/static-host target was supplied, so there is no verified HTTPS URL. Incognito/phone verification and the required 30-second screen recording are also still missing. The local pages must not be called a finished public delivery until those human/external steps pass.

## Manual check — Phase 1

Serve the folder over HTTP and open `complex.html`. The status must confirm that the official MuJoCo WASM module initialized and that the MJCF model is loading. A CDN or WASM failure appears as a red `MuJoCo/MJCF failed to load` message; it never starts a scripted substitute.

## Manual check — Phase 2

Open `complex.html` and wait about one second. The status must report `Phase 2 PASS` and cube `z=0.775 m`. Drag to orbit: inspect the base, overlapping joint housings, upper arm, forearm, wrist block, and two claws. The red cube must rest on the white tabletop without sinking or jumping.

## Manual check — Phase 3

Refresh `complex.html` and watch the automatic extended → folded → neutral actuator test. After about six seconds, the status must report `Phase 3 PASS` with the maximum joint error. Orbit during or after the test; every link, wrist housing, and both claws must remain visibly joined.

## Manual check — Phase 4 cube gate

After the automatic pose test, press **Start** and wait for all seven phases. The status must report `Phase 4 PASS` and the counter must read 1/1. Press **Reset** to restore the cube, then press **Step** repeatedly to inspect one phase at a time. During lift, the cube must be visibly off the table between both claws; during release, it must fall without any parenting or scripted transform.

## Object list and grasp notes

The measured clearing order is red cube → grey bar → purple arch → orange cylinder → red rectangular block → blue arch → black pen → tape roll. The rear stack is cleared top-down so its three free bodies do not start by supporting an object that is picked first.

- Red cube, grey bar, red block, pen: top-down flat-face pinch.
- Purple and blue arches: top-down pinch across the two outside legs; the bridge remains part of the same free body.
- Orange cylinder: top-down bilateral side pinch with object sliding friction 1.4 and a 4.05-second two-segment Move, of which the final 0.446 s holds still above the box. Friction 0.9 ejected it; 1.2 and 1.4 at the original 1.8-second Move each passed once but failed repeatability. The slower carry was the stable final lever.
- Tape roll: low bilateral pinch on a flat transparent-cylinder stand-in. The pads reach the table plane and close only to 0.04, avoiding excessive squeeze on the round surface.

## MJCF and render mapping

The worldbody contains a fixed tabletop and five fixed box geoms. The box has no body, joint, free joint, or dynamic mass: its floor and four walls are static world geoms, so contact cannot move or shake the box. The arm is a fixed base followed by six nested hinge-jointed bodies, then two finger bodies with opposing slide joints and position actuators. Every prop has a free joint and one or more matching primitive geoms. On each frame MuJoCo steps up to 1/60 simulated second; `xpos` and `xquat` are copied to one Three.js group per body beneath a single Z-up-to-Y-up root. Visual geoms are local children of those groups; script never writes a prop transform.

## Motion smoothing and box insertion

- Every phase uses minimum-jerk timing `10t^3 - 15t^4 + 6t^5`, giving zero commanded velocity and acceleration at the overall endpoints. This replaced cosine easing, whose acceleration was nonzero at the endpoints.
- Normal Move lasts 3.15 s: 2.804 s of motion plus a 0.347 s target hold. The orange cylinder uses 4.05 s: 3.605 s of motion plus a 0.446 s hold.
- The moving portion is one cubic Bezier joint path from the grasp pose toward the box, pulled through the approach target `(-0.72, 0.42, 1.56)` and ending at `(-0.92, 0.68, 1.46)`. There is no full stop at the intermediate target; the final part naturally decelerates before the hold.
- Reach is 1.3 s, Lower is 1.0 s, and Return is 1.3 s. Reach/Lower retain their second-half target hold so the arm is settled before grasping; removing that hold caused a purple-arch failure and was rejected. The temporary 1.8/1.4/1.8 s version passed physics but looked visibly clumsy and was discarded.
- Release lasts 1.4 s. The claws open with minimum-jerk easing during the first 28% (0.392 s), then the wrist holds for 1.008 s before Return so the object can fall and settle under gravity.
- The automatic extended → folded → neutral chain check now interpolates each target with the same minimum-jerk curve over the first 75% of the stage, then holds for the final 25%; it no longer sends discontinuous pose steps at page load.
- Final hinge damping values are j0=4.2, j1=4.2, j2=3, j3=2, j4=1.5, j5=1.2. The six arm position actuators additionally use `dampratio=0.5`; gains remain kp=260, 260, 230, 160, 110, 110. `dampratio=1.0` cleared 8/8 but was rejected because it lagged into the next phase and produced a 0.1239 rad/s Move endpoint.

## Studio visual treatment

- Background and fog use deep simulator blue `#18344d`, with linear fog from 4.8 m to 11.5 m, matching the supplied visual reference.
- The floor is a self-contained procedural 8×8 blue checker texture with fine pale-blue grid lines, repeated across a 12 m square plane; no external texture file is required.
- Lighting is one cool-white directional key at intensity 2.8 with 2048² PCF soft shadows and one blue hemisphere fill at intensity 1.15. ACES filmic tone mapping uses exposure 1.1.
- Arm visuals use warm-white matte-metal shells with near-black circular joint caps. The gripper uses white carriers and fingers with black rubber contact pads. Props retain lower-saturation colors. These are render materials only; MJCF geom colors and physics parameters are unchanged.
- The supplied-reference arm shell uses rounded white box links, compact white joint housings, paired near-black circular side caps, silver center bolts, a slimmer layered pedestal, and a compact white two-finger gripper with black rubber pads. These meshes remain children of the existing MuJoCo body groups and add no collision geometry, mass, joints, or actuators.
- The visual bench has a 12 cm thick matte top and four dark metal visual legs. The added legs have no MuJoCo collision geoms and cannot affect the task.
- Default camera: 40° field of view at `(3.15, 2.05, -3.65)`, aimed at `(-0.28, 0.84, 0.05)` so the white arm is the subject while the bench and box remain readable.

## Simplifications

The UR-style arm and household items are geometry stand-ins, not CAD models. The tape roll is a transparent flattened cylinder rather than a hollow torus collider. Joint targets come from a two-link analytic wrist aim and a scripted finite-state task plan; this is actuator-driven MuJoCo contact physics, not a learned policy, IK library, or hardware controller.

## Manual check — Phases 5 and 6

Refresh `complex.html`, wait for the pose self-check, and press **Start**. Watch each status label and the counter. Every object must visibly rise with the gripper, travel over the box, and fall only after the claws open. The final status must confirm that all eight objects were grasped by contact friction and measured inside the box, and the counter must read 8/8.

## Phase 7 and 8 verification record

- Smoothing final run 1: PASS, measured 8/8, no browser error/warn.
- Reset followed by smoothing final run 2: PASS, measured 8/8, no browser error/warn.
- Across the latest two runs: all eight object names appeared in both Move sets; maximum Move endpoint joint-speed norm 0.0499 rad/s, down 63.2% from the original 0.1356 rad/s; maximum late-dwell object displacement was 11.4 mm in run 1 and rounded to 0 mm in run 2.
- CSS-only polish followed by run 3: PASS, measured 8/8, no browser error/warn.
- MuJoCo studio visual-only restyle: PASS, measured 8/8, no browser error/warn; MJCF and task/control/counter hashes unchanged.
- Industrial arm-shell restyle: PASS, measured 8/8, no browser error/warn; neutral and final poses kept every visual arm section connected, and MJCF plus task/control/counter hashes remained unchanged.
- Supplied-reference visual match: PASS, measured 8/8, no browser error/warn; blue grid/fog, white shells, black joint caps, lower camera, and long cast shadow were verified in initial and final screenshots.
- Rounded reference-arm replacement: PASS on a fresh page load, measured 8/8 with 0 mm late displacement and no browser error/warn. The MJCF and the complete task/control/counter code hashes are unchanged from the pre-replacement backup. Two earlier Reset-timing attempts did not clear every object, so they are not reported as successful runs; the accepted verification is the independent fresh-load run.
- Wobble correction final runs 1 and 2: PASS at measured 8/8 with 0 mm late displacement and no browser error/warn. With actuator `dampratio=0.5` and minimum-jerk startup poses, the maximum Move endpoint joint-speed norm was 0.0331 rad/s versus the prior 0.0499 rad/s; seven of eight objects ended Move at 0.0078 rad/s or lower.
- Scripted fallback regression after physics work: PASS, `Self-test PASS` and measured 1/1.

Manual check: open `complex.html`, wait for the automatic connected-chain pose check, press **Start**, and wait for 8/8. Press **Reset** and repeat without reloading. Orbit during the run and inspect the fully extended arm, folded arm, wrist, and two claws for gaps.

## Phase 9 handoff

Publish `index.html`, `complex.html`, `README-index.md`, and this README together on any HTTPS static host. Open both public pages in a private/incognito window and on a phone; for `complex.html`, confirm the WASM status does not show an error and run to 8/8. Record one full run for about 30 seconds (or the complete run if longer), keeping the controls, live phase status, arm, objects, box, and final 8/8 counter visible.

Delivery note draft:

> This project provides a scripted single-cube fallback page and a separate MuJoCo WebAssembly page that clears eight tabletop objects with actuator-driven contact physics. The UR-style arm and household props are primitive-geometry stand-ins, and the task plan is scripted rather than learned. Next, publish the static files over HTTPS, verify the public link in incognito and on a phone, and attach a screen recording of a complete 8/8 run.

## Backup note

At project start the folder contained only `SKILL.md`; there was no existing `complex.html`, `mujoco.html`, or `index.html`, so no original HTML existed to copy. After the first known-good versions were created, `complex.backup.html` and `index.backup.html` were maintained before later edits. No `mujoco.html` was created because the requested physics deliverable is `complex.html`.
