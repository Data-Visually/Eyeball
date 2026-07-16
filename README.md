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

- **GitHub Pages**: enable Pages for this repo → open the URL on your phone. Zero setup.
- **A tunnel**: `npx ngrok http 8000` (or `cloudflared tunnel --url http://localhost:8000`)
  and open the generated `https://` URL on your phone.

Then tap **"Tap to start eye tracking"**, allow camera access, and you should
see green dots locked onto your irises. Open the remote-debug console
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

## Next steps toward "Look and Tap"

The iris landmarks this prototype extracts are the input for gaze estimation:
compare iris centers against eye-corner landmarks (33/133 right, 362/263 left)
to derive a normalized gaze vector, calibrate with a few on-screen tap targets,
and map gaze to screen regions.
