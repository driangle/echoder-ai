---
name: echoder
description: Write, run, and verify Echoder sketches — real-time generative video and audio built on the Echoder DSL (audio, vis, shader, math, time). Use when the user asks to create, edit, debug, or render an Echoder sketch or project, mentions echoder.yaml or the `echoder` CLI, or asks for an audio-reactive visual, shader field, MIDI-driven visual, or generative animation in this environment.
---

# Echoder

Echoder is a real-time creative coding environment. Sources, effects and sinks are
wired into a **reactive node graph** that runs at 60+ FPS. A sketch is a plain
`.js` file; the DSL namespaces (`sketch`, `shape2d`, `lfo`, `audio`, `vis`,
`shader`, `math`, `time`, …) are globals — there is nothing to import.

## Procedure

Follow all five steps every time. Step 4 is the one that is tempting to skip and
must not be: **exit 0 does not mean the sketch looks right.**

1. **Look up the API.** `echoder types path` prints the absolute path to the
   CLI's bundled `@echoder/types` `dsl/` directory, plus the CLI version. Read
   the `.d.ts` files there for exact signatures. Never write type stubs into the
   user's project.
   These definitions are hand-maintained and **lag the runtime** — `vis.buffer`,
   `signal.mix`, `signal.noise` and others are real but undeclared. A name that
   is absent from the types is not proof it does not exist; check
   `references/recipes.md` and the user's existing sketches before concluding it.
2. **Write the sketch** as a `.js` file next to the project's other sketches.
3. **Run it.** `echoder run <sketch> --duration=2 -v` must exit 0. `-v` prints
   the real error instead of just a non-zero exit.
   Add `--renderer=browser` whenever the sketch touches a GL namespace —
   `vis`, `shader`, `shape3d`, `image`, `video`, `screen`, `compose`. Without it
   the run fails with `vis is not defined`, because `--renderer=auto` resolves to
   the Node-native 2D runtime for `run`.
4. **Look at it.** `echoder render <sketch> -o ./frames --duration=2 --fps=4`
   (same `--renderer=browser` rule), then `Read` one of the PNGs and describe
   what it actually shows. If it is blank, off-centre, one pixel wide, or the
   wrong colour, fix it and render again. Delete the frames directory when done.
5. **Report honestly**, naming everything from *Verification limits* below that
   applies.

For a whole project: `echoder check <dir>` smoke-runs every runnable sketch and
prints a pass/fail/skip summary. It only picks up files the project considers
**runnable** — the manifest's `main`, any file with a `kind: sketch` header, and
any subdirectory's `main.js`. Files without a header are library code and are
skipped silently, so `no sketch files matched` usually means a missing header,
not a missing file.

## Verification limits

Say which of these apply. Do not describe unverified behaviour as if you saw it.

| Path | Status |
| --- | --- |
| 2D visuals (`sketch`, `shape2d`, `text`, `system`, `lfo`, `math`) | Fully verifiable — run and render. |
| GL visuals (`vis`, `shader`, …) | Verifiable via `--renderer=browser` (headless Chromium + SwiftShader). Multi-file projects included — the browser renderer resolves `import` the same way the 2D one does. |
| Audio output | `echoder render <sketch> -o out.wav --duration=4` prints `peak <n>`. `peak 0.000` with exit 0 means silence — read the number, do not trust the exit code. You cannot listen to it. |
| Audio-*reactive* visuals | **Not verifiable.** Visual runs wire `audio` to an unrendered context, so analysers (`song.amplitude`, `.spectrum()`) sit at **zero**. Files are still decoded, so a bad path fails — but a rendered frame shows the sketch's *idle* state only. |
| MIDI, mouse, keyboard, camera | **Not verifiable.** The runtime registers these against idle hosts: they run, and read as at-rest forever. |
| `vision` (pose/face/hands) | Never available headlessly. |

Because analysers and inputs idle at zero, **give every reactive sketch a
non-blank idle state** — an LFO floor under the spectrum bars, a `.default()` on
every MIDI signal. It is what makes the sketch verifiable, and it is also what
makes it look alive before the user plays anything.

If the browser renderer reports `The browser renderer needs Chromium`, tell the
user to run `echoder install-browser` (add `--with-deps` on Linux) and say the
GL sketch is unverified until they do. Do not suggest
`npx playwright install chromium` — the CLI only launches the Chromium revision
its pinned `playwright-core` matches. Never quietly drop back to the 2D
renderer.

## Project anatomy

A directory-form project:

```
my-project/
  echoder.yaml     # manifest: name, version: 1, main, canvas, audio.bpm, runtime.fps
  main.js          # the default entry — always runnable
  scenes/intro.js  # runnable IF it carries a `kind: sketch` header
  lib/palette.js   # no header → library code, import-only
```

- **Runnability is per file.** A file is runnable when it is `manifest.main`, has
  a header comment (`kind` defaults to `sketch`), or is a subdirectory's
  `main.js`. Adding the header is the deliberate act that makes a file runnable.
- **The header** is the first block comment, fenced YAML:
  `/* --- \n name: …\n kind: sketch\n tags: [audio]\n --- */`. Media a sketch
  needs is declared in `files: [{ path, type }]`.
- **Imports are project-relative only**: `./x.js`, `../x.js`, `/x.js` (project
  root, not filesystem root). Bare specifiers (`lodash`) do not resolve. Modules
  have the same full DSL access the entry does.
- Sketches are **JavaScript**, not TypeScript.

## Idioms

These are the things nothing on disk will teach you.

**Signals, not per-frame mutation.** Bind a parameter to a `Signal` and it
updates itself. Do not hand-roll a frame loop.

```js
radius: lfo.sine({ period: 3 }).mapTo(40, 160)   // yes
```

**`lfo.*` outputs `[-1, 1]`.** Always `.mapTo(lo, hi)` before using one as a
radius, a width, or a coordinate. Bound raw, a radius spends half its cycle
negative and renders as a single pixel — with exit code 0.

`.mapTo()` needs range metadata. LFOs and MIDI signals carry one; `time.seconds`
does not (`Signal.mapTo: this Signal has no range defined`). Use `.map()`,
`.mul()`/`.add()`, or `.withRange([lo, hi])` for those.

**Signals compose.** `.add() .sub() .mul() .div() .mod() .floor() .sin()
.map() .default(v) .withRange([lo,hi])`, and `signal.mix(a, b, t)`. One source
should drive several properties — colour *and* motion *and* scale — that is what
makes the graph visible in the output. `sketch.width` / `sketch.height` are
Signals but also arithmetic-safe (`sketch.width / 2` works).

**`.to()` runs before `setup()`.** User code executes first and builds the graph;
the runtime then calls `setup()` → `start()` → `tick()`. So
`song.to(speakers)` on line 2 is normal and correct — connections are deferred.

**Node methods are immutable.** `song.loop({ start: '16m', end: '17m' })` and
`.noLoop()` on audio, `.slice()/.loop()/.trim()/.speed()` on video, and the
sequencer's `.shift()/.reverse()/.invert()` all return a **new** node and leave
the receiver alone. Assign the result; calling one for its side effect does
nothing. Audio variants share the decoded buffer, so this is cheap.

**Buffer feedback is the trails idiom.** `vis.buffer()` is a render target that
reads as the *previous* frame. Route a chain that reads it back into it and you
have trails, echoes and evolving fields — no manual history:

```js
const a = vis.buffer();
const content = vis.noise.simplex({ scale: 2.5, speed: 0.4 });
a.scale(1.02) // zoom grows trails outward
  .effect("brightness", { amount: 0.94 }) // decay — below 1.0 or it blows out
  .layer(content, { blend: "add", opacity: 0.35 })
  .to(a);
sketch.add(a);
```

**Canvas coordinates are top-left origin, Y down**, pixels, 800×600 by default
(`sketch.setCoordinateOrigin("center")` / `setCoordinateYDirection("up")` change
this). Angles are radians unless `sketch.setAngleMode(DEGREES)`.

**Do not shadow a DSL global.** `const fx = …` throws
`Identifier 'fx' has already been declared`. Every namespace is reserved, and the
short ones are easy to reach for by accident: `fx`, `seq`, `signal`, `text`,
`system`, `force`, `random`, `log`, `color`, `math`, `time`, `point`, `xy`,
`vec2` — plus every colour constant (`red`, `gold`, `teal`, `plum`, `tan`, …).

**Timing.** `time.bpm(124)` sets project tempo. Every scalar time — loop points,
durations, envelope stages, note lengths — is written in one dialect whose base
unit is the **cycle** (one 4/4 measure): `'1c'`, `'2m'`, `'4b'`, `'16n'`,
`'1/16'`, `'2m4b'`, plus the absolute `4`, `'4s'`, `'300ms'` and a
`Signal<number>` of seconds. A bare number is always seconds. Tempo-relative
units without a tempo throw rather than guessing.

## Sync with Studio

Write the file and Studio's watcher picks it up within about a second (Electron,
directory-form projects). There is no reload step.

## Scope

- **`.echo` bundles and web-mode (`localStorage`) projects are out of scope.**
  Tell the user to open it in Studio and save it to a directory first.
- **`echoder init <dir>` only on the fresh-project path** — when the user asked
  to start something new. It scaffolds `echoder.yaml`, `main.js`,
  `package.json`, `jsconfig.json`, `echoder-env.d.ts` and `.gitignore`. Never run
  it against an existing project, and never write `package.json` or
  `node_modules` into one. If a project looks uninitialized, say so once and
  carry on — the API lookup in step 1 does not need it.

## References

- `references/recipes.md` — ten canonical patterns, every snippet verified with
  `echoder run`. Start from the closest one.
- `references/errors.md` — failure text mapped to cause and fix, including the
  ones that exit 0.
