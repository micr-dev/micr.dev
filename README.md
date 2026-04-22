# micr.dev

![portfolio](https://github.com/user-attachments/assets/1ec0295e-3ea7-4a8c-961d-9bcf9ca4d1eb)

A small [3D portfolio site](https://micr.dev) made with [Spline](https://spline.design).

Looks cool, runs smooth... *if* your browser's hardware acceleration is on.

## Tech Stack

- **3D rendering**: [Spline](https://spline.design) — interactive 3D scene
- **ASCII animation**: custom ZIP-based loader using [fflate](https://www.npmjs.com/package/fflate)
- **Audio**: [Howler.js](https://howlerjs.com/) — spatial audio
- **Fonts**: [opentype.js](https://opentype.js.org/) — typography rendering
- **CRT effects**: custom WebGL post-processing shader

## Features

- Interactive 3D scene with orb lineup and refresh animations
- ASCII ZIP-based frame animation at 30fps
- Spatial audio with Howler.js
- Custom CRT/retro post-processing effects
- WebAssembly-powered OpenType font rendering
- Responsive layout with hardware-accelerated rendering

## Browser Requirements

Hardware acceleration must be enabled in your browser for the 3D scene and ASCII animation to render smoothly. Most modern browsers have this on by default.

## Project Structure

```
├── index.html          # Main entry point
├── run.js              # ZIP-based ASCII animation loader
├── runtime.js          # Core runtime and scene management
├── howler.js           # Audio engine (Howler.js)
├── crt-effect.js       # CRT post-processing shader
├── crt-integration.js  # CRT effect setup and uniforms
├── opentype.js         # OpenType font rendering (WASM)
├── process.js          # Asset processing pipeline
├── assets/             # Static assets (frames ZIP, fonts, favicons)
├── about/              # About page
└── tree/               # Directory tree visualization
```

## Local Development

```bash
# Serve locally (any static server works)
npx serve .

# Or use Python
python3 -m http.server 8000
```

Then open `http://localhost:8000` in your browser.

## License

**© micr.dev 2025 ∷ all rights reserved.**

All code, design, writing, and media assets in this repository are fully owned by me.
Nothing in this repo may be copied, reused, modified, or distributed without explicit written permission.
