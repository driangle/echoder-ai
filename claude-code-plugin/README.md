# Echoder plugin for Claude Code

A Claude Code plugin that teaches Claude to write, run, and **verify** Echoder
sketches — the real-time creative coding environment for generative video and
audio.

## Install

The plugin is published to the public **[`driangle/echoder-ai`](https://github.com/driangle/echoder-ai)**
marketplace repo. In Claude Code:

```
/plugin marketplace add driangle/echoder-ai
/plugin install echoder@echoder-marketplace
/reload-plugins
```

The skill then appears as `/echoder:echoder` and activates on its own when you
ask for Echoder work.

### From a local checkout

Working on the plugin itself? Point the marketplace at this repo instead — the
manifest at the repo root is the same one that gets published:

```
/plugin marketplace add /path/to/echoder
/plugin install echoder@echoder-marketplace
/reload-plugins
```

Re-run `/reload-plugins` after editing `skills/echoder/SKILL.md`.

## Usage

Ask for Echoder work in a project directory and the skill picks it up:

- "Make me an audio-reactive visualizer for `track.wav`."
- "Why is my feedback loop washing out?"
- "Add a shader field behind the existing shapes."

Claude looks up DSL signatures via `echoder types path`, writes the sketch,
runs it with `echoder run`, renders a frame with `echoder render`, and looks at
the frame before handing it back.

### What it can and cannot check

- **2D visuals** — fully: rendered to PNG and looked at.
- **GL visuals** (`vis`, `shader`, …) — fully, through a headless-Chromium
  renderer. Needs `echoder install-browser`; if Chromium is missing Claude
  surfaces the command rather than silently rendering without GL.
- **Audio** — the render reports a peak level, so silence is caught. Claude
  cannot listen to it.
- **Audio-reactive visuals, MIDI, mouse, keyboard, camera, `vision`** — *not*
  checkable. Analysers and inputs idle at zero headlessly, so a rendered frame
  shows the sketch's resting state. Claude is instructed to say so rather than
  describe motion it never saw.

The skill carries two references it consults as it works:
`skills/echoder/references/recipes.md` (ten verified patterns) and
`skills/echoder/references/errors.md` (failure text → cause → fix, including the
failures that exit 0).

**Requires the [`echoder` CLI](https://github.com/driangle/echoder):**

```bash
brew install driangle/tap/echoder
```

The skill's first step is `echoder types path`, which landed **after**
`cli-v0.1.0`. On an older CLI it fails with `unknown command: types` — check
`echoder --version` and `brew upgrade driangle/tap/echoder`.

## Scope

Directory-form projects (`echoder.yaml` + `main.js` + sketch files) and loose
`.js` sketches. For a `.echo` bundle or a web-mode (`localStorage`) project,
open it in Studio and save it to a directory first.

## Updating

```
/plugin marketplace update echoder-marketplace
/reload-plugins
```

## Development

This plugin is generated from the Echoder source repository. Everything it needs
is **physically inside this directory** — it is copied on install and cannot
reach outside itself at runtime. Edits made to the published copy are overwritten
on the next release, so please open issues upstream.
