# Scripted fallback — scope and usage

## Phase 0 decisions

- Deliverable: `index.html` is built and verified first as the guaranteed fallback. It contains one red cube only. Later physics work must not modify it.
- Mode: deadline mode — ship the smallest working fallback before the physics page.
- Gripper: one wrist block with two opposing black claws. Both claws remain attached to the wrist and move symmetrically.
- Arm: one parent-child chain from base through six joints and links to the wrist and both claws. No arm part is positioned as a free scene object.
- Loop: approach → lower → close → lift vertically → move over the box → release → return.
- Honesty: this page is scripted Three.js animation, not contact physics. It may parent the cube to the wrist while carrying because it is explicitly the scripted fallback.
- Controls: Start runs the loop, Step advances one phase, Reset restores the cube and counter, and mouse drag orbits the camera.
- Counter: the cube counts only after its measured world position is inside the box region.

## Phase status

- Phase 0: PASS — scope is recorded and was confirmed in the project prompt.
- Phase 1 fallback gate: PASS — the page loaded from a local static server, Start completed all seven phases, the measured box-region check produced `Cleared 1 / 1 objects`, Reset restored 0/1, and Step advanced one phase.
- Scripted self-test: PASS — phase order, lift motion, release, and measured in-box position passed in the browser.
- Physics parameters: not applicable. This page is intentionally scripted and is never described as MuJoCo physics.

## Manual check

1. Serve this folder over HTTP and open `index.html`.
2. Press **Start**. Watch the red cube follow approach → lower → close → vertical lift → box move → release → return; confirm the status reads `Self-test PASS` and the counter reads `Cleared 1 / 1 objects`.
3. Press **Reset** and confirm the cube returns to the tabletop and the counter returns to 0/1.
4. Press **Step** and confirm exactly one named phase runs.
5. Orbit the camera and inspect the base, six joint groups, wrist, and both claws; all are one Three.js parent-child chain.

## Simplifications

The arm uses analytic two-link aiming plus three visual wrist joints. The cube is parented to the tool only on this explicitly scripted fallback page, and its fall is animated. There is no collision solver, contact friction, or rigid-body dynamics here; those belong only to `complex.html`.
