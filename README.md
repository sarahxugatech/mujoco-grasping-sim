# MuJoCo Grasping Sim

A browser-based robot-arm grasping simulation that recreates a tabletop-clearing task. The scene uses a UR-style six-axis arm, a symmetric two-finger gripper, a workbench, an open box, and primitive-geometry household objects. Everything runs as static files with Three.js and the official MuJoCo WebAssembly build loaded from CDNs; there is no build step or backend.

## Live demo

- Scripted fallback: https://sarahxugatech.github.io/mujoco-grasping-sim/
- MuJoCo physics simulation: https://sarahxugatech.github.io/mujoco-grasping-sim/complex.html

HTTPS hosting is recommended for reliable WebAssembly and ES-module loading.

## Pages

- `index.html` is an intentionally scripted fallback. It runs a single red-cube grasp loop and temporarily parents the cube to the wrist while carrying it. It is not a physics simulation.
- `complex.html` is the multi-object MuJoCo version. The arm and gripper move through actuator targets, each prop is a free MuJoCo body, grasping uses contact friction, and released objects fall under gravity. It does not attach objects to the gripper or script their carried transforms.

The MuJoCo MJCF model is embedded in `complex.html`, so no separate XML file is required.

## Controls

- **Start** runs the complete clearing sequence.
- **Step** advances one phase of the current pick-and-place cycle.
- **Reset** restores the initial scene and counter.
- Drag with the mouse to orbit the camera.

The cycle is reach, lower, close, vertical lift, move over the box, release, and return. An object is counted only when its measured MuJoCo body position is inside the box.

## Objects in the current physics scene

The bottle is not part of this project. The current eight-object sequence is:

1. Red cube
2. Grey bar
3. Purple arch block
4. Orange cylinder
5. Rear red rectangular block
6. Rear blue arch block
7. Black pen
8. Tape roll

The arches are compound free bodies made from three box geoms. The tape roll is represented by a transparent flattened cylinder rather than a hollow torus collider.

## Physics parameters

The current `complex.html` uses MuJoCo WASM 3.17.0 with these tuned values:

| Parameter | Value |
|---|---|
| Simulation step | 0.0025 s |
| Gravity | 9.81 m/s² downward |
| Integrator / solver / cone | `implicitfast` / `Newton` / `elliptic` |
| Solver settings | `impratio=10`, `iterations=20`, `ls_iterations=20` |
| Table and fixed-box friction | `0.8 0.01 0.0002` |
| Finger-pad friction | `1.2 0.015 0.0002` |
| Normal object friction | `0.9 0.01 0.0002` |
| Orange-cylinder friction | `1.4 0.01 0.0002` |
| Finger mass / slide-joint armature | 0.035 kg each / 0.002 |
| Arm hinge damping, j0–j5 | 4.2, 4.2, 3.0, 2.0, 1.5, 1.2 |
| Arm actuator gains, a0–a5 | 260, 260, 230, 160, 110, 110 |
| Arm actuator damping ratio | 0.5 |
| Gripper actuator gain / force range | 180 / ±12 N |
| Normal carry duration | 3.15 s, including a 0.347 s final hold |
| Orange-cylinder carry duration | 4.05 s, including a 0.446 s final hold |

Object masses are 0.020 kg for the red cube, 0.015 kg for the grey bar, 0.025 kg for each arch, 0.018 kg for the red block, 0.012 kg for the orange cylinder, 0.010 kg for the pen, and 0.020 kg for the tape roll.

The open cardboard box is made from fixed world geoms, so it cannot move when an object lands. Motion targets use minimum-jerk timing and a continuous cubic Bezier carry path. Release opens the claws for 0.392 s and then holds the wrist for 1.008 s while the object settles.

## Current verification

- The scripted fallback completed its measured single-object run at 1/1.
- The MuJoCo page completed two consecutive Reset-to-8/8 runs using actuator-driven contact physics.
- The accepted final run reported no browser errors or warnings, no detached arm parts, and 0 mm late-settle displacement.

These results describe the current checked-in pages. They are not claims about arbitrary browsers, modified parameters, or real hardware.

## Simplifications

The arm and props are primitive-geometry stand-ins rather than robot CAD and scanned object models. The pick sequence and actuator targets are scripted; there is no learned policy, perception system, or real-robot controller. `index.html` is a visibly labeled animation fallback, while only `complex.html` uses MuJoCo contact physics.

For detailed model structure, grasp notes, tuning history, and manual checks, see `README-complex.md`. For the fallback implementation, see `README-index.md`.
