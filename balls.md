# Spinning Balls

This is the maintainer workflow for adding a new `tree` entry that uses the same spinning-ball treatment as the existing site previews.

## What this changes

Each ball needs two things:

1. A `tree/index.html` entry in the `spheres` array.
2. A transparent animated GIF in `tree/images/` that loops as a spinning sphere.

The `tree` page does not generate the ball effect on its own. The GIF is the asset.

## Output conventions

Match the current ball assets:

- File location: `tree/images/micrdev<slug>.gif`
- Format: animated GIF
- Size: `250x250`
- Length: `150` frames
- Duration: `3s`
- Background: transparent
- Motion: texture should drift to the right

Check an output with:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,nb_frames,duration \
  -of default=noprint_wrappers=1:nokey=0 \
  tree/images/micrdev<slug>.gif
```

Expected:

```text
width=250
height=250
duration=3.000000
nb_frames=150
```

## 1. Capture the real site

Use Playwright for the screenshot. `agent-browser` is fine for inspection, but in this repo Playwright was the reliable path for the actual capture.

Example:

```bash
mkdir -p /tmp/micr-tree-captures

node --input-type=module <<'EOF'
import { chromium } from 'playwright';

const browser = await chromium.launch({ headless: true, args: ['--no-sandbox'] });
const page = await browser.newPage({
  viewport: { width: 1440, height: 900 },
  colorScheme: 'dark', // remove or change if the page should be light
});

await page.goto('https://example.micr.dev/', {
  waitUntil: 'networkidle',
  timeout: 30000,
});

await page.screenshot({
  path: '/tmp/micr-tree-captures/example.png',
  fullPage: false,
});

console.log(await page.title());
await browser.close();
EOF
```

Notes:

- Use the exact route you want represented, for example `https://mullgate.micr.dev/docs`.
- If the page should be dark mode, set `colorScheme: 'dark'`.
- Keep the capture at `1440x900` unless a page needs something different.

## 2. Turn the screenshot into a spinning ball

This inline Python generator is the current working recipe. Replace:

- `SOURCE_PNG`
- `OUTPUT_GIF`
- `CENTERING`
- image adjustments if the page needs them

```bash
python3 - <<'EOF'
from pathlib import Path
import math
import shutil
import subprocess
import tempfile

import numpy as np
from PIL import Image, ImageEnhance, ImageFilter, ImageOps

SOURCE_PNG = Path('/tmp/micr-tree-captures/example.png')
OUTPUT_GIF = Path('/home/ubuntu/workspace/micr.dev/tree/images/micrdevexample.gif')

CANVAS = 250
TEXTURE_SIZE = 720
FRAMES = 150
FPS = 50
RADIUS = 92.0
CENTERING = (0.5, 0.5)

center = (CANVAS - 1) / 2.0
px = np.arange(CANVAS, dtype=np.float32)
py = np.arange(CANVAS, dtype=np.float32)
xx, yy = np.meshgrid(px, py)
dx = (xx - center) / RADIUS
dy = (yy - center) / RADIUS
rr2 = dx * dx + dy * dy
mask = rr2 <= 1.0
zz = np.zeros_like(dx)
zz[mask] = np.sqrt(1.0 - rr2[mask])
edge = np.clip(RADIUS + 0.85 - np.sqrt((xx - center) ** 2 + (yy - center) ** 2), 0.0, 1.0)

light = np.array([-0.42, -0.38, 0.82], dtype=np.float32)
light /= np.linalg.norm(light)
view = np.array([0.0, 0.0, 1.0], dtype=np.float32)
half_vec = light + view
half_vec /= np.linalg.norm(half_vec)

def bilinear_sample(texture, sx, sy):
    h, w, _ = texture.shape
    sx = np.clip(sx, 0.0, w - 1.001)
    sy = np.clip(sy, 0.0, h - 1.001)
    x0 = np.floor(sx).astype(np.int32)
    y0 = np.floor(sy).astype(np.int32)
    x1 = np.clip(x0 + 1, 0, w - 1)
    y1 = np.clip(y0 + 1, 0, h - 1)
    wx = (sx - x0)[..., None]
    wy = (sy - y0)[..., None]
    top = texture[y0, x0] * (1.0 - wx) + texture[y0, x1] * wx
    bottom = texture[y1, x0] * (1.0 - wx) + texture[y1, x1] * wx
    return top * (1.0 - wy) + bottom * wy

src = Image.open(SOURCE_PNG).convert('RGB')
fitted = ImageOps.fit(
    src,
    (TEXTURE_SIZE, TEXTURE_SIZE),
    method=Image.Resampling.LANCZOS,
    centering=CENTERING,
)
fitted = ImageEnhance.Brightness(fitted).enhance(1.0)
fitted = ImageEnhance.Contrast(fitted).enhance(1.1)
fitted = ImageEnhance.Color(fitted).enhance(1.05)
fitted = fitted.filter(ImageFilter.UnsharpMask(radius=1.2, percent=125, threshold=2))
texture = np.asarray(fitted, dtype=np.float32) / 255.0

workdir = Path(tempfile.mkdtemp(prefix=f'{OUTPUT_GIF.stem}-frames-'))

try:
    for frame in range(FRAMES):
        angle = (2.0 * math.pi * frame) / FRAMES
        cos_a = math.cos(angle)
        sin_a = math.sin(angle)

        # This sign makes the texture drift to the right.
        x_rot = dx * cos_a - zz * sin_a
        y_rot = dy

        sx = (x_rot * 0.5 + 0.5) * (TEXTURE_SIZE - 1)
        sy = (y_rot * 0.5 + 0.5) * (TEXTURE_SIZE - 1)
        sampled = bilinear_sample(texture, sx, sy)

        diffuse = np.clip(dx * light[0] + dy * light[1] + zz * light[2], 0.0, 1.0)
        spec = np.clip(dx * half_vec[0] + dy * half_vec[1] + zz * half_vec[2], 0.0, 1.0)
        spec = np.power(spec, 28.0) * 0.18
        rim = np.power(1.0 - np.clip(zz, 0.0, 1.0), 1.8) * 0.12
        shading = 0.58 + 0.34 * diffuse + 0.12 * zz - rim

        rgb = np.zeros((CANVAS, CANVAS, 3), dtype=np.float32)
        rgb[mask] = np.clip(sampled[mask] * shading[mask, None] + spec[mask, None], 0.0, 1.0)

        alpha = np.zeros((CANVAS, CANVAS), dtype=np.float32)
        alpha[mask] = edge[mask]

        rgba = np.empty((CANVAS, CANVAS, 4), dtype=np.uint8)
        rgba[..., :3] = np.clip(rgb * 255.0, 0, 255).astype(np.uint8)
        rgba[..., 3] = np.clip(alpha * 255.0, 0, 255).astype(np.uint8)
        Image.fromarray(rgba).save(workdir / f'frame_{frame:03d}.png')

    palette = workdir / 'palette.png'

    subprocess.run([
        'ffmpeg', '-y', '-v', 'error', '-framerate', str(FPS),
        '-i', str(workdir / 'frame_%03d.png'),
        '-vf', 'palettegen=reserve_transparent=1:stats_mode=diff',
        str(palette),
    ], check=True)

    subprocess.run([
        'ffmpeg', '-y', '-v', 'error', '-framerate', str(FPS),
        '-i', str(workdir / 'frame_%03d.png'),
        '-i', str(palette),
        '-lavfi', 'paletteuse=dither=bayer:bayer_scale=3:alpha_threshold=96',
        str(OUTPUT_GIF),
    ], check=True)

    print(f'generated {OUTPUT_GIF}')
finally:
    shutil.rmtree(workdir, ignore_errors=True)
EOF
```

## 3. Add the `tree` entry

Add a new object to the `spheres` array in `tree/index.html`.

Example:

```js
{
  img: "images/micrdevexample.gif",
  url: "https://example.micr.dev/",
  desc: "Short description here.",
},
```

Use the exact site URL and keep the copy short enough to fit the tooltip cleanly.

## 4. Verify locally

Serve `tree` locally:

```bash
python3 -m http.server 4173 --directory /home/ubuntu/workspace/micr.dev/tree
```

Then verify in Playwright:

```bash
set +H
node --input-type=module <<'EOF'
import { chromium } from 'playwright';

const browser = await chromium.launch({ headless: true, args: ['--no-sandbox'] });
const page = await browser.newPage({ viewport: { width: 1440, height: 1200 } });

await page.goto('http://127.0.0.1:4173/', {
  waitUntil: 'networkidle',
  timeout: 30000,
});

await page.click('#music-no');
await page.waitForTimeout(500);

const result = await page.evaluate(() => {
  const items = Array.from(document.querySelectorAll('.sphere img')).map((img) => ({
    src: img.getAttribute('src'),
    naturalWidth: img.naturalWidth,
    naturalHeight: img.naturalHeight,
    complete: img.complete,
    alt: img.getAttribute('alt'),
  }));

  return {
    broken: items.filter((item) => item.complete === false || item.naturalWidth === 0),
  };
});

console.log(JSON.stringify(result, null, 2));
await browser.close();
EOF
```

You want `broken` to be empty.

## Notes from the `thinko` / `mullgate` update

- `thinko` used the root page capture.
- `mullgate` looked better from `/docs` in dark mode than from `/`.
- The crop bias matters. Use `CENTERING` to keep the subject inside the sphere.
- The wrong rotation direction is easy to spot. If the ball drifts left, the sign on `x_rot` is wrong.
