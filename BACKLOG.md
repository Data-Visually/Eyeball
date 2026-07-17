# Eyeball — Backlog / things to look at

A running list of deferred improvements and open questions, newest concerns
first. Not commitments — a place so nothing gets lost between sessions.

## Reach / hands
- [x] **First-person perspective** — DONE. Hands render from MediaPipe *world*
      landmarks (true 3D) rotated 180° about the vertical ("stepping through
      the screen"): right hand stays right, palm turns to face into the scene.
      🔃 Flip-hands toggle swaps the 180° if needed. Confirmed correct on real
      hardware.
- [ ] **Hand detail** — the hands are still primitives (spheres + capsules +
      palm blob), a chunky "robot" look. Real fix: a **rigged glTF/VRM hand
      mesh** skinned to the landmarks (proper anatomy, skin). Bigger job —
      sourcing/hosting a model + bone mapping. Do this when looks matter.
- [ ] **Grab feel tuning** — spring stiffness, throw strength, grab radius,
      pinch thresholds. Needs real-hand feel testing.
- [ ] **Reach depth** — hand shape now has real world-landmark depth, but the
      hand's overall distance sits on a fixed plane; grab is screen-space.
      Consider mapping hand distance (size) to depth so reaching "further in"
      pushes deeper toward the objects.

## Interaction / features
- [ ] **Wand** — track the hand/fingertip as a wand tip, buffer the recent
      trajectory, match a swish/flick gesture → fire a spell (projectile /
      particle burst). Reuses hand tracking; no new model. Likely next build.
- [ ] **Physics** — gravity + collisions for throwable objects (cannon-es or a
      light custom integrator), so objects fall, stack, and bounce properly.
- [ ] **Rigged avatar (VRM)** — swap the pose-driven capsule figure for a real
      rigged character; face blendshapes + hands could drive it too.

## 3D window / corridor
- [ ] **Presets** — one-tap Realistic / Playful / Extreme (intensity + depth).
- [ ] **Metric calibration** — a known-distance reference step so the "1× =
      life-like" scale is exact rather than approximate.

## Cross-cutting
- [ ] **Performance** — running 3 MediaPipe models + WebGL is heavy; profile
      framerate on real devices; consider disabling unused models per mode.
- [ ] **Scanner project** (separate) — depth-camera point cloud vs monocular
      depth estimation; MediaPipe as semantic registration on measured geometry.
