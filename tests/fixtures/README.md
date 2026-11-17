# Test fixtures

The videos used for tests and benchmarks, and what each one stands for. Every file is stored through
Git LFS (`*.mp4` and `*.gif`, see `.gitattributes`). How they are used (clip profiles, pipeline
stages) is described in `docs/direction.md`.

## Metadata

From ffprobe (2026-09-29).

| ID | File | Resolution | fps | Duration | Frames |
| --- | --- | --- | --- | --- | --- |
| `easy/ball` | `easy/BounchingBall.gif` | 600×450 | 100/9 (11.1) | 0.72 s | 8 |
| `easy/complex` | `easy/Complex.mp4` | 1280×720 | 24 | 10.0 s | 240 |
| `easy/overlap` | `easy/Overlap.mp4` | 384×376 | 30 | 8.6 s | 259 |
| `easy/sequence` | `easy/Sequence.mp4` | 1280×720 | 24 | 10.0 s | 240 |
| `medium/badapple` | `medium/Bad Apple.mp4` | 1440×1080 | 60 | 219.1 s | 13140 |
| `medium/fish-short` | `medium/Don't watch fish spinning anymore.mp4` | 1280×720 | 29.97 | 16.5 s | 491 |
| `medium/fish` | `medium/Fish spinning.mp4` | 1920×1080 | 60 | 178.6 s | 10709 |
| `hard/birdbrain` | `hard/Birdbrain.mp4` | 1920×1080 | 24 | 255.7 s | 6134 |
| `hard/machinelove` | `hard/Machine Love.mp4` | 1920×1080 | 30 | 286.5 s | 8593 |
| `hard/rollback` | `hard/rollback.mp4` | 1920×1080 | 24 | 155.1 s | 3720 |
| `hard/minecraft` | `hard/This is Minecraft.mp4` | 1920×804 | 24000/1001 | 103.5 s | 2481 |

## What each fixture represents

Scene-cut counts are estimates from ffmpeg's `scene` filter at threshold 0.3 on a 320-pixel-wide
downscale.

### Easy: flat motion graphics

Few objects, flat colours, white background. A correct result is a handful of sprites and a short
command list.

| ID | Content | What it tests |
| --- | --- | --- |
| `easy/ball` | 8-frame GIF (content not reviewed when this table was written) | Very short input; GIF decoding |
| `easy/complex` | Two circles that move, overlap with blended colours, fade, and one that grows until it fills the screen | Semi-transparent objects whose overlap is a blended colour, not a third object; choosing the largest, sharpest view of an object as its source image |
| `easy/overlap` | A ring passing over a filled dot | Layer order between two objects that cross |
| `easy/sequence` | Circles that move, change colour gradually, scale, overlap, and leave the frame | Colour change expressed as a command; objects entering and leaving the frame |

### Medium: one difficulty at a time

| ID | Content | Scene cuts | What it tests |
| --- | --- | --- | --- |
| `medium/fish-short` | Static line-art panels with captions; only a mouth and a small fish on a phone screen change | not counted | Small patches on a static background |
| `medium/fish` | One 3D-rendered fish rotating on a white background, slowly shrinking | 1 | Repetition that is close but not exact; reuse of an image at a different size; animation memory limits |
| `medium/badapple` | Two-colour silhouettes that morph continuously, soft edges, 60 fps | 3 | Shapes that never hold still; only the boundary between black and white changes; large flat regions |

### Hard: several difficulties at once

| ID | Content | Scene cuts | What it tests |
| --- | --- | --- | --- |
| `hard/birdbrain` | A flat-drawn character dancing over handheld real footage | 110 | An object that changes pose every drawing but is clearly one sprite; noisy, shaking background; clean edges when the background changes |
| `hard/machinelove` | Real footage of a laptop whose screen plays animation, with subtitles on top | 150 | Sprites that survive a scene cut (subtitles); a tilted screen-in-screen; pixelated glitch frames |
| `hard/rollback` | Dense motion graphics with film grain, blur, glow, camera shake, and tilted 3D planes | 127 | Very short scenes (about 1.2 s); full-screen effects; hundreds of small elements; perspective that sprites cannot express |
| `hard/minecraft` | A camera flying through a 3D world, with particles and flashes to white | 63 | True 3D motion, where nothing slides as a flat piece; flashes handled as an additive layer |

`hard/minecraft` and the 3D parts of `hard/rollback` are expected to fall back to redrawing most
frames. They are in the set to check that the fallback stays correct, not to be compressed well.
