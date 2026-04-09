# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

A personalized 3D WebGL portfolio site ("My Igloo") based on a static mirror of [igloo.inc](https://www.igloo.inc/). Originally built with Svelte + Vite + Three.js r165, compiled into minified production bundles. This repo contains the production output (not source code), customized with new text content, branding, and assets.

Deployed via **Vercel** (auto-deploys on push to `main`). Config in `vercel.json`.
Live at the domain configured in Vercel dashboard.

## Running Locally

```bash
python3 server.py
# Serves at http://localhost:3000
```

`server.py` is a multi-threaded Python HTTP server with:
- Correct MIME types for `.drc`, `.ktx2`, `.wasm`, `.ogg`, `.exr`
- CORS headers (`Access-Control-Allow-Origin: *`)
- COOP/COEP headers (required for SharedArrayBuffer / Basis transcoder performance)

Do **not** use a generic file server — MIME types and COOP/COEP headers are critical for WASM, KTX2 texture decoding, and Web Workers.

## Deploying

```bash
git add -A && git commit -m "description" && git push
```

Vercel auto-deploys from `main`. The `vercel.json` configures headers (COOP/COEP, CORS) and MIME types for production, plus a rewrite for SPA routing.

## Architecture

No build step, no `package.json`, no `node_modules`. Static site served as-is.

### Key Files

- `index.html` — Entry point, loads `assets/index-2d900c2d.js` as a module script.
- `assets/index-2d900c2d.js` (16KB) — Svelte bootstrap. CSS injection, loader animation, viewport setup. Dynamically imports `App3D-5907d20f.js`. Modified from original: base URLs changed from `https://www.igloo.inc/` to relative paths for local serving.
- `assets/App3D-5907d20f.js` (1.5MB) — Entire application: Three.js r165, Svelte, GLSL shaders, animations, audio, router, UI. All minified. **This is the only file you edit for content changes.**

### Asset Directories

| Path | Content | Format |
|------|---------|--------|
| `assets/geometries/` | 3D models (root level + `cubes/` and `igloo/` subdirs) | Draco `.drc` |
| `assets/images/` | Textures (subdirs: `cubes/`, `igloo/`, `ui/`, `uv/`, `volumes/`, `noises/`) | `.ktx2`, `.png`, `.exr` |
| `assets/audio/` | Sound effects and music | `.ogg` |
| `assets/fonts/` | MSDF font atlas | `.ktx2` + `.json` |
| `assets/libs/draco/` | Draco WASM decoder | `.js` + `.wasm` |
| `assets/libs/basis/` | Basis Universal transcoder | `.js` + `.wasm` |

### Site Data Object (`Be`)

All text content, portfolio data, colors, and links are in `assets/App3D-5907d20f.js` in the config object `const Be={...}`. Current personalized values:

```
Be.manifesto.title    → "////// About"
Be.manifesto.text     → "My Igloo is a creative technology studio..."
Be.copyright          → "// Copyright © 2025"
Be.rights             → "My Igloo\nAll Rights Reserved."
Be.social             → [{name:"WEB",...}, {name:"MAIL",...}]
Be.cubes              → [{title:"...ORKA",...}, {title:"...SAW Studio",...}, {title:"...CasaStudio",...}]
Be.links              → [{title:"Contact", url:"mailto:ataberkekalender@gmail.com", vdb:"peachesbody_64",...}]
```

Portfolio cubes mapping (do NOT change `obj`/`innerobject` — these reference geometry and texture filenames):
- Cube 1: `obj:"cube3"`, `innerobject:"pudgy"` → ORKA
- Cube 2: `obj:"cube1"`, `innerobject:"overpass_logo"` → SAW Studio
- Cube 3: `obj:"cube2"`, `innerobject:"abstractlogo"` → CasaStudio

### Geometry Loading (Two Contexts)

The main cube class loads from `cubes/` subdirectory:
```
zt.load(`cubes/${obj}.drc`)        → assets/geometries/cubes/cube3.drc
zt.load(`${innerobject}.drc`)      → assets/geometries/pudgy.drc
```

The detail view class loads from root geometries path:
```
zt.load(`${obj}.drc`)              → assets/geometries/cube3.drc (must also exist here!)
```

**Both paths must have valid `.drc` files** — the root-level and `cubes/` subdirectory versions.

### Texture Loading

Textures load via `le.load()` with base path `assets/images/`. Key texture sets per cube:

| Texture | Path | Purpose |
|---------|------|---------|
| Color | `cubes/${innerobject}_color.ktx2` | Inner object color |
| Roughness | `cubes/${obj}_roughness.ktx2` | Cube surface roughness (PBR) |
| Normal | `cubes/${obj}_normal.ktx2` | Cube surface normals (PBR) |
| Dark color | `${innerobject}_dark_color.ktx2` | Detail view inner object shader |

Scene textures: `igloo/`, `shattered_ring_color.ktx2`, `shattered_ring_ao.ktx2`, etc.
Volume textures for footer 3D shapes: `volumes/*.ktx2` (64x64x64 or 32x32x32, RGBA, zstd-compressed KTX2).

### Routes

Hash-based SPA routing:
- `/` → Home scene (igloo on mountain, shattered ring)
- `/portfolio/:project` → Detail view (`orka`, `saw-studio`, `casastudio`)

## Making Modifications

### Text changes (easy)
Find/replace strings in `assets/App3D-5907d20f.js`. All user-visible text is in the `Be` config object.

### Creating 3D volume textures (for footer shapes)
Volume textures are 64x64x64 RGBA voxel grids stored as zstd-compressed KTX2. The RGB channels encode a directional gradient field (not just occupancy), and alpha encodes density. See commit `8cba425` for the Python script that generates a V-shape texture by analyzing the original X texture's encoding. Key: R and G channels encode opposing arm directions, B is constant 128, A ranges 123-159.

### Color scheme
Colors in `const Be={...}`:
- Background: `#A0A5B1` (also `--bgColor` in `index-2d900c2d.js`)
- Title: `#3C3C54`, Body text: `#ffffff`
- Project title: `#67707E`, Project text: `#A1AAB7`

## Known Issues and Gotchas

- **Corrupt `.drc` files**: When scraping igloo.inc, some URLs return HTML (SPA fallback) instead of binary data. Always verify `.drc` files start with `DRACO` magic bytes and `.ktx2` files start with `AB 4B 54 58 20 32 30 BB`. Files that are exactly 1410 bytes and start with `<!doctype html>` are corrupt.
- **Geometry duplication**: Cube geometries must exist at BOTH `assets/geometries/cubes/cube{1,2,3}.drc` (main view) AND `assets/geometries/cube{1,2,3}.drc` (detail view). Missing either causes fallback rendering.
- **Font/glyph geometries**: `medium_32.drc`, `peachesbody_64.drc`, `x_64.drc` and their `.json` counterparts in `assets/geometries/` are corrupt HTML from the original scrape. These don't exist on the original site at that path either — they appear to be unused or loaded differently.
- **KTX2 fallback**: When any KTX2 texture fails to load, the Basis transcoder error handler falls back to `uv/uvchecker-srgb.ktx2` (the checkered pattern). If you see UV checker anywhere, a texture is missing or corrupt.
- **server.py is gitignored**: The local dev server is in `.gitignore`. It's not needed for Vercel deployment but required for local development.
