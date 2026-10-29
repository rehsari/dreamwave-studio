# DreamWave_Studio

Drop in a 3D model, give it a VHS / PS1 look, and export a PNG, GIF or video.
Everything runs in the browser: models never leave the user's computer.

## Run it
It needs to be served over http (double-clicking index.html won't load the model):

    npx serve .        # then open the printed URL

## Files
- `index.html` – the whole app (HTML + CSS + one JS module).
- `assets/dreamwave-studio.svg` – original logo.
- `model.glb` – the sample character (Glitter), loaded on start and by "Sample character".
- `vendor/three/` – three.js r170, only the 11 files this app imports, with its MIT license.
- `vendor/fonts/` – the VT323 web font (latin and latin-ext subsets).
- `assets/icon-32.png`, `assets/icon-180.png` – favicon and home-screen icon, resized from `DreamwaveIcon.png`.
- `.gitignore` – keeps `.DS_Store`, `Icon`, `.claude/`, `review/` and the duplicate root logo out of a repo.

There is no build step and no dependencies to install. Edit `index.html`, reload the page.

## Working on the code
`index.html` is one file: the styles, the markup and a single ES module, in that order. The module is
long, so it is cut into labelled sections — search for a banner like `/* ---------- model ---------- */`
to jump to one. A map of all the sections sits at the top of the module.

Two ideas carry most of the design:

- **`S` is the state.** One flat object holds every user-facing setting. `SCHEMA` is a declarative
  list describing each control, and the panel, the saved looks and undo/redo are all generated from
  it. Nothing keeps a second copy of a setting.
- **`applyAll()` pushes `S` into the scene and shader; `syncUI()` pushes `S` back into the controls.**
  After changing state, call both.

Where to make common changes:

| To do this | Go here |
| --- | --- |
| Add or retune a preset (VHS, PS1, …) | `PRESETS` in the *settings* section |
| Add a slider, toggle or dropdown | add one entry to `SCHEMA`, then read it in `applyAll` |
| Change the retro look itself | the fragment shader in *renderer & scene* |
| Change how models are posed or oriented | *model* |
| Change PNG / GIF / video output | *export rendering*, and *GIF encoder* for the encoder |

## Privacy and hosting
The page makes no third-party requests: three.js and VT323 are served from `vendor/`, so there is
no CDN or font host to log visitors, and the app works offline once loaded. To move to a newer
three.js, replace `vendor/three/` and keep the import map in `index.html` pointing at it.

A `Content-Security-Policy` meta tag limits the page to its own origin plus `blob:`/`data:` for the
user's files and the canvas. Scripts and styles still need `'unsafe-inline'` because they live inside
`index.html`, so the tag's real value is the network restriction, not script control. `frame-ancestors`
is omitted on purpose: it is ignored in a meta tag and only works as a real HTTP header, so add it at
the host if you want to stop other sites from framing the studio.

The link-preview tags use a relative `og:image`, which many scrapers ignore. Once the site has a real
address, change `og:image` and `twitter:image` in `index.html` to absolute URLs.

Dropped models never reach the network. A `.gltf`, `.obj` or `.mtl` can reference an absolute URL,
which would hand the visitor's IP address to that host, so only files dropped alongside the model are
served to the loaders; anything else is skipped, logged to the console and left as a missing texture.

## How it works
- **Undo and redo:** the panel header has **Undo**/**Redo**, with <kbd>Ctrl</kbd>/<kbd>Cmd</kbd>+<kbd>Z</kbd>
  and <kbd>Ctrl</kbd>/<kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>Z</kbd> (or <kbd>Ctrl</kbd>+<kbd>Y</kbd>).
  History stores snapshots of every setting: effects, presets, saved looks, background and floor
  options, and model position/rotation/scale. One slider drag is one step, and 60 steps are kept.
  Text and number fields keep the browser's own undo while they have focus. Camera orbit is not
  undoable, loading a file is not undoable, and loading a model clears the history because it
  re-frames and re-poses the scene.
- **Loading:** GLB, GLTF (+ .bin + textures), FBX, OBJ (+ .mtl + textures). Dropped files are read
  locally; companion files are served to the loaders by file name through a LoadingManager URL modifier.
  A .gltf and its .bin files are packed into an in-memory GLB first.
- **Orientation:** for rigged characters the first animation frame is posed and the model is rotated so
  the head is above the feet and it faces the camera. Static models are centered with their base on the floor.
- **Position:** choose **Move model** to drag world-space X/Y/Z handles, or type positions in model
  units. Drag outside the handles to orbit. **Reset position** returns to the original placement;
  **Place on ground** lowers the current pose to the ground while keeping X/Z. Position survives
  animation and orientation changes; loading a new model resets it. Turntable exports spin around
  the positioned model, and editing handles are never exported. Resizing the panel and switching
  animation preserve the camera; loading, framing changes and the flip/turn buttons re-frame it.
- **Rotate and scale:** **Rotate model** shows rotation rings and degree fields; **Scale model**
  shows scale handles and X/Y/Z factors (1 = original size, supported range 0.01–100).
  Both use the model's center. Use **Place on ground** after a tilt or scale change if needed.
  Reset buttons affect the selected transform only. Rotation/scale survive animation changes and
  are included in exports and viewer configuration; loading another model resets them.
- **Keyboard:** G selects Move, R selects Rotate, S selects Scale; then drag a handle or type an
  amount. X/Y/Z constrain an axis (press it again to clear). For typed movement or rotation,
  choose an axis first: `G X 0.5 Enter` moves 0.5 model units; `R Y 90 Enter` adds 90° around
  world Y. `S 2 Enter` doubles all scale factors; `S X 0.5 Enter` halves local X only.
  Enter commits a typed command; Escape cancels the current command or handle drag, and a second
  Escape returns to Orbit. Shortcuts are inactive while typing in form fields or exporting.
  These are tool-selection and numeric-command shortcuts, not Blender's free-mouse modal transforms.
- **Materials:** every material is converted to an unlit flat material (texture or color) or a stepped
  toon material. Models without textures switch to toon automatically.
- **Look:** the scene renders into a low-resolution render target (nearest filtering), then one
  full-screen shader adds line jitter + rolling tracking band, red/cyan split, YIQ chroma bleed,
  saturation/contrast, posterize with 4×4 Bayer dither, scanlines, grain and vignette.
  PS1 vertex wobble snaps vertices to a screen grid (injected with onBeforeCompile).
  The effect is defined for a 900px-tall view and scales with height, so exports match the screen.
  Set `fixedScale: false` in the viewer configuration to keep the effect at a constant pixel size instead.
- **Exports:** PNG (1080/2048/720 px tall, optional transparent background), GIF (built-in encoder:
  per-frame median-cut palette + LZW, frames rendered deterministically so the loop is seamless:
  one full turn and/or whole animation cycles), video (MediaRecorder on the canvas, MP4 where
  supported, otherwise WebM; recorded in real time for one loop).
  Inside Claude, saving goes through the platform's downloads capability; elsewhere it's a normal download.
- **Styles:** presets VHS, PS1, Game Boy, Arcade, Comic, ASCII, Chunky, Clean.
  Palettes (Game Boy, 4-gray, 1-bit, CGA, PICO-8, or imported .hex/.txt/.gpl/image) matched by color or
  brightness with Bayer dithering. ASCII (glyph atlas, one character per cell) and CMYK halftone styles.
  CRT curve, glow and phosphor mask. PS1 affine texture warping (uv*w / w trick injected into the
  materials) and draw-distance fog. Toon outlines from a depth texture (Laplacian of 1/z, so flat
  surfaces don't outline).
- **Scene:** backdrop color, flat image (cover-fit), or panoramic sky. Load a seamless 2:1
  equirectangular panorama through **Backdrop image…**, choose **Panoramic sky**, and adjust
  **Sky rotation**. This is a surrounding background, not environment lighting; regular photos may
  stretch or show seams. Flat image and sky modes reuse the same loaded image.
  The floor supports off / grid / checker / image and a soft contact shadow that follows placement.
  **Ground tile size** ranges from 0.1× to 10×; larger values enlarge each tile while preserving
  the image's aspect ratio. Backdrop and floor render inside the scene, so effects apply to them.
- **My looks:** saved to localStorage (older saved looks are migrated automatically), including ground tile size, backdrop mode and sky rotation.
  Image files must be reselected after a reload. Looks do not change model position.
  The collapsed **Advanced** section contains "Copy viewer configuration", which outputs a
  VIEWER_CONFIG block (including position) for websites running this viewer; it does not include model or image files.
- **Interface:** classic gray desktop controls, beveled buttons and an inset viewport. The existing control order and responsive side-panel / bottom-panel layout are preserved.

## License
Two licenses, in `LICENSE`:

- **Code** (`index.html` and everything else) — MIT. Do what you like with it.
- **Artwork** (`model.glb`, the logo, the icons) — CC BY-NC 4.0. Free to use, remix and share with
  credit, but not for anything commercial. The character "Glitter", the logo and the icon were made
  by Rafael C Sanchez and belong to him.

Renders you make from your own models are yours.

## Ideas for next steps
- Save / reopen a complete project (model, images, camera, transforms and effects)
- Undo / redo for scene edits
- Camera angle and output-size presets
- Share links (settings encoded in the URL hash)
- Sprite-sheet export (8 directions) for game developers
- Background video, lighting controls, camera angle presets
- Draw the tape overlay text into exports
- Hosted embeds (needs storage, accounts, moderation)
