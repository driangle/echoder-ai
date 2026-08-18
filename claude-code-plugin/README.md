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
the frame before handing it back. It reports what it could not verify — live
input (MIDI, mouse, keyboard, camera) cannot be exercised headlessly.

**Requires the [`echoder` CLI](https://github.com/driangle/echoder):**

```bash
brew install driangle/tap/echoder
```

## Scope

Directory-form projects (`echoder.yaml` + `main.js` + sketch files) and loose
`.js` sketches. For a `.echo` bundle or a web-mode (`localStorage`) project,
open it in Studio and save it to a directory first.

## Development

The source of truth for this plugin lives in the Echoder monorepo at
`claude-code-plugin/`, alongside `.claude-plugin/marketplace.json` at the repo
root. Everything the plugin needs must be **physically inside this directory** —
it is copied on install and cannot reach `../docs` or `../examples` at runtime.

Validate both manifests:

```bash
pnpm check:plugin
```

This also runs as part of `make check-static`, the pre-commit hook, `pnpm check`,
and CI.

Publishing and versioning: see `docs/site/guide/claude-plugin.md` in the monorepo
(not linked — this README is copied into the published marketplace repo, where
that path does not exist).
