# Stereo 3D (layered stereo)

CannonBall DX can render OutRun in stereoscopic 3D, in the spirit of the
"layered" 3D used by later official 3D re-releases of the game. Scenery, traffic
and the road recede into the screen, the Ferrari and the HUD sit on the screen
plane, and objects closer than the car can come slightly out of the screen.

This is phase 1 of the PC-VR work: the engine renders both eyes and packs them
into one frame. That frame works today on 3D TVs/monitors, with cross-eye free
viewing, and in VR through desktop/cinema apps that offer a 3D side-by-side
screen mode. A native OpenXR big-screen renderer is the next phase.

## Quick start

1. Set the engine resolution to **2x or higher** (each eye is packed at half
   resolution, so 2x keeps full arcade detail per eye).
2. Turn off CRT / NTSC filtering, which would blur the two eyes into each other.
3. Press **F12** to cycle the stereo output mode. **Shift+F12** cycles the depth.

## Modes (`video.stereo`)

| Value | Mode | Use with |
| ----- | ---- | -------- |
| 0 | Off | Normal 2D |
| 1 | Half side-by-side, left eye on the left | 3D TVs, VR desktop/cinema apps in 3D SBS mode |
| 2 | Half side-by-side, eyes swapped | Cross-eye free viewing |
| 3 | Half top-and-bottom, left eye on top | 3D TVs / players that prefer TAB |

## Settings in `config.xml`

| Setting | Range | Default | Meaning |
| ------- | ----- | ------- | ------- |
| `video.stereo` | 0-3 | 0 | Output mode (table above) |
| `video.stereo_depth` | 0-48 | 10 | Separation of the far distance, in native (320-wide) pixels. Higher = deeper. |
| `video.stereo_popout` | 0-200 | 50 | How far in front of the screen objects may come, in % of the car's depth. 0 = nothing pops out. |

Start with the default depth. On a large virtual screen, values around 8-12
feel natural; small physical screens can take more.

## How it works

Every layer of the OutRun video hardware already carries depth information:

- **Sprites**: the engine's zoom index. Apparent size, and so inverse depth, is
  proportional to `47 * zoom + 111`; zoom `0x7F` is the player's Ferrari. Each
  hardware sprite is tagged with its zoom in an otherwise unused word of its
  sprite entry. The Ferrari's shadow and passengers use the car's depth.
- **Road**: each scanline selects a pre-rendered road ROM row. The width of the
  road in that row is proportional to inverse depth on flat road, hills and
  splits alike. Widths are measured from the ROM once at start-up.
- **Calibration**: scenery and traffic standing on the road give, every frame,
  pairs of (scanline, zoom). Matching the road's depth to those sprites gives
  the road width at the car's depth automatically, so the road, scenery and car
  line up without hand tuning.
- **Tilemaps**: distant backdrop, placed near infinity while the road is on
  screen. Menus, the map and Music Select stay flat.
- **Text / HUD**: always on the screen plane.

Each frame is drawn twice with every element shifted horizontally by half of
its stereo separation (left eye left, right eye right for things behind the
screen), then packed. Because the packing happens on palette indices before the
renderer, every existing scaler and display mode keeps working.

## Reporting depth problems

Run with the environment variable `CANNONBALL_STEREO_DEBUG=1` to log the road
ROM row widths at start-up and the calibration every 120 frames, for example:

```
Stereo: road 0 row widths: fit ok (slope ..., intercept ...), row0=... row255=...
Stereo: ref width 365.2 (fallback 380, 14 samples) inv@112=0.21 inv@160=0.58 inv@223=1.03
```

`inv@223` should be close to 1.0 during normal driving (the road just in front
of the car is at the car's depth). If the road and roadside objects appear to
float relative to each other, include these lines when reporting it.

## Known limitations

- Half-resolution packing point-samples each eye (palette indices cannot be
  blended). Use 2x engine resolution or higher.
- Pixel scalers and CRT/NTSC filters run on the packed frame and can bleed a
  pixel or two across the seam between the eyes.
- Sprites are flat cut-outs placed at depth: by design, like other layered 3D
  versions of the game.
