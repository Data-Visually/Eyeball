# Eyeball — Backlog / things to look at

A running list of deferred improvements and open questions, newest concerns
first. Not commitments — a place so nothing gets lost between sessions.

## Reach / hands
- [ ] **Hand detail** — the current hands are a placeholder built from
      primitives (spheres + capsules + a palm blob), which reads as a chunky
      "robot" hand. The real fix is a **rigged glTF/VRM hand mesh** skinned to
      the 21 landmarks: proper anatomy (thumb origin ~5 o'clock, knuckle
      creases, tapered fingers, skin). Bigger job — sourcing/hosting a model +
      bone mapping. Do this when the hands become the focus.
- [ ] **Mirror / first-person feel** — hands are now mirrored (right hand on
      your right). Confirm on real hardware it tracks the correct way; if
      reversed, one-line flip of the `(1 - x)` mapping.
- [ ] **Grab feel tuning** — spring stiffness, throw strength, grab radius,
      pinch thresholds. Needs real-hand feel testing.
- [ ] **True depth in Reach** — hands currently sit on a plane with a small
      per-landmark z wobble; grab is screen-space. Consider real 3D depth from
      hand size / landmark z so reaching "deeper" works.

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
