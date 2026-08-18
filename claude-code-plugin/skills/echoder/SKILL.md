---
name: echoder
description: Write, run, and verify Echoder sketches — real-time generative video and audio built on the Echoder DSL (audio, vis, shader, math, time). Use when the user asks to create, edit, debug, or render an Echoder sketch or project, mentions echoder.yaml or the `echoder` CLI, or asks for an audio-reactive visual, shader field, MIDI-driven visual, or generative animation in this environment.
---

# Echoder

> **Status: stub.** The procedure below is a placeholder so the plugin validates
> and installs. The authored content — project anatomy, the DSL idioms nothing on
> disk teaches, the recipe and error references — lands in task `01m09qhf8`.

Echoder is a real-time creative coding environment: sources, effects, and sinks
wired into a reactive node graph that runs at 60+ FPS. Sketches are plain
JavaScript files in a directory-form project (`echoder.yaml` + `main.js` +
sketch files).

## Procedure

1. **Look up the API.** Run `echoder types path` to get the absolute path to the
   bundled `@echoder/types` `dsl/` directory, and read the `.d.ts` files there
   for exact signatures. Never write type stubs into the user's project.
2. **Write the sketch** as a `.js` file in the project.
3. **Run it.** `echoder run <sketch> --duration=2 -v` must exit 0. `-v` surfaces
   the actual error rather than just a non-zero exit.
4. **Look at it.** `echoder render <sketch> -o ./frames --duration=2 --fps=4`,
   then `Read` a rendered frame to confirm it looks like what was asked for.
5. **Report honestly**, naming anything you could not verify — live input (MIDI,
   mouse, keyboard, camera) cannot be exercised headlessly.

## Scope

- **Directory-form projects and loose `.js` sketches only.** For a `.echo` bundle
  or a web-mode project, the answer is: open it in Studio and save it to a
  directory first.
- Studio's file watcher picks up writes within ~1s, so no reload step is needed.
