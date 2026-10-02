# Cricket wicket tracker

Personal project: a phone-camera web app for casual/club cricket that decides whether a white ball would have hit the stumps (LBW-style "hitting / missing / umpire's call").
First trials planned at PA nets, then a pitch to Jersey Cricket.

## How it's built
- Everything lives in a single `index.html` (HTML + CSS + JS inline). No build step, no framework.
- Hosted on Netlify, auto-deployed from this repo's main branch. Publish directory is the repo root.
- Camera needs https (Netlify provides it). It does not work inside Claude artifact previews.

## How it works
- Camera setup: phone fixed 2–4 m behind the stumps, about stump height, looking down the pitch, 1× lens. The phone must not move after setup.
- User drags a virtual wicket from the base of the middle stump to the top of the stumps. Stumps are 0.711 m high and 0.2286 m wide.
- Distance to the stumps = focal length (px) × 0.711 / stump height (px). The default focal length assumes a 65° horizontal field of view on the long side. Entering a tape-measured distance calibrates the focal length (`kFocal`).
- Ball detection runs on frames downscaled to 640 px on the long side. A candidate is a pixel that is both white (bright, low colour spread) and moving (frame difference). Each candidate is then grown over the white mask to measure the full ball. Regions over the max size are rejected as shirts, pads or sightscreens.
- Tracking uses nearest-neighbour matching with constant-velocity prediction and a size-ratio check.
- Ball distance from camera = focal × 0.072 / ball diameter (px).
- Judging: the last ≤10 points are back-projected to 3D, a straight line is fitted against time, and the path is solved for the moment it reaches the stumps' distance. That point is tested against the stump rectangle, with the ball radius as the umpire's-call margin. If the ball was stopped short (pad/bat), the path is projected on.
- Settings, wicket and zone are saved in localStorage key `wkt-v2`.

## Status
- Tested only on a synthetic camera feed (headless Chrome with a fake y4m camera): distance 3.3 m vs 3.31 m true, and correct hitting/missing calls. Not yet tested on real footage.
- Expected real-world issues: white pads, shoes and clothing as false positives; small ball size at distance; dusk or floodlight lighting; camera shake.

## Working conventions
- Keep it a single self-contained file unless there's a strong reason to split it.
- Test changes before pushing. A fake-camera test (Playwright + `--use-file-for-fake-video-capture`) worked well.
- Commit with clear messages and push to main. Netlify deploys automatically.
