# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

A static mirror of [igloo.inc](https://www.igloo.inc/) — a 3D WebGL portfolio site for Igloo Inc. (Web3 company behind Pudgy Penguins, Overpass, Abstract). The site was originally built with Svelte + Vite + Three.js r165 and compiled into minified production bundles. This repo contains the downloaded production output, not source code.

## Running Locally

```bash
python3 server.py
# Opens at http://localhost:3000
```

`server.py` is a custom Python HTTP server with correct MIME types for `.drc`, `.ktx2`, `.wasm`, `.ogg` etc. and CORS headers. Do not use a generic file server — MIME types matter for WASM and module scripts.

## Architecture

This is **not** a source-code project. There is no build step, no `package.json`, no `node_modules`. It is a static site served as-is.

### Key Files

- `index.html` — Entry point. Loads `assets/index-2d900c2d.js` as a module script.
- `assets/index-2d900c2d.js` (16KB) — Svelte bootstrap. Contains the CSS (injected at runtime), loader component (ASCII animation), viewport setup, and dynamically imports `App3D-5907d20f.js`.
- `assets/App3D-5907d20f.js` (1.5MB) — The entire application: Three.js r165, Svelte components, GLSL shaders, animation engine, scene management, audio system, custom router, UI rendering. All minified.

### Asset Loading System

The app uses `window.location.origin` as its base path (variable `q.absolutePath` in the bundle). All assets load relative to origin:

| Path | Content | Format |
|------|---------|--------|
| `assets/geometries/` | 3D models | Draco compressed `.drc` |
| `assets/images/` | Textures | `.ktx2` (Basis Universal), `.png`, `.exr` |
| `assets/audio/` | Sound effects | `.ogg` |
| `assets/fonts/` | MSDF font atlas | `.ktx2` + `.json` |
| `assets/libs/draco/` | Draco WASM decoder | `.js` + `.wasm` |
| `assets/libs/basis/` | Basis transcoder | `.js` + `.wasm` |

### Site Data Object

All text content, portfolio data, colors, and social links are embedded in `App3D-5907d20f.js` in a config object (`const Be={...}`). Key fields:

```
Be.manifesto.title    → "////// Manifesto"
Be.manifesto.text     → "Our mission is to create..."
Be.copyright          → "// Copyright © 2026"
Be.rights             → "Igloo, Inc.\nAll Rights Reserved."
Be.social             → [{name:"X", link:"..."}, {name:"LI", link:"..."}]
Be.cubes              → [{title, hash, date, obj, innerobject, interior:{content,...}, social, links}, ...]
```

Portfolio items (`Be.cubes`): Pudgy Penguins (`hash:"pudgy-penguins"`, obj:`cube3`, innerobject:`pudgy`), Overpass (`hash:"overpass"`, obj:`cube1`, innerobject:`overpass_logo`), Abstract (`hash:"abstract"`, obj:`cube2`, innerobject:`abstractlogo`).

### 3D Model Pipeline

Each portfolio cube loads geometry via: `zt.load("${obj}.drc")` for the outer cube and `zt.load("${innerobject}.drc")` for the inner object. Textures load via: `le.load("cubes/${innerobject}_color.ktx2")`. The Draco decoder at `assets/libs/draco/` decompresses `.drc` files at runtime.

### Routes

Hash-based routing with two routes:
- `/` → Home scene (igloo on mountain)
- `/portfolio/:project` → Detail view (project = `pudgy-penguins`, `overpass`, or `abstract`)

## Making Modifications

### Text changes (easy)
Find/replace strings in `assets/App3D-5907d20f.js`. Text content is stored as plain strings in the `Be` config object. Search for the exact text to locate it.

### Replacing 3D models (moderate)
1. Obtain a `.glb` model
2. Convert to Draco-compressed `.drc` using `gltf-pipeline` or Three.js DRACOExporter
3. Replace the corresponding file in `assets/geometries/` (keep the same filename)
4. For textures, convert to KTX2 using `toktx` from KTX-Software, or provide a `.png` fallback

### Modifying animation/shader logic (very difficult)
All code is in a single 1.5MB minified bundle. Variable names are mangled. Changing behavior requires finding the relevant section, understanding the minified logic, and making surgical edits without breaking anything. There is no source map.

### Color scheme
Colors are defined in `const Be={...}` inside `App3D-5907d20f.js`:
- Background: `#A0A5B1` (also set as CSS `--bgColor` in `index-2d900c2d.js`)
- Title: `#3C3C54`
- Body text: `#ffffff`
- Project title: `#67707E`
- Project text: `#A1AAB7`
