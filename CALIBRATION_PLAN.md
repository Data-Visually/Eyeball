# Calibration Quality Plan — making the cursor line up

Research + plan for closing the gap between where you look and where the
cursor lands. Ordered by expected payoff per effort. The playground is the
proving ground; anything that wins graduates to `index.html`.

**Status:** Phase 0 and Phase 1 are built in the playground — the Test
mode (held-out error map), basis families incl. cross terms, held-out-dot
CV with 🧪 Auto-tune, and robust per-dot trimming + fixation gate. On the
synthetic testbench, CV correctly ranked basis families against held-out
error and auto-tune improved held-out accuracy; the cross basis needs
more distinct targets than 9 dots to shine, which is exactly Phase 3's
job. Awaiting real-hardware measurement.

## Where we are

- 9-point tap calibration, ~1 s of frames per dot (~200+ samples).
- Features: iris offsets (per-eye or averaged), eyeLook* blendshapes,
  z-proxy head pose, facing-direction head pose, iris size; 22-dim in the
  main app.
- Mapping: standardize → quadratic basis → ridge (fixed λ = 1e-3·n) →
  per-axis normal equations.
- Runtime: clamp + speed-adaptive smoothing; continuous tap-recalibration
  in the playground; head-sweep collection mode.

Reference points from the literature: WebGazer-style click-mapping lands
around 4° of visual error; classic 9-point polynomial calibration on good
features reaches ~1.4°; smooth-pursuit calibration has hit ~0.3° offsets.
So there is likely a 2–5× accuracy win available before we hit the floor
imposed by webcam resolution.

## Phase 0 — Measure first (foundation, do before everything)

**Accuracy Test mode in the playground.** A grid of ~12 test dots (offset
from the calibration positions, so it's a true held-out test): look at
each, it samples ~0.5 s, records predicted-vs-true, then renders an error
map — arrows from each dot to its mean prediction, per-dot error in % of
screen and (using iris-size distance estimate) approximate degrees, plus
overall mean/worst. One tap to run after any calibration.

Why first: every phase below is a hypothesis. Without held-out
measurement we can't tell a real win from placebo, and training RMS is
misleading (it flatters overfitting). This also finally answers "how far
off are we, in numbers?"

## Phase 1 — Better fitting on the data we already collect (quick wins)

No new calibration UX; the playground can refit stored samples, so these
are testable on one recorded session.

1. **Interaction terms in the basis.** Today's basis squares two features
   and has no cross terms. But eye-in-head and head pose combine
   *multiplicatively* (a rotation composition), so the model that maps
   `eye + head` additively fundamentally cannot line up at poses between
   calibration dots. Add: `irisX·irisY`, and eye×head products
   (`irisX·headFx`, `irisY·headFy`, blend×head). Keep the basis
   configurable in the playground so A/B is one toggle.
2. **Leave-one-dot-out cross-validation.** Frames within one dot are
   near-duplicates, so random CV lies. Hold out entire dots, fit on 8,
   test on the 9th, rotate. Use it to (a) pick λ from a grid instead of
   the hardcoded 1e-3·n, and (b) auto-pick the best feature config —
   the playground's config toggles become a searchable space ("Auto"
   button that reports the winner).
3. **Robust per-dot sampling.** Currently every frame in the window
   counts equally. Add: drop frames with landmark velocity above a
   threshold (saccades/jiggle), trim outliers per dot by median absolute
   deviation, and reject dots whose feature variance stays high (user
   wasn't fixating — re-prompt that dot instead of poisoning the fit).

## Phase 2 — Better features (the head-pose interaction, done right)

4. **Port the full per-eye feature set to the playground.** ✅ DONE — lid
   balance + openness are now the "Eyelid signals" toggle (default on),
   and the blink threshold was lowered so looking-down frames aren't
   discarded. This directly targets the field report "the lower I look,
   the worse it is". Continuous taps also got ×12 weighting (Phase 4's
   item 11 brought forward) after the field report that taps did nothing.
5. **Head-normalized eye features ("data normalization").** The
   appearance-gaze literature's standard trick: use the facial
   transformation matrix to rotate iris offsets into *head* coordinates,
   so the eye features mean the same thing at every head pose. Then the
   regression learns `screen = g(head pose, eye-in-head)` instead of
   entangling both in camera space. This is the principled version of
   what the head-sweep calibration is trying to teach the model by brute
   force — and it should make the sweep data dramatically more effective.
6. **Geometric gaze ray as a feature (hybrid model).** Build an explicit
   3-D gaze direction: eyeball-center estimate (from eye-corner landmarks
   + fixed offset behind the plane) → iris center → ray → intersect the
   screen plane. It needs per-user constants (kappa angle, screen pose),
   which the ridge regression absorbs — feed the intersection point in as
   two features. Geometric models extrapolate across head pose far better
   than pure regression interpolation; the regression then only fixes up
   residuals. This is the biggest single idea; slot it after 5 because it
   reuses the same head-coordinate machinery.

## Phase 3 — Better calibration data

7. **13-point pattern option.** 9 points leaves the quadratic model's
   corners under-constrained; add mid-edge + inner-ring dots. Cheap to
   try; measure whether the extra 20 s pays for itself.
8. **Smooth-pursuit calibration mode.** A dot glides a Lissajous/rounded
   rect path for ~25 s while you follow it; every frame is a training
   sample with a *unique* target — hundreds of distinct positions vs 9.
   The literature reports excellent offsets. One correction is essential:
   pursuit tracks the target with ~100–150 ms lag, so labels must be
   time-shifted (fit the lag as the value that minimizes CV error).
9. **Head-sweep + normalization combo.** Once 5 lands, re-test the
   sweep: eyes-on-dot head rotation becomes exactly the data that
   teaches the head-compensation term.

## Phase 4 — Staying calibrated at runtime

10. **Quality-gated continuous samples.** Only accept a tap as a training
    label when the eye signal was in a stable fixation for the preceding
    ~300 ms (variance gate) — clicks made without looking (typing,
    muscle-memory taps) are the poison WebGazer suffers from.
11. **Recency weighting.** Weight continuous samples by age (exponential
    decay, floor at seed weight) so the model tracks posture drift
    without forgetting the calibrated geometry.
12. **Fixation-aware cursor.** During detected fixation, freeze-then-ease
    the cursor instead of micro-wandering; it reads as "lined up" even at
    equal numeric error — perceived accuracy is part of accuracy.

## Expected trajectory

Phase 1 alone typically buys a solid fraction (cross-terms + tuned λ +
outlier trimming are the classic 30–50 % RMS cuts); Phase 2 is the
structural fix for "calibrate here, look from there"; Phase 3's pursuit
mode is how we approach the webcam floor; Phase 4 keeps it there over a
session. Measured at every step by Phase 0's test mode — numbers, not
vibes.
