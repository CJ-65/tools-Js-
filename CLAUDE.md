# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo shape

A collection of independent sections, each with its own installer and docs. There is no root build, no shared setup, no test suite, no linter. Do not add cross-section dependencies.

- `minimax-h3/` — ComfyUI workflows, custom nodes and patches for running MiniMax H3 (video + synchronized audio) locally. Bash installers, Python nodes.
- `skills/hand-drawn-canvas-animation/` — an agent skill (`SKILL.md` + assets/scripts/references) that makes short hand-drawn films from Canvas 2D JavaScript.

Numbers in READMEs are measured on 4x RTX 3090 (see root `README.md`). Don't "update" benchmark figures without a real run.

## minimax-h3

Commands (run from `minimax-h3/`):

```bash
./install/setup.sh check                              # read-only: GPUs, CUDA, power limits, recommends a set
./install/setup.sh standard --comfy /path/to/ComfyUI  # workflows into an existing ComfyUI
./install/setup.sh multi-gpu --dir ~/comfy-raylight   # builds a separate Raylight stack
./install/download-models.sh --dir /path/to/ComfyUI/models   # ~60 GB of weights
```

Nothing here can run in a GPU-less session; only `check` is safe to execute, and it just reports missing hardware.

Architecture (needs several files to see):

- **Two workflow sets.** `workflows/standard/` (7 graphs, DisTorch2, every loader on `cuda:0`, extra cards add capacity not speed) vs `workflows/multi-gpu/` (4 graphs, Raylight, one Ray worker per GPU, sequence-parallel). The multi-gpu set installs into its own directory because Raylight pins torch 2.8.0, older than current ComfyUI.
- **Workflows are ComfyUI graph JSON**, with guide cards embedded on the canvas. Edit them as JSON carefully (node ids/links must stay consistent); they are normally authored in the ComfyUI UI.
- **Turbo LoRA patch chain** (multi-gpu only). The turbo LoRA's tensor names are not recognised by ComfyUI's loader, which silently loads zero keys. Fix spans three places:
  - `patches/h3_turbo_lora_ray.py` / `h3_turbo_ray.py` — rebuild the name map; 208 backbone modules via bypass injection (`base(x) + lora(x)`), 51 adaln modules re-injected at run time.
  - `patches/sitecustomize.py` — Ray imports the worker class by `module.qualname` in each worker process, so a monkey patch in the main ComfyUI process never reaches workers. `sitecustomize` runs at start of every Python in the venv (including `ray::RayWorker`) and hooks `raylight.distributed_worker.ray_worker.RayWorker.load_lora` on import, leaving the raylight repo unmodified. Env: `H3_TURBO_LORA_MODE=bypass|merge`, `H3_TURBO_LORA_PATCH=0` disables.
  - `custom-nodes/h3-turbo-ray` — the turbo sampler as an importable module so Ray can ship it to workers.
  - Verification: the load log must contain `[H3TurboRay rank 0] ... 208 adapters, 208 hooks + 51 adaln at run time`. If missing, the LoRA is being ignored (nothing errors).
- `custom-nodes/h3-ray-kit` — `H3Resolution` (presets, short edge 768, multiples of 32) and `H3MultishotRay` (shot chaining: split script on a `---` line, last frame of shot N is keyframe of N+1, drops the duplicate frame and 1/24 s of audio, 40 ms crossfade). Frame count snaps to the 17k+5 grid at 24 fps, capped at 362.
- `custom-nodes/longform-patch` — stops the chaining node fighting MultiGPU over memory.
- `skills/` inside minimax-h3 — prompt-writing skills (`h3-t2v-prompt`, `h3-ref-prompt`, `h3-long-video`), separate from the top-level `skills/`.

Conventions: docstrings/comments in the Python patches and nodes are mostly in Russian, and some preset labels in `h3-ray-kit` are Russian strings that appear in workflow JSON — renaming a preset label breaks saved graphs that reference it. Keep the existing language when editing those files. Installer scripts are `set -euo pipefail` bash with `say/ok/warn/die` helpers; follow that style.

## skills/hand-drawn-canvas-animation

`SKILL.md` is the entry point and the real spec (file table + numbered procedure); read it, then `references/architecture.md` before editing code and `references/style.md` before drawing anything.

Commands (a film lives in its own folder; `puppeteer-core` drives system Chrome, ffmpeg must be on PATH, `CHROME=/path/to/chrome` overrides detection):

```bash
cd scripts && npm i --no-audit --no-fund              # or in a film folder after copying render.mjs + package.json
node render.mjs film.html --grid 24 --ar 1:1          # contact sheet of 24 frames (look at this first)
node render.mjs film.html --only 0,12 --ar 16:9       # spot-check PNGs
node render.mjs film.html --ar 16:9 --width 1920      # full render: mp4 + contact sheet + <film>-final.mp4 with score
```

Invariants that matter when changing anything:

- `assets/core.js` is shared and **never edited per film**; films copy it next to themselves. Changes to it affect every example in `examples/`, so re-render a few after touching it.
- `drawFrame(i)` is a pure function of the drawn-frame index: no state between frames, no clock reads. Frames are drawn on twos (12 fps).
- Scenes are `sceneX(c, tau, i)` and place things relative to `CX, CY, W, H`, never literal pixels (short side is always 1080 logical units; output size is set at render time).
- A film HTML must expose `window.__drawFrame(i)` and `window.__NDRAW` and honour `?bare=1`; `render.mjs` reads frames from the page's own canvas via `toDataURL`.
- Three optional engines sit on top of the core: `roto.js` (+ `scripts/roto.py`, found motion), `sand.js`, `paper3d.js`.
- You can't judge a frame from code; verify visual changes by rendering and opening the PNG.

The skill installs by copying the folder to `~/.agents/skills/` (or `.agents/skills/` in a project).

## Git hygiene

`.gitignore` excludes `*.safetensors`, `*.ckpt`, `*.pt`, `*.mp4`, `*.mov`, `node_modules/`, `.venv/` — model weights and rendered video never get committed.
