# Eyeball 👁️ — "Look and Tap" Eye-Tracking Prototype

A pure HTML/CSS/JavaScript web prototype that tracks your irises in real time
using the front-facing camera and [MediaPipe Face Landmarker
(tasks-vision)](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker/web_js).
No build step, no native app, no Xcode — just one `index.html`.

## What it does

1. Requests the **front camera** and shows it full-screen (mirrored, like a selfie).
2. Loads the **MediaPipe Face Landmarker** model (478 3D landmarks, iris refinement included) from CDN.
3. Runs live detection every camera frame and draws **bright green rings/dots
   on both iris centers** on a canvas overlaid on the video.
4. Logs the **right iris center coordinates to the console every 60 frames**
   (normalized + pixel coordinates + relative z-depth).
5. **Gaze calibration**: a 9-point routine — look at each yellow dot and tap;
   the app samples your iris-vs-eye-corner geometry for ~1 s per dot and fits
   a quadratic regression from gaze features to screen coordinates.
6. **Look and Tap**: after calibrating, a green gaze cursor follows your eyes
   across a grid of app tiles. The tile you're looking at lights up; tapping
   **anywhere** on the screen selects it (with haptic feedback where
   supported). Look chooses, tap confirms.

## Running it

The camera API (`getUserMedia`) only works on a **secure origin** — `https://`
or `localhost`.

### On your desktop (quick check)

```bash
# from the repo root — either works:
python3 -m http.server 8000
# or
npx serve .
```

Open <http://localhost:8000>.

### On your phone (the real target)

Your phone can't use your laptop's `localhost`, so you need HTTPS. Easiest options:

- **GitHub Pages** (automated, already live): the site is served from the
  `gh-pages` branch at **<https://data-visually.github.io/Eyeball/>**.
  A workflow (`.github/workflows/pages.yml`) force-syncs every push on the
  development branch to `gh-pages`, so the live site always matches the
  latest commit.
- **A tunnel**: `npx ngrok http 8000` (or `cloudflared tunnel --url http://localhost:8000`)
  and open the generated `https://` URL on your phone.

Then tap **"Tap to start eye tracking"**, allow camera access, and you should
see green dots locked onto your irises. Tap **Calibrate gaze**, follow the
9 dots, and the Look-and-Tap tile grid appears. Open the remote-debug console
(chrome://inspect for Android Chrome, Safari → Develop menu for iOS) to see
the periodic right-iris coordinate logs.

## Implementation notes

- **`playsinline` / `webkit-playsinline` / `muted` / `autoplay`** on the
  `<video>` are required for iOS Safari — without them the video opens the
  native fullscreen player and the overlay never renders.
- **Running mode**: the JS `tasks-vision` API exposes `IMAGE` and `VIDEO`
  modes; `VIDEO` + `detectForVideo()` per frame is the web equivalent of the
  mobile SDKs' `LIVE_STREAM` mode.
- **Coordinate alignment**: the canvas bitmap is set to the camera's native
  resolution and both video and canvas share identical `object-fit: cover` +
  `scaleX(-1)` CSS, so drawn landmarks stay pixel-registered with the video on
  any screen aspect ratio.
- **Iris landmark indices** (points 468–477 of the 478-point mesh):
  - Right iris: `468` (center), `469–472` (ring)
  - Left iris: `473` (center), `474–477` (ring)
- **GPU→CPU fallback**: model initialization retries with the CPU delegate if
  the GPU delegate fails (some mobile WebGL contexts do).
- **Frame gating**: inference only runs when a *new* camera frame is available
  (`requestVideoFrameCallback` where supported), saving battery and keeping
  `detectForVideo` timestamps monotonic.

## How the gaze estimation works

1. **Feature extraction** (per frame, 22 dimensions):
   - *Iris geometry, per eye* (4 features × 2 eyes): iris offset from the
     eye-corner midpoint (x and y), iris-to-eyelid balance (a second,
     independent vertical-gaze signal), and eye openness (lids droop looking
     down, widen looking up) — all normalized by eye width. Frames where the
     eyelid gap drops below 10 % of eye width are treated as true blinks and
     skipped; a single winking eye is mirrored from the open one.
   - *Blendshape gaze scores* (8 features): MediaPipe's learned
     `eyeLookIn/Out/Up/Down` blendshapes for each eye, fed individually so
     the regression weighs each one itself.
   - *Head pose & position* (6 features): cheek depth difference (yaw),
     forehead-vs-chin depth difference (pitch), roll angle of the eye line,
     nose-tip position, and inter-ocular distance (distance to screen).
     Head terms are essential: people naturally turn their head partway
     toward what they look at, so a mapping trained on eyeball rotation
     alone systematically undershoots at runtime ("the cursor won't reach
     the edges").
2. **Calibration**: 9 on-screen points. For each, the user looks at the dot
   and taps; after a 250 ms settle (tap jiggle), ~1 s of frames are recorded —
   every frame is a training sample (~200+ total).
3. **Mapping**: features are standardized, expanded with quadratic terms on
   the four iris offsets (27 basis terms total), and ridge-regressed per axis
   (normal equations + Gauss-Jordan). The training fit error is logged to the
   console.
4. **Runtime**: predicted gaze is clamped to the viewport and smoothed with a
   speed-adaptive filter — heavy while fixating (steady cursor), light during
   saccades (fast catch-up).

Known limitations (it's a prototype): large posture changes after calibrating
still degrade accuracy — recalibrate via the ↻ button. Vertical resolution is
inherently weaker than horizontal on front cameras.
