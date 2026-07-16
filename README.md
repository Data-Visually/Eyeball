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

- **GitHub Pages** (automated): this repo ships a workflow
  (`.github/workflows/pages.yml`) that deploys the site to Pages on every
  push. Once the action has run, open
  `https://data-visually.github.io/Eyeball/` on your phone.
  ⚠️ **This repo is currently private**, and GitHub Pages on private repos
  requires a paid GitHub plan — so the deploy step fails until you either
  (a) make the repo public (Settings → General → Danger Zone → Change
  visibility), or (b) upgrade the plan. After that, re-run the
  "Deploy to GitHub Pages" action once (Actions tab → Re-run) and it will
  self-enable Pages and deploy on every push from then on.
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

1. **Feature extraction** (per eye, per frame): the iris center's offset from
   the midpoint of the eye corners (right eye: 33/133, left eye: 362/263),
   normalized by eye width. Normalizing by width — not eyelid gap — keeps the
   feature stable through blinks; frames where the eyelid gap drops below 12 %
   of eye width are treated as blinks and skipped. Both eyes are averaged.
2. **Calibration**: 9 on-screen points. For each, the user looks at the dot
   and taps; ~1 s of features are averaged into one sample.
3. **Mapping**: per-axis least-squares fit of a quadratic polynomial basis
   `[1, fx, fy, fx·fy, fx², fy²]` (normal equations + Gaussian elimination,
   tiny ridge term for stability). 9 samples over 6 unknowns.
4. **Runtime**: predicted gaze is clamped to the viewport and smoothed with an
   exponential moving average (α = 0.3) before driving the cursor and tile
   hit-testing.

Known limitations (it's a prototype): accuracy degrades if you move your head
significantly after calibrating — recalibrate via the ↻ button. Vertical gaze
resolution is inherently weaker than horizontal on front cameras. A production
version would fold in the head-pose transformation matrix and a Kalman/One-Euro
filter.
