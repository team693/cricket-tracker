# Cricket wicket tracker

Personal project: a phone-camera web app for casual/club cricket that decides whether a white ball would have hit the stumps (LBW-style "hitting / missing / umpire's call"), and calls wides.
First trials planned at PA nets, then a pitch to Jersey Cricket.

## How it's built
- Everything lives in a single `index.html` (HTML + CSS + JS inline). No build step, no framework.
- Hosted on Netlify (project "crickettracker", https://cricketracker.netlify.app), auto-deployed from this repo's main branch. Publish directory is the repo root.
- Camera needs https (Netlify provides it). It does not work inside Claude artifact previews.
- Live camera only, by design: there is no video-file option (removed at the user's request). For testing, use Chrome's fake camera with a y4m file.

## How it works
- Camera setup: phone on a stand at the **bowler's end**, behind the bowler's stumps where the umpire stands, looking down the pitch at the batter's stumps. The user enters how far behind the bowler's stumps the phone is (default 2 m); far stumps = 20.12 m + that.
- Setup is kept deliberately simple; tools sit directly under the picture:
  - Draw wicket: tap the far stumps (picture zooms 3× around the tap via a CSS transform on `#view`), then drag a box around them. Box height sets the wicket; it sits on the box's bottom edge. Stumps are 0.711 m high and 0.2286 m wide.
  - Wide markers: tap the batter's crease (zooms in), then tap the wide line on each side.
  - Optional extras in Detection tuning: Learn ball colour (tap a still ball) and Detection zone.
- Focal length comes from the drawn wicket and the known distance (`focalProc = stumpsDist × wicket px height / 0.711`), so any camera zoom works. Where the browser supports it (Chrome on Android), a Camera zoom select applies `zoom` constraints; changing zoom clears the wicket. Size limits (minA/maxA) are set from this geometry when the wicket is drawn.
- Ball detection runs on frames downscaled to 960 px on the long side (camera requested at 1920×1080). A candidate is a pixel that is both white (bright, low colour spread) and moving. Each candidate is then grown over the white mask to measure the full ball. Regions over the max size are rejected.
- Shake tolerance: each frame the global camera shift (±8 px) is estimated against the previous frame (coarse search every 2 px on a 16 px grid, then ±1 px refine on an 8 px grid), and a white pixel only counts as moving if it differs from all 3×3 neighbours of the shifted previous frame. Frames where over 40% of the picture changes are skipped.
- Drift following: when the wicket is drawn, the grey frame is kept as a reference. Every 15 frames the live frame is matched against it (±24 px, brightness-change tolerant); once two checks agree, an offset `off` moves the wicket and wide markers. They are stored in reference coordinates and saved re-based (`wkNow()`/`widesNow()`).
- Frame timing uses the camera's `captureTime` from requestVideoFrameCallback when available. A screen Wake Lock is held while the camera runs, and a camera track that ends shows a restart message.
- Per-frame cost is about 13–17 ms at 960×540 in headless Chrome (the moving-pixel list means candidate search only visits moving white pixels). `window.__wkt().perf` has the timing breakdown.
- Tracking: nearest-neighbour matching with constant-velocity prediction and a size-ratio check; a candidate at a similar speed in any direction is also accepted so tracks survive the bounce.
- Only deliveries count: the ball must shrink by ≥15% along the track (travelling away towards the far wicket), a wicket must be drawn, and the projected path must head towards the stumps (±8 wicket widths) or, with wide markers, within half a marker-gap outside them. Everything else is silently ignored. The live pink trail is only drawn for tracks that look like a delivery.
- Judging: depth Z(t) is a straight line in time fitted to per-frame size depth (focal × 0.072 / diameter), weighted by diameter⁴ so close sightings dominate (speed must be 5–60 m/s). The bounce is the lowest image point; the post-bounce points (or pre-bounce if too few) are back-projected and fitted with X linear and Y linear + gravity. Solve for Z = far stumps: test against the wicket rectangle with the ball radius as the umpire's-call margin. Wides: same path at the batter's popping crease (1.22 m in front of the far stumps) compared with the markers.
- Settings, wicket, wide markers and zone are saved in localStorage key `wkt-v3` (v2 was the old camera-behind-the-batter's-stumps layout).
- `window.__wkt()` exposes deliveries/tracks for automated tests.

## Status
- Tested only on synthetic camera feeds (headless Chrome with a fake y4m camera, scene rendered from the bowler's end at 960×540 "2× zoom", gravity, bounce, ±2 px shake, textured ground): speed 29–31 m/s vs 30 true, impact within a few cm sideways and ~5–10 cm in height; correct hitting/missing/wide; balls thrown back or rolling across are ignored. Not yet tested on real footage.
- Expected real-world issues: the ball is only a few pixels across near the far wicket (use 2× zoom); white pads, shoes and clothing (bowler's run-up, batter) as false positives; dusk or floodlight lighting; processing speed on older phones (about 30 fps in headless Chrome after the speed-ups).

## Working conventions
- Keep it a single self-contained file unless there's a strong reason to split it.
- Test changes before pushing. A fake-camera test (Playwright + `--use-file-for-fake-video-capture`) worked well; generate y4m clips of the scene with a small Node script.
- Commit with clear messages and push to main. Netlify deploys automatically.
